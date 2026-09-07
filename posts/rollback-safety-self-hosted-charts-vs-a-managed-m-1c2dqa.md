# Rollback Safety: Self-Hosted Charts vs a Managed Metrics API for Checkout KPIs

Decision rule: for a marketplace checkout-failure dashboard, choose the path that preserves a versioned failure event across deploy and rollback. Use self-hosted BI for changing questions, a managed metrics API for a stable set of application-owned KPIs, and database-adjacent charts when the source data can safely absorb the read load.

| Option | Pick this when | Rollback and operating trade-off | Cost question to answer |
|---|---|---|---|
| Self-hosted BI, with Metabase or Redash as candidates | Operators need to change queries and investigate rows | The dashboard can query durable history after an application rollback, but the team owns another service and its query access | Who patches it, backs it up, and handles expensive queries? |
| Database-adjacent charts, with Supabase Charts as a candidate to evaluate | KPI data already lives beside the application data and the chart workflow fits the release process | Fewer data hops can simplify recovery; dashboard reads now share a failure domain with operational storage | What happens to database load at peak checkout traffic? |
| Managed metrics API | The product exposes a small, stable KPI contract | The application must retain enough source detail to replay metrics if a release changes their meaning | Are ingestion, retention, querying, and high-cardinality dimensions billed separately? |
| Application-owned rollup endpoint | The team needs exact control over definitions and rollback behavior | Maximum control, plus full ownership of aggregation, retention, access control, and chart delivery | What is the engineering and on-call cost of maintaining it? |

“Cheapest” means the complete path, not the smallest visible invoice. Include compute, storage, backups, upgrades, query isolation, incident response, and the work required to correct a bad release.

## Should self-hosted charts or a managed metrics API show in-app SaaS KPIs?

Start with who is allowed to change the question. A fixed customer-facing panel might always show checkout attempts, payment declines, inventory conflicts, and completed orders. Those definitions belong in application code, get reviewed with the checkout change, and can be represented by a narrow metrics contract. A managed metrics API fits that shape because the UI asks known questions. An application-owned endpoint fits it too when control matters more than offloading operations.

Self-hosted BI fits a different job: an operator needs to slice failures by marketplace, payment method, release, or failure category, then revise the query during an investigation. Metabase and Redash are candidates in that operating model. The useful capability is the exploratory query layer; the trade-off is running it, securing its data access, and preventing analytical reads from disturbing checkout storage.

Database-adjacent charts sit between those choices. They can be attractive when the data is already in one database and the evaluation confirms that embedding, permissions, query behavior, and deployment ownership match the application. The catch is coupling. If a dashboard query competes with checkout writes, the short data path is no longer an advantage. I’m not sure any one option is universally cheapest without traffic, retention, cardinality, staffing, and recovery targets; your mileage may vary.

Keep browser performance separate from business outcomes. Core Web Vitals define LCP, INP, and CLS and assess them at the 75th percentile. Those measures can tell you that the KPI screen is slow or unstable, but they cannot tell you why a checkout failed. Two signal families. Two contracts.

## Pick by the rollback boundary, not the screenshot

The key question is not “Which tool draws a line chart?” Every option can eventually put pixels on a screen. Ask what remains true when release `checkout-2026-08-11.2` is rolled back while events from that release are already stored.

Here is the diagram in words: checkout transaction -> durable failure event -> aggregation boundary -> KPI read model -> in-app chart. Put the release identifier and event schema at the durable boundary. Keep the chart behind a read model rather than pointing it at mutable checkout tables. Then a rollback changes producers and readers without erasing what happened.

An event should describe an outcome, not a chart. `checkout_failed` with a bounded `reason` is durable. `red_bar_incremented` is not. Avoid customer email, card data, raw exception text, and other unbounded or sensitive values in metric dimensions. A high-cardinality checkout identifier can live in protected diagnostic storage while the KPI stream carries the small categories needed for aggregation.

Consider a concrete two-release drill. Release A emits `schemaVersion: 1` with `reason: "payment_timeout"`. Release B adds the optional `retryable` flag, deploys at 10:00, and emits both `retryable: true` and `retryable: false` until 10:12, when an operator restores A. The durable stream now contains old v1-shaped events, extended v1 events, and new events from the restored producer. The KPI reader must count all three shapes exactly once. It must not assume that a missing `retryable` means `false`, because A never made that claim; it can report the retry split only for events where the field exists. The release dimension lets an operator compare the windows without rewriting history, while the immutable `eventId` lets a rebuild discard duplicate delivery. This is where a pretty dashboard can lie — and it’s easy to miss — if its aggregation treats field absence as a business value or resets a counter during rollback. The safe result is less dramatic: the total failure series stays continuous, the optional breakdown has explicit incomplete coverage, and a later read-model rebuild produces the same totals.

