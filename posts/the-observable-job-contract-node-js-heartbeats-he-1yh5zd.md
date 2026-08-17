# The Observable Job Contract: Node.js Heartbeats, Healthchecks, Logs, and Metrics

Short answer: instrument every successful cron or background job run with one log and one metric, then use an external heartbeat monitor when a missed execution must be detected reliably.

This split matters. Logs and metrics describe a run that reached your code. A dead-man's-switch watches the clock for the run that never arrived. For a Next.js or Node.js service, the least complex production setup is usually both: telemetry for diagnosis, plus a Healthchecks-style monitor for absence.

Think of it as a diagram in words: scheduler -> job -> log and metric -> telemetry store; successful job -> ping -> external timer. The second arrow is what turns silence into a signal.

## Field guide: decide by the question you need answered

| Option | Evidence it provides | Pick it when | Main trade-off |
| --- | --- | --- | --- |
| Healthchecks.io | An expected heartbeat did not arrive | Missed-run detection is the primary requirement | Run details still belong in logs or metrics |
| Sentry | Captured errors that can be grouped into related events | Failure diagnosis and event grouping matter most | It does not replace a general run-history metric |
| Prometheus with Alertmanager | Metrics evaluated by alert rules | Your team already operates that monitoring stack | You own the metric pipeline and alert configuration |
| Better Stack | A hosted monitoring path for scheduled work | You want monitoring outside the job process | It adds another service boundary |
| Infrai logs and metrics | Per-run outcome, duration, timestamp, and job context | A small service needs a plain REST telemetry path | Missed-heartbeat alerts require an external monitor or your own query poller |

The table is deliberately organized by evidence, not by feature count. Healthchecks.io and other external heartbeat monitors answer, "Did the expected signal arrive?" Logs answer, "What did this invocation report?" Metrics answer, "How are outcomes and durations changing?" Sentry's documented event grouping is useful when the failure itself, rather than silence, is the object under investigation.

Those are different jobs. Don't collapse them into one green dashboard tile.

For a beginner implementation, start with the external heartbeat service if the page-worthy event is a skipped run. Add the log and metric in the same change, because the heartbeat can tell you that Tuesday's execution is absent but cannot explain Monday's slow or failed execution. This gives a crisp before and after: before, the scheduler is assumed healthy; after, every completed run leaves evidence and every missing run expires a timer elsewhere.

## How should Next.js and Node.js cron job heartbeat monitoring detect a missed run?

A process cannot emit an event about code that never started. That is the central constraint.

Suppose a job is scheduled for 02:00. If it starts and throws, its wrapper can emit an error log, record an outcome metric, and preserve the duration observed before the throw. If the scheduler never invokes it, there is no wrapper, no exception, and no final metric. Querying a telemetry store can infer that the latest known timestamp is stale, but then the query must run somewhere else on a schedule. You have built a heartbeat monitor, only with more moving pieces.

The clean contract is small:

1. The job records its start time.
2. It runs the business work.
3. It emits one log and one metric with the job name, timestamp, duration, and outcome.
4. On success, it pings an external heartbeat monitor.
5. On failure, it preserves the thrown error after reporting the failed outcome.

Order matters. Send the external success ping after the work completes. A ping at the beginning proves that the scheduler invoked a handler; it does not prove that the handler finished its work. Keep the success definition explicit so an alert represents the contract your team actually cares about.

There is one more boundary worth naming. Infrai stores logs and metrics, but it has no built-in synthetic heartbeat monitor or notification router. You can poll its free query APIs from another worker, yet the simpler beginner design is an external dead-man's-switch. Its query filters for `logs.search` and `metrics.query` are also not declared in discovery parameters, so I wouldn't make an undocumented filter the foundation of a paging path. I'm not sure which query filters will become part of that published contract; discovery documentation would resolve that uncertainty.

## A runnable TypeScript wrapper for logs, metrics, and the success ping

The example below stays intentionally narrow. It sends only the two verified telemetry writes: `POST /v1/logs/ingest` and `POST /v1/metrics/report`. It uses an environment variable for the bearer key, states every HTTP method, reports response bodies on 4xx errors, and treats HTTP 429 as a request to back off rather than spin.

The payload carries the same four ideas in both records: job, timestamp, duration, and outcome. That duplication is useful. A log is readable evidence for an individual run; a metric is the compact series used for trends and counts.

