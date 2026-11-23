# Node.js SMS Alerts: Audit Evidence for Passwordless Backup and US/EU Account Notifications

A storefront must be able to explain why it sent a signup verification link, even after the link has expired. That requirement changes how to choose an SMS alert service for US/EU shoppers. TL;DR: compare Twilio, Vonage, and Telnyx on the evidence your application can preserve. Keep authorization, delivery observations, and account verification separate. For passwordless backup alerts and account notifications, a transport receipt answers only one part of the review.

Start with evidence.

The useful decision axis is compliance evidence: can a reviewer reconstruct one signup without retrieving its live credential? Define that record before choosing the transport. A low quoted SMS rate cannot answer this question, and this note does not rank providers by unmeasured delivery performance.

## Decide what the reviewer should be able to reconstruct

Imagine a review of a single e-commerce account signup. The reviewer needs the purpose of the message, the relevant authorization record, the transport observation, and the application's verification outcome. Those records can share an attempt identifier without sharing the phone number or verification link. This is a proposed application design, not a claim about any provider's audit schema. Its practical benefit is that a delivery investigation can follow the attempt while credentials stay out of ordinary logs.

Before: the dashboard has one green badge labeled “success.” After: it has separate evidence for request authorization, transport observation, and verified signup. Diagram in words: shopper requests account → application records purpose → SMS carries verification link → transport observation enters the ledger; separately, valid link redemption → application records verification. The two branches meet at an attempt identifier.

Delivery isn't authentication.

GDPR Article 7 requires demonstrable consent where processing relies on consent. It does not decide the legal basis of every transactional signup message. Have the responsible reviewer establish that basis and the retention policy, then make the implementation preserve the evidence that decision requires. A policy version helps locate the governing rules; it does not prove a shopper agreed to them. Likewise, approval to send a signup link should not silently become approval for unrelated account messaging.

OWASP's Forgot Password Cheat Sheet recommends securely generated, single-use recovery tokens with appropriate expiration. Use that guidance as a security review lens for signup links. Keep the token and complete verification URL out of the evidence stream. A searchable log containing a working link creates another place from which the credential could be used.

## Can a Node.js dashboard keep those facts separate?

Yes. Make the separation visible in the data model before connecting a provider. First collect a transport observation for an existing SMS message. Set `SMS_API_ORIGIN` to your configured HTTPS service origin without a version suffix, supply `SMS_MESSAGE_ID`, and inject `INFRAI_API_KEY` through your secret manager. Save this TypeScript as `observe.ts` and run it with your project's TypeScript runner. The explicit GET is read-only. It reports observation time without guessing response fields; interpret the returned payload using the discovered schema before recording a delivery outcome.

```ts
async function observe(): Promise<void> {
  const origin = process.env.SMS_API_ORIGIN;
  const id = process.env.SMS_MESSAGE_ID;
  const key = process.env.INFRAI_API_KEY;
  if (!origin || !id || !key) throw new Error("Missing service configuration");
  const url = new URL(`/v1/sms/status/${encodeURIComponent(id)}`, origin);
  if (url.protocol !== "https:") throw new Error("HTTPS required");
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
      redirect: "error",
      signal: AbortSignal.timeout(10000),
    });
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const dateDelay = header === null ? NaN : Date.parse(header) - Date.now();
      const retryDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(dateDelay) ? Math.max(0, dateDelay) : 0;
      await response.body?.cancel();
      const delay = Math.max(retryDelay, 1000 * 2 ** attempt);
      if (delay > 60000) throw new Error("Reschedule observation after retry delay");
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const body = await response.text();
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${body}`);
    JSON.parse(body);
    console.log(JSON.stringify({ observedAt: new Date().toISOString() }));
    return;
  }
}

observe().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Configure the origin only from trusted deployment settings. Error bodies belong in restricted, appropriately redacted operational logs. This example discards the successful payload after checking JSON syntax; an observation timestamp alone does not establish delivery.

The next TypeScript example runs locally and uses synthetic storefront events. Save it as `evidence.ts` and run it with a TypeScript runner in your Node.js project. It makes no network requests and uses no vendor response fields.

```ts
type Evidence = {
  eventId: string;
  attemptId: string;
  at: string;
} & (
  | { kind: "send_authorized"; purpose: "signup_link"; policyVersion: string }
  | { kind: "transport_observed"; observationRef: string }
  | { kind: "signup_verified"; validationRef: string }
);

type Review = {
  authorized: boolean;
  transportObserved: boolean;
  verified: boolean;
};

function project(attemptId: string, events: Evidence[]): Review {
  const seen = new Set<string>();
  const review: Review = {
    authorized: false,
    transportObserved: false,
    verified: false,
  };
  for (const event of events) {
    if (event.attemptId !== attemptId || seen.has(event.eventId)) continue;
    seen.add(event.eventId);
    switch (event.kind) {
      case "send_authorized":
        review.authorized = true;
        break;
      case "transport_observed":
        review.transportObserved = true;
        break;
      case "signup_verified":
        review.verified = true;
        break;
    }
  }
  return review;
}

const attemptId = "store-signup-223";
const events: Evidence[] = [
  {
    eventId: "authorization-1",
    attemptId,
    at: "2026-11-20T09:00:00Z",
    kind: "send_authorized",
    purpose: "signup_link",
    policyVersion: "store-signup-v3",
  },
  {
    eventId: "observation-1",
    attemptId,
    at: "2026-11-20T09:00:30Z",
    kind: "transport_observed",
    observationRef: "restricted-record-17",
  },
];

const beforeRedemption = project(attemptId, events);
if (beforeRedemption.verified) {
  throw new Error("Transport evidence must not verify a signup");
}
events.push({
  eventId: "verification-1",
  attemptId,
  at: "2026-11-20T09:01:00Z",
  kind: "signup_verified",
  validationRef: "restricted-validation-8",
});
console.log(JSON.stringify({
  attemptId,
  beforeRedemption,
  afterRedemption: project(attemptId, events),
}));
```

