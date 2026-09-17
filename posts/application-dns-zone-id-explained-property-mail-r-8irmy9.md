# Application DNS Zone ID Explained — Property Mail Record Lookup Without Drift

Store the DNS zone ID as a replaceable locator on the zone inventory row, never as the application's primary key.

Short answer: a property management application should give each managed domain its own internal ID, keep the current DNS zone ID beside it, and resolve record operations through that binding. This lets SPF, DKIM, and DMARC publication survive a provider move, an account change, or a retry without confusing an external identifier for durable business identity.

The distinction matters because the application owns intent while DNS exposes published state. A leasing team may intend to send mail for `notices.oak.example`; a DNS account may organize that name under `oak.example`; and a later portfolio transfer may change the zone locator without changing the property's identity. One overloaded primary key makes those changes look equivalent. They aren't.

## Should a Node.js application store the DNS zone ID for record operations?

Yes, when record writes require that ID, but it belongs in a binding rather than in the primary key. The primary key should identify the application's managed domain. The zone ID answers a narrower question: where should the current adapter perform a lookup or mutation?

Start with a small data flow. A property is linked to a managed mail domain. That domain holds the desired SPF, DKIM, and DMARC records. A zone binding says which external zone currently publishes them. A reconciliation run loads the domain by its local ID, reads the binding, compares desired records with observed records, and applies only the planned changes. The provider-shaped value never escapes that boundary.

Here is a runnable TypeScript model of the decision. The in-memory adapter is deliberately boring; the useful part is that `managedDomainId` remains stable while `externalZoneId` can be replaced.

```ts
type RecordKind = "TXT";

type DesiredRecord = {
  fqdn: string;
  kind: RecordKind;
  value: string;
};

type ManagedDomain = {
  id: string;
  propertyId: string;
  apex: string;
  desiredRecords: DesiredRecord[];
};

type ZoneBinding = {
  managedDomainId: string;
  providerKey: string;
  externalZoneId: string;
  boundApex: string;
  revision: number;
};

type PublishedRecord = DesiredRecord & { externalRecordId: string };

interface DnsAdapter {
  listRecords(externalZoneId: string): Promise<PublishedRecord[]>;
  putRecord(externalZoneId: string, record: DesiredRecord): Promise<void>;
}

class MemoryDnsAdapter implements DnsAdapter {
  private readonly records = new Map<string, PublishedRecord[]>();

  async listRecords(externalZoneId: string): Promise<PublishedRecord[]> {
    return this.records.get(externalZoneId) ?? [];
  }

  async putRecord(
    externalZoneId: string,
    record: DesiredRecord,
  ): Promise<void> {
    const current = await this.listRecords(externalZoneId);
    const unchanged = current.some(
      (item) =>
        item.fqdn === record.fqdn &&
        item.kind === record.kind &&
        item.value === record.value,
    );

    if (unchanged) return;

    const next: PublishedRecord = {
      ...record,
      externalRecordId: `${externalZoneId}:${current.length + 1}`,
    };
    this.records.set(externalZoneId, [...current, next]);
  }
}

async function reconcileMailRecords(
  domain: ManagedDomain,
  binding: ZoneBinding,
  dns: DnsAdapter,
): Promise<number> {
  if (binding.managedDomainId !== domain.id) {
    throw new Error("Zone binding does not belong to this managed domain");
  }

  if (binding.boundApex !== domain.apex) {
    throw new Error("Bound apex does not match the managed domain");
  }

  const published = await dns.listRecords(binding.externalZoneId);
  let writes = 0;

  for (const desired of domain.desiredRecords) {
    const exists = published.some(
      (item) =>
        item.fqdn === desired.fqdn &&
        item.kind === desired.kind &&
        item.value === desired.value,
    );

    if (!exists) {
      await dns.putRecord(binding.externalZoneId, desired);
      writes += 1;
    }
  }

  return writes;
}

const domain: ManagedDomain = {
  id: "domain_2048",
  propertyId: "property_oak_17",
  apex: "oak.example",
  desiredRecords: [
    {
      fqdn: "_dmarc.oak.example",
      kind: "TXT",
      value: "v=DMARC1; p=none",
    },
  ],
};

const binding: ZoneBinding = {
  managedDomainId: domain.id,
  providerKey: "primary-dns",
  externalZoneId: "zone_781",
  boundApex: domain.apex,
  revision: 3,
};

const writes = await reconcileMailRecords(
  domain,
  binding,
  new MemoryDnsAdapter(),
);

console.log({ managedDomainId: domain.id, writes });
```

The example uses one DMARC TXT value to keep the mechanism visible, not to prescribe a rollout policy. RFC 7489 defines DMARC as a layer over SPF and DKIM authentication that adds identifier alignment, published handling policy, and reporting. Its policy choices include `none`, `quarantine`, and `reject`. Choosing among them is a mail-security rollout decision; it should not be smuggled into the persistence model.

