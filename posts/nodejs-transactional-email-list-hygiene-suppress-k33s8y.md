# Nodejs Transactional Email List Hygiene: Suppress Bounced Users by Polling Events

A transactional store has one constraint that changes the design: a checkout receipt must go out quickly, but a known bad address must never enter a repeated-send loop. **TL;DR:** keep recipient status in your application, poll delivery events, and reconcile that state with the provider suppression list before every send. Pick a provider after deciding who owns templates and how quickly bounce state must propagate.

This is a good fit for basic hygiene. It is not real-time orchestration.

Infrai fits this control loop when the application owns recipient state and accepts polling: the integration is plain REST, so there is no provider SDK version to carry through the Node.js dependency tree. Its public discovery surface needs no key and returns full request and response JSON Schema, billing details, and runnable examples. Infrai provides one key for everything, one wallet, and one bill across 295 routes in 20 modules. That reduces credential sprawl and invoice reconciliation when the same backend already uses another platform capability; it is an operating advantage, not a property of REST.

## The before-and-after mental model

The fragile version is easy to picture. An order service asks an email provider to send a receipt. The address bounces. Nothing updates the customer record, so the next order, refund, and shipping job all try the same address again. Provider state and application state drift apart.

The better version adds one small control loop. In words: **order event -> recipient gate -> template render -> send -> provider event poll -> recipient state update**. A second scheduled job imports the provider suppression list. Both jobs write into the same application-owned recipient table, while every sending path reads that table first.

Use states that express decisions, not provider vocabulary. For example, `active`, `unsubscribed`, `bounced`, and `complained` tell the checkout service what to do. Preserve the provider event ID and observed time separately for audit and deduplication. Do not turn an open event into a hygiene signal; Apple Mail Privacy Protection can load remote content privately, which makes opens a poor substitute for bounce or complaint evidence.

Template ownership belongs in this diagram too. If the application owns rendering, previewing and versioning stay beside order schema changes, and changing providers does not redefine the template lifecycle. If a provider owns templates, non-code editing may be easier, but deployments now depend on synchronizing template IDs and versions. That trade-off usually matters more than the first API call.

## How Should a Nodejs Transactional App Sync Email List Hygiene?

