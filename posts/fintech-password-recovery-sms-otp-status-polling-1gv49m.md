# Fintech Password Recovery: SMS OTP Status Polling With Observable Expiry

| Integration choice | Pick this when | Boundary to inspect |
| --- | --- | --- |
| Verification lifecycle interface | Your application needs send, resend, and verification through one contract | Whether expiry and attempt limits survive resends |
| SMS transport interface | Your application already owns challenge creation and validation | Your team must also own retry coordination and challenge state |
| Email recovery path | An established recovery policy already permits email | Sender requirements and recovery policy remain separate concerns |

TL;DR: A fintech password reset needs one fixed challenge deadline across repeated SMS OTP messages. Poll delivery status for feedback. Require a separate successful verification before authorizing the password change. If challenge management already exists, a transport adapter can be a smaller integration; otherwise, evaluate the effort of owning that state before choosing it.

The useful comparison is operational: can a developer explain a delayed message from the application's events without guessing whether the user has authenticated? The example below makes that distinction explicit. Its numeric settings are illustrative application policy, not measured delivery performance or standards requirements.

## How should an SMS OTP API report status during recovery?

Two questions matter: did the message progress, and did the person prove control of the recovery channel? Give them different fields. A delivery observation must never become permission to reset a password. The same distinction matters for SMS OTP during 2FA login, although account recovery and login should retain separate authorization policies.

Diagram in words: recovery request → challenge with fixed deadline → message attempt → delivery observation. A second branch runs from submitted code → verification decision → narrowly scoped password-reset authorization. Only the second branch can authorize a password change.

Keep them separate.

For this design, the challenge lifetime is 180 seconds; the resend cooldown is 30 seconds. Each resend creates a message attempt under the original challenge. It does not move the deadline. The trade-off is less convenience in exchange for a recovery window that operators can explain and tests can check. The exact lifetime belongs to the application's recovery policy; the important integration question is whether the selected interface can express it without additional state machinery.

A poll response can report an unresolved delivery observation while verification is already successful, or delivery can be reported while verification remains pending. Neither combination is contradictory. Record the observation time alongside the state so the UI and support tooling can distinguish an old observation from a current one.

## Pick a verification lifecycle when challenge ownership is new

Choose this interface shape when the application does not already have challenge storage, verification attempt controls, and atomic consumption. Ask for evidence of the behavior you need: a resend under the same deadline, a verification result independent of delivery, and a consumed challenge that cannot authorize another reset. Treat those as acceptance criteria for evaluation, not presumed features of every implementation.

The integration-effort trap is counting endpoints.

One send call looks small. The surrounding work can include concurrency control, secret handling, retries after ambiguous outcomes, and on-call evidence. A lifecycle interface reduces application work only to the extent that its documented contract actually covers those responsibilities. An undocumented edge case still becomes your team's code.

For an application serving the US and EU, evaluate the intended destinations separately. Keep region as an explicit test dimension. The CTIA resource concerns messaging interoperability and compliance; it is useful context for that review, not evidence that a particular integration satisfies every destination's requirements. Do not infer geographic suitability from a successful local test.

## Pick transport when recovery state already exists

Choose an SMS transport interface when your application already owns a durable, reviewed challenge lifecycle. Keep the transport adapter narrow: request a message attempt and obtain delivery observations. Let the recovery service decide expiry, verification, and consumption. That division makes the logs teachable because every state transition has an owner.

Email is a separate recovery-policy choice.

If the application already permits it, evaluate that path with its own sender setup and acceptance tests. The email sender guidelines in References address sending practices; they do not make email and SMS interchangeable authentication factors. Adding another channel adds a recovery branch to test, so it should earn its integration cost.

## Implement the observable polling boundary

Here is a focused TypeScript example for a Node.js service. It shapes an authenticated status read; it is not an SMS client or a complete recovery service. The injected lookup must enforce access to the challenge. The delivery lookup returns the latest stored observation, populated by a separate adapter. Keeping polling on stored state makes its work visible and avoids tying every browser refresh to another transport request.

