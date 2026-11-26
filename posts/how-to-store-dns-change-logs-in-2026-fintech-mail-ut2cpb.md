# How to Store DNS Change Logs in 2026 (Fintech Mail Custody)

| Zone custody | Pick this when | Where the change evidence belongs |
| --- | --- | --- |
| Customer-owned | The fintech company controls its mail domain and approves DNS changes. | A customer-controlled audit destination, with platform events correlated into it. |
| Platform-owned | The platform operates the authoritative zone and executes approved changes. | The platform's operational audit trail, with exportable evidence available to the customer. |

Customer-owned zones preserve direct control of company mail; platform-owned zones move more of the execution responsibility into the platform. **TL;DR: Keep the record of who changed what in durable, searchable audit logs under an explicit custody agreement.** Capture the actor at the authenticated write boundary, then correlate the approved MX change, execution result, and later DNS observations. For a fintech company's mail cutover, choose custody by who must produce that evidence during review.

Start with custody.

An MX answer can show a mail destination. It cannot tell a reviewer which employee approved that destination or which service account submitted the change. DNS record data and a change audit trail serve different purposes. This guide treats the latter as the deliverable: a traceable account of one authorized company-mail change, including the cases where its outcome remains uncertain.

## Where should DNS change auditing logs live?

Use evidence custody as the decision axis. Before choosing who hosts the zone, identify the team that must answer this question: which authenticated principal changed the MX record, what did they request, and what evidence supports the reported outcome? Make the answer independent of one engineer's console access. A reviewer should be able to retrieve the same event after a handoff.

Diagram in words: approver → authenticated executor → authoritative DNS service. In parallel, the executor sends correlated events → audit storage → reviewer. A DNS observer contributes a separate observation to that same trail. Keeping those paths distinct prevents a successful lookup from becoming an invented account of who performed a write.

RFC 1035 defines the MX record's preference and exchange fields. RFC 2136 defines a DNS UPDATE message and its response codes. Neither record content nor an update response supplies a complete organizational approval history. Design that history into the execution workflow. RFC 5424's structured logging model offers useful vocabulary for carrying identifiers, but the choice of a log format alone does not establish retention or access controls.

For this workflow, record an event identifier, a stable change identifier, the authenticated actor, the executing principal, the zone, the record owner, and the before-and-requested-after MX sets. Include an approval reference and timestamps. Keep resolver observations separate from acknowledged writes. Those distinctions matter when the approved plan exists but the executor loses its response: the audit trail must preserve uncertainty instead of labeling the change successful.

## Pick customer-owned zones when mail authority stays with the company

Choose customer custody when the company needs to retain direct authority over its mail domain and its own team approves or executes changes. Put the audit destination under that team's control. A platform can contribute a request identifier and execution events, but the customer should define how those events join its approval record.

This choice adds coordination. The company has to connect its DNS writer, approval process, and evidence store; a request made through one interface cannot account for a manual edit through another. Inventory every permitted write path. Either capture its events or remove it from the approved workflow. Treat an uninstrumented administrative edit as an evidence coverage gap, even if the resulting MX answer looks correct.

Set the handoff explicitly: who supplies the before-state snapshot, who authenticates the actor, and who can retrieve the resulting trail? Avoid logging credentials or mailbox contents. The useful object here is a record change, with enough identity and authorization context to explain it.

## Pick platform-owned zones when execution belongs to the platform

Choose platform custody when the platform operates the zone and performs the DNS writes. Require customer identity to remain linked to the platform's executing service account. Otherwise, every edit may be attributable only to the same machine principal, leaving the reviewer unable to connect an operation to its requester.

The trade-off is evidence access. Specify export, retention, deletion authority, and retrieval after the customer relationship ends before relying on this arrangement. A searchable operations screen is useful for investigation; the custody agreement must also cover what evidence survives account and team changes.

Retention is a policy decision.

Do not invent a universal compliance period for fintech mail changes. Use the company's applicable obligations and evidence policy to set it, then test retrieval against that period. Include storage and operational effort in the decision: exporting events, controlling access, and investigating missing correlations all create work regardless of who owns the zone.

## Implement the evidence path before changing MX

Go deep on the customer-owned option: the company approves the MX change and retains the evidence. The following two steps build a small local demonstration of event capture and independent observation. Run the TypeScript with a runtime that supports TypeScript and Node built-ins. The reserved example names are placeholders. These snippets do not submit a DNS change, authenticate a user, or create a protected compliance archive.

1. Persist the approved intent before handing it to an executor.

This runnable example writes one JSON event per line and requests synchronization before returning. Its identities and approval are explicit fixtures; in a deployed writer, obtain identity from the authenticated request context and verify the approval before constructing the event. Capture the previous RRset from the actual read path. Do not trust a caller-supplied actor string as proof of identity.

