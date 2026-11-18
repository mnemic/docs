# `ANALYZE` and the cost-based planner

> **In this section** — the `ANALYZE` statement, the `[Mnemic].[Statistics]` catalog, the `EXPLAIN` `EstimatedRows` column, and how the planner uses stats to bypass an index when a sequential scan is cheaper.

The Mnemic planner uses fresh statistics to compare eligible index paths and sequential scans. `ANALYZE` supplies selectivity and physical-width evidence: a broad index path can cost more than a sequential scan, while intersecting selective indexes can reduce heap reads. Missing statistics preserve the deterministic eligibility fallback.

## Running `ANALYZE`

```sql
ANALYZE [App].[Orders];
```

Scans the collection once and writes one stats row per declared or indexed population — the primary key, every secondary index (single-column, compound, partial), and full-collection scalar rows for compound components — to `[Mnemic].[Statistics]`. Each row carries the distinct-value count (a full scan is exact until the bounded distinct-key dictionary saturates; an incremental row uses the merged HyperLogLog), the null count, the total non-null count, the encoded min and max, the collection's document count at analyze time (used for staleness detection), plus these estimators:

- **Equi-depth histogram** (100 buckets, byte-encoded) for range selectivity.
- **MCV list** (top 20 values + frequencies) for heavy-hitter equality selectivity.
- **HyperLogLog sketch** for bounded NDV estimation and incremental merge. The sketch is accumulated for every non-null observation; the configurable non-null row threshold (100 K by default) controls when it is persisted, independently of when the exact distinct-key dictionary saturates.
- **Compound stats** row per scalar compound secondary index or declared field group, keyed by a collision-free length-framed ordered path list, carrying joint NDV via HLL and a `CompoundCorrelationRatio` (product of single-column NDVs ÷ joint NDV; values > 1 mean correlated columns). Joint keys use the physical index's escaped, terminated component encoding, so variable-width binary values and embedded zero bytes remain distinct. The ratio stays in the finite `[1, FLOAT64_MAX]` domain: approximate NDV crossings clamp to 1 and mathematically larger results saturate instead of making `ANALYZE` fail on a legal wide compound index.
- **Partial-index stats** row per partial secondary index, scoped to rows satisfying the index's filter and tagged with the canonical filter text. That text participates exactly in row identity: identifier case remains insensitive, but literal case is preserved because Base64 `BYTES` literals that differ only by case decode to different values. Compound correlation always uses the physical index population: every key component is present. For a partial compound index, its component NDVs and joint NDV additionally use only filter-satisfying rows; missing or excluded rows cannot inflate its marginals.

Scalar equality and range estimates for an eligible partial index prefer its filter-scoped statistics. Full-collection statistics are a fallback only when the partial row is absent. A stale partial row leaves the estimate unknown, even if full-collection statistics are fresh; `EXPLAIN` reports that stale evidence.

For an eligible partial-index `ORDER BY` scan, the estimate counts the selected population's non-null keys before residual filtering. The same missing-row fallback and freshness rules apply. Ordinary full-index ordered scans need no `ANALYZE`: without a predicate they report a standard collection's `DocumentCount`, and with one, or over a temporal collection, `null`.

Scalar partial-index key-width estimates also prefer the filter-scoped row. If that row exists but lacks a width measurement, costing uses the normal width default; only an absent partial row permits the full-column width fallback. INCLUDE payload widths continue to use full-column measurements.

Array indexes produce separate **component posting statistics** with `IsArrayComponent = TRUE` and `IsCompound = FALSE`. Each path counts its distinct typed values once per source row, plus its empty/null coverage posting. Thus `TotalCount` counts postings and can exceed `CollectionDocCountAtAnalyze`; `NullCount` is zero because coverage is an encoded key. MCV frequencies count row membership for each value, not repeated occurrences within an array. Scalar paths in a compound array index also keep ordinary scalar statistics. Statistics for `PINNED REFERENCES` fields compare the referenced id in ordinary scalar, compound tuple, and array-component populations: different pinned revisions of one id share one distinct value. Empty String and Bytes ids remain values; a NULL array-index component retains its coverage posting. Component populations have separate identities from scalar rows and from each other's partial filters. They never claim a joint Cartesian distribution or same-element correlation. They are always rebuilt by a full scan.

