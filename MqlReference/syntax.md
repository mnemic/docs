# Syntax conventions

> **In this section** — identifiers, string literals, comments, special values, multi-statement execution.

## Identifiers

Schema identities use **mandatory bracket syntax**. This includes collection namespaces and names, database names, enum and reusable-type names, field/path segments, CTE names, relation/output aliases, and named windows:

```
[Namespace].[Collection]
[FieldName]
[Parent].[Child].[GrandChild]
```

Every user-defined name segment requires brackets in both declarations and references. Relation aliases following `FROM`, `JOIN`, `APPLY`, and `JOIN LATERAL` follow the same rule:

```sql
FROM [Orders].[Orders] [o] JOIN [Store].[Products] [p] ON [o].[ProductId] = [p].[Id];
```

Use `[Namespace].[TypeName]` for a qualified enum or reusable type and `[TypeName]` for a top-level type. Database DDL uses `CREATE DATABASE [Name]`; the `ON DATABASE` scope of the [user and role administration](auth-admin.md) statements also accepts a plain identifier (`ON DATABASE myshop`), and brackets work there too. Named windows use `WINDOW [w] AS (...)` and `OVER [w]`.

Built-in type keywords (`STRING`, `INT32`, `ARRAY`, and so on), function names, session-parameter keys, and grammar options remain unbracketed. String literals and parameter values remain values; brackets apply to names in identifier positions.

## Strings

String literals use **single quotes**. Escape a literal single quote by doubling it:

```sql
'Hello'
'O''Brien'
'value with (parens) inside'
''   -- empty string
```

The NUL character (`U+0000`) is **rejected everywhere in MQL statement text**, including string literals, bracketed identifiers, and comments; the lexer throws `MqlParseException`. NUL is reserved for internal statement framing, so admitting it into parsed text would make those frames ambiguous. String values that genuinely need NUL (or any byte the lexer cannot represent) must travel through a [parameter](parameters.md); parameters ride the binary protocol and preserve every byte of the original string, in equality and `IN` lists alike.

MQL statement text must also be well-formed UTF-16 at the in-process parser boundary (and valid UTF-8 on the wire). An unpaired high or low surrogate is a parse error, including inside a string literal or bracketed identifier; a valid surrogate pair such as an emoji is accepted. This fail-closed rule prevents a later UTF-8 conversion from replacing distinct malformed inputs with U+FFFD. The public programmatic `StringLiteral` AST node enforces the same invariant at construction, so bypassing the text parser does not widen the accepted domain.

Statement text has no lexer-imposed character cutoff. Long literals and identifiers remain valid; token storage grows with the tokens actually present rather than reserving proportionally to text length. One statement may contain at most 65,536 non-EOF tokens; denser input fails with a parse error before its token array can grow without bound.

## Numbers

Numeric literals use decimal notation (`42`, `3.14`, `99.99`); hexadecimal, octal, and scientific notation are not accepted. Literals written without a decimal point become `INT64` only when their value fits that type. A scale-zero value outside the `INT64` range and every literal written with a decimal point remain exact `DECIMAL` values. The shared classifier inspects decimal scale without allocating, including on per-row window-filter evaluation.

An ordinary numeric value literal outside Decimal's exact range or precision is rejected before execution with `INVALID_VALUE`, including a nonzero fraction that would round to zero. Redundant leading or trailing zeros are accepted when the value remains exact. For floating-point values beyond that literal domain, use a [numeric string or typed parameter](document-literals.md).

## Comments

Line comments start with `--` and continue to end of line:

```sql
-- This is a comment
FROM [Store].[Products];  -- End-of-line comment
```

Block comments use `/* ... */` and can span lines or nest up to 100 levels. They act as whitespace between tokens; delimiters inside strings or bracketed identifiers remain literal content. An unterminated comment or deeper nesting is a parse error.

## Special values