```ts
import { open } from "node:fs/promises";
import { randomUUID } from "node:crypto";

async function main() {
  const event = {
    eventId: randomUUID(),
    changeId: "mail-cutover-2026-042",
    occurredAt: new Date().toISOString(),
    kind: "approved-intent",
    actor: "fixture:mail-approver",
    executor: "fixture:dns-writer",
    approvalRef: "fixture:approval-042",
    zone: "treasury.example",
    owner: "treasury.example",
    type: "MX",
    before: [{ preference: 10, exchange: "old-mx.treasury.example" }],
    requestedAfter: [
      { preference: 10, exchange: "new-mx.treasury.example" }
    ]
  };
  const file = await open("mail-audit.jsonl", "a", 0o600);
  try {
    await file.writeFile(JSON.stringify(event) + "\n");
    await file.sync();
  } finally {
    await file.close();
  }
  console.log(JSON.stringify({ recorded: event.eventId, kind: event.kind }));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The decision rule is simple: if intent persistence fails, stop before submission. A successful run proves only that this demonstration recorded an intent. It says nothing about a DNS write. In production, emit a separate execution event from the actual response, preserving the same change identifier and the writer's native request identifier when one exists. Record rejection as rejection; classify a lost response as unknown until reconciliation supplies evidence.

That last distinction is easy to miss. A retry after an ambiguous result can make the final DNS state look reasonable while leaving two attempted operations in the history. Keep both attempts. Preserve the change identifier across them and give each event its own identifier so investigation can follow the sequence without silently collapsing it. Consider a test in which the executor submits the approved replacement, then loses its connection before receiving a response. The approved intent is still available, but the executor cannot establish acceptance from that fact alone. A later resolver observation might match the requested destination. Keep that observation attached to the change without rewriting the earlier uncertainty: another permitted writer could have submitted the same values. Reconcile against the authoritative service's available execution evidence before claiming who completed the operation. If that evidence cannot resolve the attempt, leave its outcome unknown and record the decision about retrying. This is the trade-off for honest evidence: the reviewer sometimes sees an unresolved attempt instead of a tidy success label, but can distinguish what the system requested, what it observed, and what it could actually attribute.

2. Add a DNS observation without upgrading it to execution proof.

Run this second snippet after the first. It reads the latest recorded intent, queries MX through the runtime's resolver path, and appends an observation. RFC 2606 reserves `.example` for examples, so the fixture does not demonstrate a working company-mail destination. Substitute a domain you control for an actual observation exercise.

```ts
import { open, readFile } from "node:fs/promises";
import { resolveMx } from "node:dns/promises";
import { randomUUID } from "node:crypto";

type Mx = { preference: number; exchange: string };
type Intent = {
  kind: string;
  changeId: string;
  owner: string;
  requestedAfter: Mx[];
};

function key(records: Mx[]): string {
  return records.map(({ preference, exchange }) =>
    `${preference}:${exchange.toLowerCase().replace(/\.$/, "")}`
  ).sort().join("|");
}

async function main() {
  const lines = (await readFile("mail-audit.jsonl", "utf8"))
    .trim().split("\n");
  const intent = lines.map((line) => JSON.parse(line) as Intent)
    .reverse().find((event) => event.kind === "approved-intent");
  if (!intent) throw new Error("No approved intent recorded");

  let outcome: object;
  try {
    const answer = await resolveMx(intent.owner);
    const observed = answer.map(({ priority, exchange }) => ({
      preference: priority, exchange
    }));
    outcome = {
      status: key(observed) === key(intent.requestedAfter)
        ? "match" : "mismatch",
      observed
    };
  } catch (error: unknown) {
    outcome = {
      status: "lookup-error",
      message: error instanceof Error ? error.message : String(error)
    };
  }

  const event = {
    eventId: randomUUID(),
    changeId: intent.changeId,
    occurredAt: new Date().toISOString(),
    kind: "resolver-observation",
    owner: intent.owner,
    ...outcome
  };
  const file = await open("mail-audit.jsonl", "a", 0o600);
  try {
    await file.writeFile(JSON.stringify(event) + "\n");
    await file.sync();
  } finally {
    await file.close();
  }
  console.log(JSON.stringify(event));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

A match means this lookup returned the requested MX set.

It doesn't identify the writer.

A mismatch or lookup error calls for investigation; neither identifies a human actor or establishes what happened at the write boundary. Preserve the observed set and observation time. A single resolver result does not establish that every cache has the same answer.

Before deployment, exercise failed intent storage, rejected writes, lost responses, and lookup errors in a controlled environment. Check that each result leaves an honest event sequence. Then test retrieval as a reviewer with read access: starting from the change identifier, can that reviewer find the approval, submitted values, execution outcome, and observations without asking the operator to reconstruct events from memory? That is a more useful acceptance test than merely seeing the new exchange in a lookup.

For observability, derive counts of rejected writes, unresolved execution outcomes, and overdue observations from events. Put change identifiers in searchable logs; avoid turning every actor and record name into a metric label. Define observation deadlines in the deployment policy, with expected cache behavior considered, rather than declaring all mismatches incidents immediately. Keep audit evidence after the alert closes.

## Limits of this evidence path

The local file is a teaching artifact. Its owner can edit or remove it, synchronization does not make it tamper-proof, and the example has no retention enforcement or export authorization. A production implementation needs storage and access controls that match the chosen evidence policy. Include collection failures in monitoring so missing evidence is visible.

MX verification also has a narrow boundary: it checks mail-routing records. It does not prove message delivery or DMARC alignment. RFC 7489 describes DMARC's relationship to SPF and DKIM; changing an MX target does not by itself establish those authentication results. Keep delivery and authentication checks in the mail cutover plan, with their own evidence.

The remaining acceptance criterion is custody: the responsible team can retrieve an attributable change history, including uncertain outcomes, after the cutover and after the people involved have moved on.

## References

The sources below support record semantics, update responses, logging structure, reserved examples, and mail authentication. The custody choices and acceptance checks above are engineering recommendations.

## Sources

- RFC 1035, domain names and MX resource records: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 2136, dynamic updates in DNS: https://datatracker.ietf.org/doc/html/rfc2136
- RFC 5424, the syslog protocol: https://datatracker.ietf.org/doc/html/rfc5424
- RFC 2606, reserved example names: https://datatracker.ietf.org/doc/html/rfc2606
- RFC 7489, DMARC: https://datatracker.ietf.org/doc/html/rfc7489