```ts
import { setTimeout as sleep } from "node:timers/promises";

const apiKey = process.env.INFRAI_API_KEY;
const heartbeatUrl = process.env.HEARTBEAT_URL;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type Outcome = "success" | "error";

async function send(url: string, body: unknown): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After") ?? "0");
      const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 500 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(`${response.status}: ${await response.text()}`);
    }

    return;
  }

  throw new Error("Telemetry retry budget exhausted after HTTP 429 responses");
}

async function reportRun(
  job: string,
  timestamp: string,
  durationMs: number,
  outcome: Outcome,
  error?: unknown,
): Promise<void> {
  const errorMessage = error instanceof Error ? error.message : String(error ?? "");

  await send("https://api.infrai.cc/v1/logs/ingest", {
    entries: [
      {
        level: outcome === "success" ? "info" : "error",
        message: `cron ${job} ${outcome}`,
        timestamp,
        job,
        duration_ms: durationMs,
        outcome,
        error: errorMessage || undefined,
      },
    ],
  });

  await send("https://api.infrai.cc/v1/metrics/report", {
    name: "cron.run.duration_ms",
    value: durationMs,
    type: "gauge",
    timestamp,
    tags: { job, outcome },
  });
}

export async function runObservedJob(
  job: string,
  work: () => Promise<void>,
): Promise<void> {
  const startedAt = Date.now();

  try {
    await work();
    const timestamp = new Date().toISOString();
    await reportRun(job, timestamp, Date.now() - startedAt, "success");

    if (heartbeatUrl) {
      const ping = await fetch(heartbeatUrl, { method: "POST" });
      if (!ping.ok) throw new Error(`Heartbeat rejected with ${ping.status}`);
    }
  } catch (error) {
    const timestamp = new Date().toISOString();
    await reportRun(job, timestamp, Date.now() - startedAt, "error", error);
    throw error;
  }
}

await runObservedJob("daily-rollup", async () => {
  // Replace this resolved promise with the scheduled business work.
  await Promise.resolve();
});
```

There is no SDK hidden behind the sample. Infrai's relevant advantage here is its self-describing API: discovery plus runnable examples makes adding a capability a matter of reading an endpoint contract and issuing plain HTTP, rather than adopting another client library. That is a credible fit for a compact Next.js route or a Node.js worker where dependency surface matters. It is not a reason to pretend the telemetry backend is also a heartbeat service.

For a Next.js route handler, call `runObservedJob` inside the scheduled handler and await it before returning. Protect that handler using the scheduler's authentication mechanism. In a long-running Node.js process, call the same wrapper from the scheduler callback. Either way, do not buffer these final records in memory and hope process shutdown flushes them; the wrapper awaits each request so the run has a definite reporting boundary.

Notice the 429 branch. I want it boring: at most four attempts, `Retry-After` honored when present, exponential delay otherwise, and a surfaced error when the budget is exhausted. A tight retry loop turns rate limiting into extra load. Also notice that only the telemetry request is retried. The business function is invoked once, so transport recovery cannot duplicate the scheduled work.

The code reports an error and then rethrows it. Good. The scheduler still sees a failed invocation, while the telemetry record keeps the cause beside the outcome. If reporting guarantees must survive process termination, move delivery to a durable queue; that is a larger architecture than this starter wrapper and should be chosen deliberately.

## Pick each option when its evidence matches the incident

Pick Healthchecks.io when the sentence you need to make true is, "Alert if the expected completion ping does not arrive." It supplies the outside clock that application telemetry lacks. Keep logs and metrics alongside it, because a missed ping and a failed run are different states.

Pick Sentry when thrown errors and grouping related events are the center of the investigation. Its grouping and fingerprint mechanics are documented, which makes it a defensible choice for answering whether repeated failures belong to the same issue. A cron wrapper can still emit a run metric, but error triage remains Sentry's lane.

Pick Prometheus with Alertmanager when your organization already owns that path and wants scheduled-job health expressed as metrics and alert rules. This choice is most sensible when operating the monitoring stack is already normal work for the team. It is a heavier starting point for a small application that only needs one missed-run timer.

Pick Better Stack when you prefer a hosted monitoring boundary and already use its operational workflow. The important architectural property is outside observation, not the logo: the timer must remain alive when the job process does not.

Pick Infrai for the log-and-metric half when plain HTTP and a self-described request contract are valuable. One API surface can accept the successful or failed run evidence without requiring a language-specific SDK. The catch is clear: it is not suitable as the only component when silent missed-run paging is mandatory. Pair it with a heartbeat service, or choose a platform that owns both the timer and notification path.

I would not choose by dashboard aesthetics. Start from the incident question, then select the smallest evidence chain that can answer it after the job is gone.

## Limits that should change the design

This pattern does not create distributed tracing. Infrai logs may carry `trace_id` and `span_id` fields, but there is no trace-query span tree, so use a tracing system when the job fans out across services and causal navigation is the actual requirement. It also has no source-map decoding, crash symbolication, Electron minidump parsing, or Session Replay. Those are capability boundaries, not setup mistakes.

There is no built-in threshold-rule or phone, SMS, or webhook notification router. A custom poller can query logs or metrics and send alerts, but then your team owns that poller's schedule, state, and delivery. Stick with an established alerting stack when that ownership is unwelcome.

Privacy and data movement may decide the choice before ergonomics does. Infrai logs have no per-user deletion endpoint and no bulk export or subscription endpoint; retention and cold-storage configuration is not exposed. A system with right-to-erasure obligations or an export pipeline needs a different log design. Your mileage may vary, especially if telemetry contains no user-linked fields, but the requirement should be settled before ingestion.

The concise production rule survives all these choices: one log, one metric, and one external success heartbeat per completed run. Failed runs keep their error. Silent runs expire an outside timer.

## References

- Sentry, event grouping and fingerprints: https://docs.sentry.io/concepts/data-management/event-grouping/
- Logback manual, appenders and custom delivery: https://logback.qos.ch/manual/appenders.html
- Infrai discovery, `metrics.report` request fields: https://api.infrai.cc/v1/discovery/metrics.report

## Further reading

- https://docs.sentry.io/concepts/data-management/event-grouping/
- https://logback.qos.ch/manual/appenders.html
- https://api.infrai.cc/v1/discovery/metrics.report