| Value | Description |
|-------|-------------|
| `TRUE` / `FALSE` | Boolean literals (case-insensitive) |
| `NULL` | Null literal |
| `DEFAULT` | A field's declared default, or NULL when it declares none, as a field's whole value in a written document or SET list only ([DEFAULT](dml.md#default)) |
| `NOW` | Current UTC time (case-insensitive). Accepted everywhere a datetime is expected: `AS OF`, `IN BETWEEN`, `WHERE`, correlated subqueries, JOIN residuals, window filters, computed projections, `VALID FROM`, `VALID TO`, `DELETE … AT`, and `CANCEL VERSION AT`. Resolves at execute time, not parse time. It is statement-pinned: every occurrence of `NOW`, `CURRENT_TIMESTAMP()`, and `CURRENT_DATE()` in one execution derives from exactly one instant, while the next execution gets a fresh instant. Cached ASTs never cache the clock value. |

## Multi-statement execution

A wire request contains one statement and may end with one semicolon. Extra statements, empty statements, and repeated semicolons are rejected. Newlines and indentation are whitespace, so formatting cannot change statement boundaries.

The .NET client's `ExecuteAsync` splits scripts at top-level semicolons and sends each statement sequentially in the same session. The final statement may end at EOF. Semicolons inside strings, bracketed identifiers, nested comments, parentheses, and a trigger's `AS BEGIN ... END` body do not split the script. Trigger `CASE ... END` expressions retain their own nesting. Comment-only input is a no-op; malformed quoting or comments are errors.

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [Zone].[Zone] (Temporal: TRUE)
WITH SCHEMA (
    [ZoneId] INT32 PRIMARY KEY AUTOINCREMENT,
    [Name] STRING NOT NULL
);
COMMIT MIGRATION;
```

## See also

- [Document literals](document-literals.md) — the `(...)` / `[...]` value syntax for INSERT / UPDATE / REPLACE
- [Reserved keywords](keywords.md) — the full keyword list

## Detailed syntax and execution

Canonical field paths bracket each segment separately. Compact single-token paths also normalize to that form: `[Parent.Child]` becomes `[Parent].[Child]`, and `[Tags[]]` becomes `[Tags][]`. Empty logical segments and nested bracket syntax inside one path token are rejected.

<a id="syntax-identifiers"></a>

See [Identifiers](#identifiers).

<a id="syntax-strings"></a>

See [Strings](#strings).

<a id="syntax-numbers"></a>

See [Numbers](#numbers).

<a id="syntax-booleans--null--now"></a>

See [Special values](#special-values).

<a id="syntax-comments"></a>

See [Comments](#comments).

<a id="syntax-multi-statement-scripts"></a>

See [Multi-statement execution](#multi-statement-execution).

<a id="syntax-session-execution-ownership"></a>

### Session execution ownership

One connection or embedded MqlExecutor is one session. Concurrent top-level submissions on the same session serialize so transaction/migration state, cancellation, snapshots, and SET parameters never race. Cancellation while waiting returns `Statement cancelled.` without executing the statement. The gate is released when `MqlResult` returns, before any `RowsAsync` drain; use separate sessions for genuinely parallel statements.

<a id="syntax-case-sensitivity"></a>

### Case sensitivity

`EXACT_EQUALS(a, b)` provides ordinal, case-sensitive STRING equality. It is NULL-safe: two NULL operands are equal; one NULL operand is unequal. Other argument types are rejected. It does not change index or ordinary `=` semantics.

- All keywords: case-insensitive.
- Field / collection names in the schema: case-insensitive.
- String comparisons in `WHERE` / `ORDER BY` / index probes / expression `=`/`<`/`>` / `POSITION` / `REPLACE`: Unicode-aware **case-insensitive** — the active .NET invariant simple-lowercase mapping, then ordinal compare (`'café' = 'CAFÉ'`). This is neither linguistic collation nor Unicode Default Case Folding: ignorable chars stay significant, no normalization occurs, and order is by code point. The database header fingerprints the exact effective mapping and policy. If the runtime or globalization backend changes it, open durably invalidates and rebuilds all derived indexes before publishing the new profile; an interrupted rebuild is repeated on reopen. `REPLACE` uses stack/pooled fold scratch, so a no-match row returns the original string without allocating lowercase copies.

<a id="syntax-binary-comparison"></a>

### Binary comparison

BYTES equality and ranges compare exact unsigned byte sequences, including present empty values, strict prefixes, case variants, and invalid UTF-8. Bind byte parameters or use Base64 literals. Range predicates never match NULL; ORDER BY instead places NULL first ascending and last descending. The comparator is the same for field predicates and projected comparison expressions.

<a id="syntax-numeric-comparison"></a>

### Numeric comparison

Comparison across Int64, Decimal, and Double is exact: it does not round an integer or decimal through Double first. Thus Int64 `9007199254740993` is not equal to Double `9007199254740992.0`. The rule covers equality, ordering, IN, BETWEEN, NULLIF, grouping/deduplication, and window order/peer detection; large literal IN sets hash this exact identity as well, so distinct high-precision DECIMAL members do not collapse into one FLOAT64-derived bucket; mixed Decimal/Double comparison uses scaled fixed-width coefficients without allocating temporary arbitrary-precision values per probe; arithmetic retains its ordinary, order-independent promotion ladder: `FLOAT64` > `FLOAT32` > `DECIMAL` > `INT64` > `INT32` > `INT16` > `INT8`.

Arithmetic operators accept numeric operands only. Unlike comparisons, they do not parse numeric text: `1 + '2'` and `'2' + 1` both raise `TYPE_MISMATCH`. Use an explicit cast when conversion is intended. Every nonnumeric kind (including bool, GUID, the temporal types, bytes, object, and array) is rejected in either operand position — when the statement is bound if the operand's type is known, otherwise before a numeric storage slot is read; MQL has no temporal arithmetic (see `DATEADD` / `DATEDIFF`).

`POWER`, `SQRT`, and `MOD` enforce the same numeric-only boundary. Numeric text / bool inputs require explicit conversion; `MOD(7, '3')` errors rather than treating the string's inactive numeric slot as zero.

Numeric-versus-String comparison parses invariant [numeric text](document-literals.md#numeric-text) into the text's narrowest lossless numeric representation, then uses exact mixed-numeric comparison. Valid text such as `'5'` works in both `=` and `IN`, while a large integer is not rounded merely because the other operand is floating point; nonnumeric text is a nonmatch for every scalar comparison arm, including `!=`, on both indexed and full-scan plans, including when the text's UTF-8 length equals the field's native binary width. In `IN`, an unparseable numeric member contributes no match while parseable siblings retain their normal meaning. Well-formed values outside a field's width remain nonmatches for equality/IN and use the documented range-operator policy. A polymorphic extension path with different types by subtype treats a value valid for one subtype as a nonmatch on incompatible subtype instances. This active- subtype binding applies at collection level and inside Object / ARRAY OBJECT fields, including row-wise WHERE, MATCH, CHECK, UPDATE expressions, window- aggregate FILTER predicates, correlated outer references, and enum metadata used by `ENUM_LABEL`. Resolution continues through arbitrarily nested Object / ARRAY OBJECT shapes; array ANY/ALL, IN, LENGTH(predicate), and correlated MATCH all use the active schema of the specific element being tested. For a bound predicate parameter, validation accepts the binding when at least one candidate subtype declaration accepts its type; each row then uses the active declaration, so sibling declaration order cannot reject an otherwise valid binding. Literal validation is also existential per scalar or per `IN` member: different list members may target different subtype types, while a member accepted by no active type is a query error rather than a silent miss. Materialized `IN (FROM …)` members follow the identical rule. For a nonpolymorphic target, every non-NULL materialized member must convert to the target field type, except malformed numeric text follows ordinary numeric comparison semantics and contributes no match. Malformed BOOL, GUID, BYTES, and date/time text is a query error. A syntactically valid integer outside a narrower target width remains a nonmatch.

## Usage pitfalls


<a id="pitfall-block-comments-must-close"></a>

### Block comments must close

`/* ... */` comments can span lines and nest up to 100 levels. Close every opening delimiter. They are ignored only outside strings and bracketed identifiers; `--` line comments remain available.

<a id="pitfall-there-is-no-if-exists-or-if-not-exists"></a>

### There is no `IF EXISTS` or `IF NOT EXISTS`

No MQL statement accepts `IF EXISTS` or `IF NOT EXISTS`: `DROP COLLECTION IF EXISTS [App].[Old]`, `CREATE COLLECTION IF NOT EXISTS …`, and `CREATE DATABASE IF NOT EXISTS [Shop]` are parse errors. A CREATE of a taken name reports `DUPLICATE_NAME`, and a DROP or ALTER of a missing object reports `COLLECTION_NOT_FOUND` for a collection and `UNKNOWN_OBJECT` for any other object. Check first through the catalog — `FROM [Mnemic].[Collections] WHERE [Name] = '[App].[Old]'`, `LIST DATABASE` — and remember that a statement that fails inside a migration rolls the whole migration back ([migrations](migrations.md)).

<a id="pitfall-when-parser-errors-are-unhelpful"></a>

### When parser errors are unhelpful

Run the offending statement through `MqlQuery.ParseStatement` in a local test (or via the .NET typed client's `ExecuteAsync`). The `MqlParseException` includes the position; counting characters from the start of the statement points at the bad token.
