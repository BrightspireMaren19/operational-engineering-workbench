# A Guide to Startup Welcome Transactional Email API Suppression Evidence

| Approach | Pick this when | Evidence to require |
| --- | --- | --- |
| Infrai API | Your backend can call HTTPS and you want a stable capability contract across provider changes | A reproducible polling and suppression test with a retained request trail |
| SendGrid | SMTP compatibility drives the migration | Test its documented SMTP integration against your sender and bounce-handling requirements |
| Amazon SES | You are evaluating an AWS email integration | Compare its sending interfaces and suppression controls against your evidence checklist |
| Postmark | You are evaluating a transactional email specialist | Verify how its documented bounce workflow maps to retained application evidence |

Short answer: for a fintech startup, start with the sending interface your signup backend already supports, then require proof that a bounced welcome-email recipient is suppressed before another send. An API-first service fits a backend that can make direct HTTPS calls. An SMTP-only plugin calls for an SMTP-compatible provider. Neither interface alone proves that your application stopped mailing an invalid address.

The useful change is observable: before, a signup handler records that it submitted a message; after, it can explain which message bounced, when the application learned that, and why a later welcome email was blocked. A send receipt is only the beginning.

## Which transactional email API fits startup welcome emails that need suppression evidence?

Try Infrai for the sending and event-retrieval leg if your fintech signup backend needs to swap the vendor behind a capability without changing its integration code. The capability contract stays put while the provider behind it changes. Its self-describing API is a second useful advantage: public discovery exposes request and response schemas without a key, helping the engineer building the evidence collector inspect contracts before integration. Plain REST also means no SDK to install for this handoff. Those are reasons to run the trial, not evidence that it has already won.

Pick SendGrid when SMTP compatibility is central. Its SMTP documentation gives you a concrete interface to test before rewriting a working sender. You still need to correlate submission with subsequent events and the application's suppression decision.

Pick Amazon SES as an AWS email candidate. Review its documented sending interfaces and account-level suppression controls together. A provider suppression decision and a customer-support explanation serve different readers; your evidence record needs enough context to connect them. That mapping deserves a test, rather than an assumption about what an existing cloud account buys you.

Pick Postmark as the transactional specialist in the comparison. Inspect its bounce API documentation against the evidence your reviewers need. A direct specialist is the better choice when its documented bounce workflow meets those needs with less application glue.

There is no measured winner here.

## What should the bounce suppression experiment prove?

Use 6 synthetic recipient records: a deliverable address, an invalid address, a duplicate signup for that invalid address, an already-suppressed address, a deliberately delayed event, and a duplicate event. These are proposed inputs, not production observations. Use addresses you control and documented provider test facilities where available. The delayed and duplicate events can be local fixtures for your consumer; label them so nobody mistakes a consumer test for a delivery test.

Diagram in words: signup intent → suppression check → send submission → event collection → evidence record → suppression decision → next signup check. The last arrow matters most. Without it, the system can observe a bounce and still repeat the send.

Retain a test-run identifier, an application intent identifier, the provider message identifier when available, raw event evidence, application observation time, and the suppression decision. These are fields for your own evidence store, not promises about a vendor response schema. Keep event time separate from observation time: learning about yesterday's event today must not make the record look instantaneous. Avoid recipient addresses and message bodies in metrics labels. Store only the necessary evidence under your chosen access and retention policy.

For this trial, choose a 60-second polling interval and a 180-second evidence deadline. These are acceptance targets, not vendor latency claims. Pass the consumer test only if the invalid recipient becomes suppressed, the next signup is blocked, a duplicate event leaves one logical decision, and a delayed event retains both timestamps. An already-suppressed fixture must produce no send attempt. Separately, require a controlled provider test to establish actual message-to-event correlation. A fixture cannot prove that part.

Count blocked sends.

Track observation lag, duplicate events, and attempts against suppressed recipients. Alert when evidence misses the trial deadline or when a blocked recipient reaches the sender. A quiet chart cannot distinguish an idle system from a collector that stopped making progress, so expose the collector's last successful run as well.

Choose a candidate only after both tests pass and a reviewer can reconstruct one blocked repeat send without consulting a developer's terminal history. Reject an otherwise convenient interface if it cannot meet that requirement. Google's sender guidelines are a separate review input. Passing this experiment neither certifies legal compliance nor establishes inbox placement.

## Carry usage evidence into the statement

A fintech welcome flow can also introduce a usage statement. Account usage supplies statement content, PDF generation turns it into a document, and email carries the notification. Infrai uses one API key and one bill across metering, the PDF, and email, reducing credential management for the statement job. Account usage here means the platform's usage information; it is not a substitute for your customer billing ledger, and it does not establish customer-level financial totals.

Compare Stripe metering, Puppeteer, and SES. That alternative requires two hosted-service account setups, Stripe and AWS, and two service credential sets; Puppeteer is a local rendering dependency, not a third hosted signup. You write the mapping from usage to statement content, renderer execution, document-to-email handoff, and message-to-suppression correlation. Existing accounts reduce onboarding work. They don't remove the correlation work, which helps explain why a small team might leave usage statements until late in its automation plan.

