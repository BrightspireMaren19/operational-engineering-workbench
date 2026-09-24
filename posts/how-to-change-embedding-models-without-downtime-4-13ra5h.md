# How to Change Embedding Models Without Downtime — 4 Cost Controls

Short answer: to reindex after changing an embedding model without downtime, build a second index, backfill it from the source of truth, shadow a fixed set of searches, then switch one small alias or configuration value. Keep the old index readable until the new path has survived both retrieval checks and rollback practice. For a healthtech team adding semantic search to product content, this blue-green approach is the least complex option that preserves availability while making index cost visible.

Start with the migration shape, because the wrong shape can double spend for much longer than expected.

| Approach | Pick this when | Availability during rebuild | Peak index footprint | Main risk |
|---|---|---:|---:|---|
| In-place rebuild | Search can be paused or degraded | Low | About one generation | Mixed or missing vectors |
| Blue-green index | Reads must continue and a clean rollback matters | High | About two generations | Temporary duplicate storage |
| Partitioned migration | The corpus is too large to duplicate at once | High, with careful routing | Tunable by partition | Queries crossing model generations |

The default here is blue-green. It buys a clean invariant: a query uses one embedding model and one compatible index generation from start to finish. The four cost controls are deterministic chunking, bounded write concurrency, delta capture, and an explicit retirement deadline for the old generation.

## Which migration shape fits the index budget?

An in-place rebuild is attractive when cost dominates every other concern. It avoids holding two complete generations. It also creates the hardest failure mode: documents already rewritten with the new model can sit beside old vectors, even though their dimensions and similarity behavior may differ. A maintenance window makes this option honest. Without one, it is difficult to reason about search results or rollback.

Use a blue-green index when product pages, dosage-device instructions, compatibility notes, and safety FAQs must remain searchable throughout the migration. Every live read stays on `content-v1`; writes are mirrored to `content-v1` and `content-v2`; a backfill fills the remaining gaps; then a single logical pointer moves to `content-v2`. The duplicate footprint is deliberate and temporary.

Partitioned migration is the pressure valve for a corpus that cannot be duplicated. Move a tenant, locale, or stable document-key range at a time. Routing must bind both query embedding and document lookup to the same generation. Cross-partition ranking is suspect when scores come from different embedding spaces, so avoid merging those scores as if they were calibrated.

There is no magic fourth architecture. If neither duplicate capacity nor mixed-generation routing is acceptable, schedule downtime. Make that trade-off explicit.

## How can you reindex after changing an embedding model without mixing generations?

Treat the model identifier as data, not deployment trivia. A generation record should bind the logical index name, embedding model ID, vector dimension, chunker version, source revision, and lifecycle state. Retrieval resolves that record once per request. It must not read a mutable alias again halfway through the operation.

Here is the diagram in words: source documents flow into a deterministic chunker; each chunk goes to the old and new embedders during the migration; those outputs land in separate physical indexes; query traffic still enters through one resolver; shadow queries fork after retrieval input capture but before user-visible ranking; finally, the resolver changes generations atomically.

The following example uses generic interfaces. It stores no clinical records, and its sample documents are public product content rather than patient data.

```ts
type Generation = "content-v1" | "content-v2";

type ProductPage = {
  id: string;
  revision: number;
  title: string;
  body: string;
};

type Chunk = {
  id: string;
  documentId: string;
  revision: number;
  text: string;
};

type VectorRecord = Chunk & {
  modelId: string;
  values: number[];
};

interface Embedder {
  readonly modelId: string;
  embed(texts: string[]): Promise<number[][]>;
}

interface VectorIndex {
  upsert(generation: Generation, records: VectorRecord[]): Promise<void>;
  removeDocument(generation: Generation, documentId: string): Promise<void>;
  count(generation: Generation): Promise<number>;
}

interface GenerationStore {
  active(): Promise<Generation>;
  compareAndSet(expected: Generation, next: Generation): Promise<boolean>;
}

const chunk = (page: ProductPage): Chunk[] => {
  const sections = page.body.split(/\n\n+/).filter(Boolean);
  return sections.map((text, offset) => ({
    id: `${page.id}:${page.revision}:${offset}`,
    documentId: page.id,
    revision: page.revision,
    text: `${page.title}\n${text}`,
  }));
};

async function indexPage(
  page: ProductPage,
  generation: Generation,
  embedder: Embedder,
  index: VectorIndex,
): Promise<void> {
  const chunks = chunk(page);
  const vectors = await embedder.embed(chunks.map((item) => item.text));
  if (vectors.length !== chunks.length) {
    throw new Error("Embedding response count does not match chunk count");
  }

  await index.removeDocument(generation, page.id);
  await index.upsert(
    generation,
    chunks.map((item, position) => ({
      ...item,
      modelId: embedder.modelId,
      values: vectors[position],
    })),
  );
}
```

Stable chunk IDs make retries idempotent for one document revision. Removing the document before its replacement also prevents stale chunks from surviving when an editor shortens a page. In a real store, make replacement atomic if the engine supports it; otherwise write a new revision, mark it complete, and let reads filter to that revision.

