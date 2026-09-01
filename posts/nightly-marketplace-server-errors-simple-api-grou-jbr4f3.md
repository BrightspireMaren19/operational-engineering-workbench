# Nightly Marketplace Server Errors: Simple API Grouping, US/EU Search, Detail, and Resolve

Short answer: choose the error-tracking API that can prove, with your own nightly marketplace events, that it preserves tenant cost labels from ingestion through grouping, search, event detail, and resolution; a compact API is enough when that loop passes, while a broader platform is justified when the same test exposes a real requirement for paging, release analysis, browser diagnostics, or trace exploration.

| Pick this shape | Pick it when | The catch |
| --- | --- | --- |
| A focused grouping API | One small team needs to capture server exceptions, find a group, inspect an original event, and resolve it through code | The team may need separate alert delivery, long-term analytics, and pipeline monitoring |
| A full error-tracking platform | Several responders need release context, assignment, notification policy, and a rich investigation interface | More workflow and configuration can make tenant-level cost attribution harder to audit |
| Structured logs plus a thin grouping service | Existing logs already carry stable tenant, run, region, and cost labels | The team owns fingerprint rules, lifecycle state, and the operator experience |

The decision axis is cost attribution, not feature count. For a nightly marketplace pipeline, every captured failure should answer one operational question without a spreadsheet join: which tenant, import run, region, and billable stage produced this work?

## How should a small B2B SaaS compare a simple error grouping API for server errors?

Start with one acceptance trace. In words, the diagram is: scheduler starts a marketplace import; a worker attaches `tenant_id`, `run_id`, `region`, and `pipeline_stage`; a rejected catalog row emits a structured event; the tracker assigns a fingerprint; search returns the group; event detail preserves the original labels; a human fixes the cause; the group moves to resolved. Then the next recurrence either reopens the same group or creates a new one according to a documented rule.

That's the loop.

Run it against every serious option with the same data. Rollbar, Bugsnag, Sentry, and a smaller alternative can all remain on the worksheet, but their names shouldn't determine the score. Their current behavior, contracts, deployment choices, and commercial terms can change. Record evidence from a trial and the current agreement instead of filling cells from memory.

The test needs more than a screenshot. Capture the request used to ingest an event, the query used to find it, the response that supplies event detail, and the state transition used to resolve the group. Check that pagination is deterministic, timestamps retain their timezone meaning, and unknown fields fail or pass through in a documented way. If US or EU processing is a contractual requirement, verify the offered region, subprocessors, transfer terms, retention controls, and deletion path with the provider and counsel. I'm not sure any product label alone can settle a particular company's residency obligation; the signed terms and actual data flow resolve that uncertainty.

Use a short scorecard and make each row falsifiable:

1. Can an engineer find all failed `price_index` events for one tenant and one nightly run?
2. Does event detail return the exact cost labels sent at capture time?
3. Does the fingerprint collapse repeated manifestations of one defect without merging unrelated tenants or stages?
4. Can automation resolve a group idempotently, and what happens when a matching event arrives later?
5. Can the team export enough data to reconcile usage with an internal cost ledger?
6. Are the required US/EU handling, retention, access, and deletion controls in the contract?

This method deliberately separates claims from proof. A checkbox marked “search” says almost nothing: substring search, exact label filtering, time-window filtering, and stable pagination support very different operating loops. Event detail is equally slippery. A rendered stack trace is useful, but it doesn't replace the raw tenant and run identifiers needed to attribute pipeline spend.

## Make cost ownership part of the event contract

Treat logs as event streams. The Twelve-Factor App's logging guidance says an application should write its event stream without managing log-file routing or storage itself. That boundary fits this pipeline: application code emits a structured event, while the execution environment routes it to the chosen destinations.

The payload still needs discipline. `tenant_id` identifies the B2B customer account, `run_id` identifies one scheduled execution, `pipeline_stage` narrows the failed unit of work, and `region` records the execution location. Add `error_code` for a stable machine category and `fingerprint_version` so grouping changes are reviewable. Keep the human message useful, but don't make it the only grouping input; messages often contain row ids, timestamps, URLs, or counts that split one defect into hundreds of groups.