This also sharpens the cost comparison. Public observability pricing can separate log ingestion from indexing, which is evidence that “store it” and “make it queryable” may be different cost dimensions. Build a workload sheet before choosing: events per day, bytes per event, retained days, active dimensions, query frequency, peak readers, backup work, and on-call ownership. Don’t turn that sheet into a guessed universal ranking.

Rollback safety needs an explicit compatibility rule: old readers ignore new optional fields, new readers accept old events, and changing the meaning of an existing field requires a new schema version. If release B adds `retryable`, release A must still be able to consume the event after rollback. If B redefines `inventory_conflict`, use a new reason or version instead of silently changing the series.

## Implement an event contract that survives rollback

The deepest implementation is deliberately boring. Store the checkout state change and its observability event in one transaction. Publish or aggregate from that durable record later. A process crash after commit cannot leave the business failure recorded while its KPI evidence disappears, and a publish failure does not require the checkout request to call an external dashboard synchronously.

This TypeScript example keeps infrastructure behind small interfaces. The event is versioned, dimensions are bounded, and the optional field is additive. Adapt the interfaces to the transaction and outbox primitives already used by the application.

```ts
type FailureReason =
  | "payment_declined"
  | "inventory_conflict"
  | "payment_timeout"
  | "unknown";

interface CheckoutFailureV1 {
  eventName: "checkout_failed";
  schemaVersion: 1;
  eventId: string;
  occurredAt: string;
  checkoutId: string;
  marketplaceId: string;
  releaseId: string;
  reason: FailureReason;
  retryable?: boolean;
}

interface Transaction {
  markCheckoutFailed(checkoutId: string, reason: FailureReason): Promise<void>;
  appendObservabilityEvent(event: CheckoutFailureV1): Promise<void>;
}

interface CheckoutStore {
  transaction<T>(work: (tx: Transaction) => Promise<T>): Promise<T>;
}

interface RecordFailureInput {
  checkoutId: string;
  marketplaceId: string;
  releaseId: string;
  reason: FailureReason;
  retryable?: boolean;
}

async function recordCheckoutFailure(
  store: CheckoutStore,
  input: RecordFailureInput,
): Promise<CheckoutFailureV1> {
  const event: CheckoutFailureV1 = {
    eventName: "checkout_failed",
    schemaVersion: 1,
    eventId: crypto.randomUUID(),
    occurredAt: new Date().toISOString(),
    checkoutId: input.checkoutId,
    marketplaceId: input.marketplaceId,
    releaseId: input.releaseId,
    reason: input.reason,
    ...(input.retryable === undefined
      ? {}
      : { retryable: input.retryable }),
  };

  await store.transaction(async (tx) => {
    await tx.markCheckoutFailed(input.checkoutId, input.reason);
    await tx.appendObservabilityEvent(event);
  });

  return event;
}
```

The aggregator should deduplicate by `eventId`. Group the in-app KPI by bounded fields such as `reason`, `marketplaceId`, and `releaseId`; keep `checkoutId` available for controlled diagnostic lookup, not as a chart series. During a rollback, leave events immutable and deploy the prior producer. Rebuild the read model from the versioned event log if an aggregation definition changed.

That last step matters. A dashboard assembled only from irreversible counters cannot recalculate yesterday’s result after discovering that release B classified timeouts differently. Durable events cost more storage than counters, but they buy auditability and safe recomputation. Teams with strict storage limits can retain detailed events for a defined recovery window, then keep coarser aggregates, provided that policy matches their investigation and compliance needs.

## Test failures before the checkout release

Exercise the rollback path in staging with a small matrix: produce version 1 events, deploy a reader that understands the optional field, produce mixed events, then restore the old reader. Assert that totals do not double, unknown optional fields are ignored, and the read model can be rebuilt from the retained source. Inject a duplicate `eventId` and verify one count. Inject an unrecognized reason and verify it lands in `unknown` rather than vanishing.

Also test absence. If no `checkout_failed` event arrives, that might mean perfect conversion, a stopped producer, or a broken aggregation path. Pair the business KPI with freshness metadata from the read model and an independent signal that the checkout workflow is running. Don’t display a stale zero as good news.

Ship the contract first.

It’s safer.

## Limits and a practical decision rule

A fixed metrics path is not suitable when operators must invent joins during an incident; use an exploratory query layer over a protected replica or warehouse in that case. Self-hosted BI is a poor fit when nobody can own upgrades, backups, access reviews, and query isolation. Database-adjacent charts should be rejected when their reads threaten the checkout database or their permission model cannot enforce tenant boundaries. An application-owned rollup is the wrong choice when the team cannot commit to its retention and on-call burden.

For rollback-sensitive marketplace checkout KPIs, preserve versioned events first and choose the display path second. Stable product questions favor a narrow metrics contract. Changing investigative questions favor BI. Data locality can favor database-adjacent charts, but only after load and tenant isolation tests pass. The lowest-cost answer is the one that meets the recovery target after operations and correction work are included.

## References

- Core Web Vitals: https://web.dev/articles/vitals
- Datadog pricing: https://www.datadoghq.com/pricing/
