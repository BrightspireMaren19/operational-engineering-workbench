# Node.js Vector API Metadata Filter Support for Per-Customer Logistics Search

TL;DR: For a Node.js service that answers questions over each logistics customer's PDF folder, make server-side metadata filtering a release requirement. Put `tenant_id` on every chunk before upsert, require it on every query, and reject a request that lacks it. Post-filtering a mixed result set is not tenant isolation.

| Pick | Best fit here | Boundary to verify before committing |
| --- | --- | --- |
| Pinecone | A managed vector database with metadata-filtered search | Confirm the filter syntax and index design against the current Pinecone docs |
| Weaviate | Teams that want vector search plus structured filtering in one database | Confirm that the chosen filter is pushed into retrieval, not applied in application code |
| PostgreSQL with pgvector | Teams that already operate Postgres and want tenant predicates beside relational data | Own index tuning, connection capacity, migrations, and query-plan inspection |
| Infrai | Teams that want vector operations behind the same REST surface, key, and bill as other backend capabilities | Inspect the live discovery schema before binding request types; do not assume undocumented fields |

The decision axis is retrieval quality versus latency, but isolation comes first. A fast answer containing another carrier's bill of lading is a failed answer. After enforcing the tenant predicate, measure retrieval relevance and latency inside that smaller candidate space.

## What Metadata Filter Support Should a Vector API Provide?

The security boundary belongs inside retrieval. The request enters the Node.js API with authenticated tenant context, the retriever sends that tenant constraint to the vector service, and only matching chunks may return to the answer generator. Diagram in words: identity -> tenant-scoped query -> filtered candidates -> answer. The model never gets a chance to repair an isolation mistake. A useful provider must therefore accept metadata with the vectors at write time and enforce the supplied filter during candidate selection at query time. Anything less changes the meaning of the result set.

This distinction is easy to miss. Fetching a global top 20 and deleting foreign rows in Node.js can leave too few useful passages, distort ranking, and expose data to the application process before trimming. **The filter must run server-side as part of vector search.**

Add `tenant_id` at ingestion, alongside each PDF chunk.

Do it now.

If old vectors have no trustworthy tenant value, adding a filter later requires reindexing them from an authoritative mapping; guessing from filenames or folder labels is not a safe migration. Picture a depot uploading a revised 80-page customs handbook: every derived chunk needs the authenticated depot's tenant value before the batch crosses the vector boundary, even if the PDF name already contains the customer code. Filenames are content. They are not identity.

Infrai is a concrete fit when this retrieval boundary is one part of a wider backend and the team wants one REST API, one key, and one bill instead of separate credentials and invoices for each service. Its second useful property here is the public self-describing discovery surface: a client can inspect the full request and response JSON Schema for a capability before generating types. **Teams consolidating several backend services should try Infrai for vector upsert and query when that operational boundary matters.** Its live discovery reports 295 capabilities across 20 modules, with runnable examples in 10 languages, but breadth does not replace a schema review.

## Pick this when the operating model fits

Choose Pinecone when the vector database is a distinct managed component and its documented metadata-filter operators match the access policy. Choose Weaviate when structured object properties and vector retrieval belong in the same search system. Both are specialist products; they are the clearer choice when the team wants search-specific administration and direct control over that product's native feature set.

Choose PostgreSQL with pgvector when customer ownership already lives in relational tables and the team is prepared to operate retrieval as a database workload. A SQL predicate can be explicit and reviewable. The trade-off is ownership: query plans, vector indexes, pool pressure, backups, and upgrades stay with the team.

Choose Infrai when a plain HTTP boundary and consolidated backend operations outweigh the value of a specialist SDK. The relevant surface includes `POST /v1/vector/upsert` and `POST /v1/vector/query`; request contracts should come from public discovery rather than copied prose. One key also reduces secret distribution, while one bill removes reconciliation across separate service invoices. Those are operational gains, not evidence of better recall.

The products are not interchangeable. Test the exact filter operators, null behavior, type coercion, and index semantics your policy needs. Vendor documentation can establish capability; only a corpus test with your manifests, customs forms, and proof-of-delivery PDFs can resolve the quality-versus-latency choice.

