# Unified LLM API Chatbot Fallback with Application Recovery over Centralized Failover

Application-owned recovery is the better default when a unified LLM API drives a multi-provider chatbot that turns property sales calls into CRM actions. Keep the model choice behind a small application policy, then retry only work that is safe to repeat. A centralized router still fits a team that needs one operational control plane across many applications and accepts less local context.

| Option | Pick it when | Main trade-off |
| --- | --- | --- |
| Application-owned recovery | The chatbot knows call type, CRM state, and action risk | More code and more tests in the application |
| Centralized routing | Several teams need shared provider policy and shared telemetry | The router may not see enough business context to choose a safe retry |

The key boundary is not “which model is best?” It is “who can tell a transient generation failure from a dangerous duplicate CRM write?” For property sales calls, that answer usually lives beside the action executor.

## How should a unified LLM API route a multi-provider chatbot fallback?

Start with the output contract. A call summary is cheap to retry if it has no side effect. “Create a follow-up task for the prospect” is different: replaying it can create two tasks, two notifications, or two owners.

That split gives the application a useful policy. Route or retry extraction when the response is missing, malformed, or too slow. Before any CRM mutation, require an idempotency key derived from the call record and action type. The fallback path should receive the same transcript, instructions, schema, and correlation ID. A change in provider must not quietly become a change in business rules. In a unified API, preserve those fields in the adapter contract instead of letting each provider-specific branch invent its own defaults.

A centralized layer remains reasonable when the organization needs a single place to enforce timeouts, quotas, audit fields, and provider credentials. It becomes a poor fit when it decides based only on an HTTP error while the application knows that a “send brochure” action is lower risk than “change lease status.”

## What does automatic failover actually protect?

Failover protects availability only for the part of the workflow that can be repeated safely. It cannot repair an ambiguous side effect.

Think of the flow as a sentence: transcript in, structured proposal out, validation, CRM action, receipt. Recovery belongs around each verb. A timeout before a structured proposal can trigger a bounded retry. A timeout after the CRM accepted the request must trigger reconciliation, not blind replay.

The most common trap is treating every failure as a provider failure. A valid response can still be unusable: the action name may be unknown, a required contact ID may be absent, or the summary may exceed the CRM field limit. Those are contract failures. Send them to a repair or review path; do not rotate providers and hope the next answer has the missing field.

For token-sensitive transcripts, measure the input before routing. The `tiktoken` library is an official BPE tokenizer library, and its repository documents that tokenization depends on the encoding used. That makes token counts a planning signal, not a universal latency guarantee. Keep the exact transcript available for audit, but pass a deliberately bounded representation to each attempt when the workflow permits it.

**A small recovery policy that stays visible.**

The policy below keeps provider selection separate from CRM mutation. It is intentionally boring. Boring is useful when a leasing coordinator is waiting for a follow-up task to appear.

```ts
type Draft = {
  summary: string;
  actions: Array<{
    type: string;
    contactId: string;
    idempotencyKey: string;
  }>;
};

type Attempt = {
  provider: string;
  timeoutMs: number;
};

async function makeDraft(
  input: string,
  attempts: Attempt[],
  generate: (provider: string, input: string, timeoutMs: number) => Promise<unknown>,
): Promise<Draft> {
  let lastError: unknown;

  for (const attempt of attempts) {
    try {
      const candidate = await generate(attempt.provider, input, attempt.timeoutMs);
      return validateDraft(candidate);
    } catch (error) {
      lastError = error;
      if (!isRetryable(error)) throw error;
    }
  }

  throw new Error(`draft_generation_failed: ${String(lastError)}`);
}

function validateDraft(value: unknown): Draft {
  if (!value || typeof value !== "object") {
    throw new Error("invalid_draft");
  }

  const draft = value as Partial<Draft>;
  if (typeof draft.summary !== "string" || !Array.isArray(draft.actions)) {
    throw new Error("invalid_draft");
  }

  return draft as Draft;
}
```

The omitted pieces deserve explicit tests: `isRetryable` must distinguish a timeout from a validation error, and the CRM client must deduplicate by `idempotencyKey`. Do not hide either decision in a generic “fallback” helper. When an incident happens, the log should answer which attempt produced the draft, which validation failed, and whether the action executor ran.

Retrying is not recovery.

## How should teams measure quality versus latency?

Record separate timestamps for transcript preparation, each generation attempt, validation, CRM execution, and final acknowledgement. One end-to-end number cannot tell you whether a long wait came from a large input, a retry, or a slow CRM.

Track action precision in a review sample, not only successful HTTP responses. A summary can look fluent while assigning the wrong property or contact. Track fallback rate, validation-failure rate, duplicate-prevention hits, and human-review rate beside latency percentiles. Set a decision rule before launch: for example, a slower path can be acceptable for lease-status changes if it materially reduces incorrect actions, while a low-risk brochure task can use a tighter deadline and a review queue. The threshold belongs to the business action, not to the model brand.

Use structured logs with a correlation ID and a redacted input reference. Store enough metadata to reproduce the policy decision: schema version, attempt number, timeout, validation result, and action key. Avoid logging full call transcripts by default; sales calls can contain personal information, and observability should not become an accidental second CRM.

## Limits of the recommendation

Application-owned recovery adds surface area. It needs contract tests for every fallback path, a replay tool for ambiguous CRM results, and an explicit manual-review state. A centralized router can reduce duplicated platform work and make organization-wide limits easier to enforce.

Neither design guarantees a good summary. Routing can preserve availability; validation and review protect the business action. Choose application ownership when local context decides whether repetition is safe. Choose centralized control when policy consistency across many applications is the harder problem. Keep those boundaries visible in code and in telemetry.

## References

- https://github.com/openai/tiktoken
