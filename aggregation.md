# Aggregation — GROUP BY, HAVING, aggregates

> **In this section** — bucketing rows with `GROUP BY`, post-aggregation filtering with `HAVING`, the supported aggregate functions and their type rules.

`GROUP BY` buckets rows by one or more scalar expressions and emits one output row per distinct group. Aggregate functions in the SELECT list compute a single value per group. Omitting SELECT publishes the grouping columns alone, and so does a lone wildcard without modifiers, qualified or not (`SELECT *`, `SELECT [l].*`). A lone wildcard with modifiers (`SELECT * EXCLUDE ([Id], [Sku])`), and any wildcard among other items, expands the source's declared data fields without `Meta`, applies its modifiers, and requires every expanded field to be a grouping key: over `[App].[Lines]` (`Id`, `OrderId`, `Sku`, `Qty`), `GROUP BY [OrderId] SELECT * EXCLUDE ([Id], [Sku], [Qty])` returns `OrderId`, while `SELECT * EXCLUDE ([Id], [Sku])` reports `Qty` as neither a grouping key nor an aggregate (`GROUPING_ERROR`). The same holds beside a grouping expression or an aggregate argument, whose internal input columns are never a wildcard field: `GROUP BY [OrderId], [Qty] % 2 SELECT *` publishes `OrderId` and `Group2`, as an omitted SELECT does, and `GROUP BY [OrderId] SELECT * EXCLUDE ([Id], [Sku], [Qty]), SUM([Qty] * 2) AS [S]` returns `OrderId` and `S`. This schema is available without reading rows, including for empty input, scalar subqueries, CTEs, and set-operation branches. Joined subqueries retain their source qualifiers through WHERE, grouping, HAVING, projection, and window binding; a qualified key remains distinct from another relation's same-named field.

Groups, `DISTINCT`, and `COUNT(DISTINCT)` use the same value equality as set operations, joins, and window partitions: case-insensitive strings, exact numerics (`1.0` and `1.00` are one group), the UTC instant of a `DATETIME`, one NaN, `-0` equal to `+0` (also inside `VECTOR` components), and NULL distinct from a present empty value. The emitted grouping value is one input representative of its group.

```sql
-- Count rows per region
FROM [Shop].[Orders]
GROUP BY [Region]
SELECT [Region], COUNT(*);

-- Multi-field grouping plus several aggregates
FROM [Shop].[Orders]
GROUP BY [Region], [Status]
SELECT [Region], [Status],
       COUNT(*),
       SUM([Amount]),
       AVG([Amount]),
       MIN([Amount]),
       MAX([Amount]);

-- Filter GROUPS after aggregation with HAVING
FROM [Shop].[Orders]
WHERE [Status] = 'paid'
GROUP BY [Region]
HAVING [Revenue] > 10000
SELECT [Region], SUM([Amount]) AS [Revenue];

-- Scalar aggregation (no GROUP BY) — one output row total
FROM [Shop].[Orders] SELECT COUNT(*), SUM([Amount]);

-- Aggregate with explicit column alias
FROM [Shop].[Orders]
GROUP BY [Region]
SELECT [Region], SUM([Amount]) AS [Revenue];
```

## Projecting grouped results

An explicit grouped SELECT returns only its selected columns. Grouping keys remain available to expressions even when they are not returned. Expressions can read grouping keys and aggregate output aliases, including through scalar subqueries or window functions; they cannot read ungrouped source fields or ungrouped source values. Sibling computed aliases use SELECT dependency order. Evaluation follows HAVING and precedes final ORDER BY and SKIP / LIMIT. Without DISTINCT, ORDER BY may also use an unselected grouping key or aggregate output. After a JOIN, HAVING and grouped expressions resolve grouping keys and aggregate aliases against the completed groups. An aggregate alias takes precedence over a same-named source field; qualify the grouping key to address that key. WHERE, aggregate arguments, and aggregate FILTER conditions continue to use source fields and reject ambiguous unqualified references. For a single source, grouping keys, aggregate arguments, aggregate-local ordering and FILTER synthesize referenced virtual computed values from the captured source schema. A declared `Body` Object keeps its descendants in this source scope.

A unique emitted grouping name belongs to the completed groups, even if another JOIN input has an ungrouped field with that name. Aggregate Object outputs retain their children. For example, `MIN([r].[Detail]) AS [Body]` makes `[Body].[Code]` a result-field reference. The qualified spelling `[r].[Body].[Code]` can address that aggregate child when no exact grouping key or grouped parent claims it. These result references preserve the actual output root `Body`; they do not apply source shorthand again. A qualified ungrouped source field cannot borrow another grouping key merely because their leaf names match.

With DISTINCT, ordering can use only selected values. A grouping key selected through a direct alias retains its source identity, and a selected whole Object retains its children. An omitted qualified key cannot use a same-named aggregate output or an unrelated output Object's child as a substitute, and a path under a `Body` the source declares never names a selected key, key alias or aggregate merely because its leaf spells that name: over such a source, `GROUP BY [Id] ORDER BY [Body].[Id] SELECT DISTINCT [Id], COUNT(*) AS [N]` is refused (`UNKNOWN_FIELD`), never ordered by the selected `Id`, while `GROUP BY [Body].[Size] ORDER BY [Body].[Size] SELECT DISTINCT [Body].[Size] AS [K], COUNT(*) AS [Size]` orders by the selected key `K`, never by the `Size` count.

