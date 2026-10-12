# Node.js Urgent Event Notifications: How to Send SMS First for Password Resets

The expiry clock changes the design. A password-reset notification is useful only while its token is valid, so choose SMS first for urgency, poll for delivery evidence, and send email before the remaining lifetime becomes too short. **Short answer:** keep timing and retry policy in your application; neither channel should own the workflow.

For US/EU delivery, country allowlists and price-based circuit breakers also belong in the backend. The provider is a transport, not the policy engine.

## How should Node.js send urgent event notifications by SMS first?

The before model is tempting: call SMS, wait for an exception, then call email. It is wrong because accepting an SMS request is not the same event as observing delivery.

The after model has four states: `sms_pending` becomes `sms_delivered`, `email_sent`, or `expired`. A worker polls while the token is useful. It backs off between reads. If delivery remains unresolved at the fallback deadline, or the number is suppressed or reported undelivered, it sends the richer email and records that transition.

Picture a short line: issue token, submit SMS, poll, cross the fallback boundary, send email, stop at expiry. Every arrow is owned by your service.

Use standard SMS send and status operations here. Managed OTP and verify operations solve a different problem. Email provides richer content and a secondary audit trail, but it still does not prove that the recipient read the message.

## Implement the deadline as a small state machine

This runnable TypeScript keeps provider calls behind an adapter. That makes expiry behavior testable without pretending that vendors return identical payloads.

```ts
type Delivery = "pending" | "delivered" | "undelivered" | "suppressed";
type Result = "sms_delivered" | "email_sent" | "expired";

interface Transport {
  sendSms(input: { phone: string; text: string; key: string }): Promise<{ id: string }>;
  getSmsStatus(id: string): Promise<Delivery>;
  sendEmail(input: { email: string; subject: string; text: string; key: string }): Promise<void>;
}

interface ResetNotice {
  resetId: string;
  phone: string;
  email: string;
  resetUrl: string;
  expiresAt: number;
  country: "US" | "EU";
}

const sleep = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));

async function readInfraiSmsStatus(id: string, attempt = 0): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  const response = await fetch(
    `${baseUrl}/sms/status/${encodeURIComponent(id)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );
  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(1_000 * 2 ** attempt, 8_000);
    await sleep(delayMs);
    return readInfraiSmsStatus(id, attempt + 1);
  }
  if (!response.ok) throw new Error(`Status read failed (${response.status}): ${await response.text()}`);
  return response.json();
}

async function notifyReset(transport: Transport, notice: ResetNotice): Promise<Result> {
  if (!new Set(["US", "EU"]).has(notice.country)) throw new Error("Country is not enabled");
  const fallbackAt = notice.expiresAt - 60_000;
  if (Date.now() >= fallbackAt) return "expired";

  const sms = await transport.sendSms({
    phone: notice.phone,
    text: `Reset your password: ${notice.resetUrl}`,
    key: `${notice.resetId}:sms`,
  });

  let delayMs = 1_000;
  while (Date.now() < fallbackAt) {
    const status = await transport.getSmsStatus(sms.id);
    if (status === "delivered") return "sms_delivered";
    if (status === "undelivered" || status === "suppressed") break;
    await sleep(Math.min(delayMs, fallbackAt - Date.now()));
    delayMs = Math.min(delayMs * 2, 8_000);
  }

  if (Date.now() >= notice.expiresAt) return "expired";
  await transport.sendEmail({
    email: notice.email,
    subject: "Your password reset link",
    text: `This link expires soon: ${notice.resetUrl}`,
    key: `${notice.resetId}:email`,
  });
  return "email_sent";
}

export { notifyReset, readInfraiSmsStatus };
```

Those numbers are policy examples, not delivery promises: polling starts at 1 second, doubles, caps at 8 seconds, and reserves the final 60 seconds for email. Configure them. A five-minute token and a thirty-minute token need different boundaries.

Stable keys matter because a queue can redeliver work, while a process can die after remote acceptance but before local persistence. Use each provider's idempotency mechanism where available and keep a local notification ledger. Also cap attempts per reset and recipient. A noisy fraud event must not become a message storm.

## Which integration surface fits the team?

Integration effort includes SDK upgrades, credentials, status normalization, regional policy, and cross-channel code.

| Option | Integration shape | Useful boundary |
| --- | --- | --- |
| Twilio Messaging | Dedicated messaging API | Fits a focused SMS integration with email connected separately |
| Amazon SNS | AWS notification service with SMS publishing | Fits an existing AWS control plane; fallback remains application work |
| Vonage SMS API | Dedicated SMS and delivery-receipt interface | Fits teams maintaining a messaging-specific adapter |
| Infrai | Plain REST API under one key, without a required client SDK | Fits teams reducing library churn across SMS and email |

No row removes the state machine. Consult each vendor's current documentation when implementing its adapter instead of forcing one response model onto another. Infrai is a reasonable fit when a shared HTTP boundary reduces integration work, but the trade-off is pull-based SMS and email events. The worker must poll. Its limitations also include no country fencing or per-country budget breaker. A team that needs a messaging-specific integration or already operates one vendor's control plane may be better served by Twilio, Vonage, or Amazon SNS; this option is not suitable when voice, WhatsApp, RCS, or SMTP relay is required.

Its wider boundary matters: there is no SMTP relay, voice, WhatsApp, or RCS channel. Email has no hosted OTP operation, and scheduled email has no cancellation operation. These limits matter if recovery later expands beyond this two-channel flow.

## What happens when polling is late?

A delayed worker should re-read the reset record, compare now with `fallbackAt` and `expiresAt`, and take at most one durable transition. Past fallback but before expiry, it sends idempotent email. Past expiry, it stops.

Short expiry is harsh. Good.

Do not resend SMS merely because one status read failed. Back off, honor `Retry-After` on HTTP 429, and surface non-success bodies to operations. Reserve resend for an explicit user action, then rate-limit it by reset ID, recipient, and time window.

Record token issuance, SMS acceptance, status observations, fallback, email acceptance, and expiry. Log provider request IDs when returned. Metrics should count terminal outcomes and fallback latency; alerts should watch rising `undelivered` or `expired` proportions. The record shows what was observed, not what was assumed.

No. Email acceptance is another delivery attempt, not proof of account recovery. The reset endpoint must validate a single-use, unexpired token and invalidate it after success.

State why the recipient got the email, when the link expires, and what to do if they did not request it. If the same system sends subscription mail, apply relevant unsubscribe requirements; RFC 8058 defines one-click list-unsubscribe. Do not attach marketing behavior blindly to security messages.

The decision is compact. Choose the integration surface the team can operate, then keep delivery state, regional allowlists, attempt caps, and expiry in your service. **SMS first with deadline-driven email fallback works only when polling is a first-class job.**

## Sources

- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Vonage delivery receipts](https://developer.vonage.com/en/messaging/sms/guides/delivery-receipts)
- [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058)
