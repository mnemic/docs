# Enums

> **In this section** — `CREATE ENUM`, using enum values in INSERT / UPDATE / WHERE, `ALTER ENUM` (members, values, underlying type), renaming/moving enums, `DROP ENUM`.

> **Migration-only DDL.** `CREATE`, `ALTER`, and `DROP ENUM` are structural changes — they must appear between `BEGIN MIGRATION` and `COMMIT MIGRATION` (see [Migrations](migrations.md)). The examples below elide the wrapper for brevity; the executor rejects each form with `"<Action>Enum requires a migration context. Use BEGIN MIGRATION / COMMIT MIGRATION."` outside one.

## CREATE ENUM

```sql
CREATE ENUM [Billing].[ChargeStatus] (INT32)
VALUES (1 = 'Pending', 2 = 'Authorized', 3 = 'Completed');

CREATE ENUM [Priority] (INT16)
VALUES (1 = 'Low', 2 = 'Medium', 3 = 'High');

CREATE ENUM [Category] (GUID)
VALUES (
  '550e8400-e29b-41d4-a716-446655440000' = 'Electronics',
  '6ba7b810-9dad-11d1-80b4-00c04fd430c8' = 'Books'
);
```

Supported underlying types: `INT8`, `INT16`, `INT32`, `INT64`, `GUID`. Anything else (e.g. `STRING`) is rejected — at parse time for MQL text, and again at the executor's DDL boundary for programmatically-built statements, so an unsupported type can never reach the enum catalog. The `VALUES` list must contain at least one member. Canonical printing rejects an empty programmatic AST instead of emitting `VALUES ()`, which is not valid MQL and could not be replayed. The persisted registry stores each enum's member count as a signed 16-bit integer, so one definition accepts at most 32,767 members. Programmatic ASTs above that limit are rejected before canonical printing or value canonicalization; dense textual statements may reach the statement-wide token limit first.

Member values are parsed into their underlying type and persisted in one canonical form: invariant decimal for integer types and lowercase `D` format for GUIDs. Lexical aliases such as `1` and `01`, or two GUID spellings for the same GUID, are therefore the same member and are rejected as duplicates. Parsing, validation, and stored-value rewriting all use that canonical form; the host's current culture cannot reinterpret a leading ASCII `-` or emit a different negative-sign spelling. Descriptions are `ENUM_VALUE` labels, so they must also be unique under case-insensitive (`OrdinalIgnoreCase`) comparison. `CREATE ENUM` and `ALTER ENUM ... ADD VALUE` reject a duplicate value or label before publishing the definition. Programmatic AST member entries, underlying values, and descriptions must be non-null; both strings must be well-formed UTF-16. Canonical printing and direct execution reject an invalid member before traversal or registry mutation, so printer output is replayable and strict UTF-8 persistence cannot change an identity. Non-canonical member values are accepted and stored canonically.

Enum names require brackets in CREATE / ALTER / DROP ENUM and field declarations: `[Namespace].[EnumName]`, or `[EnumName]` for a top-level enum. Bare names and partially bracketed qualified names are rejected. Built-in underlying type keywords such as `INT32` remain unbracketed.

Enum value descriptions support escaped quotes: `1 = 'It''s active'`.

## Using enum values in INSERT / UPDATE / WHERE

Enum fields in document literals and filter predicates must use the underlying **value** (integer or GUID), not the description string. `INSERT INTO [Items] ([Pri]: 3)` works on a `Priority INT32` enum; `INSERT INTO [Items] ([Pri]: 'High')` is rejected at conversion time because the description is not resolved to the underlying value. Walk the nested `Values` array on `[Mnemic].[Enums]` to look up the underlying value for a given description.