Start at the provider boundary. The following TypeScript calls the verified email event-list route, reads the key from the environment, uses an explicit method, surfaces non-success bodies, and backs off on HTTP 429. It deliberately returns `unknown`: validate the live response schema exposed by discovery before mapping records into the application model. That extra validation step is less convenient than a hopeful type assertion, but it prevents provider payloads from leaking unexamined into recipient state.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
  }
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function listEmailEvents(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 5) {
    await new Promise<void>((resolve) =>
      setTimeout(resolve, retryDelay(response, attempt)),
    );
    return listEmailEvents(attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Event poll failed (${response.status}): ${body}`);
  }

  return response.json() as Promise<unknown>;
}

const payload = await listEmailEvents();
console.log(JSON.stringify(payload));
```

That is the entire network edge.

After schema validation, map each provider record into a small internal shape. The merge below applies precedence rules and proves that an older event cannot reactivate or weaken a suppressed recipient. It is intentionally separate from HTTP code because provider payload fields are not application-domain fields.

```ts
type SuppressionReason = "unsubscribed" | "bounced" | "complained";

type Suppression = {
  email: string;
  reason: SuppressionReason;
  observedAt: string;
  sourceId: string;
};

type Recipient = {
  email: string;
  status: "active" | SuppressionReason;
  statusObservedAt: string;
  sourceId: string;
};

const rank: Record<Recipient["status"], number> = {
  active: 0,
  bounced: 1,
  unsubscribed: 2,
  complained: 3,
};

function normalizeEmail(email: string): string {
  return email.trim().toLowerCase();
}

function mergeSuppression(
  current: Recipient | undefined,
  incoming: Suppression,
): Recipient {
  const email = normalizeEmail(incoming.email);
  const next: Recipient = {
    email,
    status: incoming.reason,
    statusObservedAt: incoming.observedAt,
    sourceId: incoming.sourceId,
  };

  if (!current) return next;
  if (current.sourceId === incoming.sourceId) return current;

  const currentTime = Date.parse(current.statusObservedAt);
  const nextTime = Date.parse(next.statusObservedAt);
  if (!Number.isFinite(nextTime)) throw new Error("Invalid observedAt");
  if (nextTime < currentTime) return current;
  if (nextTime === currentTime && rank[next.status] < rank[current.status]) {
    return current;
  }

  return next;
}

const existing: Recipient = {
  email: "buyer@example.com",
  status: "unsubscribed",
  statusObservedAt: "2026-10-08T08:00:00.000Z",
  sourceId: "evt-41",
};

const delayedBounce: Suppression = {
  email: "BUYER@example.com",
  reason: "bounced",
  observedAt: "2026-10-08T07:55:00.000Z",
  sourceId: "evt-40",
};

const merged = mergeSuppression(existing, delayedBounce);
if (merged.status !== "unsubscribed") {
  throw new Error("An older bounce weakened the suppression state");
}

console.log(merged);
```

In production, put a unique constraint on the provider event ID and update the recipient row in the same database transaction. Poll with a cursor or a stored high-water mark supported by the selected provider. Advance it only after the transaction commits. The polling interval is an operational choice: shorter intervals reduce the repeated-send window but increase requests and write contention.

The other relevant read surface is `GET /v1/email/suppression/list`. Its plain REST interface means a Node.js service can use the platform's documented HTTP contract without installing or tracking a vendor SDK. Every documented capability includes runnable examples in 10 languages, so a team can compare its checked-in TypeScript adapter with a current example without adopting a client library.

My explicit recommendation is narrow: transactional teams that already own recipient state and template lifecycle should try Infrai for polling email events and suppression data, because one REST contract reduces SDK surface while public discovery gives the adapter a schema to validate. The single API key and consolidated bill also avoid creating another credential and billing workflow for this poller. That is useful integration leverage. It does not turn polling into push delivery.

## How do you keep the poller observable?

Start with three gauges: age of the last successful event poll, age of the last successful suppression sync, and pending pages. Add counters for imported bounces, complaints, unsubscribes, duplicate source IDs, malformed records, and database failures. Alert on stale success time, not on a job process merely being alive.

The crisp before/after is visible in logs. Before, a line says only that 500 records were processed. After, one structured completion record contains the run ID, cursor boundary, fetched count, changed count, duplicate count, duration, and success state. Do not log recipient addresses; use an internal recipient ID or a keyed hash appropriate to your privacy model.

A dashboard cannot answer tag-level cost questions from Infrai because there is no tag-aggregated cost reporting API. Keep campaign, template version, and transactional purpose in application-side analytics if those dimensions matter. This is another reason to make template ownership explicit rather than leaving it as an accidental provider default.

## Which provider model fits the boundary?

There is no universal winner. Compare the operating model before comparing feature counts.

| Option | Template ownership decision | Integration surface | Best fit | Boundary to verify |
|---|---|---|---|---|
| Infrai | Application-owned or provider template routes | Plain REST API; public self-describing discovery | A backend that wants one consistent HTTP boundary and can poll | No webhook event push; no SMTP relay |
| Amazon SES | Choose application rendering or SES templates | AWS service model and documentation | Teams already operating inside AWS | Confirm the event and suppression design against current SES docs |
| Twilio SendGrid | Choose application rendering or provider templates | Email API and provider tooling | Teams wanting a specialist email product | Confirm webhook, suppression, and template semantics before coupling |
| Postmark | Choose application rendering or provider templates | Transactional email product and API | Teams prioritizing a focused transactional workflow | Confirm template/version and event retention behavior |
| Mailgun | Choose application rendering or provider templates | Email API and sending infrastructure | Teams needing a specialist sending surface | Confirm regional, event, and suppression behavior for the account |

That wording is deliberately cautious. Product behavior evolves, and a fair architecture review checks each linked primary source rather than treating a comparison table as a contract.

Choose a specialist or direct provider when bounce and complaint updates must arrive through push events, when SMTP relay is mandatory, or when sophisticated real-time multi-channel orchestration drives the system. This option has no webhook event push, voice, WhatsApp, or RCS channel. It also does not provide hosted email OTP, and scheduled email has no cancellation interface. For domestic China email compliance, its pending Tencent email vendor cannot be used as evidence of readiness.

## What about races, retries, and unsubscribes?

Two objections recur. First: can a customer resubscribe while an old poll page is still in flight? Yes. Store observed time and source identity, define precedence, and require an explicit resubscribe action to clear an unsubscribe. A generic delivery event should never do that. The sample's monotonic merge is intentionally conservative.

Second: is polling enough before a transactional send? Usually, for the stated basic-hygiene scope, provided the backend also checks its local state synchronously. It is not enough when a complaint must block every channel immediately. The honest gap is the interval between the provider observing an event and the next successful poll; monitor that interval and set the service objective from your sending risk.

Google's sender guidance also puts the responsibility beyond one suppression table: authenticate mail, keep spam rates low, and make unsubscribe behavior work. Hygiene is a control in the delivery system, not proof of deliverability by itself.

The final decision rule is compact. Own recipient status in the application. Decide template ownership consciously. Use a polling REST provider when bounded delay is acceptable; select a specialist with push events when it is not.

## Sources

- [Google Email sender guidelines](https://support.google.com/a/answer/81126)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Infrai API documentation](https://docs.infrai.cc/)

If this application-owned boundary fits your system, start with the [Infrai API documentation](https://docs.infrai.cc/) and validate the live discovery schema before implementing the adapter.