For a single source, HAVING and ORDER BY can refer to an exact grouping key using its source path, source alias, collection name, or namespace and collection name. For example, after `GROUP BY [Detail].[Code]`, both `[Code]` and `[Rows].[Detail].[Code]` address the emitted key for collection `[App].[Rows]`. A declared Object named `Body` is part of the source path. Optional `Body` shorthand applies only when that source has no declared Body root; a missing child of a declared Body cannot resolve to an unrelated root field. The same source spelling retains its meaning in grouped SELECT, HAVING, ORDER BY and window expressions: over a source that declares `Body`, `ORDER BY [Body].[Code]` or `HAVING [Body].[N] > 0` names a path under that Body, never the emitted `Code` key or `N` aggregate. This also applies to derived, CTE, and joined inputs. The completed groups do not publish such a path. One rule refuses it, as every other name they do not publish, in HAVING, in an ORDER BY without DISTINCT, in a SELECT item and in a window's `PARTITION BY` or `ORDER BY`, qualified by its relation or not, whatever the SELECT selects, beside a grouping expression, an aggregate argument or a `JOIN_STRING` separator parameter as beside plain keys, operands and separators, and for a scalar aggregate too: a name the grouped relation declares (an ungrouped field, a path under a declared Body, a subtype's field, an expanded wildcard slot no key groups) is a `GROUPING_ERROR`, and any other name, a SELECT alias HAVING reads included, an `UNKNOWN_FIELD`. A grouping key's value carries the children its declaration declares, read through the key's source path or its emitted name (`[Geo].[Lat]` after `GROUP BY [Address].[Geo]`), so a path under a grouped Object key that the declaration lacks (`[Dims].[Nope]` after `GROUP BY [Dims]`) is an `UNKNOWN_FIELD` of that rule as well, while over a single source a SELECT expression's input (a window's arguments and `FILTER` included) reads it as null, as outside grouping; a JOIN refuses it there too, as it refuses an undeclared input outside grouping. The text names the clause (`HAVING`, `ORDER BY`, `SELECT` or `Window`) and quotes the name as written, beside an item list and a wildcard alike, never by the address a joined row places it at: over a source aliased `[i]`, `HAVING [i].[Flag] = TRUE` reports `HAVING field 'i.Flag' is neither a GROUP BY key nor an aggregate output.` and `ORDER BY [i].[Nope]` reports `ORDER BY field 'i.Nope' does not exist in the source schema.`, and a qualified JOIN wildcard's slot is named by its relation's qualifier. The name is refused while its clause binds, before a key or leaf written earlier is admitted: `ORDER BY [A], [Nope]` reports the unknown `Nope` before the Array-typed aggregate `A`, which no ordering sorts by. An ORDER BY of a DISTINCT statement reads its selected values alone and reports such a name as no selected value (`UNKNOWN_FIELD`, `Cannot sort by 'i.Nope': field does not exist in the SELECT DISTINCT result schema.`). Beside a wildcard, outside HAVING, a name of a column the projection publishes under a name of its own (a `RENAME` or `REPLACE` output, a computed alias) reads that column in an ORDER BY and a SELECT expression, over a join too where a joined relation declares a field named like it, and a window's keys, which read no SELECT alias, refuse it. Over a single source, whose binding consumes the qualifier, a qualified spelling reads what its unqualified spelling reads: beside a wildcard over `[i]`, `ORDER BY [i].[Qty]` orders by the output column `Qty` after `RENAME ([Flag] AS [Qty])` or `REPLACE (... AS [Qty])`, and `[i].[U]` reads a computed alias `U`. Over a join a qualifier selects its relation's field, so there `ORDER BY [i].[Qty]` is an ungrouped field and `[i].[U]` names no computed alias; only where the relation declares no such field does `ORDER BY [i].[F]` after a wildcard's `RENAME ([Flag] AS [F])` over `[i]` order by the output column `F`, as over a single source. Beside an item list a qualified spelling names no computed alias or aliased key, and HAVING, which reads the groups before the projection publishes these columns, refuses them. Beside those inputs too, `ORDER BY [Group2]` orders by the generated output name of a grouping expression an omitted SELECT or a lone wildcard publishes, as beside the plain key a SELECT aliases `[Group2]`; an unqualified grouping key's name reads that key beside a wildcard as beside an item list, even where another joined relation declares that name; a named window's `PARTITION BY` and `ORDER BY` read grouping keys and expressions as an inline window's do, and a named window no `OVER` uses is read by no grouped operator, as outside grouping. EXPLAIN uses the same binding, including on empty input; these spellings do not make ungrouped source fields available. A source alias and a same-named Object remain separate path components: `[r].[r].[Code]` addresses the `Code` field inside Object `r` from source alias `r`, and in HAVING and ORDER BY `[r].[Code]` names the field `Code` of source alias `r`, refused when the groups do not publish it, never the child `Code` of a grouped Object `r`. Over a join, a relation aliased `[outer]` or `[inner]` likewise names its own fields.

This scope also applies when a complete output expression has the same spelling as a grouping expression after source qualifiers are removed. With `GROUP BY [d].[Reading] + 0` and `SUM([d].[Id]) AS [Reading]`, `[Reading] + 0` uses the aggregate result, while `[d].[Reading] + 0` uses the grouping key.

```sql
FROM [Shop].[Orders]
GROUP BY [Region]
ORDER BY [Label]
SELECT [Region] AS [Label], SUM([Amount]) AS [Revenue],
       [Revenue] * 2 AS [Doubled];
```

Declare the aggregate as its own SELECT item, then reference its alias in an expression. The aggregate input still uses source fields; the expression uses the completed aggregate result. A scalar aggregate also permits constant and aggregate-derived expressions, including on empty input.

## Supported aggregates

Every aggregate except COUNT(*) accepts a scalar input expression, such as `SUM([Price] * [Quantity])` or `COUNT(DISTINCT LOWER([Name]))`. GROUP BY accepts the same scalar expressions; SELECT may use the identical expression as a key, for example `GROUP BY LOWER([Name]) SELECT LOWER([Name]) AS [Key], COUNT(*)`. These inputs bind to the source, independently of SELECT aliases. Direct fields, scalar expressions, grouping keys and aggregate-local ORDER BY use the same source address: for a source declaring Object Body, `SUM([n].[Body].[Amount])` and `SUM([n].[Body].[Amount] + 0)` both read its nested Amount. A root Amount cannot substitute for it. Without a declared Body root, Body remains optional source shorthand. Missing-child and type checks apply during EXPLAIN and for empty input as well. Nested aggregates and window functions are not permitted in those inputs. JOIN_STRING's separator is a STRING literal or STRING parameter, resolved once per statement. Internal argument and grouping columns cannot capture authored source names. Adding a sibling expression preserves each input's existing value or missing-field behavior, even when its spelling resembles an internal column. Statement validation, `EXPLAIN` and execution report a statement with expression inputs the same first mistake: each aggregate's `FILTER` before `HAVING`, and a `HAVING` field that names neither a grouping key nor an aggregate output with the code and text the statement with plain keys and operands reports (`GROUPING_ERROR` for a field the source declares, `UNKNOWN_FIELD` for any other name), over a polymorphic collection and a system collection read as the single `FROM` source too.


`COUNT(*)`, `COUNT([Field])`, `COUNT(DISTINCT [Field])`, `SUM([Field])`, `AVG([Field])`, `MIN([Field])`, `MAX([Field])`, `COLLECT_ARRAY([Field] [ORDER BY [F] [ASC|DESC], …])`, `JOIN_STRING(expression, separator [ORDER BY [F] [ASC|DESC], …])`. `COUNT(*)` counts every row in the group; `COUNT(DISTINCT [F])` counts the number of distinct non-null values; all other aggregates skip null inputs. One query may project at most 64 aggregate expressions. One `GROUP BY` tuple may contain at most 64 fields.

On a polymorphic collection, a field path used by `GROUP BY`, an aggregate, or an aggregate's inner `ORDER BY` must have one structurally identical schema in every subtype that declares that path. Same-named subtype extensions may legally have different types for row-wise filtering and `MATCH`, but aggregation cannot assign those bytes one output type or one grouping order; an ambiguous path is rejected instead of interpreting every subtype through the first matching subtype's schema.

This rule includes `COUNT([Field])` and `COUNT(DISTINCT [Field])`, even though their output is always INT64. Incompatible nested or collection-level subtype declarations are rejected on empty input, when FILTER excludes every row, and during `EXPLAIN`. `COUNT(*)` does not bind an input field.