Cost attribution should use the same stable dimensions. For example, a pipeline ledger can aggregate compute or request units by `tenant_id`, `run_id`, and `pipeline_stage`, while error search uses those identical labels. The error system does not become the billing ledger. It becomes a navigable view into why a ledger line was wasteful. One failed tenant import that retries 40 times is then visible as one operational problem tied to 40 attempts, rather than 40 apparently unrelated messages.

There is an important privacy boundary here. Don't put buyer email addresses, raw listing descriptions, access tokens, or payment data into a fingerprint or searchable label merely because search is convenient. Prefer opaque internal identifiers and keep the mapping in the system that already owns authorization. Access to error detail should follow least privilege, and retention should match the data classification rather than the longest period a tool happens to offer.

Sharp edges matter. A global fingerprint such as `error_code + pipeline_stage` makes aggregate triage easy but may blend tenants whose failures require different owners. Adding `tenant_id` prevents that blend but can multiply groups and hide a shared regression. A practical middle ground is to keep tenant identity as a filterable attribute and reserve it for the fingerprint only when tenant-specific configuration changes the remediation. Document the choice. Otherwise the on-call engineer will discover the grouping policy during an incident.

## Implement one portable grouping boundary in TypeScript

Put product-specific transport behind a narrow adapter. The application should create a stable event; an adapter can then map that event to the selected ingestion contract. This keeps cost labels testable and prevents business code from depending on a dashboard's field names.

```ts
type Region = "us" | "eu";
type PipelineStage = "catalog_fetch" | "normalize" | "price_index";

type PipelineFailure = {
  occurredAt: string;
  tenantId: string;
  runId: string;
  region: Region;
  pipelineStage: PipelineStage;
  errorCode: string;
  errorName: string;
  message: string;
  attempt: number;
  fingerprintVersion: 1;
};

type ErrorGroup = {
  id: string;
  fingerprint: string;
  status: "open" | "resolved";
};

interface ErrorGroupingPort {
  capture(event: PipelineFailure, fingerprint: string): Promise<void>;
  search(filters: {
    tenantId: string;
    runId: string;
    pipelineStage?: PipelineStage;
  }): Promise<ErrorGroup[]>;
  getEventDetail(groupId: string): Promise<PipelineFailure[]>;
  resolve(groupId: string): Promise<void>;
}

function stableFingerprint(event: PipelineFailure): string {
  return [
    `v${event.fingerprintVersion}`,
    event.pipelineStage,
    event.errorCode,
    event.errorName,
  ].join(":");
}

async function reportFailure(
  port: ErrorGroupingPort,
  event: PipelineFailure,
): Promise<void> {
  await port.capture(event, stableFingerprint(event));
}

async function verifyTriageLoop(
  port: ErrorGroupingPort,
  expected: PipelineFailure,
): Promise<void> {
  const groups = await port.search({
    tenantId: expected.tenantId,
    runId: expected.runId,
    pipelineStage: expected.pipelineStage,
  });

  if (groups.length !== 1) {
    throw new Error(`Expected one group, received ${groups.length}`);
  }

  const events = await port.getEventDetail(groups[0].id);
  const matchingEvent = events.find(
    (event) =>
      event.tenantId === expected.tenantId &&
      event.runId === expected.runId &&
      event.pipelineStage === expected.pipelineStage,
  );

  if (!matchingEvent) {
    throw new Error("Event detail lost required attribution labels");
  }

  await port.resolve(groups[0].id);
}
```

The interface is intentionally boring. Good. It names the four capabilities the application truly consumes and leaves assignment, dashboards, notification rules, and vendor query syntax outside the domain model. An adapter test can use a synthetic tenant such as `tenant_test_17`, a fixed run id, and an error code such as `CATALOG_SCHEMA_422`; after capture, it polls with a bounded deadline, checks the preserved detail, resolves the group twice to test idempotency, and submits one recurrence to observe the documented reopen behavior.

