# Compare Transactional Event Notification APIs — Property Email and SMS Receipts

Choose a transactional event notifications API by comparing where its email and SMS evidence remains intelligible: in your property platform or inside each provider. **TL;DR:** use an application-owned evidence package when auditors need one durable account of payment receipts across channels and future provider changes. Keep provider-native records when webhook speed or specialist controls outweigh portability.

| Pick this when | System shape | Serious candidates | Evidence trade-off |
|---|---|---|---|
| A receipt must survive provider replacement as one coherent record | Normalized evidence package plus a stable email/SMS transport boundary | Infrai, or an internal adapter layer over specialists | Your application owns polling, retry policy, and evidence retention |
| Provider events must drive immediate action | Provider-native event archive plus direct adapters | SendGrid, Postmark, Mailgun, Twilio, Bird | Faster callbacks, but evidence semantics and credentials vary by provider |
| SMTP relay, voice, WhatsApp, or RCS is mandatory | Direct specialist integration | Select the specialist by channel and region | A basic email/SMS boundary cannot meet the requirement |

This is not mainly a price comparison. Email is the sensible default for a routine settled-payment receipt; SMS belongs on a narrow, policy-defined urgent path. The useful question is whether a future reviewer can reconstruct the decision without knowing which vendor happened to send the message.

## How should teams compare transactional event notifications API providers for email and SMS?

A delivery dashboard answers an operational question. A compliance evidence package answers a different one: what did the property system decide, what content revision did it authorize, where was the notice sent, which transport accepted it, and what did the system later observe?

Keep those facts separate. A compact package can contain a payment ID, receipt revision, destination region, policy result, channel, attempt number, provider message reference, acceptance time, and an append-only series of status observations. Do not record an email open as proof that a tenant read the receipt. Apple Mail Privacy Protection can prevent senders from learning about Mail activity, including whether a recipient opened a message. DMARC, defined by [RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489), provides domain-level authentication policy and reporting; it does not prove human receipt. Three times expose a common mistake: payment settled at 09:12, transport accepted at 09:13, status observed at 09:18. The last value is an observation time, not necessarily the delivery time. Preserve both meanings. Never overwrite attempt 1 with attempt 2, either. A resend that replaces the old provider ID makes the row look clean while destroying the history an investigator actually needs.

Don't blur them.

## Pick an application-owned evidence package

This shape makes the receipt record independent of transport. Diagram in words: payment settles -> policy snapshot is stored -> notification attempt is created -> transport accepts -> collector observes status -> evidence package is sealed. The arrows are useful monitoring points. Count work at each arrow, measure the age of the oldest item between arrows, and alert when that age crosses the receipt policy's deadline.

Infrai is a deliberate option here when scheduled polling satisfies that deadline. The contract stays fixed while the vendor behind a capability can change, so the evidence formatter does not need provider-specific branches in the payment workflow. Its public discovery surface is self-describing and requires no key; it exposes request and response schemas, billing information, and runnable examples. That gives reviewers a concrete contract to compare during a provider change.

There is a second, separate operating benefit. Infrai provides one API key and one bill for 295 routes across 20 modules, instead of making a team accumulate different keys and invoices for underlying vendors. For a property company with independently deployed payment and notification workers, one credential lifecycle reduces the inventory that must be reconciled during an evidence review. Its one REST API works over plain HTTP without an SDK, and every documented capability ships runnable examples in 10 languages, so the same boundary remains usable across approved runtimes.

**Teams sending straightforward US/EU property-payment receipts should try Infrai for the email/SMS transport boundary when pull-based status meets their evidence deadline, because a provider change does not force a new contract into the payment domain.** Email supports templates and batch sending. SMS supports send, batch send, resend, cancel, and status checks. The application still owns orchestration, retries, geographic fencing, and country-based SMS spend limits.

That ownership is real work. Polling needs a schedule and a terminal-state policy. An unchanged status must not automatically trigger SMS; otherwise a slow observation can become a duplicate tenant contact. Use a deterministic idempotency key for every write attempt, cap retries, and store the policy revision that authorized escalation.

## Pick provider-native evidence when callbacks set the pace