Compatible grouping leaves beneath differently shaped parent Objects retain their values beside scalar aggregate expressions and within mixed field/expression grouping tuples. Subtypes without a key contribute to the NULL group. An extension field of a polymorphic JOIN relation groups as that field, a materialized column of its declared type, whether the grouping sits beside a grouping expression or a wildcard, and whichever relation of the join declares it. SELECT names and column order remain as written, and HAVING and ORDER BY retain their grouping-key and aggregate-alias scopes. Static EXPLAIN checks the same key compatibility without reading rows.

This also applies to inner sort keys inside nested polymorphic Objects, with single-key and multi-key ordering validated even when no source rows match. Subtypes that do not declare the field contribute NULL sort keys. Compatible sort-key leaves retain their values when their enclosing subtype Objects have different layouts, including aggregates with scalar expression arguments such as `JOIN_STRING(UPPER([Name]), '|' ORDER BY [Component].[Reading])`. Plain-field aggregates retain those same compatible subtype values when mixed with expression aggregates, such as `SUM([Component].[Reading])` beside `SUM([Id] + 0)`. Their explicit and default output names remain unchanged. Static `EXPLAIN` and nested query schema discovery perform the same ordering-key validation, including when an empty outer query never executes the aggregate.

Base fields on polymorphic collections retain their declared values and NULL handling for scalar totals and grouped aggregates, across every subtype and execution strategy. Source metadata such as `[Meta].[Revision]` may participate alongside base and extension fields. Adding a metadata aggregate preserves sibling totals, grouping keys, and aggregate-local ordering, including NULL and absent subtype values.

`SUM` and `AVG` over a **floating-point** field (`FLOAT32` / `FLOAT64`) have an **unspecified evaluation order**: the engine may sum the values in any order (it uses a SIMD batch reduction), so the low-order bits of the result can differ from a strict left-to-right sum. This is the same contract every parallel SQL engine carries — floating-point addition is not associative. The result is Kahan-Babuška-Neumaier-compensated (high accuracy) and still saturates to `±Infinity` on overflow rather than erroring. `SUM`/`AVG` over **integer** fields are exact and order- independent. If you need a byte-stable floating-point total, round the result to a fixed precision or sum a fixed-scale integer/decimal column.

`COLLECT_ARRAY` collects every non-null value of `[Field]` in the group into a fresh `ARRAY` whose element schema mirrors the complete source field shape, including nested OBJECT/ARRAY structure, enum identity, reference metadata, and nullability, and uses the canonical public element name `Element`. Collecting pinned scalar references preserves every identifier-and-revision pair in a pinned result array. Repeated identifiers with different revisions remain separate elements. Collecting an existing pinned array produces an array of those complete arrays. An empty Object `()` is a non-null value: COUNT includes each occurrence, COUNT(DISTINCT) counts it once, and COLLECT_ARRAY preserves it as an empty Object element. It is also a valid aggregate-local ORDER BY key, distinct from NULL. A NULL parent or a field absent from the active subtype contributes NULL instead. The complete path must exist in that row's active layout: for example, `[Box].[Payload].[Value]` is NULL when that subtype's Payload is a scalar or an empty Object with no Value child. A present ancestor does not turn the missing child into an empty value. The same rule applies to aggregate operands, GROUP BY keys and aggregate-local ordering keys. `JOIN_STRING` converts every non-null value to text using the built-in `TO_STRING` rules, then inserts the resolved STRING separator between each pair. Numbers use their canonical text (FLOAT32 first widens to FLOAT64, a negative zero reads `0`), and a NaN or infinite value, which has no text, is `INVALID_VALUE`; BOOL uses `true`/`false`, temporal values use round-trip text, and STRING content is unchanged. BYTES, OBJECT, and ARRAY use Base64 of their complete payload; VECTOR uses its component-list format. A pinned scalar reference contributes its identifier without the revision; a pinned array retains its full payload. Present empty text, BYTES, and empty OBJECT values still contribute a position and its separators. Replacing the scalar `TO_STRING` function does not change this built-in aggregate conversion. Both return SQL `NULL` for an all-null group. The optional inner `ORDER BY` clause sorts the buffered values before emit; without it, values appear in whatever order the engine scanned them. The `JOIN_STRING` separator must be a STRING literal or parameter; field references are rejected so the separator stays stable across rows.

```sql
-- Concatenate per-group tags, alphabetised
FROM [App].[Tags] GROUP BY [Bucket]
SELECT [Bucket], JOIN_STRING([Tag], ',' ORDER BY [Tag] ASC) AS [Tags];

-- Collect per-group scores into a sorted INT32 array
FROM [App].[Tags] GROUP BY [Bucket]
SELECT [Bucket], COLLECT_ARRAY([Score] ORDER BY [Score] DESC) AS [Scores];
```

## GROUP BY rule

Every non-aggregated SELECT field must appear in the GROUP BY list — matching SQL. `SELECT [Region], [Name] GROUP BY [Region]` is rejected; add `[Name]` to GROUP BY or wrap it in an aggregate.

A grouping key, an aggregate argument and an aggregate-local ordering key each read one value per row, so an array element path is none of them: `GROUP BY [Tags][]`, `COUNT([Tags][])`, `MIN([Lines][].[Sku])` and `JOIN_STRING([Name], ',' ORDER BY [Tags][])` are rejected with `FEATURE_NOT_SUPPORTED`. Test a row's count of matching elements with a compared `LENGTH([Tags][] = 'x') > 0` (the uncompared form is no value; see [`LENGTH(predicate)`](queries.md#pitfall-lengthpredicate-count-filter--lengthfield-scalar)), or restrict an aggregate with `FILTER (WHERE [Tags][] = 'x')` ([element paths read no single value](queries.md#array-element-paths-read-no-single-value)).

A nested or alias-qualified grouping key defaults to its leaf name as the output column (`GROUP BY [Address].[City]` emits `[City]`). An explicit SELECT can give same-leaf keys distinct names or omit them:

```mql
FROM [App].[Pairs]
GROUP BY [Left].[Id], [Right].[Id]
SELECT [Left].[Id] AS [LeftId], [Right].[Id] AS [RightId], COUNT(*) AS [Total];
```