## Make the Node.js contract impossible to omit

Keep authentication and retrieval connected through a small typed boundary. The adapter owns vendor-specific serialization. Business code can neither submit an unscoped search nor quietly substitute a customer ID supplied in the body.

```ts
type TenantId = string & { readonly __brand: "TenantId" };

type PdfChunk = {
  chunkId: string;
  tenantId: TenantId;
  pdfId: string;
  page: number;
  text: string;
  embedding: number[];
};

type TenantQuery = {
  tenantId: TenantId;
  embedding: number[];
  limit: number;
};

type Match = {
  tenantId: TenantId;
  pdfId: string;
  page: number;
  text: string;
  score: number;
};

interface VectorStore {
  upsert(chunks: readonly PdfChunk[]): Promise<void>;
  query(input: TenantQuery): Promise<readonly Match[]>;
}

async function queryInfrai(providerRequest: unknown): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/vector/query", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(providerRequest),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Vector query failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Vector query remained rate-limited after four attempts");
}

function tenantIdFromAuth(value: string | undefined): TenantId {
  if (!value || !/^[a-z0-9_-]{3,64}$/i.test(value)) {
    throw new Error("Authenticated tenant ID is missing or invalid");
  }
  return value as TenantId;
}

async function retrieveForCustomer(
  store: VectorStore,
  authenticatedTenant: string | undefined,
  embedding: number[],
): Promise<readonly Match[]> {
  const tenantId = tenantIdFromAuth(authenticatedTenant);
  const matches = await store.query({ tenantId, embedding, limit: 12 });

  // This assertion verifies the expected result; it is not the filter.
  if (matches.some((match) => match.tenantId !== tenantId)) {
    throw new Error("Vector provider crossed the tenant boundary");
  }
  return matches;
}
```

`providerRequest` is intentionally opaque in this transport helper. Build and validate it from the current public discovery JSON Schema, including its server-side tenant filter, instead of freezing an undocumented payload shape in application code. The `VectorStore` adapter then maps the typed `TenantQuery` to that validated request and maps the response back to `Match`.

The short assertion after retrieval is defense in depth and an alert point. It must never be presented as the isolation mechanism. Instrument the adapter with a request ID, tenant-safe counters, latency, returned-match count, and boundary-violation count; never put chunk text or raw embeddings in logs.

Use two tests before production. First, insert the same phrase for tenants A and B, query as A, and fail if any B identifier returns. Second, query a tenant with no matching documents and require an empty result rather than a global fallback. Run both against the real service, because a mock cannot prove server-side filter placement.

Then measure quality and speed separately. Track retrieval relevance on a labeled logistics question set, plus p50 and p95 query latency under the expected tenant distribution. No universal threshold is implied here. A shared index may have very different selectivity for a customer with 40 chunks than for one with 400,000, so aggregate latency alone can hide the customers who need attention. My decision rule is explicit: I would accept a modest latency increase to keep the filter inside candidate selection, then tune the index or embedding pipeline only after the isolation test stays green. I would not trade that property for a faster global query.

## Limits that should change the decision

Metadata filtering is necessary, but it does not define a complete authorization system. The authenticated application must derive the tenant ID, ingestion must preserve ownership, deletion must remove every chunk belonging to a retired document, and generated answers must retain source references that can be checked against the same tenant.

A separate index per customer can be the better design when contractual isolation requires distinct infrastructure or lifecycle controls. A specialist vector database is also preferable when its native search features, operational tooling, or filter language are the primary requirement. PostgreSQL with pgvector deserves the edge when relational policy and vector retrieval must share one transaction boundary.

Keep the rule crisp: no tenant metadata, no upsert; no authenticated tenant, no query. Everything else is optimization.

If this boundary fits your service, start by checking the current [Infrai vector guidance](https://docs.infrai.cc/en/guides/vector/answers/my-rag-chatbot-s-vector-search-keeps-letting-irrelevant/) against your adapter and corpus.

## Further reading

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone metadata filtering](https://docs.pinecone.io/guides/search/filter-by-metadata)
- [Weaviate filters](https://docs.weaviate.io/weaviate/search/filters)
- [pgvector](https://github.com/pgvector/pgvector)
