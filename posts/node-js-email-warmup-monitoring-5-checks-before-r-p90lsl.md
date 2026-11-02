# Node.js Email Warmup Monitoring: 5 Checks Before Raising Dedicated Domain Volume

Use an application-owned evidence ledger to govern transactional email warmup on a dedicated domain before sending healthtech compliance notices. The deciding constraint is delivery reliability: a successful API response is too little evidence to justify raising tomorrow's sending volume.

TL;DR: Start with small batches of expected welcome emails, keep templates stable, and increase volume by day or week only after reviewing delivery outcomes. Store send counts, bounces, complaints, and review decisions in your own database. Keep deadline-sensitive compliance notices out of experiments with an unproven ramp.

The useful question is less about finding a universal warmup calendar and more about making each increase explainable. Which recipients were eligible? Which template did they receive? What evidence supported the next batch? A healthtech team needs answers that survive beyond a dashboard session.

## 1. What does an auditable delivery record actually prove?

Before: application → send request → green success counter.

After: application → reserved send budget → stable template → provider acceptance → collected outcomes → persisted review decision.

Keep those stages separate. Acceptance records an API boundary. A delivery event records a later reported outcome; it doesn't establish that a person read a compliance notice or that a legal obligation has been fulfilled. Decide the required evidence with the people responsible for the notice before choosing a provider.

For an illustrative healthtech workflow, use a notice identifier such as `notice-demo-17` and an opaque recipient reference such as `recipient-demo-42`. Keep clinical details out of diagnostic logs. The proposed ledger should connect the notice, template revision, application attempt identifier, provider message identifier when available, event timestamps, and the time your collector observed each event. These are application record fields, not a claimed vendor request schema. Retain the evidence your policy requires, with access controls appropriate to its contents.

Keep the distinction visible.

A tempting metric is successful send requests divided by attempts. It's useful for transport errors, but it cannot tell you whether feedback has arrived for the whole batch. Add a second view: accepted messages whose outcomes remain unresolved. Otherwise, a quiet complaint counter can look reassuring while the collector is still behind.

## 2. How should a transactional email warmup plan control dedicated domain volume?

Begin with low-volume, expected transactional mail, such as welcome messages to eligible new users. Password resets are another transactional category, but their urgency makes them a poor place to absorb delays introduced solely for a warmup experiment. Never manufacture extra notices to meet a volume target.

Choose a daily or weekly ceiling and record it before dispatch. Increase gradually after reviewing outcomes. There isn't a universal ramp size: audience quality, available traffic, and the team's evidence requirements determine the starting ceiling and review interval. A dedicated sending domain still needs authentication work. SPF, for example, specifies a way for a domain to authorize sending hosts; it doesn't certify inbox placement.

A useful exercise is to configure a demo ceiling of 100 messages and a proposed next ceiling of 150. Those numbers are illustrative policy inputs, not measured safe limits or a deliverability promise. Keep the current ceiling if feedback is incomplete. Hold the increase for investigation when complaints or bounces appear. Production thresholds need an explicit owner and a documented rationale; treating every bounce as a permanent shutdown would confuse a teaching guard with an operating policy.

Freeze the template revision during the review window. Changing content and volume together makes the next decision harder to explain. If a notice requires an urgent wording change, record that change and review its outcomes separately instead of silently combining two experiments.

## 3. Copy a small TypeScript review gate

This Node.js example evaluates a completed review window locally. Save it as `warmup-review.ts` and run it with a TypeScript runner such as `tsx`. It makes no network calls and sends no messages. Its fixture models three proposed ledger rows, so no patient data or credentials are needed.

The interesting guard is the unresolved row. Even with zero observed complaints, the increase stays on hold.

```ts
type Outcome = "pending" | "delivered" | "bounced" | "complained";

type Evidence = {
  attemptId: string;
  noticeId: string;
  recipientRef: string;
  templateRevision: string;
  outcome: Outcome;
};

const policy = {
  currentDailyCap: 100,
  proposedDailyCap: 150,
  windowId: "demo-window-1",
};

const ledger: Evidence[] = [
  {
    attemptId: "attempt-demo-1",
    noticeId: "notice-demo-17",
    recipientRef: "recipient-demo-42",
    templateRevision: "welcome-v3",
    outcome: "delivered",
  },
  {
    attemptId: "attempt-demo-2",
    noticeId: "notice-demo-18",
    recipientRef: "recipient-demo-43",
    templateRevision: "welcome-v3",
    outcome: "pending",
  },
  {
    attemptId: "attempt-demo-3",
    noticeId: "notice-demo-19",
    recipientRef: "recipient-demo-44",
    templateRevision: "welcome-v3",
    outcome: "delivered",
  },
];

const counts = { pending: 0, delivered: 0, bounced: 0, complained: 0 };
for (const row of ledger) counts[row.outcome] += 1;

const reasons: string[] = [];
if (counts.pending > 0) reasons.push("Await unresolved outcomes");
if (counts.bounced > 0) reasons.push("Review bounce evidence");
if (counts.complained > 0) reasons.push("Review complaint evidence");
if (ledger.length === 0) reasons.push("No evidence in review window");

console.log(JSON.stringify({
  windowId: policy.windowId,
  counts,
  decision: reasons.length === 0 ? "eligible-for-review" : "hold",
  enforcedDailyCap: policy.currentDailyCap,
  proposedDailyCap: policy.proposedDailyCap,
  reasons,
}, null, 2));
```