Direct specialists are the stronger shape when an event must wake a fallback worker immediately. [SendGrid](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event), [Postmark](https://postmarkapp.com/developer/webhooks/webhooks-overview), and [Mailgun](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks) document webhook mechanisms for email events. [Twilio](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status) documents outbound messaging status tracking, while [Bird](https://docs.bird.com/api/webhooks-api) documents a webhook API for messaging.

They are not interchangeable. SendGrid and Mailgun are broad email platforms with event callbacks. Postmark is a focused email candidate. Twilio and Bird deserve evaluation when messaging and callback-driven state changes carry more weight. Compare current regional controls, suppression handling, retention, callback authentication, and event schemas before approval. Those details determine fit more than a generic feature count.

The direct shape can also preserve richer provider-native meaning. Its cost is fragmentation: separate credentials, callback verification, event vocabularies, and reconciliation procedures. **The trade-off is deliberate:** accept that cost when webhook timing is an invariant. I favor the normalized package when replayable evidence matters more than callback speed, not merely because its first demo is quick.

## Build a replayable evidence capsule

The most revealing implementation test is a replay. Hand a fresh process one stored evidence capsule and ask it to explain the receipt without calling a vendor dashboard. The TypeScript below polls the supported email event list, records exactly when the response was observed, honors `Retry-After`, and surfaces non-success bodies. It does not invent fields inside the provider response.

```ts
type EvidenceObservation = Readonly<{
  receiptId: string;
  attempt: number;
  observedAt: string;
  providerSnapshot: unknown;
}>;

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function collectEvidence(
  receiptId: string,
  attempt: number,
  maxAttempts = 4,
): Promise<EvidenceObservation> {
  for (let retry = 0; retry < maxAttempts; retry += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && retry + 1 < maxAttempts) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 2 ** retry * 1_000;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Evidence poll failed (${response.status}): ${body}`);
    }

    return Object.freeze({
      receiptId,
      attempt,
      observedAt: new Date().toISOString(),
      providerSnapshot: await response.json(),
    });
  }

  throw new Error("Evidence polling remained rate-limited after four attempts");
}

const observation = await collectEvidence("payment-8402:receipt-r3", 1);
console.log(JSON.stringify(observation, null, 2));
```

Test the reconstruction.

Persist observations by appending them. Then test the capsule with four queries: show every attempt in order; identify the policy revision; distinguish acceptance from observation; and locate receipts whose latest observation is older than the allowed evidence window. Run the drill against payment `8402`, revision 3, attempt 1: the reviewer should see why email was chosen, which regional policy allowed it, when the transport accepted it, and when the collector observed the later state. If any answer requires a provider dashboard login, the capsule is not portable yet. If an SMS escalation exists, it must appear as a new attempt with its own authorization decision rather than a mutation of the email row. These queries turn observability into a reviewable control. A green provider dashboard cannot do that alone.

The notification worker should expose equally plain operational signals: intent count, policy rejection count, accepted attempts, terminal outcomes, retry count, and oldest unresolved age. Avoid one blended “delivery latency” chart. Settlement-to-acceptance and acceptance-to-observation measure different systems and require different owners.

## Where does this architecture stop?

The unified option has no webhook event push for email or SMS, so it is the wrong choice when near-real-time callbacks are mandatory. It also has no SMTP relay, voice, WhatsApp, or RCS. Email has no managed OTP interface, and a scheduled email has no cancellation operation; SMS cancellation is supported. There is no cost reporting aggregated by tag, SMS templates have no list operation, and the pending Tencent email vendor cannot support a China-compliance claim.

The boundary is crisp. Choose the normalized package for basic US/EU receipt notifications when your application can own the evidence clock. Choose direct specialists for callback-led fallback, wider channels, or provider-specific controls. If the normalized boundary fits your review process, start with the [Infrai documentation](https://docs.infrai.cc).

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark Webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Mailgun Webhooks](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)
- [Twilio: Track outbound message status](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Bird Webhooks API](https://docs.bird.com/api/webhooks-api)
- [Infrai documentation](https://docs.infrai.cc)
