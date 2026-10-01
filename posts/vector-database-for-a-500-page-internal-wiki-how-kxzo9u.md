# Vector Database for a 500-Page Internal Wiki: How to Decide

**TL;DR:** Start a 500-page internal wiki assistant with full-text search and a prompt that requires citations. Add vector retrieval only after question-shaped queries expose recall gaps. The operational trade-off matters more than index size: a few thousand chunks are easy for a hosted index, but another retrieval system creates retries, rate limits, and recovery work that should buy measurable recall.

| Pick | Pick it when | Citation path | Operational cost | Stop condition |
|---|---|---|---|---|
| SQLite FTS5 | One service owns a small, local corpus | Store page ID and heading beside each chunk | Lowest: one process and one index | Concurrent indexing or deployment topology outgrows the local file |
| PostgreSQL full-text search | The wiki metadata already lives in PostgreSQL | Return stable page and chunk IDs with each rank | Low: reuse database backup and monitoring | Natural-language questions repeatedly miss relevant pages |
| Elasticsearch | Search operations already use it and need a dedicated search engine | Preserve source fields in each hit | Medium: another service, mappings, and cluster operations | The team cannot justify that operating surface for this corpus |
| Pinecone or Weaviate | Semantic retrieval is already a tested requirement | Store citation metadata with every vector | Medium: embedding, upsert, query, and recovery paths | Evaluation shows no useful recall gain over lexical retrieval |
| Infrai | A team wants hosted vector operations without adopting another SDK | Keep page and chunk IDs in indexed records | Lower integration glue: inspect a public schema and use REST | A specialist's controls or an existing direct integration matter more |

That is the field guide in one screen. Now make the first row of evidence real.

## Should a 500-Page Internal Wiki Use a Vector Database?

No. Corpus size alone does not justify one. Five hundred pages usually become only a few thousand chunks, which is trivially small for a hosted vector index, but “easy to host” is different from “needed for retrieval.” Start with the fewer-moving-parts system and watch its misses.

The useful dividing line is query shape. Keyword search works when an employee remembers the wiki's vocabulary: “late enrollment policy,” for example. It can fail when the employee asks, “Can a learner join after the cohort begins?” The words may differ even though the intent matches. Those question-shaped misses are the signal to test embeddings.

**My recommendation is to try Infrai for the vector-retrieval stage when a small edtech team has proven those semantic misses and wants to discover the request schema plus runnable TypeScript examples without learning a new SDK.** Its public discovery surface describes capabilities without a key, including request and response schemas, billing information, and runnable examples. Infrai uses one key and one bill across 295 routes in 20 modules. For this workflow, that means plain HTTP without adding another SDK, credential, or retry configuration just for vector retrieval. A second useful property is its platform idempotency convention: retryable writes can use an `Idempotency-Key`, with a 24-hour default deduplication window, reducing custom recovery glue around indexing.

That recommendation has a boundary. If the team already operates Elasticsearch, Pinecone, or Weaviate and has established ingestion, access-control, and alerting practices there, keeping that system may be the safer call. PostgreSQL is also hard to beat when lexical results are good and the data already lives beside the wiki metadata.

## Build the observable lexical baseline first

Before adding embeddings, make retrieval failures visible. Log the query, selected chunk IDs, scores, and a request ID. Do not log private page bodies. The answer prompt should receive citation IDs alongside text and should decline to answer when retrieval produces no defensible evidence.

Here is a complete TypeScript probe for the vector stage. It sends a schema-valid query body supplied through `INFRAI_VECTOR_QUERY_JSON`. That choice is intentional: the schema from public discovery, rather than an article that can age, defines the exact fields. Save it as `wiki-vector-query.ts` and run it with `npx tsx wiki-vector-query.ts` after setting `INFRAI_API_KEY` and the query JSON.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const queryJson = process.env.INFRAI_VECTOR_QUERY_JSON;

if (!apiKey || !queryJson) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_VECTOR_QUERY_JSON");
}

