# Property Moderation Text Classification: Instrumenting Chat Completions Routing in Node.js

Short answer: put one small, observable classification boundary between the property-management application and its model routes; accept one API key there, enforce one result schema, and record route, latency, schema, and outcome metadata without logging the moderation report itself.

Start with the least complex option that meets the operating requirement. A team that rarely changes models may need only direct adapters. A team that treats provider portability as an incident-control mechanism needs a shared gateway plus conformance tests. Credential count alone doesn't settle the design. A route change must preserve the meaning of every downstream label.

| Option | Pick this when | Main trade-off | Operational signal to require |
| --- | --- | --- | --- |
| Direct provider adapters | One model is the stable default and native features matter | Every adapter owns its own retries, schema handling, and telemetry | Outcome and latency by adapter |
| Shared chat-completions gateway | The same classification contract must move between model routes | The common envelope can hide useful provider-specific controls | Requested route, resolved route, and schema validity |
| Internal classification service | Moderation policy must be isolated from application releases | Another service owns deployment and on-call work | Contract version, policy version, and queue age |

One rule governs all three: **portability is proven by equivalent classified records, not by a successful HTTP response.**

## How can Node.js implement chat completions for text classification?

Treat routing as a boundary with three separate contracts. The input contract describes a moderation report. The output contract describes an allowed label. The routing contract decides which model alias handles the request. Keeping those contracts separate lets an operations team change a model mapping without asking every property workflow to understand OpenAI, Claude, or Gemini request details.

Picture the path in words: leasing portal -> classifier client -> shared chat endpoint -> model route -> schema validator -> human-review queue. Beside that path, telemetry receives only identifiers and measurements: report ID, property ID hash, route alias, resolved model label, elapsed milliseconds, token counts when supplied, and an outcome such as `accepted`, `invalid_output`, `timeout`, or `transport_error`.

That's the useful split.

The application should send a stable alias such as `moderation-primary`, not a dated model name. Configuration maps the alias to a provider route. A controlled change updates that mapping, while a rollback restores it. OpenAI, Anthropic's Claude, and Google's Gemini can each sit behind an adapter; their presence is evidence that the boundary has multiple targets, not a reason to claim their native APIs are interchangeable. They aren't interchangeable at the application edge unless the team has normalized and tested the behavior it actually uses.

The catch is that a shared chat-completions shape is not suitable when a workflow depends on a provider-specific capability that the common contract cannot express. Keep a direct adapter in that case. Likewise, stick with direct calls when route changes are rare and the extra gateway would create more on-call surface than it removes. An internal classification service earns its keep when policy, audit retention, or human-review integration must evolve independently of the calling applications.

## Observability is the portability test

For a property moderation queue, the label set should be deliberately boring. Suppose reports can be tagged `harassment`, `discrimination`, `fraud`, `safety`, or `other`, with a boolean that always sends uncertain cases to a person. The model may explain its decision, but prose must never become the machine contract. Structured Outputs are designed to constrain model output to a supplied JSON Schema, while JSON mode alone does not guarantee conformance to a particular schema. That distinction matters here because an unexpected label can silently route a serious report to the wrong queue.

The contract also needs explicit failure semantics. A timeout is not `other`. Invalid JSON is not `other`. A label outside the enum is not `other`. Those outcomes should fail closed into human review, retain the original report in the system of record, and increment separate counters. This gives the team a crisp before/after check during a route change: before, the candidate route must pass the same fixed corpus; after, production dashboards must show schema-validity and review-rate changes by route alias.

Run a paper trace before writing the adapter. Imagine report `rpt_1842` enters under route alias `moderation-primary` and mapping revision `rev_17`. The endpoint responds in 1,240 ms with `safety`, `needsHumanReview: true`, and resolved model label `candidate-b`. The queue write succeeds. An operator should be able to start from the queue record and find the contract version, mapping revision, duration, and accepted outcome without finding the report text in a log. Now change one fact at a time: malformed JSON should produce `invalid_output` and a review item; an 8-second deadline should produce `timeout` and a review item; an unknown label should do the same. If any branch quietly becomes `other`, the boundary is unsafe even though its availability chart may look excellent.

Trace the bad path first.

Don't log the report body. Property moderation text can contain names, addresses, allegations, and other sensitive material. Logs need enough context to join an event to the protected source record, but they do not need a copy of that record. Use an opaque report ID, hash or otherwise minimize tenant and property identifiers according to the system's threat model, and put access control around the source data rather than trying to reconstruct it from telemetry later.

I would make four measurements mandatory: request duration, outcome count, resolved-route count, and human-review count. Token usage is useful when the endpoint returns it, but it should be optional in the event schema because a portable classifier cannot assume identical accounting fields from every route. I'm not sure a global confidence threshold is meaningful across heterogeneous models; a labeled evaluation set and per-route calibration evidence would resolve that question. Until then, uncertainty belongs in the review queue.

## Use a fail-closed TypeScript client

The following client keeps policy close to the contract. It uses one credential, passes a stable model alias, requests a schema-constrained result, applies an 8-second deadline, validates again locally, and emits metadata through an injected observer. `MODEL_ROUTER_URL` is configuration because endpoint paths are deployment facts, not something a reusable client should guess.

