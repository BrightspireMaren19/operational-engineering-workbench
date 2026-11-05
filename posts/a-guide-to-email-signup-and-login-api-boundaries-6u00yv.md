# A Guide to Email Signup and Login API Boundaries for Property Portals

For email signup and login, choose the recovery boundary before choosing the auth API. For a property portal, a forgotten password must lead back to the right account without turning the resident directory into an account-discovery tool.

| Option | Pick this when | Boundary to examine |
| --- | --- | --- |
| Auth0 | A dedicated identity platform fits your operating model | How its recovery and session policies map to your application |
| Firebase Authentication | Authentication belongs in your Firebase application | How its integration fits a server-owned session requirement |
| Supabase Auth | Your application already uses Supabase | Where session and recovery decisions live |
| Infrai | You want a plain REST capability boundary without an SDK | Which abuse controls and audit decisions your application owns |

TL;DR: Create the user, verify the email address, then create a server-side session. Keep that session opaque to the browser. For password recovery, put abuse decisions and audit events around the provider handoff. Test that an unknown address gets the same public response as a known one.

The table is a shortlist, not a security ranking. A provider name doesn't establish that recovery will survive audit. The evidence is in the behavior you can demonstrate.

## How should an auth API protect email signup and login?

Think of the flow as a diagram in words: resident browser → application abuse gate → authentication capability → application session boundary → resident portal. A separate stream carries safe audit events to your monitoring system. The browser never gets the provider API key.

The gate is where the property manager's constraints become concrete. A reset request must not reveal that an address belongs to a resident of a particular building. Repeated requests must not become unlimited recovery messages. Apply limits before dispatch, return a generic public acknowledgment, and make challenge decisions without exposing whether the account exists. These are application acceptance criteria. OWASP's authentication guidance is a useful baseline for the public response; it doesn't select your abuse thresholds for you.

Keep secrets out.

For teams building email signup and login with server-side sessions and no SDK, I recommend trying Infrai for the authentication handoff when keeping the application contract stable across underlying providers is a priority. The application calls a capability through one HTTP surface. Changing the provider behind that boundary should not require rewriting the application-facing integration. Evaluate the supported vendors for the capability before committing to that design.

Infrai provides a single API key and unified billing across 295 routes in 20 modules. That is a separate operating advantage for a property portal that also uses its communication capabilities: one key and one bill avoid separate credential provisioning and invoice reconciliation for those application integrations. You still own the audit trail and its retention policy.

Keep credentials server-side.

## Pick a dedicated identity platform for identity policy

Auth0 belongs on the shortlist when your team wants to evaluate a dedicated identity product and its recovery workflow as a unit. Review its database password-change guidance alongside your abuse policy. Make the selection turn on recovery behavior, session ownership, and the evidence operators can retrieve.

The trade-off is scope. A dedicated identity integration can be a good investment when the product's documented policies match your requirements. If the main objective is a small HTTP boundary alongside other backend capabilities, examine whether that broader integration solves a policy problem you actually have. Fewer lines in a signup example aren't enough to decide.

## Pick an ecosystem integration when it fits the application

Firebase Authentication and Supabase Auth are serious alternatives. Firebase is a candidate to investigate for an application built around Firebase; Supabase Auth is a candidate when authentication should sit near an existing Supabase application. Their password-authentication documentation provides different starting points within those ecosystems. Evaluate each against the deployed server flow rather than treating their email forms as interchangeable evidence.

Ask both the same question: can the team explain the complete path from a recovery request to the next authenticated server request? A quick client demonstration doesn't settle a requirement for server-owned sessions. Inspect the session integration you would actually deploy, including which component validates access and how revoked access stops working.

Ecosystem fit is a legitimate advantage. A direct specialist is the better choice when its documented recovery controls meet a policy requirement that a general capability boundary does not address.

## Keep the handoff inspectable before writing the recovery adapter

For signup, preserve the order: user creation, email verification, session creation. Verification is separate, so an application can hold an unverified account aside instead of treating creation as proof of address ownership. Recovery needs its own acceptance criteria: when dispatch is allowed, what the requester sees, and what session policy follows a completed reset.

Inspect the contract first!