Do not copy the `message` into the fingerprint. Consider a feed rejection reading `row 812 has currency ZZ` tonight and `row 913 has currency ZZ` tomorrow. Full-message grouping creates two groups even though the corrective action is identical. Grouping only on `errorName`, however, can merge every validation failure in the stage. The stable tuple above preserves the semantic code and stage, and its version prefix makes a later policy change explicit.

Deployment needs its own checks. Run the adapter contract test in CI against a test project, but keep a deterministic fake for unit tests. Validate configuration at process start. Bound retries for capture, attach a client request id where the selected contract supports one, and route undelivered events to an owned queue rather than blocking the entire marketplace import. The exact retry policy depends on the provider's published response semantics; don't invent one generic rule and assume it applies everywhere.

## Search tests reveal the real operational cost

Search is where a “simple” integration can become expensive. Ask an engineer to investigate a concrete case: the EU `price_index` stage for one tenant emitted `CATALOG_SCHEMA_422` during last night's run. Measure neither button count nor visual polish. Observe which joins, copied identifiers, permissions, and manual exports stand between the question and the event detail.

Search earns its keep here.

A useful before/after is crisp. Before structured labels, the engineer starts with the nightly pipeline's red status, copies a run id into log search, and finds several messages whose prose happens to mention a schema problem. One message belongs to `catalog_fetch`, two belong to `price_index`, and the messages include different row numbers. The engineer opens each result, discovers that only one preserved the tenant id, looks up that tenant in the scheduler, and then checks a separate ledger to learn which retry consumed the work. The grouping screen cannot explain the cost because its fingerprint used the changing message, while the ledger cannot explain the failure because it never received the error code. After structured labels, one query selects tenant, run, stage, region, and `CATALOG_SCHEMA_422`; event detail exposes the attempt number and stable fingerprint; the ledger uses the same dimensions and can link back by run id. This exercise tests navigation, data preservation, and ownership at once. It also exposes a false economy: cheap ingestion does not help if every investigation requires three systems and a manual join. Faster investigation is plausible, but don't publish a percentage without a controlled measurement.

Also test absence. A nightly job that never started emits no application exception, so an error grouping API cannot infer the missing run from an event that does not exist. Monitor scheduler heartbeats or expected run records separately. Likewise, a feature-flag or experimentation system answers rollout and experiment questions, not server exception grouping. Keeping those signals separate makes ownership clearer even if they later meet in one incident view.

Resolution deserves a state-machine test. Define who may resolve, whether resolution is idempotent, what a recurrence does, and whether resolution changes the underlying event retention. “Resolved” should mean that the team believes the cause has been addressed or consciously accepted; it should not mean that evidence was deleted. If the API's model cannot represent your chosen semantics, the mismatch belongs in the scorecard.

Cost has three layers: vendor charges, integration labor, and recurring operational labor. Pricing can change, so capture the date and assumptions in the ADR rather than baking a transient figure into application code. A focused API may have a smaller learning surface yet require team-owned alerting and reporting. A full platform may consolidate those workflows yet demand more configuration and governance. Structured logs may reuse existing infrastructure while shifting grouping quality and lifecycle maintenance onto your team.

No row is free.

## Where is the focused approach not suitable?

Do not choose the focused boundary when responders already require multi-channel escalation, release-health analysis, browser source processing, native mobile crash diagnostics, distributed trace navigation, or a polished case-management workflow. In that situation, keep a fuller error platform on the shortlist and test those workflows directly. Don't pretend a small adapter replaces capabilities your incident process actually uses.

The logs-plus-thin-service option is also a poor fit when nobody owns fingerprint evolution, index performance, retention, and resolution semantics. Stick with a managed workflow when operational ownership would otherwise be ambiguous. Conversely, don't buy a broad interface solely because it has more rows on a feature page; choose it only when the acceptance trace or an adjacent mandatory workflow proves the value.

The durable decision record is short: required event fields, fingerprint policy, search acceptance cases, resolution semantics, data-region evidence, retention and deletion requirements, and the owner of alerts and missing-run detection. Re-run it before renewal and after a material pipeline change. That's enough to keep the tool choice subordinate to the marketplace job it must support.

## Sources

- https://12factor.net/logs
- https://www.growthbook.io/