Compound-array planning compares individual probes, cost-reducing subsets, and their full intersection against eligible scalar paths and a full scan. Posting counts and physical leaf widths price index work; candidate row estimates price heap validation. Intersections and non-point unions also include estimated spill I/O under `MaxInMemoryStreamingState`; a single exact probe streams without sorting. Different component paths use independent selectivity estimates, while repeated predicates on one path and partial intersections use the smallest marginal. Missing or stale component statistics preserve deterministic fallback. `EXPLAIN` reports the selected `CompoundArrayMembership` or `CompoundArrayIntersection` and its estimated candidate rows; all row predicates remain authoritative.

An explicit `ANALYZE` always performs a **full rescan** — an authoritative, from-scratch resample of every estimator (min/max, histogram, MCV, NDV). That is the way to force a complete refresh; the background auto-analyze may instead refresh incrementally (see [Auto-analyze](#auto-analyze)), so run `ANALYZE` explicitly when you need an authoritative planner snapshot at a fresh scan cut.

On collections with subtypes, scalar statistics read base fields and array statistics traverse the active extension. Partial filters use the complete row and subtype binding. Rows whose subtype omits an indexed array still contribute coverage postings when they satisfy the filter.

Concurrent explicit or background requests for the same collection are serialized. This prevents an older scan from overwriting a newer watermark and keeps first-time statistics-catalog creation deadlock-free. Requests for different collections remain concurrent. If the collection is dropped, renamed, or replaced while a request waits, that stale request fails instead of publishing statistics under the reused name.

Waiting for another analysis observes cancellation and the host's configured lock-acquisition deadline. Expiry returns `Transient / LOCK_TIMEOUT` before the waiting request changes statistics.

Once a scan captures its gap-free commit cut, commits to that collection wait until the scan has consumed the matching direct or temporal image. This keeps the persisted distributions and watermark on one snapshot. The source lease is released before the statistics rows are written, so analyses of different collections remain independent.

`ANALYZE` requires `MqlPermission.Ddl` — same tier as `BACKUP` / `DEFRAGMENT` / `SHRINK`. It cannot run inside a migration or an explicit transaction.

## What the planner does with stats

The legacy equality-selectivity gate fires when:

1. The query has exactly one filter predicate.
2. That predicate is `[Field] = value` on a column with stats.
3. The estimated match-set (`TotalCount / DistinctCount`) exceeds `EngineOptions.IndexSelectivityThreshold` × `TotalCount`.

When all three hold, the planner skips the index and runs a sequential scan. Otherwise it uses the index.

`EngineOptions.IndexSelectivityThreshold` defaults to `0.3` (30%). Raise it toward `1.0` to keep using the index more aggressively; lower it to bypass sooner.

The global cost chooser also compares compound equality, secondary equality, and secondary range candidates against a page-packed sequential scan. Its `FullScan` choice is authoritative when every modeled candidate has a fresh estimate and the query has no `ORDER BY` benefit outside the current cost function. This applies to multi-predicate queries: a residual predicate no longer forces an otherwise unselective index seek. Candidates with unknown estimates remain eligible, and a primary-key range remains eligible because the global chooser does not enumerate that access path.

Cross-field equality OR uses the same page-cost crossover. When every indexed OR input has fresh statistics, a union expected to touch most heap pages runs as `FullScan`; a selective union remains `BitmapOr`. If any input estimate is unknown, the planner keeps the rule-based bitmap merge. `EXPLAIN` consumes that same chooser verdict: it reports the cost-selected `FullScan` rather than advertising the rejected equality, range, or bitmap path. Primary-key / temporal point probes, array-membership probes, and the still-unmodeled primary-key range keep their runtime precedence in the static plan.

Conjunctive predicates can select `BitmapAnd`: scalar equality, range, and compound-prefix inputs with fresh estimates are intersected before one page-ordered heap pass. Partial indexes require an implied filter; overlapping predicate fields are not counted twice. The estimate multiplies independent selectivities and is capped by the smallest input. Correlation can make actual output larger or smaller. Individual indexes and a full scan remain cost competitors. Temporal/snapshot queries retain their version-aware paths.

`BitmapAnd` preparation has a retained bitmap-capacity allowance from `MaxInMemoryStreamingState`; exceeding it falls back to the heap before rows escape. All predicates and visibility are rechecked. `EXPLAIN` lists each selected input; page order does not satisfy `ORDER BY`.

`CompoundSkipScan` uses fresh scalar component distributions to price repeated searches for selective non-leading fields. Missing or costly prefix evidence keeps other paths eligible. EXPLAIN includes the exact compound field list, estimated searches, and execution allowance. Search or bitmap exhaustion falls back before output; see [compound skip scans](indexes.md#compound-skip-scans).

## `EXPLAIN` and `EstimatedRows`

`EXPLAIN` adds an `EstimatedRows` column that's populated from the catalog when stats exist for the chosen index column. The estimate is **value-aware**: it consults the MCV list for the probe value first (exact heavy-hitter count), then the histogram, then the uniform `TotalCount / NDV` fallback — the same estimator the runtime cost gate uses, so the number `EXPLAIN` shows is the number the planner acted on. For a skewed column (100 events where the value `0` appears 91 times across 10 distinct values), probing the heavy hitter reports its exact MCV count, not the uniform guess (`100 / 10 = 10`):

```sql
EXPLAIN FROM [App].[Events] WHERE [Bucket] = 0;
```

| Operation | Strategy | Index | EstimatedRows | Notes |
|---|---|---|---|---|
| Select | FullScan | _(null)_ | 91 | `Cost-based: sequential scan is cheaper than every modeled index path.` |

The strategy reflects physical cost as well as selectivity. On this small collection, fetching 91% of the heap through an index costs more than one sequential pass, but `EstimatedRows` remains the value-aware filtered-output estimate rather than reverting to the 100 rows read. On a physically larger collection a rare probe can remain `SecondaryEquality`.

Histogram counts scale from the sample to the column's non-null population. When an equality or range estimate covers the whole sample, scaling returns that exact INT64 population count without rounding it through FLOAT64. Partial-sample estimates retain the histogram's interpolation policy.

Full compound-key equality uses the frequency of the requested tuple, with schema normalization and escaped, terminated components matching ANALYZE. Partial index statistics apply only when the query implies that index's filter. For a non-temporal full scan, scalar equality conjunctions use complete declared or full-index joint populations, then multiply value-aware scalar selectivities for remaining fields. Repeated equivalent equalities count once; conflicting equalities on one field estimate zero. Missing or stale evidence preserves the scan fallback. A sole `!=` predicate subtracts the requested value's estimated frequency from the non-null population. These statistics guide cost and explain estimated output; predicate evaluation still determines the actual result.

The older selectivity threshold is an additional conservative gate: when it, rather than the page-cost crossover, rejects an equality seek, the `FullScan` note includes the estimate, total, configured threshold, and `index bypassed`.

When several equality predicates each target an indexed column, the global chooser compares the same value-aware cardinalities and physical page costs used by runtime dispatch. `EXPLAIN.Index` reports the exact selected field, including when equal row estimates are separated by different physical leaf widths. `Notes` retains `Cost-ranked N candidates; picked most-selective.` as a candidate-count summary when no chooser hint exists, or `Kept N candidates in written order; a candidate has no usable estimate.` when one of them has no estimate: an index without statistics, for example one created after the last `ANALYZE`, is incomparable rather than expensive, so neither a costed index nor helper ranking suppresses it. Production plans instead say `picked chooser-selected path.` because full physical cost, not cardinality alone, owns the choice. The exact `Index` value is authoritative for the path that executes.

Range estimates use the same representation as the runtime scan: literal bounds are parsed by field type and encoded into the index's ordered key space before they are compared with histogram buckets. Paired lower and upper bounds are combined, so a numeric `WHERE [Score] > 25 AND [Score] < 75` estimate reflects that interval rather than comparing the UTF-8 text `25` / `75` with binary index keys. Compound partial-prefix estimates likewise encode the leading equality value before probing its MCV row, so skewed hot/rare values retain their value-aware counts. When a nonzero leading estimate is narrowed by a nonzero trailing-range fraction, the result is ceil-rounded to at least one row; a reported zero is reserved for a genuinely zero component estimate.

The same exact-path rule applies when several secondary ranges or compound indexes belong to the chosen strategy family. EXPLAIN carries the chooser's selected field or ordered compound field list through to the plan row; predicate order and index-registration order cannot replace it with a different same-family candidate.

`EstimatedRows` describes the rows a plan produces, and it is a number only where evidence defends one. Several strategies report a value with or without stats — a point PK equality is `1` (`PK IS NULL` is exactly `0`), `TemporalIndex` is always `1`, `INSERT` is `1`, aggregations report the distinct-group count, and `LIMIT N` caps a known value at N (an unknown one stays `null`). A `FullScan` or `IndexOrderScan` without a WHERE predicate reports `DocumentCount` for a standard collection, the same rule a join's `Scan` leaf follows; over a temporal collection, whose `DocumentCount` counts every retained version rather than the current versions the scan reads, it reports `null`, as every scan under snapshot isolation does. A `FullScan` with a predicate reports its fresh-statistics estimate of the surviving rows (IS NULL / IS NOT NULL counts, `!=`, or the conjunction estimate above) and `null` when none applies; an `IndexOrderScan` with a predicate reports `null`, except that a partial index reports its population. A cost-selected `FullScan` instead preserves the rejected candidate's filtered-output estimate—the scan still reads the whole heap, but `EstimatedRows` describes rows produced, not pages read. A `BitmapOr` estimate sums the same value-aware per-input estimates, discounting proven repeated cached probes, and caps the union at `DocumentCount`; it can never report more distinct rows than the collection contains. The strategies that still report `null` without stats are the ones where the estimate genuinely depends on column statistics:

- `SecondaryEquality` (uses MCV / histogram / NDV)
- `SecondaryRange` (uses histogram bucket interpolation)
- `SecondaryInList` (sums the members' MCV / histogram / NDV equality estimates over distinct keys)
- `PkRange` (uses the primary-key histogram)
- `TemporalRange` (uses the retained-history primary-key distribution)
- `CompoundEquality` (uses the compound stats row)
- `CompoundRange` (uses a value-aware leading-prefix estimate and optional trailing-range statistics)
- `BitmapOr` when any secondary-index input lacks fresh statistics
- `StreamingLeftDeep` and `MaterializingBushyHashJoin` inner equality joins (use the join keys' distinct counts; see [the physical join tree](queries.md#explain--query-plan-introspection))

For those, `EstimatedRows = NULL` means no `ANALYZE` row exists, or the existing row is stale. When the row exists but is stale the EXPLAIN `Notes` column carries `Stats stale; re-ANALYZE for tighter EstimatedRows.` — distinguishes "never analyzed" from "analyzed but out of date" so operators know which action to take.

## Inspecting the catalog

`[Mnemic].[Statistics]` is queryable from userland MQL — operators can audit what the planner sees:

```sql
FROM [Mnemic].[Statistics] WHERE [CollectionName] = '[App].[Orders]';
```

Columns: `Id`, `CollectionId` (the collection's stable identity, which a rename keeps), `CollectionName` (rendered from that identity under the current name), `FieldPath`, `TotalCount`, `NullCount`, `DistinctCount`, `MinValue`, `MaxValue`, `LastAnalyzedAt`, `CollectionDocCountAtAnalyze`, the required exact-v1 state columns (`IsCompound`, `IsArrayComponent`, `CompoundCorrelationRatio`, `IsIncremental`), the nullable extended-statistics columns (`HllSketch`, `Histogram`, `Mcv`, `PartialFilterText`, `LastAnalyzedWriteCsn`), and the cost-model page-packing inputs `AvgRowWidthBytes` (collection-level average document width) + `AvgKeyWidthBytes` (per-column average encoded-key width — the planner's leaf-fan-out input). For compound rows this is the exact persisted compound key width, including component escaping and terminators.

`CompoundCorrelationRatio` is finite and at least one on compound rows; single-column rows carry exactly `1.0`. A durable row outside those domains is reported as `STATISTICS_CATALOG_CORRUPT`, not treated as absent planner input. The statistics catalog is read only in its version `1` shape, and a row of any other shape is refused ([compatibility](https://mnemic.io/docs/compatibility/)).

`MinValue` / `MaxValue` are stored in the same byte encoding the secondary index uses for keys, so range comparisons against query parameters are byte-wise without re-decoding.

DDL on `[Mnemic].*` is rejected — the catalog is rebuilt only by `ANALYZE`.

## Stale stats

Each stats row carries `CollectionDocCountAtAnalyze`. When the collection's current document count drifts more than `EngineOptions.StatsStaleWriteRatio` (default `0.5` = 50%) past that watermark, the planner ignores the row, falls back to the rule-based path, and annotates the EXPLAIN `Notes` column with `Stats stale; re-ANALYZE for tighter EstimatedRows.`

## Auto-analyze

A background loop (`EngineOptions.AutoAnalyzeInterval`, default 15 minutes) refreshes existing statistics when their committed-write watermark trails the collection or row-count drift requires reconciliation. Update-only churn is visible even when the collection size stays constant.

Committed authored row changes can wake that loop early. The per-collection threshold is `AutoAnalyzeWriteThreshold + AutoAnalyzeWriteRatio * size`, rounded up to at least one and saturated at Int64.MaxValue. Defaults are 50 and 0.1; size is sampled after analysis or reopen. INSERT, UPDATE, DELETE, and temporal row mutations count after commit; rolled-back attempts and physical relocation do not count. Wakes coalesce, and an early pass considers only collections that reached their threshold. The periodic deadline still services quieter collections. These are scheduling counters, independent of `StatsStaleWriteRatio` and the persisted write watermark that proves what an analysis has covered.

Auto-analyze may refresh **incrementally** for a non-temporal collection whose single-column rows share one usable `LastAnalyzedWriteCsn` cut and HLL. It scans live rows whose latest logical-write CSN falls in the next gap-free interval and merges their NDV contribution rather than re-reading the whole collection. Deleted rows do not appear in that delta, so prior observations remain in the additive state until reconciliation. `IsIncremental = true` identifies the resulting additive row. Drift remains measured against the last full pass, so enough insert, update, or delete churn forces a reconciling full rescan. Compound, partial, temporal, missing/mismatched-watermark, and explicit `ANALYZE` passes are always full. The distinction does not change query semantics; between full passes, min/max/histogram/MCV remain the last authoritative distributions.

The loop only refreshes existing rows. Collections that were never analyzed stay un-analyzed until an explicit `ANALYZE` — first-time analysis remains operator-driven.

Set `AutoAnalyzeInterval = null` to disable the loop entirely (operators must run `ANALYZE` manually). Set `AutoAnalyzeWriteThreshold = null` to disable only early wakes. Write counters are process-local and reset on reopen; persisted statistics and the periodic freshness check remain authoritative. Invalid thresholds, ratios, or timer intervals are rejected before opening storage. Idle timer waits round up to one millisecond; short cadences still yield and respect cancellation.

## Declaring column-group statistics

```sql
BEGIN MIGRATION 1;
CREATE STATISTICS [App].[Geography] ON ([City], [Zip]) FROM [App].[Places];
COMMIT MIGRATION;
ANALYZE [App].[Places];
FROM [Mnemic].[StatisticsDefinitions];
```

`CREATE STATISTICS [Ns].[Name] ON ([Field], ...) FROM [Ns].[Collection]` declares an ordered group of 2–1024 distinct stored scalar field paths without creating an index. Nested base-object scalar paths and stored computed fields are allowed; array elements, composite values, virtual computed sources, and polymorphic extension fields do not form one stored scalar population. Each collection permits up to 1024 declarations. Names are qualified, unique across the database ignoring case, and cannot use the reserved Mnemic namespace. Fewer than two paths reports `INVALID_STATEMENT` and more than 1024 `LIMIT_EXCEEDED`, whether the statement is written or built.

Both `CREATE STATISTICS` and `DROP STATISTICS [Ns].[Name]` require DDL permission and an explicit migration. Creation records the definition; `ANALYZE`, after commit, collects the distribution. `[Mnemic].[StatisticsDefinitions]` exposes `Name`, `CollectionName`, and the ordered `FieldPaths` array. Migration owners see their private definitions; other sessions see committed definitions.

Analysis collects full-collection scalar marginals and a joint distribution for rows where every group component is non-null. The joint distribution shares the compound statistics identity and canonical tuple encoding with a full compound index on the same ordered fields. Duplicate declarations reuse that population; a partial index remains a separate filtered population.

Fresh joint frequencies correct equality selectivity when bitmap intersection combines complete, disjoint input domains covered by a declared group. The full-scan `EXPLAIN` conjunction estimate uses the same value-aware joint estimator. Overlapping groups are considered by descending field count and then name; each input or predicate contributes once. Uncovered inputs retain independent estimates. Missing or stale groups retain the existing fallback. Statistics affect estimates and plan choice; normal predicates determine rows.

Field and parent-object renames update dependent paths; removing a group field removes its declaration. `ALTER COLLECTION` and propagated `ALTER TYPE` changes invalidate the affected source's old statistics transactionally; run `ANALYZE` again after commit. Collection renames retain definitions and move their source identity; namespace renames also move names declared in that namespace. Abort restores the prior definitions and statistics. Dropping the last declaration removes its joint population unless a full compound index still owns it. Dropping a collection removes its groups. A declaration on a [materialized view](materialized-views.md) moves to each generation a REFRESH, `ALTER MATERIALIZED VIEW` or source rebuild publishes, unless that generation no longer stores one of its fields as a scalar.

## Tunables

```csharp
new EngineOptions {
    FilePath = path,
    IndexSelectivityThreshold = 0.3,        // 30% — bypass when more than this fraction matches
    StatsStaleWriteRatio = 0.5,             // 50% — stats stale when doc count drifts more
    AutoAnalyzeInterval = TimeSpan.FromMinutes(15), // null disables all auto-analyze
    AutoAnalyzeWriteThreshold = 50,        // null preserves only periodic refresh
    AutoAnalyzeWriteRatio = 0.1            // non-negative fraction added to threshold
}
```

## Lifecycle

Statistics are keyed by stable collection ID. Dropping a collection purges its `[Mnemic].[Statistics]` rows (so a later re-create under the same name never inherits stale, possibly wrong-encoding estimates). Renaming a collection or namespace leaves statistics payloads, deterministic row IDs, and cache keys unchanged. `CollectionName` is rendered from the caller's catalog snapshot. DROP deletes the populations in its migration transaction; abort restores them. Reopen resolves each persisted owner by ID, so reusing an old name cannot redirect statistics to the new collection. A materialized view's replaced generation keeps its rows until its drop purges them, and `[Mnemic].[Statistics]` never lists them; the generation that replaced it starts unanalyzed.

## See also

- [System collections](system-collections.md) — the `[Mnemic].[Statistics]` row layout.
- [Maintenance](maintenance.md) — `BACKUP` / `DEFRAGMENT` / `SHRINK` — sibling Ddl-tier maintenance operations.

## Detailed syntax and execution

Temporal collections contribute every retained version to their statistics. `DocumentCount` already counts those retained slots. Primary-key distinctness counts identities, while its MCV frequencies describe versions per identity. `TemporalRange` uses this value-aware history estimate; interval overlap and residual predicates may reduce the actual output. Missing or stale statistics leave the estimate unknown. Point `TemporalIndex` seeks still return at most one version and estimate `1` (`PK IS NULL` estimates `0`).


<a id="syntax-array-component-statistics"></a>

### Array component statistics

`ANALYZE` emits `IsArrayComponent = TRUE`, `IsCompound = FALSE` rows for each array-index component. `TotalCount` counts distinct typed value and coverage postings per source row; `NullCount` is zero. `CollectionDocCountAtAnalyze` counts source rows. Each MCV frequency is at most the source count, though frequencies for different values may sum above it. An empty source has no postings. Full and filtered component identities remain separate from scalar statistics. Component populations always use a full analysis scan.

Fresh component statistics price single probes, cost-reducing intersections, scalar paths, and a full scan. `EXPLAIN` reports the same choice as execution: `CompoundArrayMembership` can win even when several predicates are indexed. Distinct paths use independent selectivity estimates; repeated predicates on one path and partial intersections use the smallest marginal. Statistics do not prove same-element correlation. All predicates recheck candidate rows. The cost includes estimated sorter spill I/O under `MaxInMemoryStreamingState`; a single exact probe streams directly and has no sorter cost.

Subtype collections retain their complete row for array traversal and partial filters while scalar accumulators read the base image. A subtype that omits an indexed array contributes coverage when it satisfies the filter. Collection and namespace renames preserve statistics identities on disk and in cache; DROP removes the populations transactionally.

<a id="syntax-create--drop-statistics"></a>

### CREATE / DROP STATISTICS

Statistics declaration names retain a namespace identity and local name. An ordinary namespace rename changes their displayed names without updating declaration payloads or collected statistics. Namespace merges move membership and reject conflicting declaration names in the same migration.

```sql
BEGIN MIGRATION 1;
CREATE STATISTICS [App].[Geography] ON ([City], [Zip]) FROM [App].[Places];
COMMIT MIGRATION;
ANALYZE [App].[Places];
FROM [Mnemic].[StatisticsDefinitions];
```

`CREATE STATISTICS [Ns].[Name] ON ([Field], ...) FROM [Ns].[Collection]` declares an ordered group of 2–1024 distinct stored scalar field paths without creating an index. Nested base-object scalar paths and stored computed fields are allowed; array elements, composite values, virtual computed sources, and polymorphic extension fields do not form one stored scalar population. Each collection permits up to 1024 declarations. Names are qualified, unique across the database ignoring case, and cannot use the reserved Mnemic namespace. Fewer than two paths reports `INVALID_STATEMENT` and more than 1024 `LIMIT_EXCEEDED`, whether the statement is written or built.

Both `CREATE STATISTICS` and `DROP STATISTICS [Ns].[Name]` require DDL permission and an explicit migration. Creation records the definition; `ANALYZE`, after commit, collects the distribution. `[Mnemic].[StatisticsDefinitions]` exposes `Name`, `CollectionName`, and the ordered `FieldPaths` array. Migration owners see their private definitions; other sessions see committed definitions.

Analysis collects full-collection scalar marginals and a joint distribution for rows where every group component is non-null. The joint distribution shares the compound statistics identity and canonical tuple encoding with a full compound index on the same ordered fields. Duplicate declarations reuse that population; a partial index remains a separate filtered population.

Fresh joint frequencies correct equality selectivity when bitmap intersection combines complete, disjoint input domains covered by a declared group. The full-scan `EXPLAIN` conjunction estimate uses the same value-aware joint estimator. Overlapping groups are considered by descending field count and then name; each input or predicate contributes once. Uncovered inputs retain independent estimates. Missing or stale groups retain the existing fallback. Statistics affect estimates and plan choice; normal predicates determine rows.

Field and parent-object renames update dependent paths; removing a group field removes its declaration. `ALTER COLLECTION` and propagated `ALTER TYPE` changes invalidate the affected source's old statistics transactionally; run `ANALYZE` again after commit. Collection renames retain definitions and move their source identity; namespace renames also move names declared in that namespace. Abort restores the prior definitions and statistics. Dropping the last declaration removes its joint population unless a full compound index still owns it. Dropping a collection removes its groups. A declaration on a [materialized view](materialized-views.md) moves to each generation a REFRESH, `ALTER MATERIALIZED VIEW` or source rebuild publishes, unless that generation no longer stores one of its fields as a scalar.

## Usage pitfalls


<a id="pitfall-explicit-analyze-fully-rescans-auto-analyze-may-be-incremental"></a>

### Explicit `ANALYZE` fully rescans; auto-analyze may be incremental

An explicit `ANALYZE [Ns].[Coll]` always does a full, from-scratch resample of every estimator (min/max, histogram, MCV, NDV). The background auto-analyze loop may instead refresh **incrementally** — observing live rows whose latest logical write falls after the prior gap-free cut and merging their NDV contribution. Deleted rows do not enter that delta, so old observations remain additive until drift forces a full rescan. The difference never changes query *results*, only how fresh the planner's estimates are between full passes. When you need an authoritative full-scan snapshot (e.g. right before running `EXPLAIN` to reason about a plan), run `ANALYZE` explicitly rather than waiting for the loop.

Persisted compound correlation ratios must be finite and at least one, while scalar rows carry exactly `1.0`. Values outside that domain are catalog corruption, not a signal to fall back silently to heuristic planning.

Manual and background `ANALYZE` requests for the same collection serialize; they do not race catalog creation or let an older pass overwrite a newer watermark. Different collections can still be analyzed concurrently. A queued request fails if DROP/RENAME/re-create changed the target identity while it waited.

After an ANALYZE pass captures its gap-free commit cut, commits to that collection wait for the direct or temporal scan to finish. This is intentional: the distributions and watermark must describe one collection image. The source lease is released before `[Mnemic].[Statistics]` is updated, so analyses of different collections remain concurrent.

<a id="pitfall-drop--rename-collection-purge-or-move-the-collections-statistics"></a>

### `DROP` removes statistics; `RENAME COLLECTION` preserves them

Dropping a collection removes its `[Mnemic].[Statistics]` rows, so a later re-create under the same name never inherits stale, possibly wrong-encoding estimates — it starts un-analyzed until the next `ANALYZE`. Renaming preserves statistics and their deterministic IDs in both cache and disk. An aborted DROP restores the deleted statistics.

<a id="pitfall-statistics-declarations-need-analysis-after-commit"></a>

### Statistics declarations need analysis after commit

`CREATE STATISTICS` records a group inside a migration; it creates no index and collects no observations. Commit before `ANALYZE`. Groups describe complete, non-null tuples; scalar marginals describe the full collection. Field renames rebind groups, field removal removes dependent groups, and `ALTER COLLECTION` invalidates prior observations on commit. Abort preserves both. Dropping one of several definitions for the same ordered fields preserves their shared joint population, as does a surviving full compound index.