The 30-second observation gap is invented example data, not measured latency or a polling recommendation. The output changes `verified` only after a verification event. Duplicate event identifiers are ignored within this projection, and events for a different attempt are excluded. Neither property prevents duplicate SMS sends. Dispatch retries need a stable operation identity and a documented idempotency mechanism of their own.

There is a boundary worth keeping sharp here: this code projects trusted application events. It neither validates tokens nor establishes authorization. Production code must emit the verification event only after validating the single-use token, persist evidence with access controls, and enforce the approved retention policy. The record references also need restricted backing records if a reviewer is expected to inspect them later. An identifier pointing to nothing is a dead end during review.

For observability, count authorized attempts, transport observations, and verified signups separately. Keep individual attempt identifiers in controlled logs rather than metric labels. A transport observation may describe an unsuccessful delivery; the example deliberately makes no claim about that outcome. Your adapter must interpret the documented status and preserve its meaning before the dashboard displays a delivery result.

## How should an SMS alert service compare passwordless backup and account notifications?

Give each candidate the same acceptance exercise: reconstruct a signup from an authorization record, a transport observation, and a verification outcome. Request a representative record and review the integration contract. The table compares evaluation paths; it does not assign undocumented features or certify regional compliance.

| Candidate | Concrete evaluation path | Selection boundary |
| --- | --- | --- |
| Twilio | Review its Messaging documentation against the signup evidence packet; check how the integration preserves the meaning of delivery states. | Choose when the reviewed SMS contract and application evidence meet your signup requirements. A recognizable provider name cannot establish the processing basis. |
| Vonage | Review its SMS API documentation and map the proposed receipt record into the same packet. | Choose when that mapping supports the required investigation and retention policy. Do not treat a receipt as link redemption. |
| Telnyx | Review its Messaging documentation and examine how repeated or delayed observations would enter the packet. | Choose when the resulting adapter preserves enough context for review. Do not infer authentication strength from transport events. |
| Resend | Review its email documentation for a separately evaluated email backup path. | Consider for email delivery when that channel fits the signup flow; this is not a substitute for evaluating SMS transport or application token validation. |
| Consolidated backend API | Assess shared administration alongside an explicit polling and application-owned fallback design. | Choose for straightforward SMS notifications when those responsibilities fit the team; assess richer channel orchestration separately. |

Run every inspection task against all three named providers. The different starting points are not claims that one provider lacks the others' capabilities. There is no verified delivery benchmark here, so the table cannot honestly produce a universal US/EU winner. Require evidence for sender requirements, data handling, and regional suitability in the actual deployment before approval.

Infrai offers one key and one bill, with a plain REST API and no SDK to install. This reduces the credentials and invoices the storefront team administers. The simple API covers 295 routes across 20 modules, so the team can review shared interface conventions for messaging and other backend services together. The Node.js reviewer can inspect a plain HTTP request directly without installing another client library. Public, unauthenticated discovery supplies request and response schemas, which lets a reviewer inspect the SMS contract before issuing a credential. These conveniences reduce friction in the evidence review. They do not certify the application.

For this option, use polling jobs to collect message status and events. Keep observation timestamps so a reviewer can distinguish provider information from the time the application learned it. Application logic owns email verification and cross-channel backup decisions; geographic abuse controls and country-based spending cutoffs also belong in the application design. If the requirements include voice, WhatsApp, RCS, an SMTP relay, or cancellable scheduled email, evaluate a service contract that explicitly supports them. A US/EU signup review supplies no basis for a domestic-China compliance claim.

## Would polling make the evidence too stale?

It can. Pick an acceptable observation age before picking a polling interval. For illustration, a team might review a 60-second freshness target; that is a proposed requirement, not an advertised provider guarantee. Show the last observation time in the dashboard and alert on aging unobserved attempts. Silence should stay visible.

Faster polling trades more requests for fresher evidence. On HTTP 429, honor Retry-After when present and use bounded exponential backoff; reschedule the job when the required delay exceeds its execution budget. Preserve the distinction between the event timestamp, if documented, and the application's observation timestamp. Overwriting one with the other makes a delayed observation look like a late delivery, which changes the story an investigator sees.

## Does this create an unnecessary second ledger?

It adds application responsibility. That is a real trade-off, especially for a small storefront team. Keep the record narrow: an attempt identifier, the authorized purpose and policy reference, a transport reference, and the independently validated signup outcome. The example's three event kinds are a teaching boundary, not a universal compliance checklist.

Provider records remain useful for transport investigations. The application record connects them to the shopper's requested action and its own authentication decision. Before launch, ask a reviewer to reconstruct one synthetic attempt with duplicate observations and no redeemed link. If the dashboard still labels that account verified, fix the projection before choosing a vendor. That exercise produces a concrete purchasing requirement: transport evidence must fit the application's authorization and verification story.

## Sources

References for the security guidance, consent distinction, and provider evaluation:

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- GDPR Article 7, Conditions for consent: https://gdpr-info.eu/art-7-gdpr/
- Twilio Messaging documentation: https://www.twilio.com/docs/messaging
- Vonage SMS API documentation: https://developer.vonage.com/en/messaging/sms/overview
- Telnyx Messaging documentation: https://developers.telnyx.com/docs/messaging
- Resend documentation: https://resend.com/docs/introduction