```ts
type ModerationLabel =
  | "harassment"
  | "discrimination"
  | "fraud"
  | "safety"
  | "other";

type Classification = {
  label: ModerationLabel;
  needsHumanReview: boolean;
  rationale: string;
};

type Observation = {
  reportId: string;
  route: string;
  outcome: "accepted" | "invalid_output" | "timeout" | "transport_error";
  durationMs: number;
  resolvedModel?: string;
  inputTokens?: number;
  outputTokens?: number;
};

type Observe = (event: Observation) => void;

const labels = new Set<ModerationLabel>([
  "harassment",
  "discrimination",
  "fraud",
  "safety",
  "other",
]);

const responseSchema = {
  name: "property_moderation_classification",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    properties: {
      label: { type: "string", enum: [...labels] },
      needsHumanReview: { type: "boolean" },
      rationale: { type: "string" },
    },
    required: ["label", "needsHumanReview", "rationale"],
  },
} as const;

function parseClassification(value: unknown): Classification | null {
  if (typeof value !== "object" || value === null) return null;
  const item = value as Record<string, unknown>;
  if (!labels.has(item.label as ModerationLabel)) return null;
  if (typeof item.needsHumanReview !== "boolean") return null;
  if (typeof item.rationale !== "string") return null;
  return item as Classification;
}

export async function classifyReport(
  report: { id: string; text: string },
  observe: Observe,
): Promise<Classification> {
  const endpoint = process.env.MODEL_ROUTER_URL;
  const apiKey = process.env.MODEL_ROUTER_API_KEY;
  if (!endpoint || !apiKey) throw new Error("Missing model router configuration");

  const route = "moderation-primary";
  const started = performance.now();
  let outcome: Observation["outcome"] = "transport_error";
  let resolvedModel: string | undefined;
  let inputTokens: number | undefined;
  let outputTokens: number | undefined;

  try {
    const response = await fetch(endpoint, {
      method: "POST",
      signal: AbortSignal.timeout(8_000),
      headers: {
        authorization: `Bearer ${apiKey}`,
        "content-type": "application/json",
      },
      body: JSON.stringify({
        model: route,
        messages: [
          {
            role: "system",
            content:
              "Classify the property moderation report. Escalate uncertainty for human review.",
          },
          { role: "user", content: report.text },
        ],
        response_format: { type: "json_schema", json_schema: responseSchema },
      }),
    });

    if (!response.ok) throw new Error(`Classification transport status ${response.status}`);
    const payload = (await response.json()) as {
      model?: string;
      choices?: Array<{ message?: { content?: string } }>;
      usage?: { prompt_tokens?: number; completion_tokens?: number };
    };

    resolvedModel = payload.model;
    inputTokens = payload.usage?.prompt_tokens;
    outputTokens = payload.usage?.completion_tokens;
    const content = payload.choices?.[0]?.message?.content;
    const parsed = typeof content === "string" ? JSON.parse(content) : null;
    const classification = parseClassification(parsed);

    if (!classification) {
      outcome = "invalid_output";
      throw new Error("Classification did not match the required schema");
    }

    outcome = "accepted";
    return classification;
  } catch (error) {
    if (error instanceof DOMException && error.name === "TimeoutError") {
      outcome = "timeout";
    }
    throw error;
  } finally {
    observe({
      reportId: report.id,
      route,
      outcome,
      durationMs: Math.round(performance.now() - started),
      resolvedModel,
      inputTokens,
      outputTokens,
    });
  }
}
```

There is an intentional hard edge in this example: the function throws instead of manufacturing a classification. The caller should catch that error and enqueue the report for human review. Fast failure beats quiet corruption.

Treat `429` as a distinct transport outcome in production and retry it with capped exponential backoff plus jitter. The inference request does not mutate the moderation record, but the later review-queue write does; attach the report ID as an idempotency key there so a retry cannot create duplicate work for reviewers.

One subtle issue deserves attention. `JSON.parse` can throw before local schema validation, and the `finally` block still records the observation. Good. The outcome remains `transport_error` in this compact version, however, so a production implementation should catch parse errors separately and mark them `invalid_output`. That is an implementation refinement in the sample's generic client, not a provider claim. It also creates a clean unit test: return malformed content, assert that no classification escapes, and assert that the review path receives the report.

## Evaluate telemetry before every route change

A route flip changes system behavior even when application code does not change. Give it the same controls as a release. Build a frozen evaluation corpus from approved, de-identified examples; include ordinary reports, ambiguous language, misspellings, prompt-injection attempts, and cases where the correct answer is human review. Version that corpus and the JSON Schema. Then run every candidate route against it before traffic moves.

The dashboard should compare old and candidate routes over the same window, but raw agreement is not enough. Watch label distribution, human-review rate, invalid-output rate, timeout rate, and latency percentiles. Split those signals by route alias and contract version. Alert on sustained changes that matter to the moderation operation, not on every individual model error; individual failures already have the fail-closed review path.

Roll out in steps. Shadow traffic can test parsing and latency without changing queue assignment, provided data handling permits sending the report to both routes. A small live slice follows. The final switch happens only after reviewers examine disagreements. Keep the previous alias mapping available for rollback and attach the mapping revision to each observation so an incident timeline can answer which route classified a report.

Batch processing is a separate mode, not a transparent substitute for an interactive review path. OpenAI documents that its Batch API completes within a 24-hour window, so it can fit backfills or offline evaluation but not a moderation queue with a shorter service objective. Correctness and review latency still decide the architecture.

## Trade-offs and stopping rules

Choose direct adapters when provider-native behavior is part of the product and the team can operate each integration. Choose a shared chat gateway when one credential, a stable message envelope, and reversible route mapping reduce application coupling. Choose an internal classification service when moderation policy and human-review workflow need their own release cycle.

None of these options removes evaluation work. A common API can normalize transport, credentials, and telemetry, but it cannot make different models produce identical judgments. It can also become a central failure domain, so its availability target, rate limits, credential rotation, and rollback procedure need owners before applications depend on it.

The final decision rule is short: **adopt the smallest boundary that can fail closed, preserve the label contract, and explain every route change from telemetry.** If it cannot do all three, one API key is merely tidier configuration.

## References

- https://platform.openai.com/docs/guides/batch
- https://platform.openai.com/docs/guides/structured-outputs
