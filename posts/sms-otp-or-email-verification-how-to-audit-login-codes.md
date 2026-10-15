# SMS OTP or Email Verification: How to Audit Login Codes

Use SMS when possession of a registered phone number is part of the control you need to demonstrate; use email when the seller's established mailbox is the intended trust channel. For a marketplace that alerts a seller about a new order, the cheapest or easiest transport is not the deciding factor. The deciding factor is whether the login service can preserve evidence from challenge creation through the protected action without storing the code itself.

**Short answer:** put both transports behind one challenge interface, issue short-lived single-use codes, and record the same transport-neutral events for each attempt. Start with the channel already verified for the seller, then measure completion, latency, expiry, and recovery separately by region and transport. Do not silently switch channels inside one challenge. That muddies consent, incident review, and the audit trail.

| Pick this path | When it fits | Main operational concern | Evidence to retain |
| --- | --- | --- | --- |
| SMS code | The account has a verified mobile number and phone possession is relevant to the login policy | Message length and encoding can change segmentation; delivery and entry can still fail | Challenge ID, phone reference, region, send result, attempt count, terminal state |
| Email code | The seller already relies on a verified mailbox for account communication | Sender authentication and reputation affect acceptance; mailbox access may overlap with account recovery | Challenge ID, mailbox reference, sender configuration version, send result, terminal state |
| User-selected channel | Both destinations were verified before the login began | Attackers must not be allowed to register a new destination during the challenge | Selection event, eligible-channel snapshot, challenge ID, terminal state |

This is a field guide, not a transport contest. The useful unit of comparison is a completed, policy-valid seller login with a reviewable history. One sent message is only an intermediate event.

## Should seller login use SMS OTP or an email verification code?

Begin with the control objective. A new-order alert may contain little detail and direct the seller to sign in. The protected event is then access to the order, not receipt of the alert. Keep `order_id` out of the verification message and attach it only to the server-side authorization flow after the challenge succeeds.

Start there.

Pick SMS when the marketplace has already verified a seller's mobile number and its policy calls for a phone-based possession signal. SMS has a sharp implementation trap: encoding affects segment size. GSM-7 messages allow 160 characters in one segment, while UCS-2 messages allow 70; concatenated messages have lower per-segment limits because headers consume space. A curly quote or non-GSM character can therefore change how a message is segmented. Keep the template short, test its actual character set, and log a template version rather than message content.

Pick email when the verified mailbox is the intended trust channel and the team can operate authenticated sending. Google's sender guidance calls for SPF or DKIM for all senders to Gmail accounts, with additional requirements for bulk senders. It also recommends TLS and valid forward and reverse DNS. Those controls do not prove that a particular person read a code, but they are concrete configuration evidence worth versioning and monitoring.

Offer a choice only if both destinations were verified before the login attempt. Do not let a seller type a new phone number or mailbox into the challenge screen. That turns a factor check into destination enrollment.

The diagram in words is short: new-order alert, seller opens the marketplace, login creates a challenge, an adapter sends a code, the seller submits it, the verifier consumes it, and authorization opens the order. Every arrow emits an event keyed by one opaque challenge ID. The raw destination and code stay out of those events.

## Implement one challenge state machine

The clean boundary has three responsibilities: create a challenge, send through a selected channel, and verify exactly once. The adapter knows transport details. The challenge store knows expiry and attempt policy. The order service knows nothing about either SMS or email.

Here is a compact TypeScript implementation of that core. The in-memory stores make the state transitions runnable and visible; replace them with a datastore that supports atomic conditional updates before production use.

