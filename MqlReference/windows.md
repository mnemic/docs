# Window functions — OVER, PARTITION BY, frames

> **In this section** — analytic functions that compute per-row values over a sliding window of rows: ranking (`ROW_NUMBER`, `RANK`, `DENSE_RANK`), offset access (`LAG`, `LEAD`), distribution (`NTILE`, `PERCENT_RANK`, `CUME_DIST`), and aggregates over a frame (`SUM`, `AVG`, `MIN`, `MAX`, `COUNT`). The `OVER` clause defines the window; `WINDOW [w] AS (...)` factors it out for reuse.

A window-function call appears in the SELECT list as a computed projection and **requires** an `AS [Alias]` — the output column gets the alias as its name.

```sql
FROM [Sales].[Orders]
SELECT [OrderId],
       [Customer],
       ROW_NUMBER() OVER (PARTITION BY [Customer] ORDER BY [Date] ASC)
         AS [SeqPerCustomer];
```

## OVER clause

```
OVER ( [PARTITION BY <expr> [, ...]]
       [ORDER BY <field> ASC|DESC [, ...]]
       [<frame>] )
```

- **`PARTITION BY`** — splits rows into independent groups; the function evaluates from scratch in each partition. Without it the whole input is one partition. A partition key, an ORDER BY key and a function argument each read one value per row, so an array element path (`PARTITION BY [Tags][]`, `COUNT([Tags][]) OVER (...)`) is rejected with `FEATURE_NOT_SUPPORTED`, inline or in a `WINDOW` definition ([element paths read no single value](queries.md#array-element-paths-read-no-single-value)).
- **`ORDER BY`** — defines row order inside the partition. Without it, ranking and distribution functions (`RANK`, `DENSE_RANK`, `NTILE`, `PERCENT_RANK`, `CUME_DIST`) treat the partition as one big tie and produce meaningless results; supply an `ORDER BY` so the rank reflects something. Optional for `ROW_NUMBER` (rows numbered in arrival order within each partition). Equal complete window-order tuples also retain arrival order, so add another `ORDER BY` key when a different tie order is required. Aggregate-over-window uses the default frame selected by whether `ORDER BY` is present — see [Frames](#frames).
- **`<frame>`** — see [Frames](#frames). Frame-sensitive only on aggregate-over-window calls; ranking and offset functions ignore the frame.

Each `PARTITION BY` and each window-local `ORDER BY` accepts at most 64 expressions or fields respectively. The same boundary applies to inline `OVER (...)` clauses and reusable named `WINDOW` definitions. One `SELECT` may project at most 64 window-function expressions.

`PARTITION BY` accepts ARRAY and VECTOR values. Arrays group by their complete stored payload; vectors group by equal components, with positive and negative zero equal. NULL forms a separate partition from every present value, including an empty array. Partitions use the same value identity as `GROUP BY`: numeric values of different kinds (for example a `CASE` yielding `1` or `1.0`) share a partition when equal, and values of unrelated kinds form separate partitions.

ARRAY and VECTOR fields cannot be window `ORDER BY` keys: they have no scalar ordering. The schema check rejects them before result publication, including for empty or all-NULL inputs and static EXPLAIN. A scalar distance or text projection can supply an ordering key from an enclosing query.

## Functions

Function arity is exact and is checked before execution: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `PERCENT_RANK`, and `CUME_DIST` take no arguments; `NTILE`, `PERCENTILE_CONT`, `PERCENTILE_DISC`, `SUM`, `AVG`, `MIN`, and `MAX` take exactly one; `LAG` and `LEAD` take one to three; `COUNT` takes either `*` or exactly one expression. Empty `COUNT()` is invalid and is not an alias for `COUNT(*)`. Parsed MQL and direct programmatic evaluation both reject extra arguments; they are never ignored. A wrong argument count is `ARITY_MISMATCH`, naming the count the function takes. A call with `OVER` of any other name, such as `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE` or an aggregate without a window form like `COLLECT_ARRAY`, is `UNKNOWN_FUNCTION`, whatever its arguments, and its message lists the window functions above.

### Ranking

| Function | Returns | Notes |
|----------|---------|-------|
| `ROW_NUMBER()` | `INT64` | Sequential, never ties. |
| `RANK()` | `INT64` | Skips ranks after ties (`1, 2, 2, 4`). |
| `DENSE_RANK()` | `INT64` | No skips after ties (`1, 2, 2, 3`). |
| `NTILE(n)` | `INT64` | Splits the partition into `n` near-equal buckets, returns `1..n`. |
| `PERCENT_RANK()` | `FLOAT64` | `(rank-1) / (count-1)`, in `[0, 1]`. |
| `CUME_DIST()` | `FLOAT64` | Cumulative distribution: rows ≤ current / total, in `(0, 1]`. |

### Offset

| Function | Notes |
|----------|-------|
| `LAG([Field] [, offset [, default]])` | Value from the previous row (`offset` rows back, default 1). Returns `default` (or NULL) at partition start. |
| `LEAD([Field] [, offset [, default]])` | Mirror of `LAG`, looking forward. |

`NTILE`'s bucket count and `LAG` / `LEAD`'s offset must be **constant literals** — a field reference or expression is rejected with an error on every execution plan. The `default` argument of `LAG` / `LEAD` may be any expression (it is evaluated against the current row). The literal must be an exact finite numeric integer in `INT64` range: `2` and `2.0` are accepted, while `2.9`, `TRUE`, and `'2'` are rejected rather than truncated or implicitly converted. Offsets must additionally fit `INT32` and be non-negative; bucket counts must fit `INT32` and be positive. When `n` exceeds the partition size, each row occupies one of the first buckets (`1..row_count`); unused trailing bucket numbers do not appear. This result is identical on streaming and buffered execution plans, including `n = 2147483647`.

The result schema covers both value-producing arms. Without a default, the result has the value expression's type. With a default, compatible numeric types promote to their common numeric type; a mixed-kind or otherwise uncertain pair uses `STRING` as a stable, lossless wire type. An explicit `NULL` default is type-neutral, so `LAG([GuidField], 1, NULL)` retains `GUID`; use `CAST(NULL AS STRING)` when the default should deliberately contribute `STRING`. This rule is identical in streaming, lookahead, buffered-partition, and buffered-all plans.

Object and Array results retain their complete nested/element schema when all value-producing arms have compatible layouts and reference metadata. This includes the arms of `CASE`, `COALESCE`, `IFNULL`, `NULLIF`, and `TRY` inside the value or default expression. Incompatible or unproven Object/Array shapes use `STRING`, containing the base64 representation of the selected packed value. Scalar pinned references return their identifiers; a whole pinned array retains its packed identifiers and revisions with the array's schema.

Vector results retain `VECTOR(n)` when every non-NULL value/default arm proves the same dimension. Vector literals and bound vector parameters contribute their component count. Different or unproven dimensions use `STRING`; vector values become invariant, round-trip component text such as `[1.25, -2.5]`. A String default also uses this conversion for in-range vector values.

A positive `LAG` offset retains at most `min(offset, partition size)` preceding values while still emitting each row immediately. Its circular backing capacity and growth are charged to the query's engine-wide `StreamingState` admission reservation and released on normal drain, cancellation, error, or early result disposal.

A positive `LEAD` offset delays emission until the requested following row arrives. The executor's ring retains at most `min(largest forward offset + 1, partition size)` source rows; a `ROWS` frame ending in positive `FOLLOWING` uses the same lookahead shape. This state is charged to the query's engine-wide `StreamingState` admission reservation (including ring growth) and is released on normal drain, cancellation, error, or early result disposal. Disposal also drops the ring's backing-array reference, so retaining a disposed result does not retain the former lookahead capacity. When `LEAD` has an explicit default, the executor also retains at most `min(offset, partition size)` evaluated default values in a separate admission-accounted circular ring. Its capacity and growth remain charged until normal drain, cancellation, error, or early result disposal. At exact capacity, rotation reuses the oldest slot instead of growing beyond the requested offset. Memory still scales with the requested offset, so avoid needlessly huge offsets when a bounded frame or a join expresses the same query.

A streaming bounded aggregate over `ROWS n PRECEDING` through `m FOLLOWING` retains at most `min(n + m + 1, partition size)` positioned values. Its circular capacity and growth are independently charged to `StreamingState` and released on normal drain, cancellation, error, or early result disposal. `MIN` and `MAX` also charge their monotonic extrema deque; `SUM`, `AVG`, and `COUNT` do not allocate extrema-only state in either streaming or buffered monotonic-frame evaluation. Buffered `RANK` / `DENSE_RANK` / `PERCENT_RANK` scan with two reusable ORDER BY tuples and retain one integer length per peer run, deriving ranks during emit; they allocate neither a key array nor an eight-byte rank per row. Buffered `CUME_DIST` scans adjacent peers in the already ordered partition, retaining two reusable key tuples plus one integer length per peer run and deriving cumulative upper bounds during emit; it does not retain or re-sort every row's ORDER BY tuple. Buffered aggregate peer/group maps exist only for semantics that use them: `RANGE`, `GROUPS`, and `EXCLUDE GROUP/TIES`; plain `ROWS`, `EXCLUDE CURRENT ROW`, and whole-partition aggregates do not allocate those maps or ORDER BY snapshots. Single-key peer/range frames retain scalar ORDER BY values; only multi-key peer semantics allocate per-row key tuples. The partition-streaming evaluator resolves frame bounds as it consumes them; the full-input evaluator may retain per-row bound and coverage arrays because its complete input is already resident for random access. Unfiltered buffered aggregates also retain no per-row filter-verdict list. Whole-partition buffered aggregates retain one output value instead of a copy per row. `COUNT(*)` and `COUNT(expr)` maintain one counter and retain no per-row value, filter-verdict, peer, or output lists, including when an aggregate `FILTER` is present. Buffered LAG/LEAD retain only the first/last `min(offset, partition size)` defaults that can be emitted; offset zero owns no default history.

`BufferedPartition` stores serialized source rows in authenticated spillable replay, but some functions also retain decoded values, peer maps, or per-row results. That decoded state is conservatively estimated, admission-accounted, and bounded by `MaxInMemoryStreamingState`. A partition that would exceed the cap terminates lazy row consumption with `RESOURCE_PRESSURE`; raise the host's cap (the [standalone server](../configuration.md#standalone-server-environment) keeps the default), reduce partition cardinality, or choose a constant/bounded-state window shape. Constant-state functions such as `NTILE` can still process partitions whose raw row replay spills because their decoded state does not grow per row.

Every field named in an `OVER` clause's `PARTITION BY` / `ORDER BY` must exist in the source schema — an unknown field is an error, never a silent single-partition / all-peers evaluation.

### Aggregate over window

`SUM([F])`, `AVG([F])`, `MIN([F])`, `MAX([F])`, `COUNT(*)`, `COUNT([F])` — the same aggregates as `GROUP BY`, evaluated over the window's frame instead of a group. The output type matches the GROUP BY case ([aggregation](aggregation.md)).

`SUM` and `AVG` require numeric arguments, by the rule their grouped forms apply. A known nonnumeric field or typed expression (a `CAST` to a nonnumeric type, a built-in with a fixed nonnumeric result such as `UPPER`, a concatenation), including a numeric-looking String, is rejected with `TYPE_MISMATCH` before result publication and by static EXPLAIN, even with empty/all-NULL input or an always-false FILTER. Use an explicit numeric cast when conversion is intended. An argument whose kind only its value shows (a callback whose return type is unknown, a `CASE` whose arms differ) is checked when its values contribute, with the same `TYPE_MISMATCH`; validation does not execute it.

`MIN` and `MAX` preserve the complete Object, Array, or Vector decoding schema when every value-producing arm proves a compatible layout. This metadata is available even for empty or all-NULL input, including in derived queries and static EXPLAIN. Unproven structured layouts use the existing `STRING` representation. Scalar pinned references return logical identifiers; whole pinned arrays retain their stored frames. Standalone vector literals and vector parameters retain their nominal `STRING` result type for extrema.

Array extrema compare the complete stored value by unsigned byte order, and Vector extrema compare components numerically (float total order, then dimension), matching grouped `MIN` / `MAX`. NULL values do not contribute; an empty or all-NULL frame returns NULL.

Sliding Float64 `SUM` / `AVG` use compensated accumulation. When removing a departed row would leave that state saturated or cancellation-dominated (including strongly opposing internal cells), the executor rebuilds the active frame in row order so streaming and buffered plans retain the fresh-frame numerical result.

Buffered moving-frame Int64 and Decimal aggregates likewise rebuild the next frame if an incremental update overflows. Removing several departing values cannot report overflow merely because a partial removal exceeds the numeric bound. Overflow in the fresh frame's checked accumulation remains a `User / INVALID_VALUE` error.

A SQL:2003 `FILTER (WHERE …)` suffix attached at the call position restricts which rows contribute to the accumulator. Siblings in the same SELECT list see the unfiltered row stream — the filter is per- aggregate, not per-query. Its fields, parameters, scalar function names and argument counts are validated before publishing results, even for empty inputs. EXPLAIN and nested-query schema discovery apply the same checks without invoking filter functions. A stored view validates the field/function contract while leaving its parameter values to each execution. Registered filter functions with no declared return type are checked for BOOL/NULL when their values are evaluated; planning never calls them to discover that type.

Window FILTER uses MQL equality: NULL equals only NULL, and `NULL = value` is FALSE for a non-null value. Thus `NOT ([Value] = 1)` includes a NULL `[Value]`, just as an ordinary WHERE clause does. Ordered and text comparisons with NULL remain UNKNOWN, so negating them does not make the row contribute.

Field predicates retain their declared native width. For example, literal `200` matches stored INT8 `-56` in equality and IN, as it does in WHERE; ordered comparisons and bound parameters require signed-range admission. Explicit scalar expressions such as `CAST([Value] AS INT64)` retain their mathematical value domain. Malformed binary literal arms remain nonmatches.

Each input row evaluates the filter once before the aggregate argument. FALSE or UNKNOWN skips argument evaluation while preserving the row's partition membership and frame position. This evaluation order is the same whether the window streams, buffers one partition, or buffers the complete relation. The gate also applies to scalar subqueries inside the argument; partition expressions still evaluate for every row.

The predicate is evaluated row by row. When it traverses a polymorphic Object or ARRAY OBJECT extension, both the value and its field type come from that row or element's active subtype; a sibling subtype's declaration is never used to decode the bytes.

```sql
-- Running total of A-SKU sales only — B / C rows are skipped from the
-- accumulator but the partition + frame are still defined over the
-- full stream.
SUM([Amount]) FILTER (WHERE [Sku] = 'A')
  OVER (PARTITION BY [Region] ORDER BY [Date] ASC) AS [RunningA]
```

Valid on aggregate-over-window and ordered-set aggregates (`PERCENTILE_CONT` / `PERCENTILE_DISC`). The parser rejects FILTER on ranking + offset functions (`ROW_NUMBER`, `RANK`, `LAG`, …) — those don't accumulate row contributions, so the standard does not define a filter for them.

### Ordered-set aggregates (inverse distribution)

| Function | Returns | Notes |
|----------|---------|-------|
| `PERCENTILE_CONT(p)` | `FLOAT64` | Numeric `ORDER BY` column only. Linear interpolation at rank `p × (count - 1)` between adjacent ranked values. |
| `PERCENTILE_DISC(p)` | Same type as the `ORDER BY` column | Exact ranked value at index `ceil(p × count) - 1`; any sortable scalar column is accepted. |

`PERCENTILE_DISC` rejects Object, Array, and Vector ranked fields before publishing a result, even for empty or all-NULL input. Static EXPLAIN applies the same check; a Vector's fixed dimension does not define an ordering.

The ranked column is the window's single `ORDER BY` column. NULL order-by values are excluded from the ranking. `PERCENTILE_CONT` requires a numeric ranked column; Bool and String columns do not coerce merely because their values look convertible. `PERCENTILE_DISC` returns an exact ranked value and therefore accepts any sortable scalar type. Continuous interpolation stays inside the finite endpoint range even when opposite-sign Float64 endpoints make their raw difference overflow. Equal infinities remain infinite; an interior point between a finite value and one infinity is that infinity, while opposite infinities or a NaN endpoint produce NaN.

`p` must be a finite numeric value in `[0,1]` and a row-independent expression (a literal or parameter, optionally combined through scalar arithmetic/functions). Bool and String values are not converted implicitly; use `CAST(... AS FLOAT64)` or `TO_DOUBLE(...)` when conversion is deliberate. Invalid probabilities are rejected before a result stream is returned on every execution plan. The direction of the OVER's `ORDER BY` (`ASC` / `DESC`) defines the percentile ranking direction — SQL:2008 WITHIN GROUP semantics. Endpoint probabilities are constant-space per partition: `p = 0` retains only the first filtered non-NULL ranked value and `p = 1` only the last. Interior probabilities retain the filtered non-NULL rank set needed for selection or interpolation.

```sql
-- Median order amount per region (continuous interpolation).
PERCENTILE_CONT(0.5)
  OVER (PARTITION BY [Region] ORDER BY [Amount] ASC) AS [MedianAmount]
```

## Frames

```
{ROWS | RANGE | GROUPS} BETWEEN <bound> AND <bound> [EXCLUDE <mode>]
{ROWS | RANGE | GROUPS} <bound> [EXCLUDE <mode>]
```

The single-bound shorthand ends at `CURRENT ROW`: `ROWS 2 PRECEDING` is `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`, and `ROWS UNBOUNDED PRECEDING` is `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

`<bound>` is one of:

- `UNBOUNDED PRECEDING`
- `n PRECEDING` (n is a non-negative integer literal)
- `CURRENT ROW`
- `n FOLLOWING`
- `UNBOUNDED FOLLOWING`

`ROWS` counts physical row offsets within the partition. `RANGE` counts rows whose `ORDER BY` value is within `[current ± offset]` — ties on the `ORDER BY` key fall in the same frame regardless of position. A row whose `ORDER BY` value is NULL frames its NULL peers (all NULL-key rows form one peer run — the PostgreSQL contract). `RANGE` with an offset bound requires a single `ORDER BY` column. Ordinary aggregates require that column to have a numeric type (`INT8`/`INT16`/`INT32`/ `INT64`, `FLOAT32`/`FLOAT64`, or `DECIMAL`), including for a zero offset. This check runs before result publication and during static EXPLAIN, even for empty or all-NULL input. Temporal fields have no implicit numeric interval unit. Peer-only RANGE frames and functions that ignore frames retain their existing ordering rules. `GROUPS` counts distinct `ORDER BY` peer-groups — `n PRECEDING` starts at the first row of the peer-group `n` groups before the current row's, `n FOLLOWING` ends at the last row of the peer-group `n` groups after. `GROUPS` requires `ORDER BY` (without it every row is one giant peer-group and the frame collapses to the whole partition for any non-unbounded bound).

Frame offsets are integers from `0` through `2147483647`. Bounds are resolved mathematically and clamp at the partition or numeric-key domain: a `ROWS` / `GROUPS` bound beyond the available rows or peer-groups never wraps to the opposite direction, and a `RANGE` target beyond an `INT64` or `DECIMAL` endpoint remains ordered beyond that endpoint. The streaming, buffered-partition, and buffered-all plans therefore select the same frame even at the maximum offset and at numeric storage boundaries. Decimal targets also retain their exact fraction when adding the integer offset would round to a coarser Decimal scale; that rounding cannot admit a row outside the mathematical frame. Finite `FLOAT32` / `FLOAT64` keys retain their exact binary value when applying the integer offset. Rounding cannot collapse a following boundary onto the current key or widen a frame onto a neighbouring key. NaN and infinity keep their ordinary total-order semantics.

### Default frames

- Aggregate-over-window **with** `ORDER BY` → `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — the SQL:2003 default (PostgreSQL / SQL Server / DuckDB parity). `CURRENT ROW` in RANGE mode means "the current row and its `ORDER BY` peers", so rows tied on the ORDER BY tuple all share the same running value. For strict row-by-row running totals, spell the frame explicitly: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.
- Aggregate-over-window **without** `ORDER BY` → the whole partition.
- Ranking functions and `LAG` / `LEAD` ignore the frame entirely.
- A RANGE frame with an **offset** bound (`n PRECEDING` / `n FOLLOWING`) requires exactly one `ORDER BY` column (value arithmetic); the default frame's `UNBOUNDED` / `CURRENT ROW` bounds work with any ORDER BY arity.

### EXCLUDE

Filters rows out of an otherwise-resolved frame:

| Mode | Effect |
|------|--------|
| `EXCLUDE NO OTHERS` | Default. Equivalent to omitting the clause. |
| `EXCLUDE CURRENT ROW` | Drop the current row. |
| `EXCLUDE GROUP` | Drop the current row plus every row tied with it on `ORDER BY`. |
| `EXCLUDE TIES` | Drop every row tied with the current row, keeping the current row itself. |

## Named windows — WINDOW clause

When several projections share an `OVER` spec, factor it out at the end of the query:

```sql
FROM [Sales].[Orders]
SELECT [OrderId],
       SUM([Amount]) OVER [w] AS [RunningTotal],
       RANK()        OVER [w] AS [Rank],
       LAG([Amount]) OVER [w] AS [PrevAmount]
WINDOW [w] AS (PARTITION BY [Customer] ORDER BY [Date] ASC);
```

Window names are case-insensitive and must be unique within the query. Each named definition is a full `OVER` body — including a frame if needed. Each `OVER [w]` reads its definition as an inline `OVER` body; a definition no `OVER` names is never evaluated, so the fields its keys spell are not resolved, beside GROUP BY too (an array element path there is still refused, as in every key).

One query accepts at most 13,106 named-window definitions. Parsing and direct AST validation enforce the map bound before reading or traversing another definition; each definition retains the 64-expression `PARTITION BY` and 64-field `ORDER BY` limits.

## Examples

```sql
-- Sequence each customer's orders by date.
FROM [Sales].[Orders]
SELECT [Customer], [Date], [Amount],
       ROW_NUMBER() OVER (PARTITION BY [Customer] ORDER BY [Date] ASC) AS [Seq];

-- Top three orders per region by amount.
FROM [Sales].[Orders]
SELECT [Region], [OrderId], [Amount],
       RANK() OVER (PARTITION BY [Region] ORDER BY [Amount] DESC) AS [Rank];
-- (filter to Rank <= 3 in the application or a CTE)

-- Per-customer running total.
FROM [Sales].[Orders]
SELECT [Customer], [Date], [Amount],
       SUM([Amount]) OVER (PARTITION BY [Customer] ORDER BY [Date] ASC) AS [RunningTotal];

-- Three-row centred moving average.
FROM [Sales].[Orders]
SELECT [Date], [Amount],
       AVG([Amount]) OVER (
         ORDER BY [Date] ASC
         ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS [Smoothed];

-- Lag with a default for the first row of each partition.
FROM [Sales].[Orders]
SELECT [Customer], [Date], [Amount],
       LAG([Amount], 1, 0) OVER (PARTITION BY [Customer] ORDER BY [Date] ASC) AS [Prev];

-- Quartile bucket per region.
FROM [Sales].[Orders]
SELECT [Region], [OrderId], [Amount],
       NTILE(4) OVER (PARTITION BY [Region] ORDER BY [Amount] ASC) AS [Quartile];

-- Range-frame: include all orders with the same date as the current row.
FROM [Sales].[Orders]
SELECT [Date], [Amount],
       SUM([Amount]) OVER (
         ORDER BY [Date] ASC
         RANGE BETWEEN CURRENT ROW AND CURRENT ROW) AS [SameDayTotal];

-- GROUPS frame: spans the surrounding peer-groups by ORDER BY tuple.
-- Useful when ties on the ORDER BY column should be treated as a single
-- "step" rather than counted individually.
FROM [Sales].[Orders]
SELECT [Date], [Amount],
       COUNT(*) OVER (
         ORDER BY [Date] ASC
         GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS [AdjacentDays];

-- FILTER (WHERE …) — per-aggregate row gate at the window position.
FROM [Sales].[Orders]
SELECT [Customer], [Date], [Amount],
       SUM([Amount]) FILTER (WHERE [Status] = 'paid')
         OVER (PARTITION BY [Customer] ORDER BY [Date] ASC) AS [PaidRunningTotal];

-- PERCENTILE_CONT — median order amount per region.
FROM [Sales].[Orders]
SELECT [Region], [Amount],
       PERCENTILE_CONT(0.5) OVER (PARTITION BY [Region] ORDER BY [Amount] ASC) AS [Median];
```

## Composability

Static EXPLAIN and nested result-schema discovery validate window shape, partition/order fields, numeric RANGE keys, literal offset/bucket arguments, and percentile ranked types through the same schema checks as execution. They do not evaluate a percentile probability expression to plan the query. Its value is validated when the statement executes, before publishing rows.

Window projections run directly over JOIN and APPLY results. Partition and ordering keys resolve against the merged relations; the final query SKIP/LIMIT runs after window evaluation. A CTE may name the intermediate result for reuse.

- Window functions live alongside plain field projections in the same SELECT list.
- `WHERE` filters are applied **before** the window is computed — partitions only see rows that survived `WHERE`.
- `ORDER BY` at the query level is independent of the window's internal `ORDER BY`. The query's ordering is applied last.
- `SKIP` / `LIMIT` are applied **after** the window is computed, over the full `WHERE` result — the SQL evaluation order (windowed SELECT → ORDER BY → LIMIT). A window function therefore sees every matching row even on a paginated query: `ROW_NUMBER()` keeps counting across page boundaries, and `COUNT(*) OVER ()` reports the whole-result size, not the page size.

### Single-pass page total

Because the window spans the full result, `COUNT(*) OVER ()` (empty `OVER`) emits the grand total on every row — so one query returns both a page of rows and the total count without a second counting scan:

```sql
-- Page 3 (rows 20–29) plus the total result size on each row.
FROM [Shop].[Orders]
WHERE [Status] = 'open'
ORDER BY [Placed] DESC
SKIP 20 LIMIT 10
SELECT [OrderId], [Placed], COUNT(*) OVER () AS [Total];
```

(The typed client's `QueryPageAsync` reaches the same single-pass total for the statement without its `SKIP` and `LIMIT`: it asks for the page and the total count with the request ([request options](connecting.md#request-options)), and the engine counts the statement's rows in the pass that keeps the page's; `COUNT(*) OVER ()` is the plain-MQL equivalent.)

## Streaming vs buffered evaluation

The planner picks the cheapest evaluation strategy per window spec. `EXPLAIN ANALYZE.StreamingMode` reports streaming versus buffered result delivery, while the per-stage breakdown names the window stage (`WindowStreaming`, `WindowLookahead` or `WindowBufferedPartition`); a `BufferedAll` window adds no stage row:

- **Streaming** — partition by `ORDER BY` matches the query's output order and each row can be emitted immediately; state is O(1) or bounded by a preceding-row frame. Positive-offset `LAG` history is admission-accounted as `StreamingState`; bounded aggregate value rings use the same lifecycle and release with the result iterator.
- **Lookahead streaming (`WindowLookahead` stage)** — positive `LEAD` / `FOLLOWING` needs future rows before emission. The in-memory ring is admission-accounted as `StreamingState` and releases retained entries as they become ready. An explicit LEAD default has a separately accounted value ring; result delivery remains `Streaming`.
- **BufferedPartition** — the input is partitioned but the frame needs the whole partition before first emit (e.g. `UNBOUNDED FOLLOWING`, `NTILE`, `PERCENT_RANK`, `CUME_DIST`). Raw rows use authenticated spillable replay. Decoded evaluator state is capped as described above and can terminate lazy consumption with `RESOURCE_PRESSURE`.
- **BufferedAll** — the window's `(PARTITION BY, ORDER BY)` does not align with the query's ordering, so the executor sorts the full input before walking. Its random-access batch cannot spill safely and fails before publishing rows when it exceeds `EngineOptions.MaxInMemoryStreamingState`. A separate decoded-state owner uses the same local cap for window result vectors, cached row contexts, ordering arrays, scalar-subquery input vectors, and partition scratch state. Shared string, byte-array, vector, and boxed GUID payloads are counted by object identity. Scratch payloads release between partitions; values retained in results remain accounted through output construction or lazy consumption. Exceeding this estimate raises `RESOURCE_PRESSURE`, even when source rows fit.

Buffered ordering observes statement cancellation during ordering-index preparation, every row-context lookup, and every native comparison. Empty and short inputs also check at phase boundaries. Deferred scalar-subquery windows combine statement and consumer cancellation before preparing their ordering. Cancelled preparation releases decoded-state admission and preserves the canonical `Cancellation` / `CANCELLED` outcome and operator attribution; see [statement cancellation](cancellation.md).

Buffered `ROWS`, `GROUPS`, and `RANGE` aggregates share moving-frame state when nonempty frame boundaries advance monotonically and there is no exclusion. `SUM`, `AVG`, and `COUNT` use inverse transitions; `MIN` and `MAX` use a monotonic deque. Exclusions and retreating boundaries retain row-dependent evaluation; unstable floating-point or overflowing checked transitions rebuild the active frame in order.

For a computed window over a polymorphic collection, the executor must first preserve the subtype envelope and rebuild a typed union row. Streaming and partition-buffered plans replay that source from bounded authenticated spill state. A `BufferedAll` plan needs random access to the complete rebuilt union; if the source exceeds `MaxInMemoryStreamingState`, the statement fails before returning rows with `RESOURCE_PRESSURE`. Add a query `ORDER BY` compatible with every window's `PARTITION BY` / `ORDER BY`, raise the cap, or reduce the input.

Add a covering `ORDER BY` to your query to avoid a full-input sort on large data. Over a derived, CTE, or view source the mode depends only on that ORDER BY and the window specs, not on whether the producer's rows arrive lazily or, because they fit the cap, as one resident result. For `WindowLookahead`, peak retained rows are governed by the largest forward offset even when input order is already covered.

## See also

- [Aggregation](aggregation.md) — `GROUP BY`, `HAVING`, the same aggregate functions used outside `OVER`.
- [Basic queries](queries.md) — `WHERE`, `ORDER BY`, `SELECT`.

## Detailed syntax and execution


<a id="syntax-window-functions"></a>

### Window functions

```sql
FROM [Shop].[Orders]
SELECT [Id], [Region], [PlacedAt],
       ROW_NUMBER() OVER (PARTITION BY [Region] ORDER BY [PlacedAt]) AS [Rn],
       SUM([Total]) OVER (PARTITION BY [Region] ORDER BY [PlacedAt]
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS [Running];
```

Functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, `NTILE`, `PERCENT_RANK`, `CUME_DIST`, `PERCENTILE_CONT(p)`, `PERCENTILE_DISC(p)`, plus aggregates (`SUM` / `AVG` / `MIN` / `MAX` / `COUNT`) used with `OVER (...)`.

Window projections run directly over JOIN and APPLY results. Partition and ordering keys resolve against the merged relations; the final query SKIP/LIMIT runs after window evaluation. A CTE may name the intermediate result for reuse. A WHERE with `EXISTS`, `IN` or scalar subqueries filters the FROM input before the window runs; the projection keeps the input's column declarations.

Without a window-local `ORDER BY`, `ROW_NUMBER()` numbers rows in arrival order within each partition. Rows whose complete window-order tuples are equal also retain arrival order; add another ordering key when ties need a different order.

ARRAY and VECTOR partition keys use equality grouping: complete array payloads and vector components, including equal signed zeros. NULL remains separate from present values. ARRAY and VECTOR window ordering keys reject before publication, including empty/all-NULL input and static EXPLAIN; use an explicitly projected scalar key to define order.

Arity is exact: `ROW_NUMBER` / `RANK` / `DENSE_RANK` / `PERCENT_RANK` / `CUME_DIST` take zero arguments; `NTILE` / `PERCENTILE_CONT` / `PERCENTILE_DISC` / `SUM` / `AVG` / `MIN` / `MAX` take one; `LAG` / `LEAD` take one to three; `COUNT` takes `*` or one expression. `COUNT()` and every excess-argument form are rejected by both parsed MQL and direct programmatic evaluation; arguments are never silently ignored.

`SUM` and `AVG` reject known nonnumeric field arguments before publication, including empty/all-NULL input and static EXPLAIN. Explicit numeric casts are supported; callbacks with an unknown return type keep contribution-time checks.

For `PERCENTILE_CONT` / `PERCENTILE_DISC`, `p` is a row-independent, finite numeric expression in `[0,1]`. Bool / String do not coerce; wrap a deliberate conversion in `CAST(... AS FLOAT64)` / `TO_DOUBLE(...)`. NaN, infinities, NULL, out-of-range values, and field-dependent probabilities error before a result stream is published on every plan.

`PERCENTILE_CONT` additionally requires its single `ORDER BY` field to have a numeric schema type and always returns `FLOAT64`; numeric-looking String and Bool fields do not coerce. Its interpolation is overflow-safe for finite opposite-sign Float64 endpoints. `PERCENTILE_DISC` accepts any sortable scalar field and returns the exact selected value in that field's type. Object, Array, and Vector fields are not sortable percentile inputs and reject before publication, including for empty/all-NULL data and static EXPLAIN.

Aggregate-over-window and ordered-set aggregates accept a SQL:2003 `FILTER (WHERE …)` suffix that gates which rows feed the accumulator — siblings in the SELECT list see the unfiltered row stream:
```sql
SUM([Amount]) FILTER (WHERE [Status] = 'paid') OVER (PARTITION BY [Customer] ORDER BY [Date])
```

Window FILTER fields, bound parameters, scalar function names and argument counts are checked before results are published, including empty inputs. EXPLAIN and nested-query discovery perform these checks without calling the functions. Stored-view checks defer parameter values until execution. A registered function's undeclared return type is checked for BOOL/NULL when its filter value is evaluated.

The parser rejects FILTER on ranking / offset functions. FILTER predicates are row-resolved: a path through a polymorphic Object or ARRAY OBJECT uses the active subtype's value and field type for each row or element, never a sibling subtype's declaration.

`NTILE`'s bucket count and `LAG` / `LEAD`'s offset must be constant literals — a field ref or expression errors on every plan (the `default` arg of `LAG` / `LEAD` may be any expression). The literal must be an exact finite numeric integer in Int64 range: `2` and `2.0` accept; `2.9`, `TRUE`, and `'2'` reject rather than truncating or coercing. Offsets additionally fit Int32 and are non-negative; bucket counts fit Int32 and are positive. If an NTILE bucket count exceeds the partition size, rows occupy buckets `1..row_count` and trailing buckets are unused; streaming and buffered plans return the same result even at Int32.MaxValue. Every field in `PARTITION BY` / `ORDER BY` inside `OVER` must exist in the source schema — a typo is a loud error, never a silent all-peers / one-partition evaluation. Each window `PARTITION BY` and window-local `ORDER BY` accepts at most 64 expressions or fields respectively, including inside named `WINDOW` definitions. One `SELECT` may project at most 64 window-function expressions.

`LAG` / `LEAD` result typing includes both the value and optional default arms. Compatible numerics promote to a common numeric type; mixed or uncertain kinds use a stable `STRING` wire schema so a partition-boundary default cannot be truncated or replaced with NULL. An explicit `NULL` default is type-neutral and preserves the value arm's nominal type; an explicitly cast NULL contributes its cast target type.

Object/Array offset results preserve a complete common decoding schema across all value/default expression arms. Incompatible layouts or reference metadata use base64 `STRING` output. This applies to ordinary projections, wildcard `REPLACE`, joined results, and schema-only discovery.

Vector offset results likewise require one common valid dimension, including literal and parameter arms. Otherwise they use `STRING` with invariant, round-trip bracketed component text for selected vectors.

Window `MIN` and `MAX` also preserve proven complete Object/Array/Vector argument layouts, including for empty/all-NULL inputs and schema-only discovery. Unproven structured layouts use `STRING`; standalone vector literals and parameters retain nominal `STRING` typing. Scalar pins remain logical IDs, while whole pinned arrays retain their stored frames.

Array and Vector extrema use the grouped order in every window execution mode: unsigned stored bytes for Arrays, numeric components for Vectors; NULL inputs do not contribute.

Positive-offset `LAG` retains at most `min(offset, partition size)` preceding values while continuing to emit each row immediately. Its circular backing capacity and growth are charged to the statement's engine-wide `StreamingState` admission reservation and release on drain, cancellation, error, or early result disposal.

Positive `LEAD` and `ROWS ... m FOLLOWING` delay emission and retain at most `min(maxForwardOffset + 1, partition size)` source rows. The lookahead ring (initial slots, retained row/value arrays, and growth) is charged to the statement's engine-wide `StreamingState` admission reservation and releases on drain, cancellation, error, or early result disposal. Disposal also drops the ring's backing-array reference. When `LEAD` has an explicit default, up to `min(offset, partition size)` evaluated defaults are retained in a separate admission-accounted circular value ring whose capacity and growth reservation is released on the same terminal paths. At exact capacity, rotation reuses the oldest slot instead of growing beyond the requested offset. Memory still scales with the requested forward offset.

A streaming bounded aggregate over `ROWS n PRECEDING` through `m FOLLOWING` retains at most `min(n + m + 1, partition size)` positioned values. The circular entry array and its growth are independently admission-accounted as `StreamingState` and release on drain, cancellation, error, or early result disposal. `MIN` and `MAX` additionally account their monotonic extrema deque; `SUM`, `AVG`, and `COUNT` do not allocate extrema-only state in either streaming or buffered monotonic-frame evaluation. Buffered `RANK` / `DENSE_RANK` / `PERCENT_RANK` scan with two reusable ORDER BY tuples and retain one integer length per peer run, deriving ranks during emit; they allocate neither a key array nor an eight-byte rank per row. Buffered `CUME_DIST` scans adjacent peers in the already ordered partition, retaining two reusable key tuples plus one integer length per peer run and deriving cumulative upper bounds during emit; it does not retain or re-sort every row's ORDER BY tuple. Buffered `PERCENTILE_CONT(0|1)` and `PERCENTILE_DISC(0|1)` retain only the first/last filtered non-NULL ranked value respectively; only interior probabilities retain the full filtered rank set needed for selection or interpolation. Buffered aggregate peer/group maps exist only for semantics that use them: `RANGE`, `GROUPS`, and `EXCLUDE GROUP/TIES`; plain `ROWS`, `EXCLUDE CURRENT ROW`, and whole-partition aggregates do not allocate those maps or ORDER BY snapshots. Single-key peer/range frames retain scalar ORDER BY values; only multi-key peer semantics allocate per-row key tuples. The partition-streaming evaluator resolves frame bounds as it consumes them; the full-input evaluator may retain per-row bound and coverage arrays because its complete input is already resident for random access. Unfiltered buffered aggregates also retain no per-row filter-verdict list. Whole-partition buffered aggregates retain one output value instead of a copy per row. `COUNT(*)` and `COUNT(expr)` maintain one counter and retain no per-row value, filter-verdict, peer, or output lists, including when an aggregate `FILTER` is present. Buffered LAG/LEAD retain only the first/last `min(offset, partition size)` defaults that can be emitted; offset zero owns no default history.

Window execution uses window-local `Streaming`, `BufferedPartition`, and `BufferedAll` shapes. These are not the root `EXPLAIN ANALYZE.StreamingMode`, which reports only lazy (`Streaming`) versus eager (`Buffered`) result delivery. `BufferedPartition` spills serialized source rows to authenticated replay, but decoded values, peer maps, and per-row results have no safe spill form. Their conservative high-water estimate is admission-accounted and bounded by `MaxInMemoryStreamingState`; exceeding it terminates lazy consumption with `RESOURCE_PRESSURE`. Constant-state shapes such as `NTILE` can continue when only raw row replay spills. `BufferedAll` requires a resident random-access batch and fails before publishing rows if that batch exceeds the cap. Its separate decoded-state owner also uses `MaxInMemoryStreamingState` for result vectors, cached contexts, ordering arrays, resolved scalar-subquery inputs, and partition scratch state. Retained payloads are counted by object identity so a value broadcast to many result rows is charged once. Scratch payloads release between partitions; result payloads remain accounted through output construction or lazy consumption. Source rows fitting the cap does not prove that decoded window state fits; either owner can raise `RESOURCE_PRESSURE`. Buffered `ROWS`, `GROUPS`, and `RANGE` aggregate frames share inverse-transition or monotonic-deque state when boundaries advance monotonically without exclusions. Exclusions, retreating boundaries, unstable floating-point inverse transitions, and overflowing checked transitions preserve their ordered fallback evaluation.

Frames: `ROWS` / `RANGE` / `GROUPS` `BETWEEN <bound> PRECEDING|FOLLOWING AND <bound> PRECEDING|FOLLOWING|CURRENT ROW`, or the single-bound shorthand `ROWS` / `RANGE` / `GROUPS` `<bound>` that ends at `CURRENT ROW`, plus `EXCLUDE NO OTHERS|CURRENT ROW|GROUP|TIES`. `ROWS` counts physical rows; `RANGE` counts ORDER-BY-value distance (peers share a frame); `GROUPS` counts distinct ORDER-BY peer-groups. `GROUPS` requires `ORDER BY`.

Frame offsets are integers in `0..2147483647`. An offset beyond the available `ROWS` / `GROUPS` clamps to the partition edge; it never wraps to the opposite direction. A `RANGE` target beyond an Int64 or Decimal storage endpoint stays mathematically beyond that endpoint rather than wrapping or throwing. Streaming, buffered-partition, and buffered-all plans resolve these boundary frames identically. Decimal fractional boundaries remain exact even when the target cannot be represented at the source value's scale. Floating-point boundaries also retain the exact binary key plus integer offset rather than rounding that sum into a stored key.

**Default frame** — explicit when you write a frame, but otherwise:

| Window shape | Default frame |
|--------------|---------------|
| Aggregate (`SUM`/`AVG`/…) **with** `ORDER BY` | `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (running total; ORDER-BY-tied rows are peers and share the value — spell `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` for strict row-by-row) |
| Aggregate **without** `ORDER BY` | the **whole partition** |
| Ranking (`ROW_NUMBER` / `RANK` / `DENSE_RANK`) | frame is ignored — these only consume the ORDER BY |
| Offset (`LAG` / `LEAD`) | frame is ignored — only the offset matters |
| `PERCENTILE_CONT` / `PERCENTILE_DISC` | frame is ignored — the percentile is over the whole partition |

`EXCLUDE` modes: `EXCLUDE NO OTHERS` (default — keep the current row), `EXCLUDE CURRENT ROW`, `EXCLUDE GROUP` (drop the current row and its peers), `EXCLUDE TIES` (drop peers but keep the current row). RANGE-frame bounds compare against the ORDER-BY value, not row position — over a numeric key, `RANGE BETWEEN 10 PRECEDING AND CURRENT ROW` frames the rows whose key lies within 10 below the current one. A bound is an integer literal: there is no `INTERVAL` bound, and a temporal key takes no offset ([frames](#frames)).

Named windows: `WINDOW [w] AS (PARTITION BY [X] ORDER BY [Y])` then `OVER [w]` in the SELECT. One query accepts at most 13,106 named-window definitions. Parsing and direct AST validation enforce the map bound before reading or traversing another definition; each definition retains the 64-expression `PARTITION BY` and 64-field `ORDER BY` limits.

`SKIP` / `LIMIT` apply **after** the window is computed over the full `WHERE` result (SQL window-before-LIMIT order), so a window sees every matching row even on a paginated query. `COUNT(*) OVER ()` (empty `OVER`) therefore reports the whole-result size on every row — the single-pass "page rows + grand total" idiom (no second counting scan): `… SKIP 20 LIMIT 10 SELECT [Id], COUNT(*) OVER () AS [Total]`.

See [`windows.md`](windows.md).

## Usage pitfalls


<a id="pitfall-window-default-frame-is-range--tied-rows-share-the-running-value"></a>

### Window default frame is RANGE — tied rows share the running value

`SUM([X]) OVER (ORDER BY [K])` uses the SQL:2003 default frame `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`: rows tied on `[K]` are peers and ALL get the sum that includes every peer (PostgreSQL parity, deterministic under ties). If you want strict row-by-row running totals, spell `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — but note which tied row gets which value then depends on physical row order.

<a id="pitfall-large-forward-windows-retain-rows-until-their-value-is-knowable"></a>

### Large forward windows retain rows until their value is knowable

`LEAD([X], n)` and a `ROWS` frame ending in `n FOLLOWING` cannot emit the current row until the future row arrives. Peak ring occupancy is `min(n + 1, partition size)` even when the input order is covered. Mnemic charges the ring and its growth to `StreamingState` admission and releases it on every result-disposal path, but query memory still scales with `n`; do not use an enormous offset as a substitute for a join.

<a id="pitfall-window-frame-offsets-clamp-they-never-wrap"></a>

### Window frame offsets clamp; they never wrap

Frame offsets accept `0..2147483647`. If a `ROWS` / `GROUPS` offset reaches beyond the partition, the bound clamps at the appropriate edge; Int32 arithmetic never turns a huge `FOLLOWING` into `PRECEDING`. `RANGE` targets likewise retain their mathematical order beyond Int64 / Decimal endpoints. All window plans select the same rows at these boundaries. Adding an offset cannot round a Decimal fractional boundary onto a neighbouring row and change its membership. Floating-point addition likewise cannot collapse a positive offset onto the current key. This matters when the key's representable spacing exceeds the requested offset.

<a id="pitfall-percentile-probability-is-constant-numeric-and-finite"></a>

### Percentile probability is constant, numeric, and finite

`PERCENTILE_CONT(p)` / `PERCENTILE_DISC(p)` require a row-independent numeric `p` in `[0,1]`. NaN does not pass the range check, and Bool / String values do not convert implicitly. Bind a numeric parameter or use an explicit `CAST(... AS FLOAT64)` / `TO_DOUBLE(...)`. Invalid values fail before lazy row enumeration, identically under buffered-all and buffered-partition plans.

<a id="pitfall-continuous-percentile-ranks-numeric-fields-only"></a>

### Continuous percentile ranks numeric fields only

`PERCENTILE_CONT` requires a numeric `ORDER BY` field. A String such as `'12.5'` and a Bool do not become numbers implicitly; expose an explicitly converted numeric field upstream when that interpretation is intended. `PERCENTILE_DISC` is different: it selects an existing ranked value, so any sortable scalar field is valid and the original field type is preserved. Continuous interpolation does not overflow merely because two legal finite Float64 endpoints have opposite signs and an unrepresentable difference.

<a id="pitfall-explain-validates-ordering-having-and-windows-before-reading-rows"></a>

### EXPLAIN validates ordering, HAVING, and windows before reading rows

Static planning binds each consumer against its actual input: source/computed columns for ordinary ORDER BY, aggregate output for HAVING, and the window's partition and order schema. Missing sort keys, unbound HAVING parameters, unknown functions, and invalid window shapes fail even when an enclosing input is empty. Planning does not call HAVING functions or evaluate a percentile probability expression; value-dependent checks happen during execution.

<a id="pitfall-streamingmode-describes-only-root-result-delivery"></a>

### `StreamingMode` describes only root result delivery

`EXPLAIN ANALYZE.StreamingMode` describes only root result delivery: `Streaming`, `Buffered`, or `n/a`. Internal stages have their own execution shapes and pressure policies; do not interpret the root field as a promise that every stage is stateless or uses one universal cap.

Windows locally choose `Streaming`, `BufferedPartition`, or `BufferedAll`. `BufferedPartition` can spill serialized source rows, but decoded values, peer maps, and per-row results are capped because they have no safe spill form; an oversized partition can therefore terminate lazy consumption with `RESOURCE_PRESSURE`. `BufferedAll` needs a resident random-access batch and fails before publishing rows when it exceeds `MaxInMemoryStreamingState`. Align query ordering with the window keys when possible, reduce partition/input cardinality, choose a bounded-state window shape, or raise the cap.
