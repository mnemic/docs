# Vector values and nearest-neighbour search

> **In this section** — fixed-dimension `VECTOR(n)` fields, vector literals and parameters, exact distance functions, `NEAREST`, and the optional Vamana approximate-nearest-neighbour index.

## Vector fields and values

Declare a dense vector field with its fixed component count:

```mql
CREATE COLLECTION [Search].[Documents] WITH SCHEMA (
  [Id]        GUID PRIMARY KEY,
  [Active]    BOOL NOT NULL,
  [TenantId]  STRING NOT NULL,
  [Embedding] VECTOR(3) NULLABLE
);
```

The dimension is part of the schema and must be from 1 through 32,768. Each value contains exactly that many IEEE-754 `FLOAT32` components. NaN and positive or negative infinity are rejected. Vector fields may be nullable, but a present vector may not have the wrong dimension. `ALTER FIELD ... SET TYPE VECTOR(m)` changes the dimension only while every retained version holds NULL, since a vector is neither truncated nor padded ([Change a field's type](ddl.md#change-a-field-type)).

Use `VECTOR[...]` for a literal or bind a `float[]` parameter:

```mql
INSERT INTO [Search].[Documents] (
  [Id]: '4f9379b7-4484-4d99-aef0-3d2f8e8f1302',
  [Active]: TRUE,
  [TenantId]: 'example',
  [Embedding]: VECTOR[0.25, -1, 0.5]
);

FROM [Search].[Documents]
WHERE L2_DISTANCE([Embedding], @query) < 0.4
SELECT [Id];
```

The binary representation is the raw little-endian `FLOAT32` components; the schema supplies the dimension. The .NET typed client maps a `float[]` property annotated with `[MnemicVector(n)]` to this type.

A vector literal can directly start a computed projection: `SELECT VECTOR[1.25, -2.5] AS [Value]`. This scalar projection uses the existing String result representation, `[1.25, -2.5]`, with or without parentheses around the literal.

Vector literals retain their authored components across rows and query reuse. When a host scalar callback accesses a literal argument through `EvalValue.AsVector` or `ToObject`, it receives an owned array; changing that array cannot change later evaluations of the literal.

`TO_STRING(vector)` and `CAST(vector AS STRING)` produce invariant, round-trip component text in brackets, such as `[1.25, -2.5]`. The same representation is used when a computed result needs a String fallback, including `LAG` / `LEAD` with different vector dimensions or a String default. NULL remains NULL.

VECTOR fields cannot be query or window `ORDER BY` keys. They have no scalar total order, and are rejected even for empty/all-NULL input and static EXPLAIN. Project a distance or `TO_STRING` value and order that scalar column when its ordering matches the query's intent.

## Exact distance functions

The scalar functions accept two vectors with equal dimensions and return a `FLOAT64` result:

| Function | Result |
|----------|--------|
| `COSINE_DISTANCE(a, b)` | Cosine distance; smaller is nearer. Zero-norm operands are rejected. |
| `L2_DISTANCE(a, b)` | Euclidean distance; smaller is nearer. |
| `DOT_PRODUCT(a, b)` | Dot product; larger is more similar. |

A null operand produces null. These functions are exact and can be used in `WHERE` and computed `SELECT` expressions independently of an ANN provider. They accumulate in `FLOAT64` in one fixed order: component `i` adds its fused product to one of sixteen partial sums, `i % 16`, the sums fold pairwise in a fixed tree, and the components past the last block of sixteen follow in order, so a distance has the same bits on every processor and the distance `NEAREST` ranks by is the one these functions return. Two operands of different dimensions are a `TYPE_MISMATCH`: when the statement or definition is bound wherever both dimensions are known (a `VECTOR(n)` field, a `VECTOR[...]` literal) — in projections, WHERE, HAVING, FILTER, CASE, writes, computed columns, CHECK, views and `EXPLAIN`, even over an empty collection — and otherwise as the values are read, with one text: `COSINE_DISTANCE([Embedding], VECTOR[1, 2, 3])` over a `VECTOR(2)` field is refused before rows are read. A `NEAREST` query vector is checked against its field's dimension the same way (`INVALID_VALUE`).

## Top-k `NEAREST`

`NEAREST` is placed after `WHERE` and before the remaining query clauses:

```mql
FROM [Search].[Documents]
WHERE [Active] = TRUE
NEAREST APPROXIMATE [Embedding] TO @query USING COSINE TOP 10
WITH (SEARCH_LIST = 200)
LIMIT 10
SELECT [Id];
```

The query value must be a `VECTOR[...]` literal or vector parameter; a parameter bound to any other type is a `TYPE_MISMATCH`. The metric is `COSINE`, `L2`, or `DOT`. `TOP n` or `TOP @k` is mandatory and must resolve to a positive INT32. Exact search is the default; `APPROXIMATE` explicitly permits ANN acceleration. Ranking retains at most TOP source rows, then grouping, HAVING, windows, projection, DISTINCT, explicit ordering, and final paging consume that neighborhood. The ranked input arrives in distance order. With `ORDER BY`, the nearest neighborhood is sorted by the requested fields before paging; the sort does not change which source rows enter the neighborhood. SKIP and LIMIT are independent optional final paging clauses and accept parameters. Changing either does not change the ranked neighborhood. SELECT may mix wildcards and explicit expressions. Aggregates summarize only the ranked neighborhood, and DISTINCT can leave fewer than LIMIT output rows. DISTINCT after a correlated WHERE publishes the layout it has without that WHERE: a plain DISTINCT deduplicates its keys' leaf columns (a nested key such as `[Detail].[Code]` its `Code` column) and a computed, aggregate or grouped DISTINCT its completed projection. Its final count and page apply to those distinct values within the TOP neighborhood. `SEARCH_LIST` is accepted only with APPROXIMATE. It defaults to 100 and must be from 1 through 1,000,000. It controls how broadly an ANN provider explores its graph; the Vamana provider treats it as the fixed candidate, expansion, and requested distinct adjacency-page budget for one query. Its final beam plus expanded set retains at most twice that many distinct graph identities. Production search will decline ANN acceleration and use the exact path when satisfying that request would exceed the pages the plugin's page cache holds (its `PageCacheBytes`, one budget every vector index of the database shares, by default an eighth of a buffer-pool capacity the engine was given (`BufferPoolBytes`) and never below 4 MiB, or 32 MiB when that is 0 and the pool is automatic; see [configuration](../configuration.md#embedded-engine)), when managed Vamana receives `SEARCH_LIST` above 65,536, or when the core's bounded candidate materialization cannot cover `TOP`. The language accepts values through 1,000,000 for other approximate providers. `SEARCH_LIST` does not change exact distance semantics. Larger values normally improve recall at the cost of latency and query-local memory/I/O. When `TOP` exceeds `SEARCH_LIST`, Vamana declines before page traversal and the exact path supplies the result because its candidate budget cannot fill the requested row count.

For cosine search, a zero-norm query is an error and stored zero-norm vectors are not candidates. Equal exact distances are ordered by the unsigned bytes of the primary key's canonical identity — a STRING key's case fold, so `'a'` ranks before `'B'`, and every other key type's stored bytes; versions sharing that identity, whatever their spelling, use their encoded row bytes as a further tie-breaker. Derived and joined rows use encoded row bytes because they need not retain a unique source key. Identical rows remain separate multiset entries.

`NEAREST` accepts collection, derived-query, and CTE sources, ordinary and lateral joins, and temporal `AS OF`, `IN BETWEEN`, and `LAST PER` sources. Source selection, joins and the complete WHERE predicate run before ranking. Null vectors, including null-padded outer-join values, are skipped. Temporal clauses still require a source with history. Derived sources can project away the primary key while retaining the vector. ANN acceleration applies only to a current, non-temporal physical source; other relations use exact ranking. `EXPLAIN` exposes a `VectorTopK` stage above its source access plan. The exact top-k queue is bounded by `EngineOptions.MaxInMemoryStreamingState`; reduce `TOP` or row size, or raise that cap, if the retained result exceeds it.

Collection subtype vector fields are supported, including compatible nested leaves beneath differently shaped Objects. Repeated declarations must have structurally identical leaf schemas, including vector dimensions. Missing or NULL leaves are skipped; incompatible declarations are rejected even for empty input and by `EXPLAIN`. Derived queries and CTEs must retain the vector or its polymorphic envelope: projecting it away does not leave a searchable field.

Vector paths use the source's ordinary field scope. For example, `[r].[Body].[Embedding]` names a child of a stored `Body` field when that root is declared, even if its requested child is absent. `Body` acts as shorthand only when the selected source has no `Body` field. An explicit source alias takes precedence over another source's collection name, including its short name; `EXPLAIN` applies the same binding rules.

The engine always has an exact bounded top-k scan. Snapshot reads bypass ANN. For current APPROXIMATE reads, the core validates a provider result against the same collection incarnation and mutation state that the provider searched. If no compatible ANN index exists, validation loses a race, the candidate payload is malformed or too large, the provider fails or returns stale identities, or too few candidates remain after authoritative filtering, the engine runs the exact path. When ANN is used, it returns document identities only: the core re-reads the current documents, applies visibility and `WHERE`, recomputes exact distances, and reranks before returning rows. ANN can therefore affect recall and latency, but cannot expose stale, invisible, or predicate-violating rows. Memory exhaustion and modeled process death propagate instead of falling back. Independent terminal Vamana page-read failures remain visible together through search and generation retirement, including reads outliving cancelled callers. Provider query-vector preprocessing also cannot change the original query value, its exact fallback ranking, or the caller's bound vector parameter. Mutable BYTES filter hints are likewise isolated from the original WHERE predicate and bound parameters. Their combined snapshot payload and array overhead are capped by `EngineOptions.MaxInMemoryStreamingState`; hints that do not fit are omitted before copying, and a provider requiring them declines to the exact path. Duplicate candidates are collapsed with the primary key's actual equality rules (including case-insensitive STRING identity); too few unique candidates falls back before scanning candidate rows.

## Optional Vamana index

The `Mnemic.Engine.Indexing.Vector.Vamana` plugin adds managed SSD-aware Vamana graph acceleration:

```mql
CREATE VECTOR INDEX [documents_embedding]
ON [Search].[Documents] ([Embedding]) USING VAMANA
WHERE [Active] = TRUE
WITH (
  METRIC = COSINE,
  MAX_DEGREE = 64,
  BUILD_LIST = 100,
  ALPHA = 1.2,
  FILTER_FIELDS = ([TenantId])
);

DROP VECTOR INDEX [documents_embedding] ON [Search].[Documents];

LIST VECTOR INDEXES;
LIST VECTOR INDEXES ON [Search].[Documents];
```

Vector-index DDL commits independently and is rejected while a transaction or migration is active. Commit or roll back that scope before creating or dropping a vector index. Once DROP removes the catalog definition, later page/delta reclamation failure is reported as degraded health rather than making the completed DROP ambiguous; startup or a same-incarnation, same-name CREATE retries cleanup. Index identity follows the collection incarnation rather than its mutable name: after a collection rename, a newly created collection may reuse the old name and the same vector-index name without replacing the renamed collection's index. Pending mutations are isolated by incarnation too, so work on the recreated collection does not force exact fallback on the renamed original.

A vector index on a materialized view follows the view. `REFRESH MATERIALIZED VIEW`, `ALTER MATERIALIZED VIEW` and a source change that rebuilds the view each replace its backing; the replacing migration rebuilds every vector index of the view on the replacement, as `CREATE VECTOR INDEX` would build it there, and commits, rolls back and recovers it with the replacement, so `LIST VECTOR INDEXES` names the view, and `NEAREST APPROXIMATE` uses the index, before and after the migration and after a reopen or crash. A replacement whose rows no longer satisfy an index, such as an ALTER that stops selecting its vector field, is refused with `DEPENDENT_OBJECTS_EXIST` naming the index; drop the index first and declare it again afterwards. While a migration replaces a view, `CREATE VECTOR INDEX` and `DROP VECTOR INDEX` on it from another session fail with the transient `MIGRATION_CONFLICT`; retry them after the migration commits or rolls back.

The `WHERE` predicate is optional. A partial vector index is selected only when its canonical predicate exactly matches the query `WHERE` predicate. An unfiltered index may serve a filtered query because core filtering and exact underfill fallback remain authoritative.

A vector index holds what it read from the stored rows: its vectors, its `FILTER_FIELDS` labels and the fields its `WHERE` reads. A schema change that converts such a field's stored values, `ALTER FIELD ... SET TYPE` including a vector's dimension, or that makes it virtual computed, is refused with `DEPENDENT_OBJECTS_EXIST` naming the index, as `SET PRIMARY KEY` is while an index labels the documents by their key: drop the index, change the field in a migration, and create the index again.

`FILTER_FIELDS` is optional and accepts one through four stored scalar field paths. A query can use such an index only when its top-level conjunctive `WHERE` has equality for every listed field (literal or bound parameter). Traversal is label-aware and admits only matching labels; the core still rechecks the complete predicate. `FILTER_FIELDS` handles varying equality values such as tenants or categories. It is independent of the index `WHERE`, which fixes membership to one canonical predicate. A `PINNED REFERENCES` filter field labels the referenced identifier; the pinned revision does not participate, so equality on the identifier can use the index.

`STRING` equality labels use the canonical parameter text for native scalars: round-trip dates and times, base64 byte arrays, and the wire value's invariant numeric text. Case-insensitive STRING comparison still applies. Empty byte-array parameters stringify as present empty text, distinct from `NULL`.

Numeric equality labels are available only when the operand has an exact key at the field's storage width. Integer text accepts integral decimal and exponent forms such as `'1.0'` and `'100e-2'`. Floating and Decimal text retain the ordinary numeric comparison domain, and bound numerics retain their source kind. Lossy conversions decline label filtering and leave the original predicate to exact execution.

`BOOL` text labels accept exactly `'true'` / `'false'` (case-insensitive) and `'1'` / `'0'`. Numeric equality operands receive a Bool label only when they equal exactly zero or one; other numeric values cannot supply a truthy label. Ordinary field parameter validation still applies.

`BYTES` equality labels accept base64 text and native byte-array parameters and compare the decoded payload. An empty payload, including empty base64 text, remains distinct from `NULL`.

Temporal equality labels follow the ordinary scalar conversion rules. An offset-less timestamp means UTC for `DATETIME`; an explicit offset compares by UTC instant. `DATEONLY` and `TIMEONLY` use the timestamp's written date or clock components, including offset-bearing timestamp text. Native date-only and time-only parameters retain their own logical types rather than supplying an implicit `DATETIME` anchor.

Options and defaults:

| Option | Default | Valid values | Effect |
|--------|---------|--------------|--------|
| `METRIC` | `COSINE` | `COSINE`, `L2`, `DOT` | Must match the query metric. |
| `MAX_DEGREE` | `64` | 2–512 | Maximum outgoing graph degree; higher values use more memory and build work. |
| `BUILD_LIST` | `100` | `MAX_DEGREE`–1,000,000 | Candidate-list width during construction; higher values normally improve graph quality. |
| `ALPHA` | `1.2` | 1–4 | Vamana pruning factor. |
| `FILTER_FIELDS` | none | 1–4 stored scalar paths | Builds label-aware edges and query entry postings for equality filters. |

Each option may appear at most once. The same validation runs at execution, so a programmatically constructed AST cannot bypass these ranges, the finite `ALPHA` requirement, or case-insensitive `FILTER_FIELDS` uniqueness.

`LIST VECTOR INDEXES` returns one row per vector index of the database, or with `ON` of one collection, ordered by collection and name. The definitions live in the plugin's own catalog, which `FROM` cannot read. It needs only read permission, changes nothing, and runs inside a transaction or migration; `ON` a collection the database does not have reports `COLLECTION_NOT_FOUND`.

| Column | Type | Meaning |
|--------|------|---------|
| `Name` | `STRING` | The index's name. |
| `CollectionName` | `STRING` | The collection, by the name it has now; an index follows its collection through a rename, and a materialized view through every replacement of its backing. |
| `FieldPath` | `STRING` | The indexed vector field, as a dotted path. |
| `Metric` | `STRING` | `COSINE`, `L2` or `DOT`. |
| `Dimensions` | `INT32` | The vector field's dimensions. |
| `MaxDegree` | `INT32` | `MAX_DEGREE`. |
| `BuildList` | `INT32` | `BUILD_LIST`. |
| `Alpha` | `FLOAT64` | `ALPHA`, as written. |
| `FilterFields` | `STRING` nullable | `FILTER_FIELDS`, comma-joined dotted paths; NULL for none. |
| `FilterText` | `STRING` nullable | The partial index's `WHERE` condition; NULL for a full index. |

The compact identity/four-bit-code directory stays resident while checksummed adjacency pages are read asynchronously through a page cache that every vector index of the database shares. Its budget is the plugin's `PageCacheBytes`, by default an eighth of a buffer-pool capacity the engine was given (`BufferPoolBytes`) and never below 4 MiB, or 32 MiB when that is 0 and the pool is automatic; an embedding application passes it as `VamanaPluginOptions.PageCacheBytes`, and the standalone server takes it from `MNEMIC_VAMANA_PAGE_CACHE_BYTES` for every database it opens ([configuration](../configuration.md#standalone-server-environment)). Committed writes append durable deltas and update a bounded in-memory overlay, which searches read beside the base generation. Consolidating the overlay into a new base generation is due once it holds a sixteenth of the base generation's points, at most half the points the index's 64 MiB overlay budget holds (a point counting its compact code, its prepared query, a neighbour list of 1.3 × `MAX_DEGREE` entries and fixed overhead), and never fewer than 1,024 deltas. The plugin's background worker then rebuilds the base from the committed rows while searches keep using the current base and overlay, and publishes it with an overlay of the deltas committed meanwhile; a commit that would grow the overlay to twice that threshold rebuilds the base on that commit instead. A clean reopen loads the compact directory, replays the overlay's deltas, and consolidates an overlay already past its threshold in the background. After an unclean shutdown, the first MQL vector query rebuilds from authoritative rows before ANN is reused. Corrupt or incomplete directory, page, or delta state likewise recovers or leaves exact search available. A different provider can implement the same candidate-identity seam without changing MQL.

Mutation capture is bounded. If its high safety limit is exceeded, the current engine is quarantined with `PLUGIN_RECOVERY_REQUIRED`, its clean-shutdown marker is withheld, and reopen rebuilds the vector index from authoritative rows. This fails closed instead of allowing a clean reopen to trust a graph that may have missed a committed source mutation.

The same recovery path is used when post-commit reconciliation, delta persistence, or a rebuild on a commit fails after the authoritative row has committed. A background consolidation that fails before it publishes instead leaves the overlay and its durable deltas authoritative, reports a degraded `VAMANA_CONSOLIDATION_DEFERRED` fault, and runs again when the next commit leaves the overlay due.



Computed VECTOR fields use the same exact dimensions and finite-component rules as ordinary vectors. `COSINE_DISTANCE`, `L2_DISTANCE`, and `DOT_PRODUCT` infer FLOAT64 results in schema-owned expressions. Vamana index vectors, filter labels, and membership predicates must use stored inputs; virtual computed fields (including children of a virtual computed container) require STORED before indexing.

Scalar WHERE expressions and unary NOT filter the source before NEAREST admits rows to its bounded top-k state. The source scan has no paging of its own; removing a nearer row cannot prevent the next qualifying vector from filling TOP. Native conjuncts remain eligible for ordinary source access paths.

Persistent vector-index membership cannot call runtime-replaceable functions, including built-in names that an embedded host can replace. Define the expression as a STORED computed field and filter that field instead. Stored computations bind immutable built-in implementations, so membership changes only when the source row changes and index maintenance can reconcile it.

### Vector defaults

`ALTER COLLECTION ... ADD FIELD [V] VECTOR(n) DEFAULT VECTOR[...]` backfills existing rows with the literal. `ALTER FIELD [V] SET DEFAULT VECTOR[...]` sets a default for the fields an inserted document, or an object a later write assigns, leaves out, without rewriting existing rows; `DROP DEFAULT` removes it. These forms also work through `ALTER TYPE`. A vector default must have exactly the declared number of finite FLOAT32 components. DDL rejects mismatches before mutation; executor startup validates persisted defaults again. Backfill and metadata changes roll back with the migration.

## Detailed syntax and execution


<a id="syntax-vector-top-k-search"></a>

### Vector top-k search

```mql
FROM [Search].[Documents]
WHERE [Active] = TRUE
NEAREST APPROXIMATE [Embedding] TO @query USING COSINE TOP 10
WITH (SEARCH_LIST = 200)
LIMIT 10
SELECT [Id];
```

`NEAREST` accepts a `VECTOR[...]` literal or `float[]` parameter and one of `COSINE`, `L2`, or `DOT`. `TOP n` or `TOP @k` is mandatory; `SKIP` and `LIMIT` are optional final paging. `SEARCH_LIST` defaults to 100 and is from 1 through 1,000,000; increasing it normally trades latency for ANN recall. Vamana treats it as the fixed candidate, expansion, and requested per-query distinct adjacency-page budget. The final beam plus expanded set retains at most `2 * SEARCH_LIST` distinct graph identities. It declines acceleration when the configured page cache cannot bound that working set, when `SEARCH_LIST` exceeds the managed reader's 65,536-entry beam ceiling, or when its bounded candidate materialization cannot cover `TOP`. The language still accepts values through 1,000,000 for exact execution and other providers. The core always has an exact bounded top-k path. Snapshot reads bypass ANN. For current reads, provider candidates are accepted only while their collection-incarnation and mutation validation token remains current. The provider returns primary-key candidates only; the core re-reads current rows, applies `WHERE`, recomputes exact distances, and reranks. Missing, corrupt, oversized, stale, underfilled, raced, or failed ANN state falls back to exact search. Cosine rejects a zero-norm query and omits zero-norm stored vectors. Candidate identities are deduplicated with the primary key's authoritative equality, including case-insensitive STRING identity; unique underfill falls back before candidate-row scanning. Ranking accepts collection, derived-query and CTE sources, ordinary and lateral joins, and temporal `AS OF`, `IN BETWEEN` and `LAST PER` sources. Source selection, joins and the complete WHERE run first. Ranking retains `TOP` rows, with zero default SKIP; grouping, HAVING, windows, projection, DISTINCT, explicit ordering and final paging then consume that neighborhood. ORDER BY rearranges that selected neighborhood; the ranked input arrives in distance order. Both pagination values accept parameters and their sum must fit a positive INT32. Aggregates summarize only the neighborhood; DISTINCT can leave fewer than LIMIT rows. DISTINCT after a correlated WHERE keeps the layout it has without that WHERE (a plain DISTINCT's leaf key columns, a computed, aggregate or grouped DISTINCT's completed projection) once WHERE and ranking have completed. Its exact count and final page describe those distinct neighborhood values. Null vectors, including null-padded join values, are skipped. Derived results need no primary key. AS OF remains the inherited instant for physical JOIN inners when ranking a composed relation. Current non-temporal physical sources can use ANN; other relations use exact ranking. Equal distances use compact primary-key bytes on physical sources, then encoded row bytes for multiple versions of one identity. Derived/joined rows use encoded row bytes; identical rows retain multiplicity. EXPLAIN exposes VectorTopK above the source plan. Retained exact top-k rows are bounded by `MaxInMemoryStreamingState`. If `TOP > SEARCH_LIST`, Vamana declines before adjacency-page traversal and the exact path runs because the ANN candidate budget cannot fill the neighbourhood.

<a id="syntax-vector-defaults"></a>

### Vector defaults

`ALTER COLLECTION ... ADD FIELD [V] VECTOR(n) DEFAULT VECTOR[...]` backfills existing rows with the literal. `ALTER FIELD [V] SET DEFAULT VECTOR[...]` sets a default for the fields an inserted document, or an object a later write assigns, leaves out, without rewriting existing rows; `DROP DEFAULT` removes it. These forms also work through `ALTER TYPE`. A vector default must have exactly the declared number of finite FLOAT32 components. DDL rejects mismatches before mutation; executor startup validates persisted defaults again. Backfill and metadata changes roll back with the migration.

## Usage pitfalls


<a id="pitfall-vector-dimensions-and-finite-values-are-strict"></a>

### Vector dimensions and finite values are strict

`VECTOR(n)` fixes the component count in the schema. A `VECTOR[...]` literal or bound `float[]` written to the field (INSERT, UPDATE, upsert) must have exactly that count and every component must be finite, else the write is `INVALID_VALUE`, as a wrong-shaped DEFAULT or NEAREST query vector is; unlike scalar `FLOAT32` / `FLOAT64`, vectors reject NaN and infinity. The two operands of a distance function must also have equal dimensions. A null operand returns null, but cosine rejects any present zero-norm operand.

<a id="pitfall-nearest-needs-top-ann-is-never-authoritative"></a>
<a id="pitfall-nearest-needs-limit-ann-is-never-authoritative"></a>

### `NEAREST` needs `TOP`; ANN is never authoritative

Place `NEAREST` after `WHERE`, use `COSINE`, `L2`, or `DOT`, and always give the neighbourhood size as `TOP k`. `SKIP` and `LIMIT` are optional final paging over those `TOP` rows. ORDER BY sorts that nearest neighborhood; GROUP BY and aggregates summarize only its rows. Derived sources, CTEs, joins and historical sources all work; they use exact ranking when no current non-temporal collection can supply ANN candidates. A derived projection may omit its primary key. Equal-score joined rows use encoded-row tie-breaking and retain duplicates. `SEARCH_LIST` is an ANN exploration width, not the number of rows returned; Vamana treats it as the fixed candidate, expansion, and requested distinct-page I/O budget. Its final beam plus expanded set retains at most twice that many graph identities, and exact fallback applies when the configured page cache cannot bound the requested working set. Managed Vamana also declines widths above 65,536 before allocating its beam, although MQL accepts values through 1,000,000 for exact search and other providers. When `TOP` is larger than `SEARCH_LIST`, Vamana declines before page I/O and the exact path runs; the hard ANN candidate budget cannot fill that result. A compatible provider can improve latency at approximate recall, but snapshot reads bypass ANN and the core validates current-read candidates against the searched collection incarnation and mutation state. The core re-reads candidates, applies `WHERE`, and exactly reranks them. Provider failure, malformed or oversized candidates, a validation race, stale candidates, or too few usable candidates triggers an exact scan. Candidate identities are deduplicated with authoritative primary-key equality (including case-insensitive STRING identity), so duplicates cannot postpone underfill until after candidate-row scanning. A partial Vamana index requires a canonically identical query predicate; use an unfiltered index when query filters vary. For varying tenant/category equality, declare `FILTER_FIELDS = ([TenantId], [Category])`; every listed path must have a top-level conjunctive equality in the query, and at most four paths are allowed. If the provider's high mutation-capture safety bound is exceeded, the engine is quarantined with `PLUGIN_RECOVERY_REQUIRED`; reopen reconstructs the graph from authoritative rows. Post-commit ANN maintenance failures follow the same path, except a background consolidation that fails before it publishes, which leaves the update overlay authoritative, reports a degraded `VAMANA_CONSOLIDATION_DEFERRED` fault, and runs again when the next commit leaves the overlay due. Exact top-k retention is bounded by `MaxInMemoryStreamingState`; a large `TOP` neighborhood or wide source rows can return resource pressure. Use a derived source to project away unneeded large fields before ranking when appropriate.

`CREATE VECTOR INDEX` and `DROP VECTOR INDEX` publish plugin state independently; naming a collection the database does not hold is `COLLECTION_NOT_FOUND`, as `LIST VECTOR INDEXES ON` reports it. They are rejected while a transaction or migration is active; commit or roll back that scope first. Their execution boundary repeats option validation for direct programmatic ASTs, including finite alpha, build-list/degree ordering, and case-insensitive filter-field uniqueness.

<a id="pitfall-vector-defaults-retain-the-declared-dimension"></a>

### Vector defaults retain the declared dimension

Use VECTOR[...] for VECTOR(n) defaults; scalar or wrong-width values are rejected. ALTER ADD DEFAULT backfills existing rows, while SET DEFAULT applies to the fields later writes leave out of the documents and objects they build whole ([defaults](dml.md#defaults)). Named-type propagation follows the same rules.
