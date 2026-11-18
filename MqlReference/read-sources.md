# Read sources

> **In this section** — a request for read sources asks a statement that reads rows to report, once for its whole result, which read each of its columns came from: per read how it chose its versions (current, at one instant, over a range, once per span, per enclosing row, or not known) with the instant or range it resolved, the stored collection it read, and where that read's `Meta.ValidFrom` / `Meta.ValidTo` lie in each row. It changes no row and never fails the statement.

```csharp
// The statement's text is sent as written; the request asks for its read sources.
var result = await client.ExecuteMqlAsync("FROM [App].[Orders] AS OF @at", parameters, new MqlRequestOptions { ReadSources = true });
ResultReadSources? sources = result.ReadSources;
```

## Asking for them

Read sources are a request option, not MQL: the request carries them beside the statement's text, as it carries the total count and a [page](queries.md#paging-a-statements-result), so the text stays as the user wrote it. The .NET clients take them as `MqlRequestOptions.ReadSources` on `IMnemicClient.ExecuteMqlAsync`, and any TCP client sets the read-sources flag of the `ExecuteMql` frame ([connecting](connecting.md#request-options)). They describe the whole statement: a subquery, a CTE body or a derived table asks for nothing of its own.

## Where it applies

Only a statement that reads rows takes them: a query (`FROM …`), a `WITH` statement, or a set operation (`UNION`, `INTERSECT`, `EXCEPT`), with or without a trailing [`OPTION (…)` list](cancellation.md#option---per-statement-options). Around any other statement the request is refused with `FEATURE_NOT_SUPPORTED` (`Read sources apply only to a statement that reads rows: FROM, WITH or a set operation.`) before the statement runs: `EXPLAIN` and `EXPLAIN ANALYZE`, every write with or without `RETURNING`, DDL, `LIST`, `SUBSCRIBE`, transaction control and a plugin's statement ([errors](errors.md)).

A read asked for its sources runs as the same read without them: its rows stream and are cancelled as the bare read's are, and the `TIMEOUT` and `MAXDOP` of its OPTION list hold for every row it produces ([cancellation](cancellation.md#option---per-statement-options)).

## What it reports

Asking for them changes no row, no column and no plan decision of the statement, and it never fails it. The result carries, once for the whole statement and never as a column of a row, the reads its columns came from and an entry per place in its rows that says which of them each value came from; a place whose values cannot be traced simply has no entry. The sources are built after the statement's rows are set up and before any row is sent, so a streamed result carries them too. A result whose request did not ask for them carries none. A page of the result reports exactly the statement's sources, since the page changes no column they describe.

### Reads

A read is one path from the result down to a relation that yields rows: a stored collection (a materialized view included), a system collection or a [`SERIES`](queries.md#series). A read is found by how the result reaches the relation, never by its name, so a CTE referenced twice and a self-join are two reads; reads whose reported values are all equal are reported once, since nothing reported could tell them apart (a self-join that selects no `Meta` reports one read). Only reads some reported place traces to are listed: a subquery that only filters rows (in `WHERE`) is not.

| Field | Meaning |
|---|---|
| `Mode` | How the read chose its versions (below). |
| `At` | `AsOf` only: the resolved instant, in UTC. |
| `From`, `To` | `Range` and `LastPer` only: the resolved `IN BETWEEN` bounds, in UTC, `From` before `To`. |
| `CollectionId` | The catalog identity of the stored collection read (a materialized view's own), as `[Mnemic].[Collections]` lists it; `0` for a system collection or a series. |
| `ValidFrom`, `ValidTo` | Where this read's `Meta.ValidFrom` and `Meta.ValidTo` lie in each row, as ordinal paths (below); absent when the row does not carry them. |

### Modes

| Construct | Mode | Reported with it |
|---|---|---|
| A collection that keeps no history, whatever clause the statement carries; a system collection; a `SERIES` | `Current` | nothing |
| A temporal `FROM` with no temporal clause | `Current` | nothing |
| A temporal `FROM` `AS OF` a literal, a parameter (a `DATETIME` or ISO text), `NOW`, `CURRENT_TIMESTAMP()` or an [instant expression](temporal.md#instant-expressions) | `AsOf` | `At` |
| A temporal `FROM` `IN BETWEEN a AND b` | `Range` | `From`, `To` |
| A temporal `FROM` `IN BETWEEN a AND b … LAST PER g` | `LastPer` | `From`, `To` |
| A temporal `JOIN` relation | the `AsOf` of the query it is joined in when that query reads `AS OF`; `Current` otherwise | as the mode says |
| A nested query (a scalar subquery, an APPLY body) whose instant names no column of the enclosing row | its own clause, as a `FROM` above | as above |
| A nested query or APPLY body whose `AS OF` or `IN BETWEEN` names a column of the enclosing row | `PerRow` | nothing |
| A read whose instant could not be resolved when the sources were built | `Unknown` | nothing |

- **A read with no temporal clause is `Current`, never the statement's `NOW`.** It samples the clock for each scan or probe and records no instant, so none is reported. `AS OF NOW` reports the statement's pinned `NOW`, the instant every `NOW` of the statement resolves to.
- **A join reads at its own query's `AS OF` only.** Under `IN BETWEEN` or `LAST PER` a joined temporal collection reads its current versions, not each range row's version, and a derived `FROM`'s clause is not the join's: `FROM (FROM [A] AS OF @t) [a] JOIN [B] ON …` reads `[B]` current ([temporal JOIN propagation](queries.md#join)).
- **A nested query reads at its own clause** ([nested queries](temporal.md#instant-expressions)): a scalar subquery or APPLY body with no clause of its own reads current versions whatever instant its enclosing query reads at.
- An instant is resolved from the statement's own parameters and pinned `NOW`, the values its read was resolved from.

### Entries

An entry is a place in the rows, its kind and its reads. A place is an ordinal path through the `OBJECT` fields of the result schema: a top-level column, or, under a raw or plain nested join's `outer` / `inner` envelope, the placement of one relation (`[0]` for `outer`, `[0, 1]` for `outer.inner`, …), or, for a derived table, CTE or APPLY body placed in an envelope, each of its columns. No entry's path is a prefix of another's; a value deeper than an entry (a nested field, an array element, a subtype field inside an envelope column) takes the entry whose path leads to it.

- **`Row`** — each value passed unchanged from one row of the one read: through projection, renaming, a wildcard slot, `ORDER BY`, `SKIP` / `LIMIT`, `NEAREST`, joins, a CTE, derived table or view, a scalar subquery whose single column is itself `Row`, an enclosing row's column that a nested query or APPLY body selects as it is (`Row` of the read that column came from), and `UNION ALL` when every branch's column is `Row` of reads that report equal values. That read's version paths in the same row give the version the value came from.
- **`Computed`** — each value was computed from rows of its reads and no single row stands behind it: `GROUP BY` and `DISTINCT` keys, aggregates (`MIN` / `MAX` of a key and `COLLECT_ARRAY` included), window functions, expressions (`CASE`, `COALESCE`, functions, `REPLACE` slots, literals), `UNION`, `INTERSECT`, `EXCEPT`, a `UNION ALL` whose branches read differently, and a recursive CTE's output. The reads are exact where the statement names them (a grouping key's column, an aggregate's arguments); otherwise they are every read the value's query reaches, which can only make a definite answer one that cannot tell. A nested query or APPLY body also reaches the enclosing row it reads, so an expression or an aggregate it computes, and the output of a recursive CTE it reads, names that row's read beside its own, while a grouping or `DISTINCT` key it computes names only the reads its column came from.
- **No entry** — the value has no traceable read, or the result is a raw or plain nested join of two or more edges returned as it is: its envelope follows the join tree the planner selected ([join order](queries.md#join)), so none of its places has an entry. Read it through a derived table (`FROM (…) [d]`), which keeps written order, to get an entry at each relation's placement; a page of the join keeps the envelope the plan publishes, as the join run unpaged does.

### Version paths

A read's `ValidFrom` / `ValidTo` path is where its `Meta.ValidFrom` / `Meta.ValidTo` lie in each row: the leaves of a `Meta` object the row carries for that read, or a projected `[Meta].[ValidFrom]` / `[Meta].[ValidTo]` under any alias, found by how each column reaches the read, never by its name, the first such place in column order. They are reported only where values pass unchanged from that read's rows, so a grouped or deduplicated value, or a scalar subquery's, carries none, and only at a place that holds them in every row: a set operation reports an enclosing row's bounds that a branch carries only where every branch carries them at that place, since that row's other values reach the rows of every branch, and a `LEFT JOIN`'s or `OUTER APPLY`'s relation reports none of the enclosing row's bounds it carries, since an unmatched row reads NULL there. Under `LAST PER` the row's `Meta` holds its span's bounds, so the paths give the span; the version a span shows was the one valid one tick before its end, the instant its last bucket was sampled at ([LAST PER](temporal.md#semantics)), so a version that starts exactly at the span's end is the next span's.

### Examples

Over temporal `[T].[Orders]` (`Id`, `CustomerId`, `PreviousId`, …) and `[T].[Customers]` (`Id`, `Name`, `ReferrerId`), both with versions, `t1 < t2 < t3`:

| Statement | Reads | Entries |
|---|---|---|
| `FROM [T].[Orders]` | 0: Orders `Current`, `ValidFrom` [4, 0], `ValidTo` [4, 1] | every column `Row` 0 |
| `FROM [T].[Orders] AS OF '<t2>' SELECT [Id], [CustomerId]` | 0: Orders `AsOf` t2 | [0], [1] `Row` 0 |
| `FROM [T].[Orders] IN BETWEEN '<t1>' AND '<t3>' SELECT [Id], [CustomerId]` | 0: Orders `Range` t1..t3 | [0], [1] `Row` 0 |
| `FROM [T].[Orders] [o] AS OF '<t2>' JOIN [T].[Customers] [c] ON [o].[CustomerId] = [c].[Id] SELECT [o].[Id] AS [Order], [c].[ReferrerId] AS [Referrer]` | 0: Orders `AsOf` t2; 1: Customers `AsOf` t2 | [0] `Row` 0, [1] `Row` 1 |
| the same under `IN BETWEEN '<t1>' AND '<t3>'`, adding `[o].[Meta].[ValidFrom] AS [From]` | 0: Orders `Range` t1..t3, `ValidFrom` [2]; 1: Customers `Current` | [0] `Row` 0, [1] `Row` 1, [2] `Row` 0 |
| `FROM [T].[Orders] [a] AS OF '<t2>' JOIN [T].[Orders] [b] ON [a].[PreviousId] = [b].[Id] SELECT [a].[Meta].[ValidFrom] AS [AFrom], [b].[CustomerId] AS [PrevCustomer]` | 0: Orders `AsOf` t2, `ValidFrom` [0]; 1: Orders `AsOf` t2 | [0] `Row` 0, [1] `Row` 1 |
| `FROM [T].[Orders] AS OF '<t2>' SELECT [CustomerId] UNION ALL FROM [T].[Orders] AS OF '<t2>' SELECT [CustomerId]` | 0: Orders `AsOf` t2 | [0] `Row` 0 |
| the same with the first branch `AS OF '<t1>'` | 0: Orders `AsOf` t1; 1: Orders `AsOf` t2 | [0] `Computed` 0, 1 |
| `FROM [T].[Orders] AS OF '<t2>' GROUP BY [CustomerId] SELECT [CustomerId], COUNT(*) AS [N]` | 0: Orders `AsOf` t2 | [0], [1] `Computed` 0 |
| `FROM [T].[Orders] [o] SELECT [Id], (FROM [T].[Customers] AS OF '<t1>' WHERE [Id] = [o].[CustomerId] SELECT [ReferrerId]) AS [Ref]` | 0: Orders `Current`; 1: Customers `AsOf` t1 | [0] `Row` 0, [1] `Row` 1 |
| `FROM [T].[Orders] [o] AS OF '<t2>' CROSS APPLY (FROM [T].[Customers] [c] WHERE [c].[Id] = [o].[CustomerId] SELECT [c].[Name] AS [Name], [o].[PreviousId] AS [Prev]) AS [x] SELECT [o].[Id], [x].[Name], [x].[Prev]` | 0: Orders `AsOf` t2; 1: Customers `Current` | [0] `Row` 0, [1] `Row` 1, [2] `Row` 0 |
| `FROM SERIES '<t1>' TO '<t3>' EVERY 1 DAY [d] CROSS APPLY (FROM [T].[Customers] AS OF [d].[Start] WHERE [Id] = 1 SELECT [ReferrerId]) AS [c] SELECT [d].[Start], [c].[ReferrerId]` | 0: the series `Current`, collection 0; 1: Customers `PerRow` | [0] `Row` 0, [1] `Row` 1 |

## Reading them

The sources travel in the result schema frame, once per result and before the first row, behind a result-only collection flag; a result without them sends exactly the schema frame it sends when they are not asked for. An in-process and a remote client read them alike, as `MqlResult.ReadSources`; the typed client and the LINQ provider never ask for them. They say where values were read, which is not the named type or enum identity a result column keeps for a composite or enum field ([composite types](composite-types.md)).
