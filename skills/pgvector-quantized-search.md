---
name: pgvector-quantized-search
description: >-
  Cut pgvector storage and index memory using halfvec, binary quantization or
  subvector indexing, then restore recall with a two-stage rerank. Use when the
  working set no longer fits in memory or the embedding exceeds the index dimension
  ceiling.
api: pgvector
contract: data-model/pgvector-vector.sql
operations:
  - halfvec
  - binary_quantize
  - subvector
  - '<~>'
  - '<=>'
  - bit_hamming_ops
  - vector_cosine_ops
  - hnsw
generated: '2026-08-27'
method: generated
source: https://github.com/pgvector/pgvector#scaling
---

# Quantized and reranked search with pgvector

Every function, operator and operator class below is declared in
`data-model/pgvector-vector.sql`. The two-stage query shapes are the ones the pgvector
README documents under Binary Quantization and Indexing Subvectors.

## When to reach for this

- The HNSW index no longer fits in memory and query latency has fallen off a cliff.
- The embedding is wider than the index ceiling — HNSW indexes `vector` to 2,000
  dimensions and `halfvec` to 4,000, while both types *store* up to 16,000. A 3,072
  dimension embedding is storable and not directly indexable.
- Storage cost, not latency, is the binding constraint.

Do not reach for it first. Quantization always costs recall; the rerank buys most of
it back, but not all.

## Option 1 — halfvec: halve storage, keep the metric

Cheapest change with the least recall cost. `halfvec` stores `2 * dims + 8` bytes
against `vector`'s `4 * dims + 8`, and doubles the index dimension ceiling to 4,000.

```sql
ALTER TABLE items ALTER COLUMN embedding TYPE halfvec(1536);
CREATE INDEX ON items USING hnsw (embedding halfvec_cosine_ops);
```

Every operator and function you were using has a `halfvec` overload — `<->`, `<#>`,
`<=>`, `<+>`, `cosine_distance`, `l2_distance`, `l2_normalize`, `subvector`,
`binary_quantize`, `avg`, `sum`. Only the opclass name changes.

Or keep `vector` storage and index a cast expression, if you want full precision on
disk and a smaller index:

```sql
CREATE INDEX ON items USING hnsw ((embedding::halfvec(1536)) halfvec_cosine_ops);
```

## Option 2 — binary quantization: 32x smaller index, rerank required

`binary_quantize` reduces each element to one bit. Index the expression, search by
Hamming distance, then rerank the shortlist against the original vectors.

```sql
CREATE INDEX ON items
  USING hnsw ((binary_quantize(embedding)::bit(1536)) bit_hamming_ops);
```

Single-stage — fast, and recall will disappoint:

```sql
SELECT id FROM items
ORDER BY binary_quantize(embedding)::bit(1536) <~> binary_quantize($1)
LIMIT 5;
```

Two-stage, which is the shape you actually want:

```sql
SELECT * FROM (
  SELECT * FROM items
  ORDER BY binary_quantize(embedding)::bit(1536) <~> binary_quantize($1)
  LIMIT 100
) candidates
ORDER BY embedding <=> $1
LIMIT 5;
```

The inner `LIMIT` is the knob: wider shortlist, better recall, more work in the
rerank. Start at 20x the final `LIMIT` and measure against exact search.

Note that the inner stage uses Hamming distance and the outer stage uses your real
metric. That is deliberate — the bit index only approximates the ranking; the outer
stage is what produces it.

## Option 3 — subvector indexing: index a prefix

For a wide embedding whose leading dimensions carry most of the signal (Matryoshka-style
models), index a prefix and rerank on the full vector.

```sql
CREATE INDEX ON items
  USING hnsw ((subvector(embedding, 1, 512)::vector(512)) vector_cosine_ops);

SELECT * FROM (
  SELECT * FROM items
  ORDER BY subvector(embedding, 1, 512)::vector(512) <=> subvector($1::vector, 1, 512)
  LIMIT 100
) candidates
ORDER BY embedding <=> $1
LIMIT 5;
```

`subvector(v, start, count)` is 1-indexed. The expression in the query must match the
expression in the index exactly — including the cast — or the index is not used.

## Option 4 — sparsevec, for genuinely sparse embeddings

```sql
ALTER TABLE items ADD COLUMN sparse sparsevec(30000);
INSERT INTO items (sparse) VALUES ('{1:1,3:2,5:3}/30000');
```

Format is `{index:value,...}/dimensions`, indices 1-based, strictly ascending, unique,
and zero values may not be stored — each of those constraints has its own error.
Storage is `8 * nnz + 16` bytes. HNSW indexes `sparsevec` to 1,000 non-zero elements.

## Verify the tradeoff before you ship it

Quantization is only sound if you measured what it cost. For a fixed query set:

```sql
BEGIN;
SET LOCAL enable_indexscan = off;
SELECT id FROM items ORDER BY embedding <=> $1 LIMIT 5;   -- ground truth
COMMIT;
```

Compare the overlap with your quantized-plus-rerank result. If recall@5 is
unacceptable, widen the inner `LIMIT` before abandoning the approach — the shortlist
size is almost always the variable that matters, not the quantization itself.

## Reversibility

Changing the storage type is a rewriting `ALTER TABLE` and quantization is lossy:
`vector` to `halfvec` discards precision that cannot be recovered from the column.
Keep the source embeddings, or be prepared to re-embed. Dropping and rebuilding an
*index*, by contrast, is free and non-destructive — indexes are derived, so experiment
there first.
