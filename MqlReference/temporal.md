# Temporal queries

> **In this section** — `AS OF` (point-in-time), `IN BETWEEN` (range), instant expressions (computed and outer-row instants), per-period reads over a `SERIES`, `LAST PER` (downsampling) on temporal collections.

Temporal queries work on collections created with `(Temporal: TRUE)`.

## AS OF (point-in-time)

`AS OF <instant>` returns the document version valid at the instant. The instant is a timestamp string, `NOW`, an `@parameter` or an [instant expression](#instant-expressions):

```sql
FROM [HR].[Employees] AS OF '2024-06-15T00:00:00Z';
FROM [HR].[Employees] AS OF NOW;
FROM [HR].[Employees] AS OF DATEADD('MONTH', -1, NOW);
```

Timestamp strings without an offset are UTC. For example, `AS OF '2024-06-15T14:30:00'` is exactly the same instant as `AS OF '2024-06-15T14:30:00Z'`, regardless of the server time zone. This rule also applies to `IN BETWEEN`, `VALID FROM` / `TO`, `AT`, and string-valued timestamp parameters. Explicit `Z` or `+hh:mm` offsets are preserved.

The engine returns the version where `ValidFrom <= timestamp AND (ValidTo == 0 OR ValidTo > timestamp)` — i.e. the open-ended current version (`ValidTo == 0`) matches any timestamp at or after its `ValidFrom`.

## IN BETWEEN (temporal range)

`IN BETWEEN <instant> AND <instant>` returns all document versions overlapping with the half-open time range:

```sql
FROM [HR].[Employees] IN BETWEEN '2024-01-01T00:00:00Z' AND '2024-12-31T00:00:00Z';
FROM [HR].[Employees] IN BETWEEN '2024-01-01T00:00:00Z' AND NOW;
FROM [HR].[Employees] IN BETWEEN DATE_TRUNC('MONTH', @at) AND DATEADD('MONTH', 1, DATE_TRUNC('MONTH', @at));
```

`AS OF` and `IN BETWEEN` are mutually exclusive.

The range must be non-empty. A start equal to the end is refused with `INVALID_VALUE` (`A temporal range requires start < end`) when the statement runs, `EXPLAIN` included; a start after the end is refused with `INVALID_VALUE` (`bounds are reversed`), when the statement is parsed if both bounds are literals and when the bounds are evaluated otherwise. A zero-width `SERIES` step (a [date the zone skips](queries.md#pitfall-series-skipped-date)) therefore cannot be read with `IN BETWEEN [Start] AND [End]`.

With equality on the declared primary key, `IN BETWEEN` uses the `TemporalRange` index strategy to find overlapping versions in bounded batches. Additional predicates still apply to each version. Without that key predicate, the engine scans the heap. Transaction-owned pending index changes and snapshots that require older bodies retain the complete version-aware heap path.

Unordered collection reads can stream history; sorting, aggregation, exact total counts, and snapshot reconciliation retain their existing buffering rules. `ANALYZE` describes all retained versions, including past and future intervals. The primary-key distribution estimates the requested identity's complete history; it does not estimate the fraction overlapping a narrower time window.

`AS OF` and `IN BETWEEN` follow the `FROM` relation only: a `JOIN` relation takes no temporal clause of its own (a parse error), and the `FROM` clause's `AS OF` reads a joined temporal collection at the same instant ([temporal JOIN propagation](queries.md#join)). Under `IN BETWEEN`, with or without `LAST PER`, a joined temporal collection reads its current versions, not the version each range row was valid at, and a join takes only the `AS OF` of the query it is written in: `FROM (FROM [A] AS OF @t) [a] JOIN [B] ON …` reads `[B]` current. [Read sources](read-sources.md) report the versions each read of a statement chose. The `FROM` relation must be a real collection — attaching either clause to a CTE reference is rejected with `"AS OF cannot be applied to a CTE reference. Place the temporal clause inside the CTE definition instead."` (and the equivalent for `IN BETWEEN`). Put the temporal clause inside the CTE body so the seed / recursive query reads the right window:

```sql
WITH [Active] AS (
    FROM [HR].[Employees] AS OF '2024-06-15T00:00:00Z' WHERE [Active] = TRUE
)
FROM [Active] [e] JOIN [HR].[Departments] [d] ON [e].[DeptId] = [d].[Id];
```

## Temporal mutations

`INSERT` / `UPDATE` / `DELETE` / `CANCEL VERSION` all carry temporal semantics on temporal collections — see [DML](dml.md). A statement that targets a version has one target instant: UPDATE's `VALID FROM` (its NOW when omitted), an UPSERT's proposed `VALID FROM`, SEAL's `AS OF`, CANCEL VERSION's `AT` and REPLACE's NOW. `WHERE` matches each document by its version valid at that instant, and that version is the one the statement writes, so UPDATE ... VALID FROM changes exactly one version and keeps the earlier and later history; see [UPDATE](dml.md#update). DELETE ends a document at its `AT` and cancels the versions scheduled after it.

A collection becomes temporal, or stops being temporal, through `ALTER COLLECTION ... SET TEMPORAL TRUE|FALSE` in a migration: TRUE starts each document's history with an open version from the statement's time, and FALSE keeps each current version and removes the history. See [Change temporal mode](ddl.md#change-temporal-mode).

`SEAL` / `UNSEAL` may use `AS OF` to select a retained version, but that version identifies the whole temporal identity. The toggle is applied atomically to every retained past, current, and future version. Reads expose the identity's current control state as `[Meta].[Sealed]` even when the body is selected with an earlier `AS OF`; snapshot transactions retain the seal state visible when they began. Activation, expiry, gaps, and scheduled versions keep their original validity intervals. A seal blocks authored timeline mutations but never pauses the clock.

Current-version caching observes the same identity-wide toggle, including newer current and scheduled versions when `AS OF` selects an older version. Inside a transaction, the toggle also includes versions inserted earlier in that transaction. Subsequent reads observe the same seal state on every retained version, and rollback discards the toggle with the other writes.

A bounded `INSERT ... VALID FROM ... TO ...` preserves the old version's tail after the inserted window, including when both versions have the same start. An existing `Restrict` reference therefore does not veto a bounded overlay: the parent identity remains present. `UPDATE` inherits the end of the version it supersedes at `VALID FROM` or uses `CLEAR VALID TO`; it does not accept a new `VALID TO` endpoint. `DELETE` and `CANCEL VERSION` still enforce referential actions for an actual absence interval. Wall-clock expiry cannot be postponed by `Restrict`; use `Cascade` or `SetNull` when references require automatic expiry cleanup.

A `UNIQUE` index holds at every instant of valid time, like SQL:2011 `UNIQUE (..., period WITHOUT OVERLAPS)`: versions of one document share their keys, and two documents may not hold a key in overlapping intervals. Temporal writes are therefore checked over the interval they write — a backdated `UPDATE` over its past start, a scheduled version from its future start, a widening `CANCEL VERSION ... UPDATE PREVIOUS VALID TO` over the widened interval — and a key is free again from the end of its holder's last version. See [UNIQUE on a temporal collection](indexes.md#unique-on-a-temporal-collection).

## Instant expressions

Every timestamp position takes an instant: `AS OF`, both `IN BETWEEN` bounds, and the [DML instants](dml.md#instant-expressions-in-writes) (`VALID FROM` / `TO`, `CREATEDAT`, `DELETE ... AT`, `CANCEL VERSION AT`, `UPDATE PREVIOUS VALID TO`, `SEAL` / `UNSEAL ... AS OF`). A bare timestamp string, `NOW` or `@parameter` is an instant as before; any other scalar expression is an instant expression:

```sql
FROM [HR].[Employees] AS OF DATEADD('MONTH', -1, NOW);
FROM [HR].[Employees] AS OF DATE_TRUNC('MONTH', @at);
FROM [HR].[Employees] AS OF CAST('2024-06-15' AS DATETIME);
FROM [HR].[Employees] IN BETWEEN DATE_TRUNC('MONTH', @at) AND DATEADD('MONTH', 1, DATE_TRUNC('MONTH', @at));
```

**What it may contain.** Literals, parameters, `NOW`, scalar function calls (`DATEADD`, `DATE_TRUNC`, `CAST`, `COALESCE`, …), arithmetic, concatenation and `CASE`, and, inside an APPLY body or a correlated subquery, the columns of the enclosing query's row (below). It reads no field of the rows its own clause selects, holds no nested query and no window function: a field of its own rows and a nested query are refused with `INVALID_STATEMENT`; a window function is a `SYNTAX_ERROR` in text, whose instant grammar has no `OVER`, and `INVALID_STATEMENT` only in a programmatically built statement. An instant is read at the precedence of a `BETWEEN` operand, so `IN BETWEEN a AND b` keeps `AND` as its separator; parenthesize a comparison or a logical expression.

**When it is evaluated.** Once per execution, with the call's parameters and the statement's single `NOW`, before any row is read; in an APPLY body or a correlated subquery, once per outer row. A prepared statement served from the plan cache evaluates it again for each execution, so a `NOW`-relative or parameter-reading instant never keeps an earlier execution's value.

**What it must produce.** Its value is admitted by the rule a timestamp parameter's value is: a `DATETIME` is used as given, and text is read by the DATETIME text rule (offset-less text is UTC). A `NULL` and text that names no instant are refused with `INVALID_VALUE`; any other type (a number, a `DATEONLY`, a BOOL, …) with `TYPE_MISMATCH`. A type that can never name an instant is refused before rows are read, by execution, schema discovery and `EXPLAIN` alike; a value's refusal is reported when the instant is evaluated. A DML bound also keeps its tick-zero rule.

**Reading the outer row.** In an APPLY body or a correlated subquery the instant may read the enclosing query's columns, through its alias or its unaliased collection name, so each outer row reads the version valid at its own instant:

```sql
-- Each order's lines as they were when the order was placed, and the day before.
FROM [Sales].[Orders] [o]
CROSS APPLY (
    FROM [Sales].[Lines] AS OF [o].[PlacedAt]
    WHERE [OrderId] = [o].[Id]
    SELECT [Sku], [Price]
) AS [now]
OUTER APPLY (
    FROM [Sales].[Lines] AS OF DATEADD('DAY', -1, [o].[PlacedAt])
    WHERE [OrderId] = [o].[Id]
    SELECT [Price]
) AS [before];

-- Bills whose tariff was 100 on the bill's own date.
FROM [Billing].[Bills] [b]
WHERE EXISTS (FROM [Billing].[Tariffs] AS OF [b].[At] WHERE [Id] = [b].[TariffRef] AND [Rate] = 100)
SELECT [Id];
```

The outer column is bound as the subquery's correlation parameter for each outer row, as if the body were run by hand with `@at = <that row's value>`; a body whose only outer read is its instant still runs once per outer row. An outer column must be a `DATETIME` or text: another declared type is refused with `TYPE_MISMATCH` before any row is read, and an outer row whose value is `NULL` fails the statement with `INVALID_VALUE`. A query that is not nested has no enclosing row, so a field named in its instant is refused as a field of its own relation.

**A nested query reads at its own instant.** An `EXISTS`, `NOT EXISTS`, `IN` or scalar subquery, and an APPLY body, read their own relation at their own `AS OF` or `IN BETWEEN`, and without one at its current versions, whatever instant the enclosing statement reads its `FROM` relation at: under `FROM [T].[Orders] [o] AS OF '2024-06-01T00:00:00Z'`, `EXISTS (FROM [T].[Tariffs] [t] WHERE [t].[TariffId] = [o].[TariffId])` tests the current tariff. The engine may run the subquery set-wise as a generated join or once per outer row; both read the same versions, so adding a conjunct never changes its result. A relational `JOIN` edge, by contrast, reads its relation at its query's `AS OF` instant, and at its current versions under `IN BETWEEN` or `LAST PER` ([read sources](read-sources.md#modes)).

## Per-period reads

A [`SERIES`](queries.md#series) of calendar steps drives an APPLY body whose instants are expressions over each step's `[Start]` and `[End]`, so one statement reads a temporal collection once per period. Each meaning of "per month" is one written instant:

| Read | Instant | Reads |
|---|---|---|
| Start of the period | `AS OF [m].[Start]` | the state at its first instant, a change at exactly 00:00 on the 1st included |
| Last day | `AS OF DATEADD('DAY', -1, [m].[End])` | the state at 00:00 on the period's last day |
| Closing state | `AS OF DATEADD('SECOND', -1, [m].[End])` | the state one second before the next period |
| Anyone during it | `IN BETWEEN [m].[Start] AND [m].[End]` | every version overlapping the half-open period |

```sql
-- Highest salary at the start of each month of 2026, in Stockholm
FROM SERIES '2026-01-01T00:00:00+01:00' TO '2027-01-01T00:00:00+01:00' EVERY 1 MONTH IN TIMEZONE 'Europe/Stockholm' [m]
CROSS APPLY (
    FROM [HR].[Employees] AS OF [m].[Start]
    ORDER BY [Salary] DESC, [Name]
    LIMIT 1
    SELECT [Name], [Salary]
) AS [top]
SELECT CAST([outer].[Start] AS DATEONLY) AS [Month], [inner].[Name], [inner].[Salary];
```

`AS OF [m].[End]` reads the next period's first instant, so the closing state reads one second earlier. `DATEADD`'s smallest unit is the second, so a version that starts inside a period's final second belongs to the next period in the closing-state read. With `IN TIMEZONE`, write the series start in the zone's offset to step from local midnight: the start is an instant, and a UTC start in a zone that is ahead of UTC starts each period at that instant's local time. More examples, ties included, are in [examples](examples.md#highest-salary-per-month).

## LAST PER (temporal downsampling)

Collapse a temporal range into one row per **contiguous span during which the same version was the last active version of its bucket**. Exactly what billing systems need: given "all versions valid in March", return the distinct end-of-day snapshots as spans.

For polymorphic collections, document identity comes from the declared primary key in the base fields. The engine groups and samples each identity independently of its subtype's storage envelope.

```
FROM [Namespace].[Collection]
IN BETWEEN <start> AND <end>
[WHERE expression]
LAST PER <granularity> [STARTING <weekday>] [IN TIMEZONE '<tz>']
```

- **`<granularity>`** — `MINUTE | HOUR | DAY | WEEK | MONTH | YEAR`
- **`STARTING <weekday>`** — `MONDAY | TUESDAY | ... | SUNDAY`. Only valid with `WEEK`. Default is `MONDAY` (ISO-8601).
- **`IN TIMEZONE '<tz>'`** — `'Z'`, an exact fixed offset in `'+HH:MM'` / `'-HH:MM'` form from `-14:00` through `+14:00`, or an IANA zone id like `'Europe/Stockholm'` / `'America/New_York'`. Fixed offsets require two-digit hours and minutes and whole-minute precision. Default UTC. IANA zones are DST-aware. A zone id is resolved through the host operating system's time-zone database, which supplies its rules: Linux and macOS read the IANA tz database, Windows maps the id to its own registry zones, whose history differs for some transitions (Windows places Pacific/Apia's skipped date of December 2011 and São Paulo's 2018 DST gap differently from IANA). Boundaries near such a historical transition can therefore differ between a Windows and a Linux server; a Windows zone name (`'W. Europe Standard Time'`) is accepted as well. An unknown id is `INVALID_VALUE`.

### Semantics

For each bucket in `[start, end)` the engine samples the document *at one tick before bucket-end* (so a version starting exactly at a boundary belongs to the next bucket). Consecutive buckets with the same sampled version collapse into a single output row. A version that's superseded within the same bucket (created in the morning but replaced before end-of-day) never appears.

Native WHERE comparisons use the sampled source version's metadata before the result receives its span bounds, whatever else reads the spans: a join, a correlated `EXISTS`, a function or a computed column elsewhere in the WHERE. Each `AND` term of the WHERE that reads the FROM collection's `[Meta].[ValidFrom]` or `[Meta].[ValidTo]` is evaluated on the sampled version, and only the other terms filter the spans. Such a term combines, under `AND` / `OR`, predicates that name the FROM collection's column before the value (`[Meta].[ValidFrom] >= @from`):

- over the FROM collection alone, comparisons (`=`, `<`, `LIKE`, …), memberships (`IN`) and `MATCH`es of its stored columns (a `STORED` computed column included) and metadata with values (literals, parameters, `NOW`) or with a nested query that reads no outer row, which runs once before the walk;
- under a join, only comparisons and memberships of the FROM collection's metadata, top-level scalar columns and the elements of a top-level scalar array (`[Tags][]`) with values.

A quantified comparison of an array's elements may write the value first (`'x' = [Tags][] ANY`). In a correlated body, an enclosing row's column is a value too: each outer row's run binds it as a parameter, as the same body run by hand with that parameter does ([WHERE](queries.md#syntax-where)), except as an unquantified `IN` list member or a `CONTAINS` / `STARTSWITH` operand, which makes its predicate an expression. A term that reads the validity in any other form is rejected with `INVALID_STATEMENT`, because the spans carry their own bounds:

- an expression or function: a comparison of two columns, one written with the value first (`@from <= [Meta].[ValidFrom]`), and an unquantified `IN` list or a `CONTAINS` / `STARTSWITH` operand that names an enclosing row's column, although the same body run by hand with a parameter there is accepted;
- `NOT`;
- `EXISTS`, or a nested query that reads the FROM row, its validity included;
- a virtual computed column;
- another joined relation's column;
- under a join, any other column form: a computed, nested, whole array or vector column, a `MATCH` or a nested query;
- under a join, a polymorphic FROM collection, whatever the term's form.

A term that names a column or metadata member its relation does not declare is `UNKNOWN_FIELD` before any of these refusals, under a join as over a single source, whether or not it reads the validity: a join refuses an undeclared name in every clause on every route ([JOIN](queries.md#join)).

```sql
-- The sampled version's start, beside a join and a function
FROM [Sales].[Subscriptions] [s]
IN BETWEEN @from AND @to
JOIN [Sales].[Plans] [p] ON [s].[PlanId] = [p].[Id]
WHERE [s].[Meta].[ValidFrom] >= @from AND UPPER([p].[Name]) <> 'TRIAL'
LAST PER DAY;

-- Rejected: the term mixes the validity with another collection's column
FROM [Sales].[Subscriptions] [s]
IN BETWEEN @from AND @to
JOIN [Sales].[Plans] [p] ON [s].[PlanId] = [p].[Id]
WHERE [s].[Meta].[ValidFrom] >= @from OR [p].[Name] = 'Trial'
LAST PER DAY;
```

`LAST PER` rejects native Boolean preparation above **1,024 disjunctive groups** or **8,192 predicate occurrences** when more than one group is required. Repeated mixed-field disjunctions joined by `AND` can exceed this limit even in a short query; pure conjunctions retain their ordinary authored-expression bound. EXPLAIN enforces the same limit, including when aggregation or NEAREST consumes the spans.

### Carry-forward

A version that started before `<start>` and is still active at `<start>` naturally fills the early buckets. **No back-fill** for buckets before the first sample where the document existed — gaps in the output reflect periods when the document didn't exist or didn't match `WHERE`.

### Timezone

The range bounds stay as-written (their offsets are honored); `IN TIMEZONE` shifts only bucket **boundaries**. For an IANA zone like `Europe/Stockholm`, DST transitions produce a 23-hour or 25-hour bucket on the transition day — transparently handled for `DAY` / `WEEK` / `MONTH` / `YEAR` (which step by wall-clock dates in the zone). `MINUTE` and `HOUR` step by fixed 60-second / 3600-second deltas after the first boundary aligns to the zone's wall clock, so they ignore DST transitions — a one-hour bucket that contains a DST shift stays an hour wide in UTC, which is what fine-grained audit-trail use cases want. If the start falls in a repeated DST hour, initial alignment retains that occurrence's offset; minute boundaries in the first occurrence are not skipped.

For calendar units, if an aligned boundary lands in a skipped local-time interval, the boundary moves to the first valid wall-clock instant after the gap. This applies to large historical offset/dateline changes as well as ordinary DST jumps; `LAST PER` does not fail merely because the declared zone skipped several hours or an entire local date. Subsequent boundaries retain their original calendar alignment; a shifted midnight does not shift every following day. The nominal calendar always advances, even if an offset change maps a boundary back into the preceding date or year.

The calendar is generated on demand in constant space and observes statement cancellation between boundaries. Large ranges do not allocate all their buckets before scanning candidate documents. Candidate identity deduplication retains its existing spill policy.

At the minimum representable UTC instant, a positive-offset zone can place the preceding aligned local boundary outside the timestamp domain. That prior boundary is skipped and the first bucket remains clipped to the written range start; enumeration resumes at the next representable aligned boundary. Minute and hour grids retain fractional-hour timezone alignment at both timestamp limits.

### Examples

```sql
-- Sweden, daily billing snapshot (IANA zone is DST-aware)
FROM [Sales].[Subscriptions]
IN BETWEEN '2026-03-01T00:00:00+01:00' AND '2026-04-01T00:00:00+02:00'
WHERE [CustomerId] = @cid
LAST PER DAY IN TIMEZONE 'Europe/Stockholm';

-- Simple fixed offset (no DST)
FROM [Sales].[Subscriptions]
IN BETWEEN @from AND @to
WHERE [CustomerId] = @cid
LAST PER DAY IN TIMEZONE '+01:00';

-- Weekly usage rollups, Sunday-start, US Eastern (DST-aware)
FROM [Sales].[Usage]
IN BETWEEN @from AND @to
WHERE [AccountId] = @aid
LAST PER WEEK STARTING SUNDAY IN TIMEZONE 'America/New_York';

-- Monthly — calendar-month buckets aligned to 1st at 00:00 local
FROM [Sales].[Subscriptions]
IN BETWEEN '2026-01-01T00:00:00Z' AND '2026-07-01T00:00:00Z'
WHERE [CustomerId] = @cid
LAST PER MONTH;

-- Minute granularity for fine-grained audit trails
FROM [Audit].[Sessions]
IN BETWEEN @windowStart AND @windowEnd
WHERE [UserId] = @uid
LAST PER MINUTE;
```

### Requirements

- Collection must be temporal (`(Temporal: TRUE)`). Non-temporal collections are rejected with a clear error.
- `IN BETWEEN` is required. Without a range, there are no buckets to walk.
- `STARTING` is only valid with `WEEK` — a parse error otherwise.
- `JOIN`, `GROUP BY`, aggregates, HAVING, ordering, paging and projection compose with the generated span rows. A join is composed over the spans whatever reads them, its plain merged rows included, so like any composed join its plain `SELECT` publishes flat leaf-named columns, its implicit row is the written-order `{outer, inner}` row, and the temporal side's `Meta.ValidFrom` / `Meta.ValidTo` are the span bounds outside the WHERE terms the walk samples (Semantics, above). Bucketing chooses/coalesces versions first; aggregation summarizes those spans rather than raw versions or the number of calendar buckets. `IncludeTotalCount` reports the consumer's row count after filtering/grouping and before SKIP / LIMIT. EXPLAIN accepts the same temporal joins and aggregations as execution.

## See also

- [DML](dml.md) — temporal INSERT, UPDATE, DELETE, SEAL, UNSEAL, CANCEL VERSION
- [Practical examples](examples.md#temporal-audit-trail) — audit trail, bitemporal correction, billing rollups

## Detailed syntax and execution

`[Meta].[CreatedAt]` is the authored revision's UTC creation timestamp, independent of `ValidFrom`. Backdated and future versions normally record the time they were written; `CREATEDAT` on INSERT/UPSERT can preserve an imported creation timestamp. Splitting an interval preserves the source revision's timestamp on both surviving fragments. See [creation timestamps](dml.md#createdat-creation-timestamps).

Fragments of one authored version share its revision. Each retained revision remains addressable by `PIN(id, revision)`, including a tail left when an overlay replaces the original start. New authored versions and REPLACE corrections use revisions above the identity's entire retained timeline. Removing the final fragment applies the pinned reference's `ON REMOVE` policy atomically; see [pinned references](ddl.md#pinned-references-pinned-references). The new body cannot pin a revision removed by the same mutation.


<a id="syntax-as-of--in-between-temporal-collections-only"></a>

### AS OF / IN BETWEEN (temporal collections only)

```
AS OF <instant>
IN BETWEEN <instant> AND <instant>

<instant> ::= 'timestamp' | NOW | @parameter | <instant expression>
```

```sql
FROM [HR].[Contracts] AS OF '2024-06-01T00:00:00Z';
FROM [HR].[Contracts] AS OF NOW;
FROM [HR].[Contracts] AS OF DATEADD('DAY', -1, @at);
FROM [HR].[Contracts] IN BETWEEN '2024-01-01' AND '2024-12-31';
```

Offset-less timestamp strings are UTC on every surface, including temporal literals and parameters, text an instant expression produces, DateTimeOffset document values, comparisons, `CAST(... AS DATETIME)`, and `TO_TIMESTAMP`. (`DATETIME` is not a legal primary-key type.) Explicit `Z` or `+hh:mm` offsets are preserved. An instant expression's contents, evaluation and admission are described in [Instant expressions](#instant-expressions).

Rejected on non-temporal collections. See [`temporal.md`](temporal.md).

<a id="syntax-temporal-bucketing--last-per"></a>

### Temporal bucketing — `LAST PER`

```sql
FROM [Sensors].[Readings]
IN BETWEEN @from AND @to
WHERE [SensorId] = @sensor
LAST PER MINUTE;
```

Granularities: `MINUTE`, `HOUR`, `DAY`, `WEEK [STARTING <day>]`, `MONTH`, `YEAR`. Optional `IN TIMEZONE '<tz>'` accepts `'Z'`, an exact `'+HH:MM'` / `'-HH:MM'` fixed offset from `-14:00` through `+14:00`, or an IANA zone id, resolved through the host's time-zone database ([`IN TIMEZONE`](#last-per-temporal-downsampling)). Fixed offsets require two-digit hours and minutes and whole-minute precision. Requires a temporal collection and IN BETWEEN. It samples the last effective version per declared primary key at each bucket's final tick and coalesces adjacent buckets choosing the same version into spans. Calendar boundaries inside a zone's skipped wall-time interval advance to the first valid instant after the gap, including historical/dateline-scale jumps; minute/hour stepping remains fixed-duration UTC stepping. At the minimum UTC instant, an unrepresentable prior boundary in a positive-offset zone is skipped: the first bucket stays clipped to the written range start and enumeration resumes at the next representable aligned boundary. See [`temporal.md`](temporal.md). Joins and grouping/aggregates compose with the generated spans. HAVING, ORDER BY, SKIP / LIMIT and SELECT use the ordinary consumer pipeline. COUNT(*) counts span rows, not calendar buckets; wire/client IncludeTotalCount reports the filtered/grouped consumer count before paging. EXPLAIN accepts the same temporal joins and aggregations as execution. Scalar/correlated HAVING is a residual stage after aggregation and before paging in the static plan.

## Usage pitfalls


<a id="pitfall-temporal-clauses-only-apply-to-real-temporal-collections"></a>

### Temporal clauses only apply to real temporal collections

`AS OF` / `IN BETWEEN` / `LAST PER` on a CTE reference or a `[Mnemic].*` system collection is an **error** — CTE results and system collections are snapshots with no version history. Put the temporal clause inside the CTE definition (`WITH [c] AS (FROM [T] AS OF ...) FROM [c] ...`), not on the outer reference. A `WITH` wrapper alone does not impose this restriction: if the main `FROM` still names a physical temporal collection, its temporal clauses remain valid.

<a id="pitfall-as-of--in-between-on-non-temporal-collections"></a>

### `AS OF` / `IN BETWEEN` on non-temporal collections

Rejected at the executor with a clear error. The temporal clauses only mean something on `(Temporal: TRUE)` collections. `LAST PER` on a non-temporal collection (`WRONG_OBJECT_TYPE`) or without `IN BETWEEN` (`INVALID_STATEMENT`) is rejected the same way, before any other clause's mistake, by statement validation, `EXPLAIN` and execution alike.

<a id="pitfall-there-is-no-automatic-temporal-history-retention-policy"></a>

### There is no automatic temporal-history retention policy

Past and current versions remain retained; ordinary `DELETE` closes validity and removes only versions scheduled after its end. `CANCEL VERSION` physically removes one future version, but MQL has no general historical purge short of `ALTER COLLECTION ... SET TEMPORAL FALSE`, which removes the whole history and leaves the collection standard. There is no `Retention` CREATE option and no `ALTER COLLECTION … SET/DROP RETENTION` — those are not part of the grammar.

<a id="pitfall-a-bounded-versions-end-is-not-necessarily-parent-expiry"></a>

### A bounded version's end is not necessarily parent expiry

Temporal INSERT overlays its interval and preserves the previous tail, so a referenced parent can remain continuously valid after the new version ends. Restrict does not reject that valid overlay. UPDATE inherits the end of the version it supersedes or clears it; to close parent validity, use DELETE with its ordinary referential checks. CANCEL VERSION also checks the actual gap it creates. Wall-clock expiry cannot be rejected retroactively: Restrict is reported and progress advances; Cascade or SetNull performs automatic cleanup.

<a id="pitfall-cancel-version--update-previous-cannot-create-an-invalid-timeline"></a>

### `CANCEL VERSION ... UPDATE PREVIOUS` cannot create an invalid timeline

The restored predecessor's bounded `VALID TO` must be after its `ValidFrom`. If a version follows the canceled target, restoration cannot extend past that successor; `VALID TO NULL` is rejected there because it would overlap it. The whole cancellation remains atomic on either validation failure. When `UPDATE PREVIOUS SET` rewrites the predecessor, the replacement revision is allocated above the entire live PK timeline; validity-neighbor order is not revision order, and no canceled or later-sibling identity is reused.

<a id="pitfall-a-unique-key-is-held-over-valid-time-not-only-now"></a>

### A unique key is held over valid time, not only now

On a temporal collection a version that ended, or one scheduled for the future, still holds its unique keys over its own interval. Reusing an ended document's email `VALID FROM` its end succeeds, but reusing it from an earlier start, or taking a key a scheduled version of another document will hold, fails with `UNIQUE_VIOLATION`. `ON CONFLICT` on a unique index probes the documents current at the statement's `NOW`, so a scheduled holder is not a conflict target and the insert can still fail with `UNIQUE_VIOLATION` ([UPSERT](dml.md#insert--on-conflict-upsert)).

<a id="pitfall-expiry-is-eventual-and-uses-ordinary-delete"></a>

### Expiry is eventual and uses ordinary DELETE

`WITH TTL '7 DAYS'` is an elapsed lifetime; `WITH EXPIRY` uses the indexed DATETIME as the absolute deadline. NULL has no deadline. Only one stored scalar DATETIME key is valid. Use offline CREATE INDEX inside a migration or append `WITH (ONLINE = ON)` for standalone creation. Both expiry and online options follow INCLUDE / WHERE. Altering a deadline or partial membership is rechecked before deletion. A BEFORE trigger, sealed row, or FK restriction can keep a row stored; rejected rows are retried without blocking later candidates. Temporal expiry ends the document as DELETE does: it closes the current version, cancels the versions scheduled after it, and retains history. Dropping the index removes the policy; recreating it changes the lifetime. MQL hosts run one worker per engine; an engine-only host must initialize MQL for automatic expiry.

Stored computed source values are eligible for row-local incremental materialized views. Stored computed DATETIME values may also define TTL/EXPIRY indexes.

<a id="pitfall-temporal-history-estimates-describe-retained-versions"></a>

### Temporal history estimates describe retained versions

`TemporalRange` serves `IN BETWEEN` with equality on the declared primary key. Its estimate uses the key's retained-version frequency, not the point-seek bound of one. A narrower interval or residual WHERE condition can return fewer rows. `ANALYZE` counts retained past and future versions too; those counts are not the number of currently visible identities. Missing or stale statistics leave the history estimate unknown. Pending transaction changes and snapshots needing older bodies can make execution use the version-aware heap instead.