There is another small but important choice in the code: writes are selected by the record tuple, while `externalRecordId` stays adapter-owned. A cached record ID may speed a later update, but it should be treated as another replaceable locator. The desired record remains intelligible even after every provider-specific identifier changes.

## Separate durable identity from the publication locator

Using `externalZoneId` as the row's primary key looks economical. It removes a column and one lookup. For a solo builder watching latency and database calls, that can feel attractive.

The catch is temporal coupling. If a property portfolio changes DNS accounts, the same application entity receives a new external value. A primary-key rewrite can then ripple into desired records, audit events, retry jobs, and foreign keys. Keeping an internal `managedDomainId` means the application can append or replace a binding while the domain's history stays attached to one identity. The extra indexed lookup is usually the cleaner cost to accept because it buys an explicit place to validate the relationship.

Names alone are weak locators for a different reason. The name being authenticated may be a subdomain, while the record operation belongs to the containing zone. Asking an adapter to rediscover that zone before every mutation hides a decision inside an I/O call. Store the resolved binding after onboarding, record the apex it was checked against, and revalidate it when the binding changes.

Keep the scope tight.

The binding needs the local domain ID, an adapter or account key, the external zone ID, the verified zone apex, and a revision or equivalent concurrency token. It does not need to become a dump of every response field. Provider response snapshots age quickly and make ownership less clear; retain only identifiers required for operations plus the evidence your audit policy actually calls for.

## How can a zone inventory reveal published record drift?

Treat publication as reconciliation, not as a sequence of button clicks. Desired state belongs in application storage. Observed state comes from DNS. A plan compares canonical tuples such as name, type, and value, then produces a small set of creates, replacements, or removals. Execution records which binding revision it used. Verification reads again and evaluates the result against the same desired set.

That separation prevents a nasty class of retry mistakes. Imagine an ownership transfer arrives between planning and execution: revision `3` points at the former account, while revision `4` points at the new one. A queued operation carrying only `zone_781` has no way to prove that it still represents the domain's current location. A queued operation carrying `managedDomainId` plus the expected binding revision can stop before writing when the revision changed. Drift also has more than one shape: a desired record can be absent, an unexpected value can occupy the same name and type, or a formerly desired value can remain published after intent changed. For mail authentication, those states deserve review because DMARC evaluation depends on authenticated identifiers and their alignment. The reconciler should report the difference before it edits anything, especially when a stronger DMARC handling policy is involved. I wouldn't infer control merely because a lookup returned a plausible zone, either. Bind onboarding to an explicit property and domain request, validate the returned apex, and require a fresh revision for later changes. I'm not sure one verification interval fits every property portfolio; mail volume, change frequency, and the organization's tolerance for delayed drift detection determine that schedule. What stays constant is the comparison: declared intent versus currently published records.

No guesswork.

Observability should follow those two sides. Log the stable managed-domain ID, binding revision, plan hash, operation count, and verification outcome. Avoid placing secret material or full message data in those events. A useful alert says that a domain at revision `4` still differs from its desired record set after verification; an unhelpful alert exposes only an external zone string that an operator must manually trace back to a building.

## Where is this binding model the wrong fit?

It is not suitable when the application never mutates DNS and only displays public lookup results. In that read-only case, storing an external zone ID adds lifecycle work without enabling a real operation; query by domain name and cache according to the application's freshness needs instead.

Stick with runtime discovery when zone ownership changes constantly and the DNS integration provides a trustworthy, inexpensive discovery step that you validate before every write. Even there, keep a local domain primary key. The choice is whether to persist the locator, not whether an outside system should define application identity.

The model also does not remove the need for human approval. Automated reconciliation is a poor match for organizations where DNS is centrally governed and every mail-policy change must pass a separate change window. Generate a plan and hand it to that process. The same desired-versus-observed representation still helps, but the executor belongs elsewhere.

These are real costs: another table or document, uniqueness rules, migration handling, revision checks, and an extra lookup on the write path. If there is one domain in one permanent account and no retry queue, a compact configuration entry may be enough. Don't build portfolio machinery for a single static zone.

## An operational checklist that survives handoffs

At onboarding, create the managed domain first, discover or receive the publication zone, and save the binding only after the apex and application relationship agree. Make the binding unique for the relevant account and zone pair, and increment its revision whenever its locator or ownership context changes. Keep desired SPF, DKIM, and DMARC records attached to the stable domain identity rather than to the binding.

Before each deployment, compute and review a plan from desired and observed records. At execution, require the revision used during planning, make identical retries harmless, and record the local ID in every event. After execution, read the published state again. During a property sale or DNS-account transfer, install the new binding as a deliberate state transition; do not rewrite the domain's identity or silently reuse work planned against the old revision.

Finally, test the lifecycle rather than one successful create. Cover an unchanged record, a missing record, a changed value, a stale binding revision, a subdomain mapped to its containing zone, and a provider move that preserves the local domain ID. Those tests answer the original storage question better than a schema diagram does: if the system can change publication location without losing intent or misdirecting a retry, the boundary is doing its job.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
