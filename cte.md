# Common Table Expressions — WITH and WITH RECURSIVE

> **In this section** — non-recursive CTEs, multiple CTEs, recursive CTEs for hierarchies and graphs, depth-guard patterns.

CTEs define named subqueries for use in a main query. CTE names use bracket syntax `[Name]`. One `WITH` block accepts at most 9,362 CTE definitions, the largest count representable under the statement token limit. Parsed text and direct programmatic AST execution enforce the same boundary. The bound counts written definitions only: the views a statement reads, which are expanded into definitions beside them, do not count, however long a chain of views is.

## Non-recursive CTE

```sql
WITH [ActiveProducts] AS (
    FROM [Shop].[Products] WHERE [Active] = TRUE
)
FROM [ActiveProducts] [p]
JOIN [Shop].[Categories] [c] ON [p].[CategoryId] = [c].[Id];
```

## Multiple CTEs

Comma-separated:

```sql
WITH [Expensive] AS (
    FROM [Shop].[Products] WHERE [Price] > 100
),
[TechCats] AS (
    FROM [Shop].[Categories] WHERE [CatName] = 'Tech'
)
FROM [Expensive] [e]
JOIN [TechCats] [t] ON [e].[CategoryId] = [t].[Id];
```

## Recursive CTE

For tree traversal, hierarchies, and bill-of-materials:

```sql
WITH RECURSIVE [OrgTree] AS (
    FROM [Org].[Employees] WHERE [ManagerId] = NULL
    UNION ALL
    FROM [Org].[Employees] [e]
    JOIN [OrgTree] [t] ON [e].[ManagerId] = [t].[Id]
)
FROM [OrgTree] SELECT [Id], [Name], [ManagerId];
```