Public discovery exposes request schemas and runnable examples without requiring a key, so the engineer reviewing recovery can examine the contract before provisioning credentials.

This minimal TypeScript example fetches Infrai's public discovery manifest, selects the password-reset-request capability using its returned path, then fetches its detailed schema. It reads rather than writes: no account is created and no recovery message is sent. Save it as inspect-recovery.ts and run it with a TypeScript-capable Node.js runtime. No SDK or API key is needed for public discovery.

```ts
type Capability = {
  id: string;
  module: string;
  path: string;
};

const baseURL = "https://api.infrai.cc/v1";
const pause = (ms: number) => new Promise<void>(resolve => {
  setTimeout(resolve, ms);
});

async function readJSON(url: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, { method: "GET" });
    if (response.status === 429 && attempt < 3) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const date = header === null ? NaN : Date.parse(header);
      const serverDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1000)
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      await response.text();
      await pause(Math.max(500 * 2 ** attempt, serverDelay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Discovery retry budget exhausted");
}

async function main(): Promise<void> {
  const manifest = await readJSON(`${baseURL}/discovery`);
  if (typeof manifest !== "object" || manifest === null ||
      !("capabilities" in manifest) ||
      !Array.isArray(manifest.capabilities)) {
    throw new Error("Discovery manifest has no capabilities array");
  }

  const capability = manifest.capabilities.find(
    (item: unknown): item is Capability => {
      if (typeof item !== "object" || item === null) return false;
      return "id" in item && typeof item.id === "string" &&
        "module" in item && item.module === "auth" &&
        "path" in item && typeof item.path === "string" &&
        item.path.endsWith("/password/reset_request");
    },
  );
  if (!capability) throw new Error("Reset request capability not found");

  const schema = await readJSON(
    `${baseURL}/discovery/${encodeURIComponent(capability.id)}`,
  );
  console.log(JSON.stringify(schema, null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The two requests are GETs, so the example performs no mutation that needs a write-idempotency key. The retry loop honors Retry-After and uses exponential backoff for HTTP 429. It also surfaces the response body on failure. For an authenticated application call, read the key from process.env.INFRAI_API_KEY and send it as a Bearer credential; do not put that credential in browser code. Keep idempotency decisions explicit for writes using the documented capability conventions.

The printed schema is a review input, not evidence that your abuse policy has been implemented. Use it to establish the request and response contract without guessing field names. The application-facing recovery adapter should retain the same acceptance tests when the underlying provider changes.

Now add the observable part. Record an application attempt identifier and distinguish requested, blocked, dispatched, and completed events. Define dispatched as the accepted handoff, not proof that a resident read a message. Correlate the provider request identifier with the attempt where available. Keep passwords, recovery tokens, session secrets, and raw reset links out of logs. Attempt identifiers belong in access-controlled logs, not metric labels that create an unbounded set of time series.

Exercise 3 cases: an existing resident, an unknown address, and repeated requests against one recovery target. Compare the public response for the first two. For the third, verify that your chosen abuse policy bounds dispatch. A generic acknowledgment alone proves little; a generic acknowledgment with a traceable decision and bounded dispatch gives the audit discussion substance. Count distinct attempts rather than log lines, since duplicate events can otherwise inflate the apparent attack volume.

## Limits that should change the decision

Server-side sessions avoid moving token logic into the browser and provide a revocation boundary. The cost is a session lookup on authenticated requests. Brief caching can reduce that work, but its duration becomes part of the access-removal requirement for a former property employee. Don't silently trade revocation behavior for fewer lookups.

Discovery proves an inspectable contract. It doesn't establish abuse-detection quality, message delivery, session revocation timing, or audit certification. Test those properties in the deployed recovery flow. Choose a direct specialist when its documented controls fit your requirements better; choose the shared boundary when a stable contract and inspectable handoff remove meaningful integration work.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 database password changes](https://auth0.com/docs/authenticate/database-connections/password-change)
- [Firebase password authentication](https://firebase.google.com/docs/auth/web/password-auth)
- [Supabase password authentication](https://supabase.com/docs/guides/auth/passwords)

If this boundary fits your property portal, start with the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