```ts
type Delivery = "unknown" | "submitted" | "delivered" | "failed";
type Verification = "pending" | "verified" | "consumed";

type Challenge = {
  id: string;
  expiresAtMs: number;
  resendAfterMs: number;
  verification: Verification;
  latestAttemptId: string | null;
};

type Observation = {
  state: Delivery;
  observedAtMs: number;
};

interface StatusPorts {
  // Reject unauthorized access before returning challenge state.
  loadAuthorized(actorId: string, challengeId: string): Promise<Challenge>;
  latestObservation(attemptId: string): Promise<Observation | null>;
  countRead(labels: { result: "active" | "expired" }): void;
}

async function readRecoveryStatus(
  ports: StatusPorts,
  actorId: string,
  challengeId: string,
  now: () => number = Date.now,
) {
  const challenge = await ports.loadAuthorized(actorId, challengeId);
  const observation = challenge.latestAttemptId === null
    ? null
    : await ports.latestObservation(challenge.latestAttemptId);
  const nowMs = now();
  const expired = nowMs >= challenge.expiresAtMs;
  const canResend = !expired
    && challenge.verification === "pending"
    && nowMs >= challenge.resendAfterMs;

  ports.countRead({ result: expired ? "expired" : "active" });

  return {
    expiresAtMs: challenge.expiresAtMs,
    expired,
    verification: challenge.verification,
    delivery: observation?.state ?? "unknown",
    deliveryObservedAtMs: observation?.observedAtMs ?? null,
    canResend,
    resendWaitMs: canResend
      ? 0
      : !expired && challenge.verification === "pending"
        ? Math.max(0, challenge.resendAfterMs - nowMs)
        : null,
  };
}
```

Notice the restraint: this read cannot authorize anything. Even a verified value in its response is only a snapshot. The password-reset operation must check the current challenge state and atomically consume the relevant authorization. Likewise, the resend operation must recheck its deadline and cooldown when it writes. A button enabled by an earlier poll is not a concurrency control mechanism.

Before instrumentation, a single `success` event leaves support guessing. After instrumentation, use distinct events such as `message_attempt_requested`, `delivery_observed`, `challenge_verified`, and `challenge_consumed`. These are proposed application event names. Include an opaque challenge correlation identifier in restricted logs, but exclude the OTP, reset link, message body, and recipient address or phone number. Keep identifiers out of metric labels. The example counter uses two bounded result values so aggregate read volume remains easy to interpret.

Don't stop at the happy path.

Test the deadline boundary with an injected clock: at 179,999 milliseconds after creation, a pending challenge can still be active; at 180,000 it is expired under this policy. Then test a delivery observation arriving after expiry, an observation for an earlier message attempt, and two simultaneous resend requests. The last case belongs to the write-side transaction tests, because this status function cannot demonstrate atomic resend behavior. Also test that unauthorized polling reveals no challenge state. A useful concurrency fixture holds both requests at the same stored cooldown value, releases them together, and checks that only the allowed message attempt is committed. This is a proposed test, not a reported result. Carry that fixture into an adapter replacement: unchanged response fields alone cannot show that a write-side guarantee survived the deployment, and a dashboard cannot repair a missing atomic operation after the fact.

For deployment, agree on the status schema before changing adapters. Preserve an unknown delivery state rather than inventing a successful one when evidence is absent. If the observation store cannot be read, return an explicit read error through the application's normal error contract. Alert on sustained observation lag and unexpected growth in resend requests using thresholds derived from your own traffic. A useful dashboard puts expiry counts beside verification completions, while a restricted trace explains an individual attempt without exposing its secret.

## Limits to carry into the integration review

The main limitation is scope: this example specifies a read boundary. It does not implement OTP generation, verification attempt limits, message dispatch, distributed coordination, or the password-change transaction. Treat those as required owners in the integration review. The fixed deadline also means a late message may leave little usable time; the UI should explain expiry and offer the recovery policy's next step.

Delivery evidence is useful operational context. It is insufficient authorization. Choose the interface whose documented behavior lets the team preserve that boundary with the least additional state and the clearest tests.

## References

- Email sender guidelines: https://support.google.com/a/answer/81126
- Messaging interoperability and compliance: https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