```ts
import { createHash, randomInt, randomUUID, timingSafeEqual } from "node:crypto";

type Channel = "sms" | "email";
type State = "pending" | "verified" | "expired" | "locked";

type Challenge = {
  id: string;
  sellerId: string;
  destinationRef: string;
  channel: Channel;
  codeHash: Buffer;
  expiresAt: number;
  attempts: number;
  state: State;
};

type AuditEvent = {
  challengeId: string;
  sellerId: string;
  channel: Channel;
  name: "created" | "send_accepted" | "verified" | "rejected" | "expired" | "locked";
  occurredAt: string;
  templateVersion: string;
};

interface Sender {
  send(destinationRef: string, code: string, challengeId: string): Promise<void>;
}

const challenges = new Map<string, Challenge>();
const audit: AuditEvent[] = [];
const senders: Record<Channel, Sender> = {
  sms: { send: async () => undefined },
  email: { send: async () => undefined },
};

const CODE_TTL_MS = 5 * 60 * 1000;
const MAX_ATTEMPTS = 5;
const TEMPLATE_VERSION = "seller-login-v1";

function digest(challengeId: string, code: string): Buffer {
  return createHash("sha256").update(`${challengeId}:${code}`).digest();
}

function record(challenge: Challenge, name: AuditEvent["name"]): void {
  audit.push({
    challengeId: challenge.id,
    sellerId: challenge.sellerId,
    channel: challenge.channel,
    name,
    occurredAt: new Date().toISOString(),
    templateVersion: TEMPLATE_VERSION,
  });
}

export async function createChallenge(input: {
  sellerId: string;
  destinationRef: string;
  channel: Channel;
}): Promise<{ challengeId: string; expiresAt: number }> {
  const code = randomInt(0, 1_000_000).toString().padStart(6, "0");
  const challenge: Challenge = {
    id: randomUUID(),
    sellerId: input.sellerId,
    destinationRef: input.destinationRef,
    channel: input.channel,
    codeHash: Buffer.alloc(0),
    expiresAt: Date.now() + CODE_TTL_MS,
    attempts: 0,
    state: "pending",
  };

  challenge.codeHash = digest(challenge.id, code);
  challenges.set(challenge.id, challenge);
  record(challenge, "created");
  await senders[challenge.channel].send(
    challenge.destinationRef,
    code,
    challenge.id,
  );
  record(challenge, "send_accepted");
  return { challengeId: challenge.id, expiresAt: challenge.expiresAt };
}

export function verifyChallenge(challengeId: string, candidate: string): boolean {
  const challenge = challenges.get(challengeId);
  if (!challenge || challenge.state !== "pending") return false;

  if (Date.now() >= challenge.expiresAt) {
    challenge.state = "expired";
    record(challenge, "expired");
    return false;
  }

  challenge.attempts += 1;
  const candidateHash = digest(challenge.id, candidate);
  const matches = timingSafeEqual(challenge.codeHash, candidateHash);

  if (matches) {
    challenge.state = "verified";
    record(challenge, "verified");
    return true;
  }

  if (challenge.attempts >= MAX_ATTEMPTS) {
    challenge.state = "locked";
    record(challenge, "locked");
  } else {
    record(challenge, "rejected");
  }
  return false;
}
```

There is an intentional trade-off here: the example fails closed if sending throws, but it has already persisted the challenge. That makes a retry possible without inventing a second security event. In a durable implementation, use an outbox or queue and an idempotency key derived from the challenge ID. A transport retry must not generate a new code, extend expiry, or reset the attempt counter.

One detail deserves extra attention. The example's `Map` cannot make concurrent verification atomic. A production store must change `pending` to `verified` with a conditional write so two simultaneous submissions cannot both consume the same challenge. Test that race explicitly.

## Make the audit trail answer questions

Logs should explain a single login. Metrics should explain the population. Alerts should explain when operators need to act. Mixing those jobs produces either high-cardinality metrics or logs too vague for an investigation.

For logs, retain the challenge ID, pseudonymous seller ID, channel, region, policy version, template version, event name, timestamp, and normalized failure class. Do not record the code, full phone number, full email address, or message body. Define retention and access around the evidence obligation; collecting sensitive data “just in case” weakens the design. For metrics, count created, send-accepted, verified, expired, locked, and recovery-started events. Measure creation-to-verification latency as a distribution. Segment by channel and broad region, but keep seller IDs and challenge IDs out of metric labels. Then the funnel is readable: created to accepted measures handoff, accepted to verified measures the user-visible path, and verified to order access checks the application boundary. This gives an operator a broad signal without turning a seller identifier into a metric dimension.

This distinction matters. A provider accepting a send request is not delivery, and delivery is not successful authentication. Name the event after what was observed, not what everyone hopes happened.

Keep it literal.

Alert on changes in ratios and latency against the service's own reviewed baseline. A fixed global threshold can hide a regional collapse or page on ordinary daily traffic changes. Pair each alert with a query that retrieves recent terminal states by channel, region, and template version. Fast diagnosis is part of the control.

## Test the failure paths before launch

The happy path proves very little. Exercise expiry at the boundary, five incorrect entries, a resend arriving after the original, concurrent correct submissions, sender timeout, delayed delivery, and recovery after a destination becomes unavailable. Use a fake clock and deterministic sender in unit tests; use reserved test destinations or controlled mailboxes in integration tests.

Deployment also needs a reversible shape. Ship the common state machine first, shadow its audit events, and compare event counts with the existing login flow. Enable one verified channel for a small policy cohort. Watch completion and recovery, then expand. If the channel is rolled back, preserve challenge records and policy versions so an auditor can still reconstruct what was enforced.

Do not use “fallback” to mean an automatic second message over another channel. Make recovery a separate, explicit flow with its own risk decision and audit events. Otherwise a failed SMS attempt can quietly become an email-based bypass, and the record will imply a stronger check than the user completed.

## Limits and decision rule

Neither transport turns a six-digit code into strong evidence by itself. The useful evidence comes from verified enrollment, bounded challenge state, single use, controlled recovery, and records that distinguish acceptance from completion. Transport choice remains a policy decision tied to the seller population and the protected action.

Choose the channel whose possession claim matches that policy and whose operational controls your team can continuously verify. If both qualify, run them through the same state machine and compare completed challenges, expiry, latency, and recovery burden. Keep price as a capacity input, not the security argument.

That is the boundary. The alert announces a new order; the authenticated marketplace reveals it.

## References

- Google, “Email sender guidelines”: https://support.google.com/a/answer/81126
- Twilio, “SMS character limits and segmentation”: https://www.twilio.com/docs/glossary/what-sms-character-limit
