---
name: pgvector-semantic-search
description: >-
  Stand up k-nearest-neighbour semantic search on a PostgreSQL table with pgvector -
  choose the type and metric, declare the column, build the right index, and write a
  query the planner will actually use the index for.
api: pgvector
contract: data-model/pgvector-vector.sql
operations:
  - vector
  - halfvec
  - '<->'
  - '<=>'
  - '<#>'
  - '<+>'
  - hnsw
  - ivfflat
  - vector_cosine_ops
  - vector_l2_ops
  - hnsw.ef_search
  - ivfflat.probes
generated: '2026-08-27'
method: generated
source: https://github.com/pgvector/pgvector#readme
---

# Semantic search with pgvector

pgvector is a PostgreSQL extension. There is no endpoint to call and no key to obtain.
Everything below is SQL issued over an ordinary Postgres connection. Every type,
operator, operator class and GUC named here is declared in the extension's own DDL
(`data-model/pgvector-vector.sql`); none is invented.

## 1. Install the extension

```sql
CREATE EXTENSION vector;
```

Requires PostgreSQL 13 or later. Confirm what you got:

```sql
SELECT extversion FROM pg_extension WHERE extname = 'vector';
```

If this errors, the extension binary is not installed on the server. That is an
operator task, not a SQL one — see the installation section of the README.

## 2. Declare the column with a fixed dimension

```sql
CREATE TABLE items (
  id        bigserial PRIMARY KEY,
  content   text,
  embedding vector(1536)
);
```

The dimension count is mandatory if you intend to index. A `vector` column without a
typmod raises `column does not have dimensions` at `CREATE INDEX` time.

Match the number to your embedding model's output width. A mismatch raises
`expected 1536 dimensions, not 768` on insert — the most common failure when a model
is swapped.

## 3. Choose the metric before you choose the index

The operator and the operator class must agree, because an index only serves the
metric its opclass names.

| Metric | Operator | HNSW / IVFFlat opclass |
| --- | --- | --- |
| Cosine distance | `<=>` | `vector_cosine_ops` |
| Euclidean (L2) | `<->` | `vector_l2_ops` |
| Negative inner product | `<#>` | `vector_ip_ops` |
| Taxicab (L1) | `<+>` | `vector_l1_ops` (HNSW only) |

Use cosine for normalized embeddings from a text model unless you have a reason not to.

## 4. Insert

```sql
INSERT INTO items (content, embedding) VALUES ($1, $2);
```

Pass the vector as the bracket literal `'[0.1,0.2,...]'`, not Postgres array braces
`'{0.1,0.2}'` — braces are a different type and produce
`invalid input syntax for type vector`. Every first-party client library exists
precisely to do this encoding for you; use it rather than string-building.

All elements must be finite. NaN or Infinity raises at insert time.

## 5. Build the index

```sql
SET maintenance_work_mem = '8GB';
CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops);
```

Build parameters, both optional: `m` (default 16, max connections per layer) and
`ef_construction` (default 64, candidate list size during construction). Higher
`ef_construction` buys recall at the cost of build and insert time. Setting
`ef_construction` below `2 * m` raises `ef_construction must be greater than or equal
to 2 * m`.

Raise `maintenance_work_mem` first. If the graph does not fit, the build spills and a
`NOTICE: hnsw graph no longer fits into maintenance_work_mem` appears — do not ignore
it, the build will be dramatically slower.

Watch progress with `pg_stat_progress_create_index`.

Index ceilings are lower than type ceilings: HNSW indexes `vector` to 2,000 dimensions
and `halfvec` to 4,000. Above that, index a quantized or reduced expression — see
`pgvector-quantized-search`.

## 6. Query

```sql
SELECT id, content
FROM items
ORDER BY embedding <=> $1
LIMIT 5;
```

Three conditions must all hold or the planner will not use the index:

1. `ORDER BY` is a bare distance operator, not an expression around one.
   `ORDER BY 1 - (embedding <=> $1) DESC` will sequential-scan.
2. The order is ascending (the default).
3. There is a `LIMIT`.

## 7. Tune recall per query, not per session

```sql
BEGIN;
SET LOCAL hnsw.ef_search = 100;   -- default 40
SELECT id, content FROM items ORDER BY embedding <=> $1 LIMIT 5;
COMMIT;
```

Use `SET LOCAL` inside a transaction. A bare `SET` persists for the session, and on a
pooled connection that leaks your tuning into unrelated queries.

For IVFFlat the equivalent knob is `ivfflat.probes` (default 1).

## 8. Expect fewer than k results, and handle it

This is the behaviour that surprises people, and it raises no error.

Results are capped by `hnsw.ef_search`, then reduced further by dead tuples and by any
`WHERE` clause — the filter is applied *after* the index returns its candidates. NULL
vectors, and zero vectors under cosine distance, are never indexed at all.

Remedies, in order of preference:

```sql
BEGIN;
SET LOCAL hnsw.iterative_scan = strict_order;  -- 0.8.0+
SELECT ... ;
COMMIT;
```

or raise `hnsw.ef_search`, or build a partial index matching your filter, or partition
the table. Do not silently accept a short result set as "no more matches".

## 9. Check recall against ground truth

```sql
BEGIN;
SET LOCAL enable_indexscan = off;   -- exact search
SELECT id FROM items ORDER BY embedding <=> $1 LIMIT 5;
COMMIT;
```

Compare against the indexed result. This is the only way to know whether your
`ef_search` / `probes` setting is adequate for your data.

## Errors you will actually hit

Full catalog in `errors/pgvector-problem-types.yml`. The four that account for most
failures:

| Message | SQLSTATE | Cause |
| --- | --- | --- |
| `expected N dimensions, not M` | 22000 | Model output width changed |
| `invalid input syntax for type vector` | 22P02 | Braces instead of brackets |
| `column does not have dimensions` | 22023 | Indexing an untyped `vector` column |
| `column cannot have more than 2000 dimensions for hnsw index` | 54000 | Index ceiling, not type ceiling |
