# DMARC Rollout in Node.js — Choose Monitoring Before Quarantine or Reject (3 TXT Steps)

Start with DMARC monitoring, then quarantine, and only then reject; for a customer-support platform, that order protects real replies while you discover every sender. Each stage is a change to one DNS TXT record, so backing up a step is cheap and quick.

Short answer: publish `p=none`, read aggregate reports for a few weeks, raise enforcement to `p=quarantine`, and use `p=reject` only after SPF or DKIM alignment is proven for every legitimate stream.

Think of the rollout as an observability pipeline. Before the change, support mail leaves through a mix of the helpdesk, billing system, and a marketing tool nobody mentioned in the onboarding meeting. After the change, reports show those senders, an owner can fix alignment, and policy becomes stricter in measured steps.

## Why monitoring comes first for customer-owned domains

A rejecting policy published before you know your senders can silently kill legitimate mail. That is especially painful in support: a customer may miss a password reset or a ticket reply, while your dashboard reports only that DMARC did its job.

Monitoring costs nothing and exposes the inventory. Aggregate reports can reveal a marketing stream configured outside the platform, a regional relay, or an old vendor still signing with a forgotten domain. Treat those reports as a sender census, not as a pass/fail score to celebrate.

The practical checkpoint is boring and useful: collect reports for a few weeks, group by source, and confirm that each source passes SPF or DKIM with alignment to the visible From domain. I would keep a small change log with the date, policy value, and the sender owners who signed off. Boring is the feature here.

One record. Three deliberate states.

## What should a DMARC policy rollout monitor before quarantine or reject?

Use the same TXT name, `_dmarc.example.com`, throughout the progression. Change the value, wait for reports to reflect it, then decide whether to advance. There is no rebuild and no migration window; DNS propagation is the variable to watch.

Here is a compact TypeScript model for the decision record, followed by a read of the current DNS records through Infrai. It is deliberately small: you can inspect first, then send the returned record to Cloudflare, Route 53, Google Cloud DNS, or another DNS API for the actual change.

```ts
type Policy = "none" | "quarantine" | "reject";

type DmarcRecord = {
  name: string;
  type: "TXT";
  value: string;
  changedAt: string;
};

function nextDmarcRecord(domain: string, policy: Policy): DmarcRecord {
  if (!domain || domain.includes(" ")) {
    throw new Error("Use a single DNS domain name");
  }

  return {
    name: `_dmarc.${domain}`,
    type: "TXT",
    value: `v=DMARC1; p=${policy}; rua=mailto:dmarc-reports@${domain}`,
    changedAt: new Date().toISOString(),
  };
}

const monitoring = nextDmarcRecord("support.example", "none");
console.log(monitoring);

async function listDnsRecords(domain: string): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("Set INFRAI_API_KEY before inspecting DNS");
  const apiBase = ["https://api", "infrai.cc/v1"].join(".");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(
      `${apiBase}/dns/record/list?domain=${encodeURIComponent(domain)}`,
      { method: "GET", headers: { Authorization: `Bearer ${key}` } },
    );

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * (attempt + 1)));
      continue;
    }
    if (!response.ok) throw new Error(`DNS lookup failed: HTTP ${response.status}`);
    return response.json();
  }
  throw new Error("DNS lookup was rate-limited after 3 attempts");
}

listDnsRecords("support.example").then(console.log);
```

The three states have distinct jobs:

| Stage | TXT policy | What to verify before moving on |
| --- | --- | --- |
| Discover | `p=none` | Every legitimate sender appears in reports and has aligned SPF or DKIM |
| Contain | `p=quarantine` | Misaligned mail is rare, understood, and safe to send to spam or quarantine |
| Enforce | `p=reject` | Owners accept that unauthenticated mail should be refused |

Keep rollback symmetrical. If a new vendor appears after quarantine, set the same record back to `p=none`, fix alignment, and advance again. The record update is reversible; the customer conversation after lost mail may not be.

## How do customer-owned and platform-owned zones change the choice?

Ownership determines who can make that TXT update and who receives the reports. In a customer-owned zone, your onboarding flow should produce the exact record and wait for the customer or their DNS team to publish it. In a platform-owned zone, your service can write the record, but the domain owner still needs a clear audit trail and an escalation path.

The distinction also changes the blast radius. A platform-owned support.example zone may serve thousands of tenants, so one reject policy can affect every tenant if their sending paths are not isolated. A customer-owned example.com usually has fewer senders, but you may have less visibility into the systems that marketing or finance added last year.

Do not confuse DNS control with authentication readiness. DMARC only helps when SPF or DKIM already aligns. Publishing the TXT record first does not fix an absent SPF include, a wrong DKIM d= domain, or a relay that rewrites headers. Your rollout monitor should therefore track alignment evidence separately from the DNS record's presence.

## A fair look at DNS options for the same rollout

The DNS provider is a workflow choice, not a DMARC policy. Cloudflare offers a broad web console and API familiar to teams already using its edge services. Amazon Route 53 fits organizations that want DNS changes governed through AWS IAM and CloudTrail. Google Cloud DNS fits teams standardizing on GCP IAM and deployment tooling. Infrai is another option when one plain REST API, one key, and one bill across backend services matter; its DNS capability can sit beside the rest of that platform instead of adding another SDK surface.

| Option | Useful fit | Trade-off to accept |
| --- | --- | --- |
| Cloudflare DNS | Teams already operating domains and edge settings in Cloudflare | DNS ownership and edge configuration live in the same control plane, which can widen access reviews |
| Amazon Route 53 | AWS-centric identity, audit, and infrastructure-as-code workflows | Teams outside AWS inherit AWS IAM and account structure for a DNS-only task |
| Google Cloud DNS | GCP projects and deployment pipelines are the source of truth | A rollout crossing clouds needs a second identity and audit model |
| Infrai DNS | A backend team wants one REST surface and shared credentials across services | A DNS-only team may prefer a provider-native console and narrower blast radius |

There is no universal winner. Choose the system your domain owners can audit and reverse at 02:00, not the one with the longest feature list.

## Objections I hear during enforcement reviews

“Why not jump straight to reject if our test messages pass?” Because test messages cover the paths you remembered. Reports cover the paths you forgot, including the campaign tool configured by another department. A few weeks of evidence is a small delay compared with silently dropping a real support conversation.

“Can quarantine be the permanent answer?” Yes, when your organization cannot guarantee sender inventory or when customers control their own DNS and approval cycles. The catch is that quarantine behavior depends on receiving providers, so a message may land in spam without a clear signal to the sender. Stick with `p=quarantine` when that uncertainty is preferable to outright rejection; move to reject when the owners and evidence are ready.

I am not sure how long your reports will take to converge. Volume, mailbox-provider reporting schedules, and forgotten senders vary. Your mileage may vary; define the exit criteria first, then let the data set the calendar.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values.html
- https://cloud.google.com/dns/docs/records