Do not send the entire corpus in one promise fan-out. Bound concurrency. A queue with retry limits lets operators control embedding throughput and observe permanent failures instead of hiding them inside one enormous batch.

```ts
async function backfill(
  pages: AsyncIterable<ProductPage>,
  embedder: Embedder,
  index: VectorIndex,
  concurrency = 8,
): Promise<{ indexed: number; failed: string[] }> {
  let indexed = 0;
  const failed: string[] = [];
  const running = new Set<Promise<void>>();

  for await (const page of pages) {
    const task = indexPage(page, "content-v2", embedder, index)
      .then(() => {
        indexed += 1;
      })
      .catch(() => {
        failed.push(page.id);
      })
      .finally(() => {
        running.delete(task);
      });

    running.add(task);
    if (running.size >= concurrency) {
      await Promise.race(running);
    }
  }

  await Promise.all(running);
  return { indexed, failed };
}
```

Eight workers are an example, not a universal setting. Raise or lower the value from observed queue age, embedding latency, error rate, and downstream write saturation. This is cost control in operational form: throughput has a dial.

Backfill alone is insufficient because editors keep changing product pages. Record a high-water revision before the scan, mirror new writes to both generations, then replay every change after that revision into the new index. Deletions belong in the same change stream. A successful document count can still conceal a deleted safety page that remains retrievable.

## Validate before the pointer moves

Validation has two layers. Structural checks ask whether every current source document has a completed new revision, no active record carries the old model ID, vector dimensions are consistent, and the failed-job set is empty. Retrieval checks ask whether known product questions still return the intended source chunks.

Build the retrieval set before changing models. Include exact product names, abbreviations, misspellings, negative constraints, and near-neighbor products whose instructions must not be confused. For health product content, a result that looks topically close can still be the wrong device variant. Log document IDs and revisions so reviewers can identify the mismatch without storing raw query text when it may contain sensitive material.

Shadowing means running the new retrieval path without using its answer for the user. Compare old and new top results, but do not require identical rankings; changing the embedding model would then have no point. Define acceptance from the product task: required-source recall on the fixed set, zero forbidden-source matches, latency bounds, and an investigated reason for material disagreement.

Watch four time series during the rebuild: queue age, successful chunks per minute, failure ratio, and estimated remaining chunks. Add new-index write latency and shadow-query latency. Alert on stalled progress rather than raw job duration, because a large healthy rebuild can run longer than a small broken one.

Then rehearse rollback. It should be boring.

```ts
async function cutOver(
  generations: GenerationStore,
  oldIndex: VectorIndex,
  newIndex: VectorIndex,
): Promise<void> {
  const [oldCount, newCount] = await Promise.all([
    oldIndex.count("content-v1"),
    newIndex.count("content-v2"),
  ]);

  if (newCount === 0 || newCount < oldCount) {
    throw new Error("New generation is incomplete; refusing cutover");
  }

  const changed = await generations.compareAndSet("content-v1", "content-v2");
  if (!changed) {
    throw new Error("Active generation changed during cutover");
  }
}
```

Count parity is only a guardrail, not proof of semantic quality. The compare-and-set is the important availability mechanism: concurrent deploys cannot silently overwrite one another. Keep query embedding selection behind the same generation record, so switching the index also switches the model used for queries.

## Control cost without making rollback imaginary

Peak storage is roughly the two index generations plus metadata and any engine-specific overhead. Embedding work is driven by chunk count, so calculate the migration from actual chunks, not documents. A 40-page instruction set and a two-paragraph FAQ are not equivalent units.

Four controls keep the temporary expense bounded:

1. Freeze the chunker version for the model-only migration. Rechunking at the same time changes both cost and retrieval behavior, making failures harder to attribute.
2. Set a worker ceiling and a retry budget. Permanent errors go to a review queue; they do not spin forever.
3. Capture deltas from the high-water mark. Re-embed changed pages, not the full corpus again when validation finds a late edit.
4. Give the old generation a retirement deadline after the rollback window. Track its last read, but delete it only after rollback criteria expire and retention rules permit deletion.

For a rough forecast, multiply source chunks by one embedding operation per chunk, then add a measured allowance for retries and edits during the run. Keep the estimate in counts. Avoid hard-coding a provider price into architecture documentation; rates and billing units can change, while chunk volume and retry behavior remain useful planning inputs.

The tempting shortcut is to delete `content-v1` immediately after cutover. That converts a reversible configuration change into another full rebuild. A short, explicit overlap is usually the cleaner trade: more temporary storage, much faster recovery.

## Limits and stopping rules

This pattern cannot guarantee zero downtime if the source system cannot produce a consistent scan plus ordered changes. In that case, add revisioned snapshots or accept a maintenance boundary. It also does not prove answer quality. Retrieval evaluation and grounded generation checks remain separate work, as retrieval-augmented generation combines retrieved evidence with generation rather than making retrieval correctness automatic.

Stop the migration when the new generation passes structural checks, the fixed retrieval set meets its acceptance rules, shadow traffic stays within the latency and error budgets, and rollback has been exercised. Switch once. Retain the old generation for the declared window, then remove it and stop dual writes.

That is the full move: isolate, backfill, verify, switch, observe, retire.

## Further reading

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