```sql
-- Declare the enum and a column that uses it
CREATE ENUM [Priority] (INT32)
VALUES (1 = 'Low', 2 = 'Medium', 3 = 'High');

ALTER COLLECTION [App].[Tickets] ADD FIELD [Pri] [Priority] DEFAULT 1;

-- INSERT: underlying value
INSERT INTO [App].[Tickets] ([Id]: '...', [Title]: 'Urgent outage', [Pri]: 3);

-- UPDATE: underlying value
UPDATE [App].[Tickets] SET ([Pri]: 2) WHERE [Id] = '...';

-- WHERE: underlying value
FROM [App].[Tickets] WHERE [Pri] >= 2;
FROM [App].[Tickets] WHERE [Pri] IN (2, 3);

-- Look up descriptions via the virtual collection
FROM [Mnemic].[Enums] WHERE [Name] = '[Priority]' SELECT [Values];
```

## ENUM_LABEL — value → description scalar function

`ENUM_LABEL([Field])` returns the description string declared by the field's enum type's `CREATE ENUM … VALUES (1 = 'Low', …)` for the field's stored underlying value. Saves the `FROM [Mnemic].[Enums] … WHERE … walk [Values]` join every consumer would otherwise re-implement.

```sql
-- Humanise enum values in projection
FROM [App].[Tickets]
SELECT [Title], ENUM_LABEL([Pri]) AS [Priority];

-- Combined with other scalar functions
FROM [App].[Tickets]
SELECT CONCAT([Title], ' (', ENUM_LABEL([Pri]), ')') AS [Display];

-- UPDATE expressions read the pre-update row and may persist its label
UPDATE [App].[Tickets]
SET ([PriorityLabel]: ENUM_LABEL([Pri]))
WHERE [Id] = '...';

-- In a computed column — every read materialises the label
CREATE COLLECTION [App].[Tasks] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Pri] [Priority],
  [Label] STRING AS (ENUM_LABEL([Pri]))
);
```

Semantics:

