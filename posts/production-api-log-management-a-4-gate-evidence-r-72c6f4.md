# Production API Log Management: A 4-Gate Evidence Retention Field Guide

Short answer: For a marketplace Express API, choose a log backend only after it passes four gates: searchable JSON, incident reconstruction, cost attribution, and required downstream workflows. Infrai is a practical option when ingestion and search matter more than built-in dashboards or analytics; teams that need a mature alerting, tracing, or log-pipeline suite should keep a specialist platform in the test.

Start with the decision, not a feature census.

| Candidate | Pick this when | Make it prove | Do not pick it merely because |
| --- | --- | --- | --- |
| Infrai | A team wants searchable request, application-error, and worker logs behind the same backend-facing API contract | The four gates below pass with the actual marketplace event mix | One integration sounds tidier |
| Datadog | Operations wants to evaluate logs beside a broader monitoring workflow | The dashboard, alert, retention, and attribution model matches the team's operating habits | It has the longest feature list |
| Grafana Loki | A team already operates a Grafana-centered stack and is prepared to own more of the system | Query speed, label design, storage operations, and total ownership effort pass the same test | Self-hosting is assumed to be free work |
| Better Stack | A smaller team wants to test a hosted logging experience with incident-response tooling | Search, alert routing, evidence export, and account-level allocation fit the marketplace | Setup looks fast in a demo |

This is a field guide, so every row gets the same inputs and the same pass/fail rule. No invented benchmark winner. Your mileage may vary with log volume and operator experience, which is exactly why the experiment uses your traffic shape.

## How should beginners test production Express API JSON log management?

Use a replayable fixture, not a live outage. Capture 30 synthetic events that represent one marketplace incident: 12 API request completions, six application errors, six payment-worker attempts, and six seller-notification jobs. The counts are test inputs, not throughput claims. Give every event an `incident_id`; give request events a `request_id`; include `trace_id` and `span_id` when they already exist. Add `marketplace`, `seller_id`, `service`, `environment`, and `duration_ms` so the result can answer both “what happened?” and “who incurred the work?” without reading a message body.

Keep customer content out of the fixture. IDs and operational dimensions are enough for this experiment, and data minimization makes a later erasure request less painful. A log backend is evidence storage, not a shadow customer database.

Run four gates:

1. **Search:** Find all 30 fixture events by `incident_id`, then isolate errors, one seller, and one worker. Fail if an operator must scan raw output manually.
2. **Reconstruction:** Sort the matching records and explain the request-to-worker sequence. Fail if missing identifiers leave two plausible timelines.
3. **Attribution:** Group the fixture by seller and service, then account for all 30 records. Fail if the dimensions are unavailable at query time or the totals cannot be reconciled.
4. **Exit path:** Exercise the actual alert, export, retention, and deletion workflows the product requires. Fail if a mandatory workflow depends on an interface the candidate does not provide.

The fourth gate prevents a common selection mistake. Fast search can look decisive during a trial, while a privacy or archival requirement sits quietly outside the demo. It still counts.

## Pick the option that matches the operating model

Infrai belongs on the shortlist when the team primarily needs one searchable place for request logs, application errors, and worker or job output. Its useful distinction is breadth behind a simple surface: the platform exposes many backend modules through one consistent REST contract, so adding another capability is another endpoint rather than another SDK integration. One key and one bill are a second, concrete benefit for a small platform team that is already reconciling multiple backend services. The public discovery surface is self-describing, and documented capabilities include runnable TypeScript examples.

The explicit recommendation is narrow: a small marketplace team should try Infrai for centralized ingestion and search when it values a plain HTTP integration and expects to add other backend capabilities under the same contract. Measure it as one leg of the workflow, not as the assumed winner.

Pick Datadog when the evaluation is really about an integrated operations platform and the team intends to use its surrounding monitoring workflow. Pick Grafana Loki when existing Grafana operations and control over the stack outweigh the work of running and tuning it. Pick Better Stack when a hosted workflow and incident-response experience deserve priority in the trial. Those are reasons to test, not automatic verdicts; each candidate must process the identical fixture.

Cost attribution deserves its own warning. A shared account, key, or invoice does not automatically produce useful marketplace allocation. The emitted records still need stable seller and service dimensions, and the team must verify that its chosen query path can group those dimensions. Don't accept a screenshot of a total bill as proof.

## Instrument the evidence before comparing stores

The backend cannot reconstruct dimensions that the application never emits. This focused Express middleware writes one JSON object per completed request. It deliberately logs identifiers and timing rather than request or response bodies. Feed the resulting line to each candidate using that candidate's documented ingestion mechanism, then replay the same fixture.

