# Parameterized queries

> **In this section** — `@parameters` as a substitute for literal values, the supported parameter types, and every clause that accepts them.

Use `@paramName` for parameterized values. Parameters are passed separately from the query text, preventing injection:

```sql
FROM [Store].[Products] WHERE [Category] = @category AND [Price] < @maxPrice;
INSERT INTO [Test].[Items] ([Id]: @id, [Name]: @name);
UPDATE [Test].[Items] SET ([Score]: @newScore) WHERE [Name] = @name;
```

Parameter names are exact, ordinal, and case-sensitive: `@value` binds only a key named `value`, not `VALUE`. This is identical for local execution and the binary protocol; a caller-supplied dictionary comparer cannot change language semantics. Execution and EXPLAIN use the same exact lookup. Correlated subqueries, lateral APPLY, and secondary-unique UPSERT probes preserve case-distinct caller bindings while adding their synthetic parameters; lateral schema discovery follows that same rule. Synthetic-name allocation skips every caller-owned name, so a legal binding such as `@__corr_0` is never overwritten by correlation machinery. A binding map contains at most 65,535 entries, and each parameter name is strict UTF-8 no longer than 255 encoded bytes. The in-process executor applies the same wire bounds before execution, so a locally accepted query is always representable by a remote client.

Parameters in IN clauses can be mixed with literals:

```sql
FROM [Store].[Products] WHERE [Status] IN (@status1, @status2, 'archived');
```

## Supported parameter types