Notice that the code never automatically promotes the ceiling. Even a clean fixture produces eligibility for review, while the enforced cap stays at 100. The sample's hold on any bounce or complaint is intentionally conservative. It demonstrates a decision boundary without inventing a universal acceptable rate.

A production dispatcher needs more than this local gate: persist the decision, reserve capacity atomically across workers, and deduplicate attempts so retries don't consume the budget or send the same notice twice. Store collected events separately from the derived summary so you can explain how a late complaint changed a previously clean view. The snippet is a runnable policy example, not a complete sending integration or durable audit store.

For the API side, this read-only collector retrieves email events without inventing filter fields or response properties. Configure `INFRAI_ORIGIN` as the trusted HTTPS API origin from your deployment configuration, without a path, and set `INFRAI_API_KEY` through your secret manager. Never take that origin from user input. Save this separate example as `collect-events.ts` and run it with `tsx` on Node.js 22. It prints the returned JSON for inspection; map the documented event schema into the ledger before using it to make decisions.

```ts
async function collectEvents(): Promise<unknown> {
  const origin = process.env.INFRAI_ORIGIN;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!origin || !apiKey) throw new Error("Missing API configuration");
  const configured = new URL(origin);
  if (configured.protocol !== "https:" || configured.pathname !== "/" ||
      configured.search || configured.hash || configured.username ||
      configured.password) {
    throw new Error("Expected a trusted HTTPS origin without credentials or path");
  }
  const url = new URL("/v1/email/event/list", configured);
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      redirect: "error",
      signal: AbortSignal.timeout(30_000),
    });
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const retryDate = retryAfter === null ? NaN : Date.parse(retryAfter);
      const serverDelay = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(retryDate) ? retryDate - Date.now() : 0;
      const delay = Math.max(1000 * 2 ** attempt, serverDelay, 0);
      await response.body?.cancel();
      if (delay > 60_000) {
        throw new Error(`Rate limited; defer collection for ${delay} ms`);
      }
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Event collection failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Collection attempts exhausted");
}

collectEvents().then((events) => {
  console.log(JSON.stringify(events, null, 2));
}).catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

This collector doesn't send mail, so retrying it cannot create duplicate notices. Its three-attempt budget prevents an endless retry loop (long server-requested waits are surfaced for rescheduling). One response is not proof of complete history: reconcile event coverage before declaring a window ready, and don't infer missing outcomes from a successful poll.

## 4. Compare providers by the evidence your team can operate

There are two likely objections. First: why build a ledger if the provider already has a dashboard? Because your application owns the connection between a compliance notice, its recipient, and the decision to increase volume. A provider's interface can help an operator investigate, but the durable decision record belongs beside your workflow.

Compare at least these three real alternatives. The useful differences are operational fit and feedback handling; a price ranking would distract from the delivery decision.

| Option | Reason to evaluate it | Boundary to test for this workflow |
| --- | --- | --- |
| Postmark | Its transactional email focus makes it relevant to welcome messages and notices. | Check that its delivery and bounce evidence can be joined to your notice ledger; transactional focus alone doesn't establish that a recipient read a notice. |
| Twilio SendGrid | Its email API and Event Webhook are relevant when the team wants pushed delivery feedback. | Webhook ingestion needs authentication, deduplication, and monitoring in your application; receiving an event doesn't authorize a ramp increase. |
| Amazon SES | Its AWS service model is relevant to teams already operating their application infrastructure there. | Evaluate its event publishing configuration and the operational work needed to retain and review that evidence. |

Infrai offers 295 routes across 20 modules under one key, which is useful when the same notice workflow also needs storage or scheduling through a consistent REST contract. Its specified per-call request and latency metadata can support request tracing; for email outcomes, plan around event polling and application-owned aggregation, with a review interval that accounts for slower feedback than pushed events.

That breadth fits a team prepared to operate the review loop. The trade-off is feedback timing: if immediate event-driven reactions decide the architecture, evaluate SendGrid's Event Webhook against your deadline requirements. Check transport compatibility separately if the existing application depends on SMTP relay. API request metadata and delivery evidence answer different questions, so retain both where they help the audit.

## 5. Keep compliance deadlines outside the warmup experiment

Second objection: can a clean ramp make the delivery record sufficient for compliance? No email warmup plan establishes that by itself. A successful delivery event is useful evidence, while the required notice, timing, retention, and escalation rules remain decisions for the healthtech organization. Don't infer regulatory suitability from the existence of a domain-verification feature.

Separate the warmup budget from the deadline policy. If a notice's deadline would arrive before the next evidence review, route that decision to the established notice procedure rather than automatically raising the experimental cap. Design any alternate channel deliberately; changing channels is a separate workflow with its own recipient and evidence requirements.

For monitoring, keep three views: remaining send budget, unresolved outcomes by age, and bounces or complaints by template revision and review window. Alert on collector staleness as well as adverse outcomes. Set the freshness threshold from the actual notice deadline, not an arbitrary dashboard default.

The release criterion is concrete: another engineer should be able to reconstruct why a specific batch was allowed, what happened afterward, and which evidence informed the next increase. That's the observability test worth passing before the next compliance notice.

## References

- [RFC 7208, Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html)