- **NULL → NULL** (three-valued logic; matches `COALESCE` / `NULLIF`).
- **Value not enrolled in the enum → NULL**.
- **Argument must be a bracketed field reference** — `ENUM_LABEL(1)` / `ENUM_LABEL('Low')` are rejected when the call is first evaluated for a row.
- **Argument's field must be enum-typed** — `ENUM_LABEL([Title])` on a `STRING` field is rejected (typed error, not silent NULL), also when the call is first evaluated.
- **A call evaluated for no row reports no mistake** — over an empty input or a `WHERE` that matches nothing, a statement holding either mistake succeeds with zero rows and a `STRING` column; test a new projection against data.
- **No index hook** — `ENUM_LABEL` is a function-on-field expression and can only appear in expression-aware surfaces (SELECT projection, computed columns, CHECK constraints, and an UPDATE assignment's RHS). An INSERT value cannot call it because INSERT has no existing row from which its required field reference could be read; it is refused with `INVALID_STATEMENT`, as every field reference in an INSERT value is. Scalar WHERE predicates accept `WHERE ENUM_LABEL([Pri]) = 'High'`. Filtering the underlying value (`WHERE [Pri] = 3`) can use the ordinary enum-key index.

Supported surfaces:

| Surface | Supported |
|---|---|
| `SELECT` projection | ✅ |
| Virtual computed column (`STRING AS (ENUM_LABEL([F]))`) | ✅ |
| `CHECK` constraint | ✅ |
| `UPDATE` value expression | ✅ (reads the pre-update row) |
| `INSERT` value expression | ❌ (`ENUM_LABEL` requires an existing-row field reference) |
| `WHERE` / `HAVING` | ✅ (scalar predicate evaluation) |

## ENUM_VALUE — label → value scalar function

`ENUM_VALUE('label', 'EnumName')` is the inverse of `ENUM_LABEL`: it returns the declared underlying numeric or `GUID` value for a label string in a named enum. Useful when the application carries a human-readable string (form input, OCPI extension code) that needs to land in the catalog's underlying-typed enum field.

```sql
-- Resolve a label to its stored value
FROM [App].[Tasks]
SELECT [Id], ENUM_VALUE('High', 'Priority') AS [Pri];

-- Project an INSERT column from a label string
INSERT INTO [App].[Tasks] ([Id]: 1, [Pri]: ENUM_VALUE('High', 'Priority'));

-- In a computed column — the inverse of the ENUM_LABEL example above
CREATE COLLECTION [App].[Tasks] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [LabelStr] STRING,
  [Pri] INT64 AS (ENUM_VALUE([LabelStr], 'Priority'))
);
```

Semantics:

- **Two arguments**: `(label, enumName)`. Label can be any expression evaluating to `STRING`; enumName must be a string-typed expression (literal in 99 % of calls).
- **Label comparison is case-insensitive** (`OrdinalIgnoreCase`, consistent with engine-wide string-comparison policy).
- **Labels are unambiguous**: enum DDL rejects case-insensitive duplicate descriptions, so lookup never depends on declaration order.
- **NULL label → NULL** (three-valued logic; matches `ENUM_LABEL`'s value-not-in-enum NULL).
- **NULL enum name → typed error** (the enum name is structural — a NULL is treated as a mistake, not a missing-data case).
- **Label not enrolled → NULL** (mirrors `ENUM_LABEL`'s value-not-in-enum semantic).
- **Unknown enum name → typed error** so a typo (`ENUM_VALUE('High', 'Priroty')`) is caught when the call is first evaluated for a row; like `ENUM_LABEL`'s argument mistakes, it passes unreported when no row is evaluated.
- **Enum-name spelling**: the name as `[Mnemic].[Enums]` publishes it — `'[Priority]'` at the top level, `'[Core.Facility].[ChargeState]'` in a namespace — and the same name with unbracketed segments (`'Priority'`, `'[Core.Facility].ChargeState'`, `'Core.Facility.ChargeState'`, where an unbracketed qualified name splits at its last dot) resolve to the same registered enum. A stored definition prints the call with the published spelling.
- **Return type**: `INT64` for `INT8` / `INT16` / `INT32` / `INT64` enums; `GUID` for `GUID` enums. In a projection, a literal enum name or a bound string parameter is resolved before rows are packed, so the wire schema carries that exact type. The same catalog-aware check lets a computed column declare the matching type directly. If the enum-name expression varies by row, an enclosing row's field a nested query reads included, the projection uses `STRING` as one stable, lossless schema across numeric and `GUID` enums.

Supported surfaces:

| Surface | Supported |
|---|---|
| `SELECT` projection | ✅ |
| Virtual computed column | ✅ |
| `CHECK` constraint | ✅ |
| `INSERT` / `UPDATE` value expression | ✅ |
| `WHERE` / `HAVING` | ✅ (scalar predicate evaluation) |

## ALTER ENUM

```sql
-- Add a value
ALTER ENUM [Billing].[ChargeStatus] ADD VALUE 4 = 'Refunded';

-- Remove a value (fails if in use without REPLACE)
ALTER ENUM [Billing].[ChargeStatus] REMOVE VALUE 2;

-- Remove a value with replacement for existing data
ALTER ENUM [Billing].[ChargeStatus] REMOVE VALUE 2 REPLACE 1;

-- Give a member another label; its stored value stays
ALTER ENUM [Billing].[ChargeStatus] RENAME VALUE 3 TO 'Settled';

-- Give a member another value; its label stays and every stored value follows
ALTER ENUM [Billing].[ChargeStatus] SET VALUE 3 TO 30;

-- Change the underlying integer type; every stored value keeps its number
ALTER ENUM [Billing].[ChargeStatus] SET TYPE INT64;
```

`RENAME VALUE value TO 'label'` changes one member's label. Documents store the value, so none is rewritten, and a stored `ENUM_VALUE` that named the member keeps naming it (see below). The new label must not be another member's, compared ignoring case as `ENUM_VALUE` compares labels (`DUPLICATE_NAME`); a value the enum lacks is `INVALID_VALUE`. Changing only the label's case is allowed, and naming the label the member already has changes nothing. The rename rides the migration: an abort keeps the old label, and `[Mnemic].[Enums]`, `ENUM_LABEL` and printed definitions show the new one after commit.

`SET VALUE value TO value` gives a member another value and keeps its label. Both values are literals as `ADD VALUE` takes them. The new value must be a value of the enum's underlying type (`INVALID_VALUE`) and no other member's, compared in canonical form, so `03` is `3` (`DUPLICATE_NAME`); a value the enum lacks is `INVALID_VALUE`, and naming the member's own value changes nothing. Every retained version that stores the old value stores the new one — history, expired and scheduled versions included, so `AS OF` reads it too — wherever the enum is stored: at the top level, inside objects, in the elements of an array of objects, in a collection-level or field-level subtype's extension, in a plain enum array, in the elements of an array of arrays at any depth, and in every use of a named type. Indexes over those fields are rebuilt. A default that names the member names the new value: a field's own default and a value inside an object or array default, in collections and named types alike. A CHECK, view or trigger that compares a field with the old value as a literal keeps the literal; change it in the same migration.

`SET VALUE` is refused:

- with `DEPENDENT_OBJECTS_EXIST` while a computed field, CHECK, view, materialized view or trigger names the member through a stored `ENUM_VALUE`, as `REMOVE VALUE` is (see below);
- with `FEATURE_NOT_SUPPORTED` while a primary key, or a reference, which holds a primary key's value, stores the enum, since those values identify documents;
- with `DEPENDENT_OBJECTS_EXIST` while a partial-index filter, including an expiry index's, reads a field of the enum, whose meaning the change would alter, and while a stored computed field reads one, whose stored values would no longer follow from its expression (make it virtual with `DROP STORED`, or drop it, first);
- with `CHECK_VIOLATION` when a converted version fails a CHECK that reads a field of the enum;
- with `FEATURE_NOT_SUPPORTED` when subtypes of one polymorphic object declare a field of the same name, one with the enum and one with another type.

`SET TYPE type` changes an integer enum's underlying type among `INT8`, `INT16`, `INT32` and `INT64`. Every field of the enum, in every collection and named type, takes the new width, and every retained version keeps its values in it, history included; indexes over those fields are rebuilt. Integer member values read the same in every width, so labels, `ENUM_LABEL`, stored `ENUM_VALUE` calls and CHECKs keep meaning the same members. Each member must fit the new type (`INVALID_VALUE`, naming the member); a change to or from `GUID` is refused with `FEATURE_NOT_SUPPORTED`, since no member's value would carry over; naming the type the enum has changes nothing. A primary key, a reference or a partial-index filter over the enum refuses `SET TYPE` as it refuses `SET VALUE`, and a computed field whose declared type no longer fits its expression refuses it with `INVALID_DEFINITION`.

`SET VALUE` and `SET TYPE` ride the migration: an abort restores every value, field type, named type and the enum definition, and other sessions see the change after commit. Statistics of every rewritten collection are dropped. A materialized view over a rewritten collection holds the new values: an incremental one is rebuilt from its sources, and a manual one's own stored values are rewritten with the collections. Like `REMOVE VALUE`, each closes writes on every collection that stores the enum until the migration settles, and is refused with `INVALID_TRANSACTION_STATE` when the same migration already wrote one of them.

`REMOVE VALUE` is also refused with `DEPENDENT_OBJECTS_EXIST` while a computed field, CHECK, view, materialized view or trigger names the member through a stored `ENUM_VALUE`; change that definition first, for example with `DROP CHECK` and `ADD CHECK`, in the same migration. With or without `REPLACE`, it is refused with `FEATURE_NOT_SUPPORTED`, as `SET VALUE` is, when subtypes of one polymorphic object declare a field of the same name, one with the enum and one with another type: its usage probe and its rewrite reach that field by its path, which holds the other subtype's values too.

`REMOVE VALUE` **fails closed**. The in-use probe and the `REPLACE` document rewrite must both succeed before the value is removed from the enum definition:

- If the usage probe errors on any collection, the ALTER errors — usage is unknown, and removing a value whose usage can't be proven would strand stored documents on a retired member.
- Temporal usage checks include every retained version, including expired history, scheduled rows, and a version starting on the final UTC tick. `REPLACE` must eliminate the member from that complete population. If an ordinary rewrite leaves it in history or a scheduled version, removal errors and rolls back; creating a new current version alone does not erase history.
- If the document rewrite fails anywhere — for example a `CHECK` constraint on a referencing field rejects the `REPLACE` target, or an `AFTER UPDATE` trigger body throws — the ALTER errors and the enum definition is not changed. Inside the migration (the required wrapper for `ALTER ENUM`), the auto-rollback also unwinds whatever part of the document rewrite had already applied.
- After the rewrite, a **post-condition re-probe** re-checks that the old value is gone from every collection. Rewrites cover top-level, nested `OBJECT` at any depth (each ancestor object is reassembled on the way back up), plain enum arrays, `ARRAY OBJECT`, field-level polymorphic object and polymorphic `ARRAY OBJECT` elements (both base-resident and extension-resident enum fields), arrays of arrays and arrays inside an array's objects at any depth (`ARRAY ARRAY [Priority]`, the fields of `ARRAY ARRAY OBJECT` elements, each enclosing array rebuilt on the way back up), and collection-level polymorphic storage. Wherever a `[base_len][base][extension]` envelope is involved — a collection-level row, a polymorphic sub-document, or a polymorphic array element — the rewriter patches only the half that declares the field and preserves the other half byte-identical. If a malformed or unsupported row cannot be rewritten, the re-probe prevents the enum member from being retired and the ALTER errors naming the collection.
- Array storage is validated before replacement. A truncated count or element, trailing bytes, an invalid object offset table, or a malformed polymorphic envelope/discriminator fails with `ARRAY_PAYLOAD_CORRUPT`; the migration and enum definition remain unchanged. Corrupt elements are never silently skipped.
- Before the first probe, Mnemic closes and drains writes on every collection whose schema references the enum. Those barriers remain closed through the optional rewrite, final re-probe, and `COMMIT MIGRATION` enum-definition publication (or completed rollback); returning from `ALTER ENUM` does not open them. The migration's own replacement writes use its private transaction bypass; concurrent external DML cannot land in the scan-to-publish window and is safe to retry after the migration commits or aborts. Explicit transactions remain counted until commit or rollback, not merely until their DML statement returns. `REMOVE VALUE` is rejected if the same migration already wrote a referencing collection; commit that DML in an earlier migration and retry the enum change.

## Rename or move an enum

```mql
ALTER ENUM [Core.Billing].[TariffType]
    RENAME TO [Core.Pricing].[TariffKind];
ALTER ENUM [Core.Pricing].[TariffKind] RENAME TO [TariffKind];
```

The destination is the complete identity: `[TariffKind]` means the top level. A namespace requires no separate creation. Names remain bracketed. The source must exist; the destination cannot be occupied by another enum or a type, or use the reserved `[Mnemic]` / `[Mnemic.*]` namespaces. An identical source and destination is a no-op; casing-only changes update spelling.

The ID, underlying values, labels, field references and defaults are preserved. No document rewrite is needed for ID-based consumers such as enum fields or `ENUM_LABEL`. The owner sees the new name inside the migration; other sessions see it after commit. Abort and failed migrations restore the original name. `[Mnemic].[Enums]` and generated schemas report the new name.

Stored `ENUM_VALUE(label, 'EnumName')` calls bind the enum ID when their owning definition is accepted, and a literal label binds the member it names: the call keeps yielding that member's value after `RENAME VALUE` changes its label, and prints with the current label. A literal label the enum does not have is refused when the definition is declared (`UNKNOWN_OBJECT`), since at run time it would evaluate to NULL and a CHECK comparing with it would pass every row. Computed fields, CHECKs, views, materialized views and trigger bodies/conditions retain that binding through rename and reopen, including nested schemas and named types. Their persisted text and bindings are unchanged by rename. Reusing the former name for a different enum never redirects an existing static reference. This includes stored scalar subqueries: correlating their arguments with an outer row preserves the original enum binding. Materialized rows are retained without an implicit refresh. Unrelated string literals remain ordinary data.

Calculated names such as `ENUM_VALUE('Ready', [EnumName])` remain runtime name lookups. They neither prevent rename nor acquire a static dependency. After rename, an expression that still calculates the old name follows the normal missing-name behavior; it can resolve again if a new enum acquires that name. Dependency creation in a later migration binds the committed identity; references admitted against the earlier snapshot keep the same enum ID. Dropped dependencies, incompatible definition changes and destination collisions require retry. Destination claims are checked against the merged type and enum registries at commit.

External application queries must use the new name after commit. Historical migration text is unchanged. See [migrations](migrations.md) and [type renames](composite-types.md#rename-or-move-a-type).

## DROP ENUM

```sql
DROP ENUM [Billing].[ChargeStatus];
```

`DROP ENUM` is rejected with `"Cannot drop enum '<full-name>': field '<field-path>' in collection '<collection>' uses it."` when any collection has a field declared against the enum. Drop or alter the referencing field first (or pick a `REPLACE` target via `ALTER ENUM REMOVE VALUE` instead, when the goal is just to retire specific members). A view, materialized view, trigger, CHECK constraint or computed column that names the enum through a static `ENUM_VALUE` blocks it too, with `DEPENDENT_OBJECTS_EXIST` `"Cannot drop enum '<full-name>': '<definition>' names it. Change that definition first."`.

## See also

- [Schema definition (DDL)](ddl.md) — using enum types as field types
- [System virtual collections](system-collections.md) — `[Mnemic].[Enums]`
- [Migrations](migrations.md) — required wrapper for ALTER ENUM

## Detailed syntax and execution


<a id="syntax-create--alter--drop-enum"></a>

### CREATE / ALTER / DROP ENUM

Enum members are **underlying-type literals** (INT8 / INT16 / INT32 / INT64 / GUID) with a description string — never bare words. Those five are the only underlying types; `CREATE ENUM [X] (STRING)` (or any other type) is rejected at parse time, and the executor's DDL boundary fail-closes the same whitelist for programmatically-built statements. The `VALUES` list must contain at least one member. Canonical printing rejects an empty programmatic AST instead of emitting the invalid, unreplayable `VALUES ()` form. One definition accepts at most 32,767 members, matching the signed 16-bit count in the persisted registry. Programmatic ASTs above that limit fail before canonical printing or value canonicalization; dense textual statements may reach the statement-wide token limit first. Values are canonicalized to invariant decimal or lowercase GUID `D` form before persistence and deduplication, so aliases such as `1` / `01` cannot name two physical members. Description strings are the case-insensitive labels consumed by `ENUM_VALUE`; descriptions must therefore be unique under `OrdinalIgnoreCase`. `CREATE ENUM` and `ALTER ENUM ... ADD VALUE` reject either kind of duplicate before publication. Programmatic AST member entries, underlying values, and descriptions must be non-null; both strings must be well-formed UTF-16. Canonical printing and direct execution reject an invalid member before traversal or registry mutation, so printer output is replayable and strict UTF-8 persistence cannot change an identity. Enum parsing, validation, and stored-value rewriting use that same invariant form; host culture cannot reinterpret an ASCII `-` or emit another negative sign spelling. There is no `REPLACE VALUE x = 'desc'` form; the only ALTER actions are `ADD VALUE`, `REMOVE VALUE [REPLACE n]`, `RENAME VALUE n TO 'description'`, `SET VALUE n TO m`, `SET TYPE INT8 | INT16 | INT32 | INT64`, and `RENAME TO [destination]`.

```sql
CREATE ENUM [App].[OrderStatus] (INT16) VALUES (
  1 = 'Awaiting payment',
  2 = 'Paid',
  3 = 'Shipped'
);
ALTER ENUM [App].[OrderStatus] ADD VALUE 4 = 'Cancelled';
ALTER ENUM [App].[OrderStatus] REMOVE VALUE 3;            -- errors if any document carries 3
ALTER ENUM [App].[OrderStatus] REMOVE VALUE 3 REPLACE 2;  -- rewrites stored documents first
ALTER ENUM [App].[OrderStatus] RENAME VALUE 2 TO 'Shipped'; -- relabels a member; values and stored ENUM_VALUE keep it
ALTER ENUM [App].[OrderStatus] SET VALUE 2 TO 20;         -- revalues a member; every retained version and default follows
ALTER ENUM [App].[OrderStatus] SET TYPE INT32;            -- widens every field of the enum; values keep their numbers
DROP ENUM  [App].[OrderStatus];                           -- errors while any field uses the enum
```

## Usage pitfalls


<a id="pitfall-enum_label-can-filter-an-enum-field"></a>

### `ENUM_LABEL` can filter an enum field

`WHERE ENUM_LABEL([Pri]) = 'High'` and `WHERE UPPER([Name]) = 'X'` use the scalar predicate evaluator. Filtering an enum's underlying value (`WHERE [Pri] = 3`) can use its ordinary index. ENUM_LABEL still requires an enum-typed field reference; INSERT values have no current row.

<a id="pitfall-label-an-enum-field-under-its-own-name-through-a-qualified-input"></a>

### Label an enum field under its own name through a qualified input

`FROM [App].[Tasks] SELECT [Id], ENUM_LABEL([Pri]) AS [Pri]` is `AMBIGUOUS_FIELD`, because `[Pri]` inside the projection names both the stored field and the new alias. Qualify the field — `ENUM_LABEL([Tasks].[Pri]) AS [Pri]` — or keep its slot with `SELECT * REPLACE (ENUM_LABEL([Pri]) AS [Pri])`, or choose another alias such as `[PriLabel]` ([alias rules](queries.md#alias-repeating-an-input-field)). An enum field also takes its member's underlying value, never its label, in INSERT and UPDATE documents (`[Pri]: 3`, not `[Pri]: 'High'`); [`ENUM_VALUE`](#enum_value--label--value-scalar-function) converts a label where one is needed.

<a id="pitfall-a-row-varying-enum_value-enum-name-projects-as-string"></a>

### A row-varying `ENUM_VALUE` enum name projects as `STRING`

`ENUM_VALUE('label', 'EnumName')` with a literal enum name, or a bound string parameter in a query, exposes the enum's exact underlying wire type: `INT64` for `INT*`, `GUID` for `GUID`. A schema-owned computed column can likewise declare the matching type directly. If the enum name comes from a row expression, different rows could select enums with different underlying types, so the result schema is `STRING`; compare or cast explicitly when a downstream consumer requires one underlying type.

<a id="pitfall-enum-descriptions-are-case-insensitive-identifiers"></a>

### Enum descriptions are case-insensitive identifiers

Descriptions are not merely display text: `ENUM_VALUE` resolves them with `OrdinalIgnoreCase`. Two members therefore cannot use descriptions that differ only by case; both `CREATE ENUM` and `ALTER ENUM ... ADD VALUE` reject the second label. Underlying values are canonicalized before the same uniqueness check, so integer or GUID spelling aliases cannot create duplicate physical members either.