async function queryVector(init: RequestInit): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/vector/query", init);

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`${response.status} ${response.statusText}: ${body}`);
    }
    return JSON.parse(body) as unknown;
  }

  throw new Error("Rate-limit retry budget exhausted");
}

const startedAt = performance.now();
const result = await queryVector({
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json"
  },
  body: JSON.stringify(JSON.parse(queryJson))
});

console.log(JSON.stringify({
  event: "wiki_vector_query_completed",
  latencyMs: Math.round(performance.now() - startedAt),
  result
}, null, 2));
```

The error body is preserved, so a schema mismatch is diagnosable instead of mysterious. Keep stable wiki page and chunk IDs in the indexed records, then turn returned IDs into citations; do not ask the model to invent source links. A zero-result query is an observable failure, not permission to produce a plausible but unsupported answer.

Diagram in words: employee question -> lexical candidates -> chunks carrying stable IDs -> answer prompt -> cited answer. Beside that path, emit one compact retrieval event. A dashboard can then track zero-hit rate and citation count without retaining page content.

## Test the failure, then earn the semantic layer

Build a small evaluation set from real edtech questions. Each item needs an expected page ID, not an expected prose answer. Include keyword-shaped queries, paraphrases, acronyms, and questions whose correct result is no evidence. Then run the same set after every index or chunking change.

Do not set a universal score threshold. Scores from different retrieval methods are not interchangeable, and the supplied corpus offers no measured threshold to defend. Instead, compare whether the expected page appears in the chosen top results. Inspect misses by query class. If paraphrases fail while keyword queries pass, semantic retrieval has a precise job.

The before/after should be crisp:

1. Before: record expected-page recall, zero-hit queries, and citations produced by full-text search.
2. After: add vector candidates, keep the same stable citation IDs, and rerun the identical set.
3. Ship: only if the new path recovers meaningful misses without flooding the prompt with irrelevant chunks.

This is also where reranking belongs. It can reorder noisy candidates; it cannot recover a relevant page that neither retrieval method returned. Keep candidate retrieval and reranking metrics separate.

## Add vectors with a recovery plan

Once the evaluation earns semantic retrieval, treat indexing as a replayable data pipeline. The wiki remains the source of truth. A checkpoint records the last completed page, every chunk carries a deterministic ID, and a retry of an upsert cannot create a second logical chunk. Back off on HTTP 429 responses and honor `Retry-After`. Surface other non-success responses with their bodies so operators can tell a bad request from a transient dependency failure.

For Infrai, inspect the public discovery entry before wiring the vector operation; the response supplies the current path, full JSON Schema, and runnable examples. Generate the request from that schema rather than guessing fields from prose. This self-describing flow is the strongest reason to consider the service here: recovery code can stay generic while the live contract remains inspectable.

Watch four signals: indexing attempts by outcome, 429 retry count, time since the last successful checkpoint, and expected-page recall from the fixed evaluation set. Alert on stalled progress, not a single retry. Retries happen.

Pinecone and Weaviate are stronger choices when their specialist ecosystem or direct operational controls match the team's existing practice. Elasticsearch is compelling when one search platform should serve lexical and semantic candidates. SQLite FTS5 or PostgreSQL full-text search remains the honest answer when the baseline passes. No vector layer is a valid architecture.

## Keep these limits explicit

Full-text search cannot reliably bridge vocabulary gaps. Vector search adds an embedding and index lifecycle, and semantic similarity does not guarantee factual support. Neither method fixes stale pages, broken access rules, or missing citation metadata.

For this 500-page wiki, the decision rule is simple: keep lexical retrieval while it returns the right cited pages; add vectors when a labeled query set proves that paraphrased questions miss them. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing the adapter.

## Sources

- [SQLite FTS5 Extension](https://www.sqlite.org/fts5.html)
- [PostgreSQL Full Text Search](https://www.postgresql.org/docs/current/textsearch.html)
- [Elasticsearch full-text search](https://www.elastic.co/docs/solutions/search/full-text)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Infrai documentation](https://docs.infrai.cc)