`GROUP BY SUBTYPE` groups a polymorphic collection by subtype, and `SELECT SUBTYPE` projects that key as the column `_type`; rename it with `[_type] AS [Kind]`, since `SUBTYPE AS [Kind]` is a parse error ([querying by subtype](polymorphism.md#querying-by-subtype)).

Only final output names must be unique case-insensitively. A hidden grouping key may share an aggregate alias: `GROUP BY [Amount] SELECT SUM([Amount]) AS [Amount]` is valid. Without SELECT, same-leaf keys still require an explicit projection to disambiguate their public names.

Plain SELECT items, expressions and HAVING can reference a grouping key by its source path or its unambiguous leaf name: after `GROUP BY [Address].[City]`, `SELECT [City], COUNT(*) AS [N]` publishes the key as `City`, as `SELECT [Address].[City]` does, and so do a metadata key (`[Revision]` after `GROUP BY [Meta].[Revision]`) and a key qualified by a single source's alias or a joined relation's (`[Region]` after `GROUP BY [o].[Region]`). A bare aggregate alias takes precedence over a same-named key; qualify the source key to distinguish them, for example `GROUP BY [r].[Amount] SELECT [r].[Amount] AS [Key], SUM([r].[Amount]) AS [Amount]`. This rule also applies to window and scalar-subquery expressions. SELECT aliases for computed expressions are output names, not HAVING inputs.

Scalar grouping expressions preserve these same result scopes. For example, with `GROUP BY [r].[Id] + 0`, `MIN([r].[Detail]) AS [Body]` and `SUM([r].[Id]) AS [Code]`, `[Body].[Code]` in HAVING or ORDER BY reads the aggregate Object child. It does not read the sibling `[Code]` aggregate or a source field. This also applies after a JOIN: an aggregate Object named Body keeps its child paths in HAVING and final ORDER BY. Reading an aggregate alias does not evaluate an unrelated source computed column with the same name. A qualified aggregate child such as `[r].[Body].[Code]` has the same meaning unless an actual grouped source key claims that path. DISTINCT retains selected key identities across scalar input evaluation and FILTER; omitted source keys cannot borrow same-named output columns.

## HAVING

Post-aggregation filter. Predicates reference either a grouping field or an aggregate alias. The HAVING grammar reuses `ParseFilterExpression`, so every WHERE operator works here too — including `LIKE` / `NOT LIKE` on string aggregates and `IN` / `NOT IN` lists. See [Basic queries → WHERE operators](queries.md#where-operators) for the full list. `HAVING` without a `GROUP BY` is only valid when the SELECT has at least one aggregate. `EXISTS`, `IN (subquery)` and scalar subqueries are accepted too, for example `HAVING [Revenue] > (FROM [Shop].[Targets] SELECT MAX([Amount]) AS [M])`; they are evaluated per group over the completed aggregate relation, as an OR across different grouping keys, aggregates or operators is (`HAVING [N] > 10 OR [Region] = 'EU'`).

An undeclared field or a source field omitted from GROUP BY is an error, including when the input is empty or the predicate compares with NULL. This validation applies to physical, derived, CTE, and joined sources, with or without an explicit SELECT, and agrees with EXPLAIN.

Bound parameters are checked against this post-aggregation schema, not against same-named source fields. For example, `HAVING [Min_Token] = @token` validates `@token` against the type of `MIN([Token]) AS [Min_Token]`, even if the source collection also declares a different-typed field named `Min_Token`. The same validation runs for base, joined, and virtual/CTE aggregation and on plan-cache hits.

That rule is identical for base collections, CTEs, computed virtual results, and `[Mnemic]` system collections. A standalone HAVING on a virtual source is rejected rather than ignored.

## `SKIP` / `LIMIT` on aggregate queries

`SKIP` / `LIMIT` on an aggregate query always page the aggregate **output** (the grouped result rows), never the aggregator input — SQL semantics. Every matching row feeds the aggregator first; then the grouped rows (sorted, when an `ORDER BY` is present) are sliced by `SKIP` / `LIMIT`. Adding or removing an `ORDER BY` never changes an aggregate's value.

With client/wire `IncludeTotalCount`, `TotalCount` is the number of aggregate output rows after `HAVING` and `DISTINCT`, before `SKIP` / `LIMIT`. Without DISTINCT this is one for a surviving scalar aggregate or one per surviving group; with DISTINCT it is one per unique selected tuple. It is never the number of input documents or raw JOIN pairs.

- A scalar aggregate produces exactly one output row, so `SKIP` / `LIMIT` page that single row. `FROM [C] WHERE [F] = 1 SELECT COUNT(*)` returns the full matching count; adding `LIMIT 3` keeps that one row (the value is **not** clamped to 3); adding `SKIP 1` (or any `SKIP` ≥ 1) pages past the single row and returns **zero** rows.
- A `GROUP BY` produces one row per group; `SKIP` / `LIMIT` slice the (optionally sorted) groups — the shape used to page through `GROUP BY` results.

This is plan-independent (indexed or full scan).

## Aggregating joined and CTE results

`GROUP BY`, aggregate functions, `HAVING`, and `SELECT DISTINCT` apply to the result of a `JOIN` and to a CTE / system-collection reference — not only to a single base collection. The merged join rows (or the drained CTE rows) are grouped after `WHERE` and before `ORDER BY` / `SKIP` / `LIMIT`. Reference joined fields by their alias in the grouping keys and aggregate arguments; the output column for a grouping key is its leaf name.

Source-row `[Meta]` fields may be grouping keys or aggregate arguments, including over a JOIN's outer side. The *aggregate output row* has no implicit `[Meta]` object because it no longer represents one stored version; select the metadata value into an ordinary named output column when it is required.

HAVING and ORDER BY can reference a metadata grouping key by its source path or its output name, for example `[Meta].[Revision]` or `[Revision]`. The same rule applies to ordinary nested grouping keys, including without an explicit SELECT. Ungrouped metadata fields are rejected even on empty input. An explicitly emitted field named `[Meta]` is evaluated with its actual result type and fields.

```sql
-- One row per region across the joined result, not one per join pair
FROM [Shop].[Orders] [o]
JOIN [Shop].[Items] [i] ON [o].[ItemId] = [i].[Id]
GROUP BY [o].[Region]
HAVING [Total] > 1000
ORDER BY [Region] ASC
SELECT [o].[Region] AS [Region], COUNT(*) AS [Cnt], SUM([i].[Price]) AS [Total];

-- Aggregate a CTE reference
WITH [c] AS (FROM [Shop].[Orders] SELECT [Region], [Amount])
FROM [c]
GROUP BY [Region]
SELECT [Region], SUM([Amount]) AS [Total];
```

Correlated predicates, including EXISTS, are supported on physical, derived, CTE, and system-collection sources. They evaluate against the complete input relation before grouping and paging; eligible shapes may be decorrelated.

## `FILTER (WHERE …)`

Every aggregate accepts an optional SQL:2003 `FILTER (WHERE expr)` clause that restricts which rows feed the accumulator. The clause sits between the closing `)` of the aggregate and any `AS` alias, and each aggregate carries its own filter — sibling aggregates in the same SELECT see the unfiltered row stream, so a single GROUP BY can compute opens / closeds / totals from one pass:

```sql
FROM [App].[Orders] GROUP BY [Region]
SELECT [Region],
       COUNT(*) FILTER (WHERE [Status] = 'open')   AS [Open],
       COUNT(*) FILTER (WHERE [Status] = 'closed') AS [Closed],
       COUNT(*)                                    AS [All];
```

The filter expression is the same form a top-level WHERE accepts (comparisons, AND/OR, IN, MATCH, IS NULL, and `EXISTS`, `IN (subquery)` and scalar subqueries, correlated or not, which each row evaluates) with the same NULL-safe equality and complementary IN/NOT IN rules. An OR across different fields or operators (`FILTER (WHERE [Status] = 'open' OR [Priority] > 2)`) is evaluated row by row, on a collection, a join, a derived table, a CTE and in a window aggregate alike. Only TRUE feeds the accumulator; FALSE and UNKNOWN both skip. `COUNT(*)`-with-filter returns `0` for groups where no row matched; `SUM`/`AVG`/`MIN`/`MAX` return SQL NULL.

FILTER field names refer to the source row, independently of output aliases and aggregate arguments. Qualification follows WHERE rules: a source alias claims its source first, including an alias named `[Body]`. A declared `Body` field is a real root, and a missing child below it cannot resolve to a same-named root field. Only a source without that field consumes optional `Body` shorthand. These rules also apply to joined inputs, independently of whether FILTER uses native predicates or scalar expressions. Execution and EXPLAIN validate the same field and parameter types, including repeated execution with new parameter values. A FILTER field no input declares is `UNKNOWN_FIELD` (`Query aggregate FILTER field 'Nope' does not exist in the collection schema.`); over a join the field is named relative to the input it reads, as in WHERE. Grouping keys and the arguments of every aggregate but `COUNT` are admitted and typed before any aggregate's FILTER, so a statement with mistakes in both reports the argument's, whatever the source: a collection, a join, a derived table or a CTE.

The filter is evaluated once for each source row before that aggregate's argument. Excluded rows do not evaluate the argument: for example, `SUM(8 / [Value]) FILTER (WHERE [Value] > 0)` does not divide by zero for zero-valued rows. Binding still validates the complete expression before reading rows, and sibling aggregates retain their independent filters. Internal FILTER columns cannot capture an authored argument or grouping input; those expressions retain their source binding and missing-field behavior.

Native FILTER predicates on polymorphic fields use the active subtype's layout, including leaves below differently shaped Objects and missing-subtype NULLs. They retain the same contribution rules beside expression aggregates and when the query also reads `Meta` fields. Unknown predicate fields are rejected even when the input is empty.

Arithmetic and function expressions in FILTER require compatible declarations of each referenced subtype leaf. They can read a compatible leaf beneath differently shaped parent Objects, regardless of subtype declaration order; subtypes without the leaf contribute NULL. Unknown and incompatible inputs are rejected before reading rows, including in static EXPLAIN, which does not invoke the functions. Predicate and argument callback counts keep the contribution rules above.

The same subtype leaf binding applies to comparisons beneath NOT. For a subtype without `[Component].[Reading]`, `NOT ([Component].[Reading] = 1)` is TRUE, while `NOT ([Component].[Reading] > 1)` remains UNKNOWN and does not contribute.

For plain-field aggregates with scalar FILTER expressions over a direct collection source without LAST PER, EXPLAIN shows source access, WHERE evaluation, FILTER gates, and aggregation in execution order. Composed predicates preserve the same ordering and paging boundaries as execution.

For aggregation over a `JOIN`, FILTER retains the selected source field when other aggregate inputs are rewritten. A qualified inner predicate such as `FILTER (WHERE [i].[Price] > @min)` checks `@min` against `[i].[Price]` rather than the outer collection, including on plan-cache hits.

When FILTER requires scalar evaluation, its native EXISTS, IN and scalar-subquery siblings read the complete source row before aggregation, including retained joined leaves absent from the flattened presentation. These residual gates bind native fields against their original candidate declarations. Native correlation uses the active subtype's value; scalar expressions and scalar expression-subquery dependencies require compatible declarations. EXPLAIN checks the same source ownership without executing subqueries or predicate callbacks.

## DISTINCT over aggregate results

`SELECT DISTINCT` can accompany `GROUP BY` or scalar aggregates. It removes identical selected output tuples after groups have been computed and `HAVING` has filtered them. Unselected grouping keys do not participate in deduplication. Aliased keys, scalar subquery projections, and expanded wildcards follow the same rule, beside a grouping expression or an aggregate argument too: `FROM [App].[Lines] GROUP BY [OrderId], [Qty] % 2 ORDER BY [OrderId] SELECT DISTINCT * EXCLUDE ([Id], [Sku], [Qty])` returns each `OrderId` once, in order. A nested modifier of a wildcard there (`EXCLUDE ([Dims].[H])`) rewrites the grouped key before the deduplication, so groups the rewrite makes equal are returned once. `ORDER BY` must reference the selected output, so a key a wildcard `RENAME` renames is ordered by its new name, and a key qualified by the relation the wildcard reads names that relation's own path first: `ORDER BY [i].[Flag]` addresses the source column `Flag`, which must be a selected grouping key, never an aggregate alias `Flag` or the child `Flag` of a kept Object named `i`, while `[i].[F]` after `RENAME ([Flag] AS [F])`, which the relation does not declare, names the output column `F`; final paging and requested `TotalCount` apply to the distinct result. Both aggregation and deduplication use the ordinary typed grouping equality and spill policy.

```sql
-- Different regions with the same revenue contribute one output value.
FROM [Shop].[Orders]
GROUP BY [Region]
HAVING [Revenue] > 10000
ORDER BY [Revenue] DESC
LIMIT 10
SELECT DISTINCT SUM([Amount]) AS [Revenue];

-- Scalar aggregation remains one row, including COUNT(*) = 0 for empty input.
FROM [Shop].[Orders] SELECT DISTINCT COUNT(*) AS [OrderCount];
```

DISTINCT does not waive grouping validation: source fields beside aggregates still need to appear in `GROUP BY`. `HAVING` uses a grouping column or aggregate output alias, such as `[Revenue]`, rather than repeating `SUM([Amount])`.

## Default aliases

Un-aliased `COUNT(*)` / `COUNT([F])` emit a column named `Count`, and `COUNT(DISTINCT [F])` emits `CountDistinct_F`. Other aggregates default to `{Func}_{Field}` (e.g. `Sum_Amount`). Override with `AS [Alias]`. For a nested, alias-qualified, or collection-qualified argument, `Field` is the leaf component (`MAX([o].[Address].[Score])` emits `Max_Score`). An argument that is an expression rather than a field has no field component: `SUM([Qty] * 2)` emits `Sum_`, so two un-aliased expression aggregates of one function share a name. Two aggregates sharing the same explicit or synthesized alias is a `DUPLICATE_NAME` error; use `AS` to disambiguate same-leaf paths and expression arguments.

## Output types

| Aggregate | Input | Output |
|-----------|-------|--------|
| `COUNT` | any | `INT64` |
| `SUM` | integer | `INT64` |
| `SUM` | `FLOAT32` / `FLOAT64` | `FLOAT64` |
| `SUM` | `DECIMAL` | `DECIMAL` |
| `AVG` | numeric | `FLOAT64` (or `DECIMAL` when the input is `DECIMAL`) |
| `MIN` / `MAX` | any | input type preserved |
| `COLLECT_ARRAY` | any | `ARRAY` with element schema = input type |
| `JOIN_STRING` | any | `STRING` |

`SUM` and `AVG` require a numeric input type. Known nonnumeric inputs are rejected before aggregate-result publication and in static EXPLAIN, even when the source is empty, every value is NULL, or FILTER excludes every row. This also applies to typed expression inputs; use an explicit numeric cast for an intended conversion. An untyped NULL literal or NULL parameter has the default `STRING` input type; cast it to a numeric type, such as `CAST(NULL AS INT64)`, to request a NULL numeric aggregate. Bound numeric parameters retain their numeric input type.

`SUM` / `AVG` / `MIN` / `MAX` return `NULL` when every input in the group was null and the input type is supported by that aggregate.

## Numerical precision

`SUM` / `AVG` over `INT8` / `INT16` / `INT32` / `INT64` fields use `checked` Int64 arithmetic — partition-level overflow surfaces as a `User / INVALID_VALUE` runtime error rather than wrapping silently, with the same pair whether the result is buffered or streaming. `AVG` widens the intermediate sum to `DECIMAL` so partitions whose mathematical average fits the output `FLOAT64` cannot overflow during accumulation.

`SUM` / `AVG` over `FLOAT32` / `FLOAT64` use Kahan-Babuška-Neumaier compensated summation: a parallel compensation cell tracks the round-off that IEEE-754 absorption otherwise loses (`1e16 + 1.0` rounds to `1e16` under naive addition), and the emit folds it back into the result. Total error of the emitted sum / average is bounded to first order by `O(ε·∑|xᵢ|)` instead of naive summation's `O(n·ε·∑|xᵢ|)`. Sliding-window invertible eviction (`SUM(...) OVER (... ROWS BETWEEN n PRECEDING AND m FOLLOWING)`) shares the same compensation cell across `Add` and `Subtract` on the ordinary path. A saturated or cancellation-dominated inverse state is rebuilt from the active frame in row order, including when its running and compensation cells strongly oppose. IEEE saturation to `±∞` on true overflow is preserved. If SIMD reordering creates a non-finite lane partial or horizontal-fold result, the aggregate retries the input in row order; a finite row-order result is not turned into `±Infinity` or NaN by vector-order overflow.

`SUM` / `AVG` over `DECIMAL` accumulate in CLR `decimal` arithmetic (exact within decimal's ~28-29 significant-digit precision); the sum overflows past `decimal.MaxValue` as the same `User / INVALID_VALUE` error.

## Grouping equality and string case-insensitivity

`GROUP BY` on a `STRING` field buckets rows by the same case-folded key the rest of the engine uses (WHERE `=`, ORDER BY, indexes). `'Cafe'`, `'CAFE'`, and `'cafe'` land in one group; the group-key column preserves the first stored representation encountered by the selected input path. Because source order is not a semantic property of an unordered aggregate, the exact representative may differ across plans while remaining a value that occurred in the input. `COUNT(DISTINCT [F])` on a `STRING` field follows the same equality rule — those three values count as one distinct value, not three. Other grouping fields use the engine's typed equality: numerically equal `DECIMAL` scale variants share a group, `DATETIME` values share a group by UTC instant, and floating-point NaN payloads and signed zero follow the total order. Every such group emits an encountered stored representation rather than the internal canonical identity.

`GROUP BY` and `SELECT DISTINCT` keep an empty Object `()` separate from NULL. The emitted grouping or distinct column retains its Object schema and presence, including when grouping spills to disk. A NULL parent or missing subtype field contributes to the NULL group.

`COUNT(DISTINCT [F])` on a scalar `PINNED REFERENCES` field counts referenced identifiers. Different revisions of one id count once; String identifiers use the usual case-insensitive equality. Present empty String and Bytes identifiers count as values, while NULL is excluded. Scalar, grouped, and spilled aggregates share this rule, including aggregates with their own `FILTER`.

`GROUP BY`, `SELECT DISTINCT`, and `COUNT(DISTINCT)` compare whole arrays by their complete encoded value. For pinned arrays, the element revisions remain part of that value: arrays with identical identifiers and different revisions are distinct. Empty arrays contribute a value; NULL forms a separate group and is excluded from `COUNT(DISTINCT)`. Grouped and distinct array columns retain their element and reference schema, including for empty or all-NULL input.

`MIN` / `MAX` on `STRING` also use the case-folded order. Hash aggregation's memory accounting includes the original value and any separate folded comparison key, including when an extremum is replaced. Already canonical ASCII reuses the stored value's payload for comparison. Only retained payloads contribute to the threshold for spilling new groups.

`MIN` / `MAX` on `BYTES` use unsigned lexicographic byte order. A present empty byte sequence is a value, precedes every non-empty sequence, and remains an empty BYTES result. It is distinct from the NULL result for an empty or all-NULL contributing population.

`MIN` / `MAX` retain the complete result schema of an Object, Array, or Vector input, including child fields, element types, and dimensions. An enclosing query can navigate a selected Object's children. Empty and all-NULL populations keep that declared schema while returning NULL.

Object and Array extrema compare their complete stored payload by unsigned lexicographic byte order; Array framing participates in the comparison. Vector extrema compare components numerically in the float total order, then by dimension, so `MAX` over `[1]` and `[2]` is `[2]` and vectors that differ only in signed zero tie (the first one seen is kept). Window extrema use the same order.

A present empty Object (`OBJECT ()`) remains a present `()` value when selected by `MIN` or `MAX`. Its zero-byte payload does not turn it into NULL, including when grouping uses an ordered index or spills to disk.

`MIN` / `MAX` of scalar `PINNED REFERENCES` compare and return the referenced identifier as an ordinary scalar. Stored revision numbers and pin framing are excluded from both ordering and the result. Empty identifiers remain values; String results preserve the encountered identifier's original casing. Whole pinned arrays remain complete array values, retaining their elements' identifiers and revisions and the array's reference metadata.

`SELECT DISTINCT` uses that same typed equality over the complete projected row, including when a correlated subquery cannot be decorrelated and must be evaluated per outer row. A plain DISTINCT's projected row is its selected keys' leaf columns whatever its WHERE (`SELECT DISTINCT [Dims].[W]` compares `W`, never the `Dims` parent), so a correlated predicate changes which rows are deduplicated, not the result's layout. Strings fold case-insensitively, Decimal scale variants compare equal, DateTimeOffset values compare by UTC instant, and floating-point NaN payloads and signed zero use the engine total order. DISTINCT is applied after WHERE (including the correlated predicate), aggregation, HAVING, and SELECT projection, and before ORDER BY / SKIP / LIMIT; it emits the first representation encountered by its selected input path, subject to the same plan-dependent rule as grouped aggregation.

## See also

- [Basic queries](queries.md) — `WHERE`, `ORDER BY`, `SELECT`
- [Common Table Expressions](cte.md) — staged aggregation patterns

## Detailed syntax and execution

### Aggregate-local ordering tuples

`COLLECT_ARRAY` and `JOIN_STRING` accept up to 64 inner `ORDER BY` field paths, each with its own `ASC` or `DESC`. Keys compare lexicographically using the field types' ordinary ordering. NULL sorts first for ASC and last for DESC on each key. Equal keys retain input traversal order; supply a final unique key when the result must be independent of the selected access path. Pinned reference sort keys compare their identifiers. Revisions affect ordering only when explicitly selected as an ordering field, such as `[Ref].[Revision]`. An empty Bytes identifier remains a present value and sorts separately from NULL.

```sql
FROM [App].[Tags] GROUP BY [Bucket]
SELECT [Bucket], JOIN_STRING([Tag], ',' ORDER BY [Score] DESC, [Tag] ASC, [Id] ASC) AS [Tags];
```

Sort paths can refer to computed fields, joined aliases, and CTE inputs. Per-aggregate FILTER still controls which rows contribute. Retained values and sort tuples share the aggregate spill store; sorting uses the same spillable sorter as single-key ordering. One multi-key row tuple has a 64 MiB bound, checked before allocation; this bounds transient row work independently of `MaxInMemoryStreamingState`.



<a id="syntax-group-by--having--aggregates"></a>

### GROUP BY + HAVING + aggregates

Without SELECT, GROUP BY publishes only grouping columns. Static schema binding uses that shape for empty inputs, scalar subqueries, CTEs, and set operations. Joined scalar subqueries preserve source aliases across every field-bearing clause, including grouping keys and window expressions. Scalar aggregate arguments and grouping expressions retain source binding even when an authored name resembles an internal input column; sibling expressions cannot supply values for otherwise absent inputs.

```sql
FROM [Shop].[Orders]
WHERE [Status] = 'paid'
GROUP BY [Region]
HAVING [Total] > 10000
SELECT [Region], COUNT(*), SUM([Amount]) AS [Total];
```

Aggregates: `COUNT(*)`, `COUNT([F])`, `COUNT(DISTINCT [F])`, `SUM([F])`, `AVG([F])`, `MIN([F])`, `MAX([F])`, `COLLECT_ARRAY([F] [ORDER BY [F] [ASC|DESC], …])`, `JOIN_STRING([F], 'separator' [ORDER BY [F] [ASC|DESC], …])`. One query may project at most 64 aggregate expressions. One `GROUP BY` tuple may contain at most 64 fields.

Without `AS`, aggregates take the [default aliases](#default-aliases) (`Count`, `CountDistinct_F`, `{Func}_{LeafField}`, `{Func}_` for an expression argument); same-leaf and expression-argument collisions require explicit distinct aliases.

Eligible correlated scalar equality subqueries with one unfiltered `SUM`, `AVG`, `MIN`, `MAX`, `COUNT(*)`, or `COUNT(field)` decorrelate through an aggregate CTE when the live outer and aggregate fields share a join equality domain. `SUM`/`AVG` require numeric fields and compare widened outputs through the same lossless numeric domain as ordinary MQL comparisons. `MIN`/`MAX` require equal scalar types or compatible numeric types. `COUNT` uses an internal null-preserving left form and compares against `COALESCE(grouped_count, 0)`, so only a missing group receives zero; a real `COUNT(field) = 0` stays a real group, and NULL correlation keys match NULL keys. A SUM/AVG/MIN/MAX SEMI rewrite also requires a provably non-null outer value. Other nullability shapes and cross-domain coercions remain on the exact per-row evaluator. Alias-qualified and fully qualified inner correlation, residual, and aggregate paths bind identically. An aggregate `FILTER` stays per-row and is never dropped.

For collection- or field-level polymorphism, every subtype that declares a field path used by `GROUP BY`, an aggregate, or an aggregate's inner `ORDER BY` must give that path one structurally identical schema. Same-named extension paths with different subtype schemas remain valid for row-wise evaluation, but are rejected for aggregation because they have no unambiguous output type or grouping order. Computed/window projection, wildcard `REPLACE` / `RENAME`, `ORDER BY`, and `JOIN` likewise require structurally identical schemas for a same-named extension path when they flatten or compare that leaf. Plain dotted SELECT of a nested polymorphic field preserves its containing Object and subtype envelope, so it does not reinterpret the leaf through one fixed scalar schema. Scalar and window expressions can read a compatible leaf beneath differently shaped parents: an Object subtype's `[Payload].[Amount]` remains INT32 even when an earlier subtype declares `[Payload] STRING`; the latter row yields NULL. The same leaf type is used for scalar-subquery binding and stored-view schema checks. Each projection admits at most 1,024 distinct extension inputs in private working slots, separately from its public output-column allowance. For JOIN output this covers plain fields, `AS` aliases, and field references nested inside computed projection expressions.

For aggregate queries, client/wire `IncludeTotalCount` reports aggregate output rows after `HAVING` and before paging: one per surviving group (or the single surviving scalar-aggregate row), never input documents or raw JOIN pairs.

`COLLECT_ARRAY` collects every non-null value into an `ARRAY` whose element schema mirrors the complete source field shape (including nested OBJECT/ARRAY, enum/reference metadata, and nullability) and uses the canonical public element name `Element`. Pinned scalar references retain their complete identifier-and-revision pairs in a pinned result array; collecting pinned arrays preserves them as nested array values. `JOIN_STRING` converts each non-null value using the built-in `TO_STRING` rules described above, then concatenates with the resolved STRING separator between each pair. The separator must be a STRING literal or parameter. Conversion preserves present empty values and does not change the types used to compare inner ordering keys. Both honour an optional inner `ORDER BY [F] [ASC|DESC], …` that sorts the buffered values before emit; without it, values appear in whatever order the engine scanned them. All-null groups emit SQL `NULL` (the buffer stays unallocated).

```sql
-- Concatenate per-group tags, alphabetised
FROM [App].[Tags] GROUP BY [Bucket]
SELECT [Bucket], JOIN_STRING([Tag], ',' ORDER BY [Tag] ASC) AS [Tags];

-- Collect per-group scores into a sorted INT32 array
FROM [App].[Tags] GROUP BY [Bucket]
SELECT [Bucket], COLLECT_ARRAY([Score] ORDER BY [Score] DESC) AS [Scores];
```

**`FILTER (WHERE …)`** — every aggregate accepts an optional SQL:2003 row filter that restricts which rows feed *that* aggregate's accumulator. Sibling aggregates in the same SELECT see the unfiltered row stream — the SQL-semantic difference from a top-level WHERE. The clause sits between the aggregate's closing `)` and any `AS` alias and accepts the full WHERE expression form (comparisons, AND/OR, IN, MATCH, IS NULL). Gating: only a TRUE predicate feeds the accumulator (FALSE and NULL both skip), with the same NULL-safe equality and complementary IN/NOT IN rules as WHERE (see [NULL semantics](queries.md#syntax-14-null-semantics--null-permissive-not-full-ansi-3vl)). `COUNT(*)`-with-filter returns 0 for non-matching groups; `SUM`/`AVG`/`MIN`/`MAX` return SQL NULL. Over a JOIN, parameter compatibility uses the final alias-remapped merged input schema and is rechecked on cached executions.

```sql
-- Open vs closed counts and total revenue from one pass
FROM [App].[Orders] GROUP BY [Region]
SELECT [Region],
       COUNT(*) FILTER (WHERE [Status] = 'open')   AS [Open],
       COUNT(*) FILTER (WHERE [Status] = 'closed') AS [Closed],
       SUM([Amount])                               AS [Total];
```

GROUP BY rule: every non-aggregated SELECT item must appear in `GROUP BY` (or be wrapped in an aggregate).

Grouping uses the engine-wide typed equality: strings compare case-insensitively, Decimal scale variants compare equal, DATETIME values compare by UTC instant, and floating-point NaN payloads / signed zero use the total order. Each group emits the first stored representation encountered by the selected input path rather than its internal canonical identity. The exact representative can differ across plans when source order is not semantically fixed, but it is always an equivalent value that occurred in the input.

A nested/qualified grouping key defaults to its leaf output name (`[Address].[City]` → `[City]`). Explicit SELECT can alias or omit same-leaf keys: `GROUP BY [Left].[Id], [Right].[Id] SELECT [Left].[Id] AS [LeftId], [Right].[Id] AS [RightId]`. Only final output names must be unique case-insensitively; hidden keys may share an aggregate alias. Without SELECT, same-leaf keys require an explicit projection to disambiguate their output. Expressions and HAVING bind source grouping paths and unambiguous key leaves. Bare aggregate aliases take precedence over same-named keys; qualify a source key to distinguish it: `GROUP BY [r].[Amount] SELECT [r].[Amount] AS [Key], SUM([r].[Amount]) AS [Amount]`. Window and scalar-subquery expressions follow the same binding rule. Computed SELECT aliases are not HAVING inputs. Source `[Meta].*` may be grouped or aggregated (and JOIN exposes outer Meta), but the aggregate output row itself has no implicit Meta object.

An explicit grouped SELECT returns only its selected columns. Grouping keys remain available to expressions even when they are not returned. Expressions can read grouping keys and aggregate output aliases, including through scalar subqueries or window functions; they cannot read ungrouped source fields or ungrouped source values. Sibling computed aliases use SELECT dependency order. Evaluation follows HAVING and precedes final ORDER BY and SKIP / LIMIT. Without DISTINCT, ORDER BY may also use an unselected grouping key or aggregate output.

Declare an aggregate as its own SELECT item to reference its output: `SUM([Amount]) AS [Total], [Total] * 2 AS [Doubled]`. Scalar aggregates also support constant and aggregate-derived expressions, including on empty input.

`SELECT DISTINCT` deduplicates the final projected tuple. With GROUP BY or scalar aggregates it is a separate stage after aggregation, HAVING, and SELECT; unselected grouping keys do not participate. Key aliases, scalar subquery projections, and expanded wildcards follow the same rule. ORDER BY refers to selected output columns; paging and requested TotalCount apply after deduplication. Ordinary grouping validation still applies. Both stages use the shared typed grouping and spill machinery. DISTINCT uses the engine-wide typed equality over the projected row even when a still-correlated WHERE is evaluated per outer row: strings compare case-insensitively, Decimal scale variants compare equal, DATETIME values compare by UTC instant, and floating-point NaN payloads / signed zero use the total order. DISTINCT runs after WHERE, grouping, HAVING, and projection, before ORDER BY / SKIP / LIMIT, and emits the first representation encountered by its selected input path, subject to the same plan-dependent rule as grouped aggregation. On a polymorphic collection, an extension projection is decoded from each surviving row's active subtype before that typed deduplication, including when the WHERE remains correlated and evaluates per outer row.

`HAVING` reuses `WHERE` operators (LIKE, IN, BETWEEN, etc.) over grouping columns and aggregate aliases. Its bound parameters are validated against those post-aggregation output types, even when an output alias collides with a different-typed source field. It requires `GROUP BY` or at least one SELECT aggregate for every source kind, including CTE, computed virtual, and `[Mnemic]` system sources; a standalone virtual-source HAVING is an error, never an ignored clause.

**Numerical precision.** `SUM` / `AVG` over `INT*` use `checked` Int64 arithmetic (overflow surfaces as a `User / INVALID_VALUE` runtime error); `AVG` widens the intermediate sum to `DECIMAL` so partitions whose average fits `FLOAT64` cannot overflow during accumulation. `SUM` / `AVG` over `FLOAT32` / `FLOAT64` use Kahan-Babuška-Neumaier compensated summation — first-order total error bounded by `O(ε·∑|xᵢ|)` (naive `+=` would be `O(n·ε·∑|xᵢ|)` and absorb small contributions into large running sums, e.g. `1e16 + 1.0 == 1e16`). Sliding-window invertible eviction shares the same compensation cell across `Add` and `Subtract` on the ordinary path. If removing a departed Float value leaves the two-cell state saturated or cancellation-dominated (including strongly opposing running/compensation cells), the active frame is rebuilt in row order so streaming and buffered plans retain the fresh-frame numerical result. IEEE saturation to `±∞` on true overflow is preserved.

See [`aggregation.md`](aggregation.md).

## Usage pitfalls


<a id="pitfall-skiplimit-on-aggregates-page-the-output-sql-semantics"></a>

### `SKIP`/`LIMIT` on aggregates page the OUTPUT (SQL semantics)

`SKIP`/`LIMIT` on an aggregate query always page the aggregate OUTPUT (the group rows), NEVER the aggregator input — and adding or removing an `ORDER BY` never changes an aggregate's value. Every matching row feeds the aggregates first; `SKIP`/`LIMIT` then slice the (optionally sorted) group rows. A scalar aggregate is ONE output row: `… WHERE [F] = 1 SELECT COUNT(*)` returns the full match count; `LIMIT 3` keeps that one row (the value is NOT clamped to 3); `SKIP 1` (any `SKIP` ≥ 1) pages past the single row and returns ZERO rows. Plan-independent (indexed or scan). There is no way to feed the aggregator a truncated input via `SKIP`/`LIMIT` — to bound scan work use a `WHERE` predicate.

<a id="pitfall-count-shortcut-is-conservative"></a>

### COUNT(*) shortcut is conservative

`SELECT COUNT(*)` on a non-temporal collection (no WHERE / no GROUP BY / no HAVING / no FILTER) hits the catalog row counter — O(1). Add **any** filter and the engine falls through to the full aggregator path. (Filters on aggregates with `FILTER (WHERE …)` also defeat the shortcut, by design.)

<a id="pitfall-aggregate-local-ordering-is-a-list"></a>

### Aggregate-local ordering is a list

Use `JOIN_STRING([Name], ',' ORDER BY [Rank] DESC, [Name] ASC, [Id] ASC)` or the same ordering list in `COLLECT_ARRAY`. Each direction applies to its own key; NULL sorts first ascending and last descending. Use a unique final key for a result independent of scan order. A SELECT-level ORDER BY sorts aggregate output rows and does not order the values within an array or joined string.