The recursive member must JOIN back to the CTE name and may JOIN other relations; a member without that self-join is `INVALID_STATEMENT`, and a seed whose projection omits the field the self-join reads from the CTE is `UNKNOWN_FIELD`. Static `EXPLAIN` and schema discovery report the same errors. Every JOIN key must satisfy ordinary JOIN compatibility: matching scalar equality domains, compatible numeric types, or a STRING and a number, which join on the number's canonical text; Object and Array keys are invalid. These checks apply before expansion, including when the seed is empty or contains only NULL keys, and during static `EXPLAIN`. Additional joins consume the current frontier as a bounded relation, so they may filter or project parent fields. When the seed uses a projecting `SELECT`, the recursive rows are reshaped to that seed schema, so every field the seed projects must also be producible by the recursive member. The member's plain items are columns named by their path's leaf, as the same items spelled with `AS` are, whether the member reads only the frontier's keys or its full rows: `SELECT [n].[Id], [n].[PreviousId], [p].[Ref]` publishes `Id`, `PreviousId` and `Ref`, `[n].[Detail].[Code]` publishes `Code`, and two items with one leaf are a duplicate output name (`DUPLICATE_NAME`). The seed is an ordinary query, so a JOIN seed whose `SELECT` lists only plain fields, or that has no `SELECT` list, publishes the nested `outer`/`inner` columns ([JOIN](queries.md#join)), and so does a member with an unqualified `SELECT *` that reads its full frontier; such a seed lacks the join key (`UNKNOWN_FIELD`) and such a member the seed's columns (`TYPE_MISMATCH`), and the diagnostic names that nested shape: name the columns with `AS`. Compatible names are not enough: scalar widths, vector dimensions, nested Object/Array shape, enum/type identity, reference targets, and pinned-reference layout must also match, because every emitted row is decoded through the seed schema. A seed whose output keeps a polymorphic relation's envelope (a select-less seed over a polymorphic collection) publishes that polymorphic shape for the whole expansion, and the member must share it: the same discriminator, the same subtype names and every subtype's extension layout; otherwise the statement is `TYPE_MISMATCH`. Only a member that reads just the frontier's keys keeps its rows' envelope (one without a `SELECT`, or with `SELECT *`); a member that reads its full frontier emits flat rows (the seed's columns when it has no `SELECT`) and shares none. Such a seed keeps its subtype fields inside the envelope, not as output columns, so the self-join cannot read one from the frontier (`UNKNOWN_FIELD`; project it in the seed instead). A seed projection that keeps no envelope publishes none, and a select-less member over a polymorphic collection projects the seed's columns from its rows at every level, a subtype's extension fields included (NULL for a row of a subtype without the field), whether it reads only the frontier's keys or its full rows, while a member with its own `SELECT` produces the columns that projection emits. These output checks also apply when the seed is empty or contains only NULL join keys, without executing the recursive member, and are the checks every level's rows meet. Nullability is directional: a nullable seed output may accept a non-null recursive field. A frontier JOIN's conservative nullable metadata can be narrowed only after every emitted row is checked against the non-null seed contract; an actual NULL fails before result publication. Numeric widths are not implicitly converted; project an explicitly layout-compatible value on both sides.

The recursive member is a complete query at every level. Its `WHERE`, `SELECT`, computed/window projections, `GROUP BY`, `HAVING`, `ORDER BY`, `SKIP`, and `LIMIT` remain active; the self-reference reads the current frontier (a key-only membership plan is used when sufficient). Thus an `ORDER BY … LIMIT …` in the recursive member selects that many rows independently at each level, rather than being discarded or applied once to the final expansion.

The seed query (before `UNION ALL`) runs first, then the recursive member runs level by level until no new rows are found or the **maximum depth (100 levels)** is reached. Cycle detection follows normal MQL equality for the join key: strings are case-insensitive, decimal scale is insignificant, floating-point values use the engine total order, and DATETIME values compare by UTC instant. An empty `BYTES` value is a valid frontier key distinct from `NULL`. Numeric frontier matching retains each key's stored kind and exact value at every level, including spilled frontiers. For example, `FLOAT32 0.1` matches its exactly widened `FLOAT64` value `0.10000000149011612`, but does not match `FLOAT64 0.1`, regardless of whether the recursive plan retains full parent rows. Depth-tracking is not synthesised — if you need a per-row depth value (e.g. `[Level]`), the schema must carry it as a real field. If the hierarchy is deeper than 100 levels, the statement returns an error instead of exposing an incomplete prefix. Restrict the recursion with a `WHERE` clause against a stored field that bounds traversal (e.g. `[Tier] < 5`) when the graph may exceed the cap. To generate rows rather than walk stored ones (a calendar of days or months, a range of integers) use a [`SERIES`](queries.md#series) source, which needs no recursion, no join back to a collection and no depth cap.

## Aliases in the main query

Reference the CTE with an alias and use bracketed field paths:

```sql
WITH [RecentSales] AS (
    FROM [Shop].[Orders] WHERE [PlacedAt] > '2024-01-01T00:00:00Z'
)
FROM [RecentSales] [r]
WHERE [r].[Status] = 'shipped'
ORDER BY [r].[Total] DESC
SELECT [r].[Id], [r].[Total], [r].[CustomerId];
```

Without an alias, the CTE name is the reference's qualifier, exactly as if it were written as an alias: `FROM [RecentSales] WHERE [RecentSales].[Status] = 'shipped'`. This holds in JOINs too, in `ON`, `WHERE`, `SELECT` (plain, nested and `AS` items), `ORDER BY`, `GROUP BY` and aggregates, whichever join order or bushy tree the planner selects. Like an alias, it takes precedence over another relation's collection name and is never captured by a same-named Object field: in `WITH [Items] AS (FROM [Shop].[Docs]) FROM [Shop].[Orders] [o] JOIN [Items] ON [o].[Id] = [Items].[Id]`, `[Items].[Title]` reads the CTE's `Title` even if `[Shop].[Orders]` declares an Object named `Items`. When the CTE reference has an explicit alias, its CTE name remains usable like a collection name outside `ON`: it selects the latest reference of that CTE in the statement. Inside `ON` only the alias qualifies the reference, so `JOIN [D] [d2] ON [o].[Id] = [D].[Id]` is `UNKNOWN_FIELD`; write `[d2].[Id]` ([relation qualifiers](queries.md#join)).

## Cascading CTEs

One CTE references another, and the second can drive a JOIN:

```sql
WITH [Active] AS (
    FROM [Shop].[Products] WHERE [Active] = TRUE
),
[ActiveElectronics] AS (
    FROM [Active] WHERE [Category] = 'Electronics'
)
FROM [Shop].[Orders] [o]
JOIN [ActiveElectronics] [p] ON [o].[ProductId] = [p].[Id]
WHERE [o].[Status] = 'pending';
```

## Recursive CTE with a depth guard

Bound the recursion against a stored depth-like field on the source collection (e.g. an organisational tier persisted on each employee row) to avoid hitting the 100-level cap. Mnemic does not synthesise a per-row depth column inside the CTE, so the field must already exist on the collection's schema.

```sql
WITH RECURSIVE [OrgTree] AS (
    FROM [Org].[Employees] WHERE [ManagerId] = NULL          -- seed: the CEO
    UNION ALL
    FROM [Org].[Employees] [e]
    JOIN [OrgTree] [t] ON [e].[ManagerId] = [t].[Id]
    WHERE [e].[Tier] < 5                                     -- stored field on Employees
)
FROM [OrgTree] SELECT [Id], [Name], [Tier];
```

## Materialization

EXPLAIN describes the main query's consumers separately from the CTE's producer. Its typed cardinality includes consumer paging and aggregation; residual filters and joins do not inherit a seed estimate. Recursive expansion has unknown total cardinality and exposes its seed and repeated member as child boundaries. See [query introspection](queries.md#explain--query-plan-introspection).

Non-recursive CTEs are inline-expanded — each reference executes the body (no materialization). Recursive CTEs are materialized **once per statement** via level-by-level frontier expansion, before the main query runs; every reference — the main `FROM`, a `JOIN` from a real collection, or a self-join of the CTE on both sides — resolves against the same full expansion. The main query's `WHERE`, `ORDER BY`, `SKIP`, `LIMIT`, and `SELECT` clauses apply to the CTE result like any other source. Each recursive CTE is materialized after every recursive CTE its seed or member reads, directly or through other CTEs of the WITH, and otherwise in declaration order, so a reference sees its full expansion too, whichever CTE is declared first. Recursive CTEs that read each other are materialized in declaration order, and a reference from one of them to a CTE of the cycle declared after it reads that CTE's seed rows, so the result of mutually recursive CTEs depends on their declaration order.

Mistakes are reported in the same order: a recursive CTE's mistakes, in the order the WITH materializes them, before every mistake of the main query, a repeated output name in its `SELECT` included, and a non-recursive CTE's where a reference reads it, as a derived table's are, after the reading query's repeated output names. A non-recursive CTE no reference reads is never validated, so `WITH [c] AS (FROM [Shop].[Products] SELECT [Nope]) FROM [Shop].[Categories]` runs. Execution, schema discovery and `EXPLAIN` report alike, also where a nested query in the main query's `WHERE` reads a CTE and that `WHERE` has a mistake of its own: `WITH [c] AS (FROM [Shop].[Products] SELECT [Nope2]) FROM [Shop].[Categories] WHERE EXISTS (FROM [c]) AND [Nope1] = 1` reports `Nope2` from each.

CTE names have lexical scope. A nested WITH may shadow a name for its own query, but an inherited CTE body keeps the definitions visible where that body was declared. A WITH nested inside a CTE's body sees the CTEs around it, that CTE included, so a recursive member's nested query reads its own CTE through a nested WITH as it does without one. Shadowing a recursive CTE never reuses its materialized rows for the new declaration. The same rules apply to JOIN, scalar subqueries, APPLY, and EXPLAIN, including results consumed after execution returns.

Every CTE or system/virtual result freezes its schema and polymorphic subtype metadata before the outer query composes filters, ordering, pagination, or projection over it. This includes lazy streaming pipelines and virtual collections adapted for a JOIN. A streaming producer cannot change the layout advertised for bytes it emits; later nested execution likewise cannot change the layout used to decode replayable bytes.

Accumulated rows, visited keys, and both frontier generations share the statement's `EngineOptions.MaxInMemoryStreamingState` budget. They spill to authenticated temporary storage above their slices of that budget. A spilled expansion remains replayable for multiple references in the same statement and reports its spill bytes; temporary state is released on completion, error, cancellation, or early result disposal.

The seed and each recursive level freeze one private schema and polymorphic subtype generation as soon as that member returns. Later levels cannot mutate the layout used to decode rows already accumulated; malformed or cyclic result metadata fails before those rows are exposed.

With engine-wide snapshot isolation enabled, collection reads in the seed, every recursive level, and the main query all reuse one statement snapshot. Rows committed after the seed starts cannot appear in a later frontier level of that same statement. Temporal segments resolve at that CSN before each level applies valid-time filtering.

Temporal clauses (`AS OF`, `IN BETWEEN`, `LAST PER`) cannot be applied to a CTE reference — the produced relation has no version history. Place the temporal clause inside the CTE definition instead; the outer form is rejected by both execution and static EXPLAIN/schema binding. The restriction follows the resolved `FROM` source, not the presence of a `WITH` wrapper: a main query that reads a physical temporal collection may still use its temporal clauses even when the statement also declares CTEs.

## Correlated predicates

CTEs may supply either side of a correlated `EXISTS` or `NOT EXISTS`. A lazy result retains its WITH definitions and recursive expansion until it finishes or is disposed; consuming another result with the same CTE names does not replace those bindings.

```sql
WITH [customers] AS (FROM [Sales].[Customers]),
     [orders] AS (FROM [Sales].[Orders])
FROM [customers] [c]
WHERE EXISTS (FROM [orders] WHERE [CustomerId] = [c].[Id])
SELECT [Id];
```

## Explaining joins

A physical FROM may join CTEs whose bodies are derived, grouped, paged, or recursive. Static EXPLAIN retains each complete producer plan under its `CTE input:` row of a `DeferredJoin` boundary. It executes no producer to choose a physical tree; ANALYZE nests the actual tree selected after materialization under the boundary, each CTE input under the node that reads it with the rows it held; a join whose aliases execution selected more than one tree over, such as joins over the same aliases in both branches of a set operation, whether of the same relations or not, keeps its inputs unmeasured and its measurements, each CTE input's `JoinInput:` row included, as stage rows under the statement root. A recursive seed count is never the complete CTE estimate. See [queries.md](queries.md#explain--query-plan-introspection).

## See also

- [Basic queries](queries.md)
- [Practical examples](examples.md) — hierarchical org-chart query

## Detailed syntax and execution


<a id="syntax-ctes"></a>

### CTEs

One `WITH` block accepts at most 9,362 CTE definitions, the largest count representable under the statement token limit. Parsed text and direct programmatic AST execution enforce the same boundary. The bound counts written definitions only: the views a statement reads, which are expanded into definitions beside them, do not count, however long a chain of views is. The payload of a CTE's bracketed name cannot itself begin with `[`, because that prefix is reserved for the canonical `[Namespace].[Collection]` form.

```sql
WITH [ActiveProducts] AS (
    FROM [Shop].[Products] WHERE [Active] = TRUE
)
FROM [ActiveProducts] [p]
JOIN [Shop].[Categories] [c] ON [p].[CategoryId] = [c].[Id]
SELECT [p].[Name], [c].[CatName];
```

Recursive form:

```sql
WITH RECURSIVE [OrgTree] AS (
    FROM [Org].[Employees] WHERE [ManagerId] = NULL
    UNION ALL
    FROM [Org].[Employees] [e]
    JOIN [OrgTree] [t] ON [e].[ManagerId] = [t].[Id]
)
FROM [OrgTree] WHERE [Level] = 3;
```

Recursion cap: 100 emitted levels. The executor probes one additional non-emitting level; if new rows remain, the statement errors rather than returning an incomplete prefix. Recursive cycle detection uses normal MQL typed equality for the join key (including case-insensitive strings, scale-insensitive decimals, canonical floating-point identity, and UTC DATETIME identity). A present empty BYTES value is a valid frontier key, distinct from NULL. A recursive CTE is materialized once per statement; the main query may reference it in FROM, in a JOIN from a real collection, or self-joined on both sides — every reference sees the same full expansion. Under engine-wide snapshot isolation, collection reads in the seed, every recursive level, and the main query share one statement snapshot; a concurrent commit cannot appear only in a later level. Temporal segments resolve at that CSN before valid-time filtering.

At each level the recursive member retains its complete query algebra (`WHERE`, `SELECT`, computed/window projections, grouping/HAVING, `ORDER BY`, `SKIP`, and `LIMIT`); the self-reference reads the current frontier, using a key-only membership plan when sufficient. Recursive rows must be binary-layout compatible with the seed, including exact scalar widths, nested Object/Array shape, enum/type identity, reference targets, and pinned-reference layout. A nullable seed output may accept non-null recursive input, and a member's `AS` reference to a field publishes that field's declaration; a member's plain item is a column named by its leaf, as its `AS` spelling is, whichever algebra a level runs. Frontier JOIN nullability can narrow only after checking every emitted value against non-null seed fields; an actual NULL rejects the result before publication. Numeric widths are never implicitly converted. Accumulated rows, cycle keys, and frontier generations are bounded by `EngineOptions.MaxInMemoryStreamingState`, spill to authenticated temporary storage above the cap, and remain replayable for every reference in the statement. The seed and every recursive level freeze one private schema and polymorphic subtype generation immediately on return; later levels cannot rewrite the layout used to decode already-accumulated bytes, and malformed or cyclic result metadata fails before rows are exposed. Every CTE/system virtual result obeys the same rule before the outer query composes even a lazy filter/order/page/projection pipeline; virtual JOIN adapters capture before buffered rows are borrowed or streaming rows are drained. A producer's row iterator therefore cannot retarget the schema advertised for bytes it emits.

See [`cte.md`](cte.md).

## Usage pitfalls


<a id="pitfall-set-op-equality-follows-the-engine-doctrine"></a>

### Set-op equality follows the engine doctrine

`UNION` / `INTERSECT` / `EXCEPT` dedup and probe with the SAME equality every other seam uses: case-insensitive strings and the float total order. `'Cafe' UNION 'cafe'` is ONE row (first spelling wins), `INTERSECT` matches case-differing spellings, and NaN payloads count as one value. Decimal scale variants and DATETIME values naming the same UTC instant also match. NULL and present empty BYTES do not. Dedup-bearing parallel forms partition the complete canonical row with XXH3, so low-bit patterns in valid binary values cannot strand half the workers. `UNION ALL` keeps everything, as always.

Set-op compatibility is recursive, not just the outer type keyword. Two projected `OBJECT` columns with different nested fields, or two `ARRAY` columns with different element schemas, are a layout mismatch even though their top-level types agree. Nested nullability, enum/named-type identity, reference targets, pinned-reference width, and polymorphic subtype layouts must also match. The statement errors before yielding rows instead of decoding the right branch through the left branch's schema. Top-level scalar column names, source roles and nullability may differ; the left name is the output name, and a column is nullable when either branch's is.

<a id="pitfall-set-expressions-preserve-scope-and-precedence"></a>

### Set expressions preserve scope and precedence

Set operators can be chained and used in CTE definitions, WITH main queries, derived sources, and view bodies. INTERSECT binds more tightly than UNION and EXCEPT; operators at the same precedence associate left to right. Use `FROM (<query-expression>) alias` for explicit grouping or branch-local paging. The nesting bound applies to the complete tree. Unconsumed trailing tokens still fail rather than truncating a statement.

<a id="pitfall-set-op-trailing-clauses-are-global-not-right-branch-local"></a>

### Set-op trailing clauses are global, not right-branch-local

Combined ORDER BY binds the left branch's output names and accepts only scalar sort keys. Missing fields and Object/Array keys fail before combined row enumeration. Static EXPLAIN and nested schema discovery share this validation; empty outer inputs do not bypass nested set ordering or paging-parameter checks. Stored-view dependency checks validate ordering while keeping paging values as runtime inputs.

Set-operation plans own their combination strategy: `Concatenation` for UNION ALL and `HashSetOperation` for deduplicating or multiset operators. Both branch access plans remain children, with a separate `Sort` above the combination for trailing ORDER BY. Combined rows inherit no source index or source early-exit claim. A UNION estimate is unknown when either branch estimate is unknown; ANALYZE executes the complete operator to report actual rows.

In `A UNION ALL B ORDER BY ... SKIP ... LIMIT ...`, the clauses apply after combination. The older MQL position before B's `SELECT` has the same global meaning and prints back after the second `SELECT`; it cannot be used to page only B. Without `ORDER BY`, combined paging streams and can stop at LIMIT. With `ORDER BY`, the full combined set is externally sorted under `MaxInMemoryStreamingState` and spills checksummed runs. If spill replay and reader cleanup both fail, the replay, cancellation, or corruption cause remains first in the aggregate; cleanup diagnostics follow it rather than replacing it.

Snapshot isolation applies to the composite statement, not independently to each arm: both set-op branches see one CSN. Likewise a recursive CTE's seed, frontier levels, and main query cannot mix commits from different instants. Snapshot ownership is flow-local: a concurrent nested helper cannot publish or clear its child CSN in the parent flow while the parent still owns its lease.

<a id="pitfall-recursive-cte-member-frontier-joins-and-the-seed-schema"></a>

### Recursive CTE member: frontier joins and the seed schema

The recursive member JOINs the CTE and may JOIN other collections or CTEs. Those joins consume the current frontier, including its projected parent fields. If the seed uses a projecting `SELECT`, the recursive rows are reshaped to that seed schema, so every seed-projected field must be producible by the recursive member with a compatible complete binary layout (exact scalar width, nested Object/Array shape, enum/type identity, reference targets, and pinned-reference layout included), and a seed that keeps a polymorphic envelope requires the member to share its discriminator and subtype extension layouts, as set-operation branches must (only a member that reads just the frontier's keys, without a `SELECT` or with `SELECT *`, keeps one; a seed projection without the envelope publishes none, and a select-less member then projects the seed's columns, subtype fields included, from its rows); static `EXPLAIN` and schema discovery check this too, also over an empty seed, against the layouts the seed and member emit. A nullable seed output may accept non-null recursive input. A member's `AS` reference to a field (`SELECT [c].[Id] AS [Id]`) publishes that field's declaration ([direct references](queries.md#select-projection)), so it meets a seed that selects the field plainly, and so does the plain item `[c].[Id]`, which a member names by its leaf whether it reads the frontier's keys or its rows. A member that reads the frontier's rows publishes its computed values and the columns of every relation it joins as nullable; every actual value is checked before narrowing to a non-null seed field, and an actual NULL rejects the complete result. Numeric widths are not implicitly converted. The recursive member's remaining clauses are not decoration: `WHERE`, `SELECT`, computed/window projections, grouping/HAVING, `ORDER BY`, `SKIP`, and `LIMIT` execute independently at every frontier level. Cycle detection compares the join key with normal MQL equality, not its display spelling: case variants, decimal scale variants, canonical float values, and DATETIME offsets naming one UTC instant close the same cycle. Present empty BYTES values also remain valid frontier keys; only NULL ends that branch.

The same rule is plan-independent for ordinary joins: `''` on a `BYTES` join field is a real key in both left-deep and bushy execution. Do not use it as a NULL sentinel.

Inside `ARRAY OBJECT ... WITH SUBTYPE`, `MATCH ([Payload] = '')` matches a present empty BYTES field; `MATCH ([Payload] = NULL)` does not. Element subtype resolution does not change the NULL-versus-empty rule. If new unique rows still exist beyond 100 emitted levels, the statement errors rather than returning a partial hierarchy. Accumulated rows, visited keys, and both frontier generations are spill-bounded under `MaxInMemoryStreamingState`; a spilled recursive CTE is still replayable when the main statement references it more than once.

<a id="pitfall-a-producers-plan-does-not-describe-its-cte-or-derived-consumer"></a>

### A producer's plan does not describe its CTE or derived consumer

EXPLAIN preserves the producer as a child and reports consumer operators separately. An input primary-key seek is not an index on the resulting CTE. Filtering, grouping, HAVING, DISTINCT, and joins can make the consumer estimate unknown; scalar aggregation still emits one row before HAVING on empty input. Sort and window operators preserve cardinality, and final SKIP/LIMIT applies to the consumer estimate. Recursive expansion is not bounded by its seed count. Static binding also rejects temporal clauses on CTE results, as execution does; place those clauses inside the definition where a physical version store exists.

<a id="pitfall-a-cte-join-plan-cannot-substitute-its-seed-collection-for-its-complete-input"></a>

### A CTE join plan cannot substitute its seed collection for its complete input

For queries joining CTEs or plain views, on either side of the join, static EXPLAIN retains complete producer plans under their inputs of a `DeferredJoin`. `RelationInput` rows describe physical inputs without asserting selected scan access. Equality join output is unknown unless a known-empty input empties the join, and a `LEFT JOIN` on a physical relation's primary key keeps the estimate before it ([deferred join estimates](queries.md#explain--query-plan-introspection)), and runtime may reorder joins once CTE rows are materialized. ANALYZE nests the actual `JoinPlan:` tree under the boundary, each CTE or view input under the node that reads it, when execution selected exactly one tree over that join's aliases; it does not force written join order.