`string`, `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `decimal`, `double`, `float`, `char`, `bool`, `Guid`, `DateTimeOffset`, `DateTime`, `DateOnly`, `TimeOnly`, `byte[]`, `float[]`. `float[]` is a vector parameter and retains its raw `FLOAT32` components; the field or query context validates its required dimension and finite values.

Every parameter that canonicalizes to a string (`string`, `char`, or a custom type's fallback `ToString()`) must be well-formed UTF-16. Unpaired surrogates are rejected by both embedded execution and the strict-UTF-8 wire encoder; valid surrogate pairs such as emoji are accepted. Malformed values are never silently replacement-encoded as U+FFFD.

When a bound parameter appears in a computed `SELECT` expression, its binding determines the projected wire type: `sbyte` → `INT8`, `byte` / `short` → `INT16`, `ushort` / `int` → `INT32`, `uint` / `long` → `INT64`, `ulong` → `INT64` when it fits and otherwise `DECIMAL`, `decimal` → `DECIMAL`, `double` / `float` → `FLOAT64`, `char` → `STRING`, `bool` → `BOOL`, `Guid` → `GUID`, `DateTimeOffset` / `DateTime` → `DATETIME`, `DateOnly` → `DATEONLY`, `TimeOnly` → `TIMEONLY`, `byte[]` → `BYTES`, and `float[]` → `VECTOR`. Values are not formatted through `STRING`; in particular, bytes remain raw bytes rather than Base64 text and vectors remain raw components. A null binding has no nominal type when combined with another value arm (`COALESCE([GuidField], @null)` remains `GUID`) and falls back to `STRING` when projected by itself.

`DateTime` maps to `DateTimeOffset` per its `Kind`:
- `DateTimeKind.Utc` → offset `+00:00` at the same wall-clock.
- `DateTimeKind.Local` → host's local TZ offset preserved (wall-clock honoured). Downstream wall-clock projections (`EXTRACT('day', …)`, `DATE_TRUNC`, `DateOnly` / `TimeOnly` coercion) see the user's local date components, not a UTC-shifted variant that would silently cross the midnight boundary.
- `DateTimeKind.Unspecified` → treated as UTC (offset `+00:00`), the host-machine-independent choice so cached MQL queries produce the same UtcTicks on every host. Explicit local interpretation requires `DateTimeKind.Local` or a `DateTimeOffset` literal.

A `string` used where MQL expects a DateTimeOffset, DATEONLY, or TIMEONLY is read only in the ISO 8601 forms of [temporal text](document-literals.md#temporal-text): culture-shaped text such as `"06/15/2024"` is refused, with `TYPE_MISMATCH` where it is compared and `INVALID_VALUE` where it is written. It follows the same host-independent rule: an explicit offset is preserved, while an offset-less value such as `"2024-06-15T14:30:00"` is UTC. This is consistent for filters, document values and primary keys, temporal clauses, cached parameter rebinding, `CAST(... AS DATETIME)`, and `TO_TIMESTAMP`.

DATEONLY and TIMEONLY also accept timestamp strings, including forms such as `"0001-01-01 00:00:00"`. They extract the written date or time component; text that writes only a time of day (`"10:00:00"`) is no DATEONLY and no DATETIME — it is never read as a time on the current date — and text that writes only a date is no TIMEONLY; an absent offset defaults to UTC during timestamp parsing. Valid boundary values are therefore accepted independently of the server time zone. An explicit offset is preserved, and a timestamp whose UTC instant is outside the representable range remains invalid.

Null parameters produce null values (equivalent to `= NULL`). Parameter references in document literals skip type validation (resolved at execution time). CLR types outside this set (for example custom records) fall back to invariant `IFormattable.ToString()` or ordinary `ToString()` and travel as string parameters. Convert custom values to one of the listed types client-side if you need exact wire-level semantics.

Prepared-query plan reuse is semantics-transparent. The engine verifies the exact parameter name, type, and null shape independently of binding enumeration order before reusing a plan; an internal hash collision cannot reuse a plan for a different binding shape. Plans are also bound to the exact collection schema generation: ALTER or DROP/recreate cannot reuse field encodings from a retired schema, even when another session performs the DDL and the recreated collection has the same name and integer schema version. Compatibility is revalidated for each current value even when its host-language type matches a cached binding: a prior valid narrow integer, parseable Bool / GUID / date string, or finite Decimal input does not authorize a later out-of-range, unparsable, or non-finite value of the same host type. When several values are refused, a cached-plan hit reports the refusal a cold execution of the same values reports. Parameter values and statement-pinned `NOW` are bound afresh for every execution. A parameter in a decorrelated `EXISTS` / `NOT EXISTS` inner predicate is validated against that inner collection, not against a same-named field on the outer collection; its compatibility is checked against the current inner schema on every execution. Likewise, a parameter in `HAVING` is validated against the synthesized aggregate-result schema (grouping columns plus aggregate aliases), not a same-named field in the source schema. This applies to base collections, joins, CTEs, and other virtual sources on both cold executions and cached-plan hits. Parameters inside an aggregate `FILTER (WHERE ...)` over a JOIN are likewise checked against the remapped merged input field selected by their qualifier, rather than being admitted by the outer schema or silently treated as a non-match. Ordinary post-join `WHERE` predicates and cross-side OR groups follow the same rule across left-deep, bushy, buffered, and streaming plans. Polymorphic extension fields are checked against every active subtype candidate after the predicate is routed to its envelope-aware evaluator.

Each execution pins an ownership-isolated snapshot of its parameter bindings before it starts. This includes copying byte and vector payloads. A streaming result therefore continues to observe the values supplied to that execution even if the caller reuses the dictionary or input buffer after `ExecuteAsync` returns; lazy row production cannot change an already-started statement's bindings.

### Coercion edge cases

- **Integer fields require an exact integer-valued parameter within the field range.** Floating-point fractions remain invalid even when conversion to Decimal would round them to zero or another integer. For `INT64`, the floating-point value `-2^63` is accepted and `+2^63` is rejected. Numeric strings are checked without Decimal rounding: `'1e-40'` and `'1.00000000000000000000000000001'` reject, while integral `'100e-2'` is accepted if its value fits the target field.
- **Bool fields accept the integer values `0` and `1`** (across every CLR integer type) and the strings `"true"` / `"false"` (case-insensitive) / `"1"` / `"0"`. Bool fields REJECT floating-point values — passing `1.0m` / `1.0d` / `1.0f` for a Bool field fails validation.
- **IEEE special values (`NaN` / `+Infinity` / `-Infinity`) split by target type.** Float32 / Float64 ACCEPT them natively (IEEE-storable as the parameter binds to a Float column). Decimal and Int8 / Int16 / Int32 / Int64 REJECT them — Decimal has no IEEE representation, Int* requires an integer-valued non-NaN source. String spellings (`"NaN"` / `"+Infinity"` / `"-Infinity"`) split the same way; the validator's target-type-dependent gate ensures the validator side matches the INSERT-path side rather than letting the validator accept and INSERT throw. Decimal also rejects finite floating-point values and numeric strings outside its representable range. A `double` bound to a Decimal field stores the DECIMAL its canonical text names (`0.1` stores `0.1`) and a `float` the DECIMAL of the FLOAT64 holding its exact value (`0.1f` stores `0.10000000149011612`).
- **Floating-point expressions preserve IEEE special results.** The `/` and `%` operators and float-returning `ABS`, `ROUND`, `FLOOR`, `CEILING` / `CEIL`, `POWER`, `SQRT`, and `MOD` functions return `NaN` or `±Infinity` when that is the IEEE result. Computed columns and projections preserve those values, and predicates/order/grouping apply the total order below. Exact zero divisors raise errors. SIGN(NaN) raises because INT64 cannot represent it; SIGN of infinities returns 1/-1. Unrepresentable integer/DECIMAL conversions, integer/decimal overflow, and ROUND scales outside their supported range raise. TRY and TRY_CAST recover these numeric failures as NULL. Integer arithmetic and MOD produce INT64. `MOD` projection schemas use the promoted type of both operands, and `TO_DECIMAL` projections always advertise Decimal, matching their runtime values on the wire.
- **Float comparison and ordering use a total order with `NaN` greatest** (PostgreSQL / DuckDB parity): `NaN` equals itself, sorts above `+Infinity` in `ORDER BY` (last ascending, first descending), matches `WHERE [F] > @x` for any finite `@x`, and never matches a finite-bound `<` / `<=`. Aggregates and joins follow the same order: `MAX([F])` returns `NaN` when any input row is `NaN`, `MIN([F])` returns the smallest non-`NaN` value, `GROUP BY [F]` buckets every `NaN` row into ONE group, `COUNT(DISTINCT [F])` counts all `NaN` rows as one value, and `JOIN … ON [a].[F] = [b].[F]` matches `NaN` rows to each other (all of these also treat `-0.0` as `+0.0`). The same order applies on every execution plan — indexed or full-scan, any join strategy — and in every expression context: `CASE WHEN [F] > @x …` / computed projections flag the same rows the `WHERE` path admits, window `ORDER BY` places `NaN` rows where the query-level `ORDER BY` does, window `PARTITION BY` groups all `NaN` rows as peers, and `MIN` / `MAX` `OVER (…)` return the same extremum as their aggregate forms. `WHERE [F] IN (subquery)` follows too: a `NaN` yielded by the inner query matches stored `NaN` rows exactly like `IN (@nan)` does.

## Sites that accept `@parameters`

| Clause | Example |
|--------|---------|
| `WHERE` / `HAVING` / `IN` / `MATCH` values | `WHERE [X] = @x`, `HAVING [Total] > @min`, `WHERE [S] IN (@a, @b)`, `MATCH ([P] > @min)` |
| Document literal field values | `INSERT INTO [C] ([X]: @x)`, `UPDATE [C] SET ([X]: @x) …` |
| `LIMIT` / `SKIP` | `SKIP @offset LIMIT @pageSize` |
| `AS OF` / `IN BETWEEN` | `FROM [C] AS OF @ts`, `FROM [C] IN BETWEEN @from AND @to` |
| `VALID FROM` / `VALID TO` (INSERT, UPDATE) | `INSERT … VALID FROM @vf TO @vt`, `UPDATE … VALID FROM @vf` |
| `DELETE … AT` | `DELETE FROM [C] WHERE [Id] = @id AT @cutoff` |
| `EXPECTED REVISION` (UPDATE, REPLACE) | `… EXPECTED REVISION @rev` |
| `SUBSCRIBE WHERE` predicate | `SUBSCRIBE TO [App].[Items] WHERE [Value] >= @minimum` |
| `UNSUBSCRIBE` id | `UNSUBSCRIBE @subId` |
| `WITH PASSWORD` (CREATE USER, ALTER USER) | `CREATE USER [alice] WITH PASSWORD @password` ([user administration](auth-admin.md)) |
| `COMPLETE` / `VALIDATE` text and `AT` position | `COMPLETE @text AT @position`, `VALIDATE @text` ([editor assistance](editor-assistance.md)) |

Missing parameters and type-incompatible values produce a clear error at execution time rather than silently matching nothing.

## See also

- [Basic queries](queries.md)
- [DML](dml.md)
- [Subscriptions](subscriptions.md)

## Detailed syntax and execution


<a id="syntax-parameters"></a>

### Parameters

`@name` style — alphanumeric + underscore. Sent via the binary protocol with explicit type tags. Bind via the typed-client `parameters` dict or the wire protocol's parameter frame. See [`parameters.md`](parameters.md).

Names are exact ordinal identifiers: `@value` does not bind a `VALUE` key, including in-process when the caller supplied a case-insensitive dictionary. Execution and EXPLAIN use the same exact lookup. Correlated subqueries, lateral APPLY, and secondary-unique UPSERT probes preserve those case-distinct bindings while adding their synthetic parameters; lateral schema discovery follows that same rule. Synthetic-name allocation skips every caller-owned name, so a legal binding such as `@__corr_0` is never overwritten by correlation machinery. One request carries at most 65,535 bindings; every name must be valid strict UTF-8 and at most 255 encoded bytes. In-process execution applies these same frame bounds before dispatch.

Every binding that canonicalizes to a string (`string`, `char`, or a custom type's fallback `ToString()`) must be well-formed UTF-16. Unpaired surrogates are rejected by both embedded execution and the strict-UTF-8 wire encoder; valid surrogate pairs such as emoji are accepted and malformed values are never replacement-encoded as U+FFFD.

In computed projections, a bound scalar contributes its exact parameter type (`sbyte` → `INT8`, `byte` / `short` → `INT16`, `ushort` / `int` → `INT32`, `uint` / `long` → `INT64`, `ulong` → `INT64` when it fits and otherwise `DECIMAL`, `decimal` → `DECIMAL`, `double` / `float` → `FLOAT64`, `char` → `STRING`, `bool` → `BOOL`, `Guid` → `GUID`, datetime values → `DATETIME`, `DateOnly` → `DATEONLY`, `TimeOnly` → `TIMEONLY`, `byte[]` → `BYTES`, and `float[]` → `VECTOR`). It is never stringified merely because it came from `@name`. A bound null is type-neutral beside a typed value arm and uses the `STRING` fallback when projected alone.

Prepared-query plan reuse is semantics-transparent. The engine verifies the exact parameter name, type, and null shape independently of binding enumeration order before reusing a plan; an internal hash collision cannot reuse a plan for a different binding shape. Plans are also bound to the exact collection schema generation: ALTER or DROP/recreate cannot reuse field encodings from a retired schema, even when another session performs the DDL and the recreated collection has the same name and integer schema version. Compatibility is revalidated for each current value even when its host-language type matches a cached binding: a prior valid narrow integer, parseable Bool / GUID / date string, or finite Decimal input does not authorize a later out-of-range, unparsable, or non-finite value of the same host type. Parameter values and statement-pinned `NOW` are bound afresh for every execution. A parameter in a decorrelated `EXISTS` / `NOT EXISTS` inner predicate is validated against that inner collection, not against a same-named field on the outer collection; its compatibility is checked against the current inner schema on every execution. A parameter in `HAVING` is instead validated against the synthesized aggregate-result schema (grouping columns plus aggregate aliases), never a same-named source field. This holds for base, joined, CTE, and other virtual sources on cold executions and cached plan hits. A parameter inside an aggregate `FILTER (WHERE ...)` over a JOIN is checked against the remapped merged input field selected by its qualifier on every execution, not against the outer schema.

Execution snapshots the parameter map into an ordinal name dictionary and copies byte and vector buffers before the statement starts. It also canonicalizes CLR numeric widths, `float`, `char`, and `DateTime` to the exact values decoded from the wire. Other `IFormattable` objects fall back to an invariant string; non-formattable objects use their ordinary `ToString()`. A lazy result stream therefore observes one stable, transport-independent binding set even if its caller reuses or mutates the original inputs after `ExecuteAsync` returns.

**Coercion edge cases:** Bool fields reject floating-point values (`1.0m` / `1.0d` rejected even though `1` accepts). IEEE specials split by target — Float32 / Float64 ACCEPT `NaN` / `±Infinity` (CLR and string spellings), Decimal / Int* REJECT (no IEEE representation / no integer-valued source). Decimal also rejects finite floating-point values and numeric strings outside its representable range. The validator and write paths apply the same target-specific bounds.