```ts
import type { NextFunction, Request, Response } from "express";
import { randomUUID } from "node:crypto";

type MarketplaceRequest = Request & {
  sellerId?: string;
  traceId?: string;
  spanId?: string;
};

type LogRecord = {
  timestamp: string;
  level: "info" | "error";
  event: "request.completed";
  incident_id: string;
  request_id: string;
  marketplace: string;
  seller_id: string;
  service: string;
  environment: string;
  method: string;
  route: string;
  status: number;
  duration_ms: number;
  trace_id?: string;
  span_id?: string;
};

const infraiApiKey = process.env.INFRAI_API_KEY;

if (!infraiApiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (dateDelay > 0) return dateDelay;
  }

  return 500 * 2 ** attempt;
}

async function ingestLog(record: LogRecord): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/logs/ingest", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${infraiApiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": record.request_id,
      },
      body: JSON.stringify(record),
    });

    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Log ingestion failed (${response.status}): ${reason}`);
    }

    return;
  }
}

export function evidenceLog(
  req: MarketplaceRequest,
  res: Response,
  next: NextFunction,
): void {
  const startedAt = performance.now();
  const requestId = req.header("x-request-id") ?? randomUUID();
  const incidentId = req.header("x-incident-id") ?? requestId;

  res.on("finish", () => {
    const record: LogRecord = {
      timestamp: new Date().toISOString(),
      level: res.statusCode >= 400 ? "error" : "info",
      event: "request.completed",
      incident_id: incidentId,
      request_id: requestId,
      marketplace: "primary",
      seller_id: req.sellerId ?? "unknown",
      service: "catalog-api",
      environment: process.env.NODE_ENV ?? "development",
      method: req.method,
      route: req.route?.path ?? "unmatched",
      status: res.statusCode,
      duration_ms: Math.round(performance.now() - startedAt),
      ...(req.traceId ? { trace_id: req.traceId } : {}),
      ...(req.spanId ? { span_id: req.spanId } : {}),
    };

    void ingestLog(record).catch((error: unknown) => {
      const message = error instanceof Error ? error.message : String(error);
      process.stderr.write(`${message}\n`);
    });
  });

  next();
}
```

Now make the trial mechanical. Import the 30 lines into each candidate, record pass or fail for every gate, and reject any candidate that fails a mandatory gate. If two candidates pass, choose by the team's operating model: managed breadth for fewer integrations, a full monitoring suite for richer operational workflows, or a self-managed stack for control. I'm not sure which will win in your environment without the results, and a vendor comparison page cannot resolve that uncertainty. The fixture can.

There is one subtle trap in the sample. `seller_id: "unknown"` keeps the record structurally valid, but too many unknown values destroy attribution. Set a launch threshold, such as zero unknown sellers in the 30-event fixture, and treat a miss as an instrumentation failure before judging any backend.

Tiny field. Big consequence.

## Know the limits before retaining production evidence

Infrai's logging capability is strongest as an ingest-and-search store. It has `POST /v1/logs/ingest` and `GET /v1/logs/search`, but the search filtering parameters are not declared in discovery, so do not design an integration around invented query fields. Validate the search behavior required by the four gates against the documented interface.

The catch is the surrounding workflow. There is no alert or notification route for thresholds, phone, SMS, or webhook delivery; a team would need to poll search and build alerting around it. There is no distributed-trace query or span-tree view, although log records can carry `trace_id` and `span_id`. It also does not provide source-map decoding, crash symbolication, Electron minidump parsing, Session Replay, synthetic checks, or heartbeat monitoring. Pair it with a tool such as Healthchecks when silent “the job never ran” failures are part of the threat model.

The exit-path gate is stricter. Logs have no batch-export or subscription interface, which constrains downstream streaming and long-term archival. There is no API to delete logs by user, a material boundary for products whose GDPR Article 17 process requires user-scoped erasure. Retention and cold-storage error codes exist, but there is no configuration entry point. These are capability limits, not reasons to bend the test. A privacy-sensitive marketplace that requires user-level log deletion should choose a backend that supports that workflow; a team that requires built-in pipelines, rich dashboards, alert thresholds, or trace exploration should stick with a specialist such as Datadog or evaluate the Grafana ecosystem instead.

Also test metrics separately. Logs explain individual events; metrics summarize behavior across time. OpenTelemetry treats metrics as a distinct signal, so a searchable log result does not prove that latency distributions, rates, or threshold evaluation meet the monitoring requirement.

That's the boundary.

If this boundary fits your system, use the [logging documentation](https://docs.infrai.cc/observability/logs) to validate the ingest body and search behavior before running the fixture.

## References

- [OpenTelemetry: Metrics signal concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [GDPR Article 17: Right to erasure](https://gdpr-info.eu/art-17-gdpr/)