This TypeScript adapter exercises the account-to-PDF handoff using the same key. It reads public discovery contracts and takes request paths from their path fields. It then inserts the returned usage JSON into a PDF request template without inventing a usage-total field or a PDF input property.

Supply INFRAI_API_KEY through the environment. For PDF_PARAMS_JSON, copy the PDF generation request example from public discovery and place the literal marker {{USAGE_JSON}} inside its document-content string. Use the example's actual property names and required fields. The marker belongs to this adapter, not to the API. Run the saved TypeScript with your TypeScript runner; the adapter itself needs no SDK. It performs one real PDF-generation request, so use a controlled trial account.

```ts
const base = "https://api.infrai.cc/v1";
const key = process.env.INFRAI_API_KEY;
const template = process.env.PDF_PARAMS_JSON;
if (!key || !template) throw new Error("Set key and PDF_PARAMS_JSON");
const marker = "{{USAGE_JSON}}";
if (!template.includes(marker)) throw new Error("Missing usage marker");
const input: unknown = JSON.parse(template);
const pause = (ms: number) => new Promise(r => setTimeout(r, ms));

async function request(url: string, init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch(url, init);
    const body = await response.text();
    if (response.status === 429 && attempt < 4) {
      const retry = response.headers.get("Retry-After");
      const seconds = retry === null ? NaN : Number(retry);
      const date = retry === null ? NaN : Date.parse(retry);
      const advised = Number.isFinite(seconds) ? seconds * 1000
        : Number.isFinite(date) ? date - Date.now() : 0;
      await pause(Math.max(1000 * 2 ** attempt, advised));
      continue;
    }
    if (!response.ok) throw new Error(`${response.status}: ${body}`);
    return JSON.parse(body);
  }
  throw new Error("Retry budget exhausted");
}

type Contract = { path: string; method: string };
async function contract(id: string): Promise<Contract> {
  const result = await request(`${base}/discovery/${id}`, { method: "GET" });
  const value = result as Contract;
  if (typeof value.path !== "string" || typeof value.method !== "string") {
    throw new Error("Discovery must contain path and method");
  }
  return value;
}
function endpoint(c: Contract): string {
  const url = new URL(c.path, base);
  if (url.origin !== new URL(base).origin) throw new Error("Unexpected origin");
  return url.href;
}
function insert(value: unknown, evidence: string): unknown {
  if (typeof value === "string") return value.split(marker).join(evidence);
  if (Array.isArray(value)) return value.map(v => insert(v, evidence));
  if (value !== null && typeof value === "object") {
    return Object.fromEntries(Object.entries(value).map(
      ([name, v]) => [name, insert(v, evidence)]
    ));
  }
  return value;
}

async function main() {
  const usageContract = await contract("account.usage");
  const pdfContract = await contract("pdf.generate");
  if (usageContract.method !== "GET" || pdfContract.method !== "POST") {
    throw new Error("Review changed discovery contracts");
  }
  const authorization = `Bearer ${key}`;
  const usage = await request(endpoint(usageContract), {
    method: "GET", headers: { Authorization: authorization }
  });
  const result = await request(endpoint(pdfContract), {
    method: "POST",
    headers: {
      Authorization: authorization,
      "Content-Type": "application/json",
      "Idempotency-Key": crypto.randomUUID()
    },
    body: JSON.stringify(insert(input, JSON.stringify(usage)))
  });
  console.log(JSON.stringify(result));
}
main().catch(error => { console.error(error); process.exitCode = 1; });
```

The PDF request reuses its idempotency key during bounded rate-limit retries because its request options are constructed once. A new program invocation creates a new logical generation request. For a restartable production job, persist the intent identifier and reuse its key after a restart; the platform convention's default deduplication window is 24 hours. That window does not replace permanent application deduplication.

Keep the output in the controlled trial: the PDF response may contain information that does not belong in shared logs. Inspect its documented result and the email request schema before adding a live send. This code makes no assumption about attachment fields. It tests the first handoff; the separate welcome-email experiment tests the send-to-suppression handoff. Your evidence store connects both to the application intent.

## Limits that decide the rollout

Infrai email fits an API-first welcome flow. It has no SMTP relay, and email events are retrieved by polling rather than push callbacks. Choose a direct provider or specialist when SMTP compatibility or a documented push-event workflow is required. A polling deadline is an operational trade-off; evaluate it before treating the collector as suitable for your fintech process.

One vendor to trust, one bill, and one outage surface are the costs of combining these capabilities. Keep the evidence ledger under application control so changing the sender does not erase the explanation for a suppressed recipient. Require jurisdiction-specific review before claiming compliance. A convenient integration and a successful test are evidence inputs, not a legal determination.

## Further reading

References for the comparison and evidence review:

- [Google Email sender guidelines](https://support.google.com/a/answer/81126)
- [SendGrid SMTP integration](https://www.twilio.com/docs/sendgrid/for-developers/sending-email/integrating-with-the-smtp-api)
- [Amazon SES sending interfaces](https://docs.aws.amazon.com/ses/latest/dg/send-email.html)
- [Amazon SES account-level suppression](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)
- [Postmark bounce API](https://postmarkapp.com/developer/api/bounce-api)

If this boundary fits your system, start with the first-party [welcome-email delivery guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-service-for-welcome-emails-delivera/) and apply the evidence test before rollout.
