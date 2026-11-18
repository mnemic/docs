# Schema definition (DDL)

> **In this section** — `CREATE` / `ALTER` / `DROP COLLECTION`, field types and modifiers, foreign-key declarations, `DESCRIBE COLLECTION`, `ALTER NAMESPACE`.

> **Schema-changing DDL is migration-scoped by default** — database-schema and DDL-registry mutations appear between `BEGIN MIGRATION` and `COMMIT MIGRATION` (see [Migrations](migrations.md)). That covers `CREATE/ALTER/DROP/RENAME COLLECTION`, `RENAME NAMESPACE`, `CREATE/ALTER/DROP INDEX`, `CREATE/ALTER/DROP ENUM`, `CREATE/ALTER/DROP TYPE`, `SET CREATEDBY` / `TEMPORAL` / `SKIPUNCHANGEDUPDATES` / `CACHECURRENTVERSION`, `CREATE/DROP TRIGGER`, `CREATE/ALTER/DROP VIEW`, and `CREATE/ALTER/REFRESH/DROP MATERIALIZED VIEW`. Online index DDL is the schema-changing exception: it commits independently, must run standalone, and is rejected inside a migration. A database admits one migration at a time: a second `BEGIN MIGRATION` fails with the transient `MIGRATION_IN_PROGRESS` ([migrations](migrations.md)), and a migration whose recorded dependency standalone structural work (an online index build) changed before it committed is rolled back with the transient `MIGRATION_CONFLICT`. Read-only DDL (`DESCRIBE COLLECTION`, `EXPLAIN`) is also exempt.

## CREATE COLLECTION

```
CREATE COLLECTION [Namespace].[Collection]
  [(option: TRUE|FALSE [, option: TRUE|FALSE]*)]
  [EXTENDS [Namespace].[Type] [, [Namespace].[Type]]*]
  WITH SCHEMA (field_definitions)
  [WITH SUBTYPE [value] (field_definitions)]*
  [WITH CHECK [name] (expression)]*

option := Temporal | CreatedBy | SkipUnchangedUpdates | CacheCurrentVersion
```

The option list names each of `Temporal`, `CreatedBy`, `SkipUnchangedUpdates`, and `CacheCurrentVersion` at most once, in any order and in any letter case; an omitted option is `FALSE`, a repeated one is a parse error (`Duplicate option 'Temporal' …`), and an empty list `()` is rejected. `CacheCurrentVersion: TRUE` requires `Temporal: TRUE`: a standard collection has no versions to cache, so the combination is `INVALID_DEFINITION`, as `SET CACHECURRENTVERSION TRUE` on one is. The parts follow a fixed order: the option list comes before `EXTENDS`, so `CREATE COLLECTION [App].[A] EXTENDS [App].[T] (Temporal: TRUE) WITH SCHEMA (…)` is a parse error, and both come before the `WITH` clauses. The `WITH SCHEMA`, `WITH SUBTYPE`, and `WITH CHECK` clauses may appear in any order among themselves; `WITH SCHEMA` appears exactly once.

```sql
CREATE COLLECTION [App].[Contacts] (Temporal: TRUE, CreatedBy: FALSE) EXTENDS [App].[Audited]
WITH SCHEMA ([Id] INT32 PRIMARY KEY, [Name] STRING)
WITH CHECK [named] ([Name] <> '');
```

`WITH SCHEMA` is the sole authoritative schema input. The exact-v1 `ExecuteMql` frame carries flags, MQL text, and a mandatory parameter count; there is no separate binary-schema field. Every individual schema field list accepts at most 1,024 fields. This includes top-level collection/type schemas, nested OBJECT schemas, ARRAY OBJECT element schemas, and subtype extension schemas; text and programmatic ASTs share the bound. Each collection-level or field-level subtype union likewise accepts at most 1,024 subtypes, matching the schema codec's persisted subtype-count limit.

Temporal collections retain past and current versions; ordinary `DELETE` closes validity without reclaiming history. `CANCEL VERSION` physically removes only a future version. MQL has no automatic retention policy or general historical purge.

`CreatedBy` defaults to FALSE independently of Temporal. TRUE requires an effective EXTERNAL or COLLECTION database policy and explicit actor/copy intent on authored writes. Change the option inside a migration with:

```mql
ALTER COLLECTION [App].[Items] SET CREATEDBY TRUE;
ALTER COLLECTION [App].[Items] SET CREATEDBY FALSE;
```

Enabling permits unattributed existing rows. Disabling preserves retained metadata; subsequent authored writes carry no actor. The collection setting and policy remain private until migration commit. See [attribution](dml.md#createdby-attribution).

`SkipUnchangedUpdates` and `CacheCurrentVersion` change the same way:

```mql
ALTER COLLECTION [App].[Items] SET SKIPUNCHANGEDUPDATES TRUE;
ALTER COLLECTION [App].[Items] SET CACHECURRENTVERSION FALSE;
```

`SET SKIPUNCHANGEDUPDATES` decides whether an UPDATE or REPLACE that leaves a document unchanged writes nothing unless it says `FORCE WRITE` (see [SKIP IF UNCHANGED](dml.md#skip-if-unchanged)); the migration's own writes follow it at once and other sessions after commit. `SET CACHECURRENTVERSION` turns the in-memory cache of a temporal collection's current versions on or off once the migration commits, since the cache serves every session; an aborted migration leaves it as it was, and a standard collection, which has no versions to cache, refuses it with `INVALID_DEFINITION`. Neither rewrites a document.

<a id="change-the-primary-key"></a>

`SET PRIMARY KEY` makes another field the identity of every document:

```mql
ALTER COLLECTION [App].[Items] SET PRIMARY KEY [Code];
```

The field is a top-level field of every document, stored and required (`NOT NULL`), without a default, since a key is supplied or generated (`DROP DEFAULT` first), of a key type: `INT8`, `INT16`, `INT32`, `INT64`, `GUID`, `STRING` or `BYTES`; a nested path, an element path or a subtype field (`[Address].[Zip]`) is `INVALID_STATEMENT`, another top-level field `INVALID_DEFINITION`, and a name the base schema lacks `UNKNOWN_FIELD`. Every retained version, history included, is re-keyed by its own value of the field, every index is rebuilt — a temporal unique index on its valid-time rule ([UNIQUE on a temporal collection](indexes.md#unique-on-a-temporal-collection)) — and the former key stays an ordinary required field without its generator. Revisions, creation times, attribution and seals stay with their documents. The values must identify the documents one to one, compared as keys compare (`STRING` keys ignore case): two documents sharing a value refuse the change with `UNIQUE_VIOLATION`, and one document whose versions hold different values with `OBJECT_NOT_IN_PREREQUISITE_STATE`. What holds the old identities refuses it with `DEPENDENT_OBJECTS_EXIST` until it is dropped: a reference from any collection, a CreatedBy policy whose users are documents of this collection, and a vector index; the current-version cache refuses it with `INVALID_DEFINITION`. Subscriptions to single documents by their old key stop matching. Naming the key the collection has changes nothing, whatever holds its identities.

<a id="change-the-key-generator"></a>

`ALTER FIELD [key] SET AUTOINCREMENT` and `SET SEQUENTIAL` give the primary key the generator those modifiers declare at creation, and `DROP AUTOINCREMENT` and `DROP SEQUENTIAL` take it away:

```mql
ALTER COLLECTION [App].[Items] ALTER FIELD [Id] SET AUTOINCREMENT;   -- an INT32 or INT64 key
ALTER COLLECTION [App].[Events] ALTER FIELD [Id] SET SEQUENTIAL;     -- a GUID key
ALTER COLLECTION [App].[Items] ALTER FIELD [Id] DROP AUTOINCREMENT;  -- inserts supply the key again
```

No document is rewritten, and the rules are those of creation: only the collection's primary key takes a generator, `AUTOINCREMENT` an `INT32` or `INT64` key and `SEQUENTIAL` a `GUID` key, and a key has one at most. Another field, a key of another type and a second generator are `INVALID_DEFINITION` (drop the other generator first), and so are a subtype scope and a path (`ALTER FIELD [Ship].[No]`), which name no primary key; a name the base schema lacks is `UNKNOWN_FIELD`, and `ALTER TYPE ... ALTER FIELD` refuses the four forms as a `SYNTAX_ERROR`, since a type has no primary key. An added `AUTOINCREMENT` starts after the highest key the collection retains — current, deleted and historical versions, and the rows the migration itself inserted before the statement — and never below the counter the key had before, so a generated key never repeats a stored one: after keys 1, 3 and 7 the next insert that omits the key gets 8. From then on an insert supplies a key only with [`FORCE ID`](dml.md#insert), as on a key declared `AUTOINCREMENT`. `SET SEQUENTIAL` generates a UUIDv7 for an insert that omits the key ([`SEQUENTIAL`](#sequential--timestamp-local-guid-keys)). A dropped generator makes inserts supply the key again, checked for uniqueness like any other key; `DROP AUTOINCREMENT` keeps the counter's high-water mark, so a later `SET AUTOINCREMENT` never hands out a key the counter already did. The migration's own later statements see the change at once and other sessions once it commits; an aborted migration leaves the generator as it was. Setting the generator the key has, or dropping one it lacks, changes nothing.

<a id="change-temporal-mode"></a>

`SET TEMPORAL` changes whether the collection keeps valid-time versions:

```mql
ALTER COLLECTION [App].[Items] SET TEMPORAL TRUE;   -- each document becomes an open version from now
ALTER COLLECTION [App].[Items] SET TEMPORAL FALSE;  -- each current version stays; history is removed
```

Both rewrite every retained row and rebuild every index in the migration. Other sessions wait for the collection until the migration commits, and an aborted migration leaves it as it was; the migration's own later statements already see the new mode. With `TRUE` each document's state becomes a version valid from the statement's time without an end, keeping its revision, creation time, attribution and seal; there is no earlier history, so `AS OF` an earlier instant finds nothing. With `FALSE` each document keeps the version current at the statement's time, without validity, and every closed version and every document without a current version is removed. `FALSE` is refused while a version or an end is scheduled after now, which a standard collection cannot hold (`OBJECT_NOT_IN_PREREQUISITE_STATE`: cancel or end it first); while a pinned reference targets the collection, or a temporal collection references it, since their history may name the versions it removes (`DEPENDENT_OBJECTS_EXIST`); while the current-version cache is on (`INVALID_DEFINITION`: `SET CACHECURRENTVERSION FALSE` first, in the same migration); and while a view, trigger or materialized view reads it `AS OF` or `IN BETWEEN` (`WRONG_OBJECT_TYPE`). Setting the mode the collection has changes nothing. A `UNIQUE` index takes the rule of the new mode: valid-time uniqueness once temporal, where each document's later versions share its keys ([UNIQUE on a temporal collection](indexes.md#unique-on-a-temporal-collection)), and one owner per key once standard. Neither direction can create a duplicate, because each document keeps exactly one version.

```sql
-- Standard collection
CREATE COLLECTION [Store].[Products]
WITH SCHEMA ([Id] GUID PRIMARY KEY, [Name] STRING, [Price] DECIMAL, [Active] BOOL);

-- Temporal with auto-increment
CREATE COLLECTION [HR].[Employees] (Temporal: TRUE)
WITH SCHEMA ([Id] INT64 PRIMARY KEY AUTOINCREMENT, [Name] STRING, [Department] STRING);

-- All options
CREATE COLLECTION [App].[Items] (Temporal: TRUE, SkipUnchangedUpdates: TRUE, CacheCurrentVersion: TRUE)
WITH SCHEMA ([Id] GUID PRIMARY KEY, [Name] STRING NOT NULL, [Score] DECIMAL);

-- With foreign keys
CREATE COLLECTION [App].[Orders] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [CustomerId] GUID REFERENCES [App].[Customers] ON REMOVE Restrict,
  [TagIds] ARRAY GUID REFERENCES [App].[Tags] ON REMOVE Cascade,
  [BackupRef] GUID NULLABLE REFERENCES [App].[Others] ON REMOVE SetNull
);

-- Polymorphic collection
CREATE COLLECTION [Fleet].[Vehicles]
WITH SCHEMA ([Id] GUID PRIMARY KEY, [Make] STRING, [Year] INT32)
WITH SUBTYPE [car] ([Doors] INT32, [Sunroof] BOOL)
WITH SUBTYPE [truck] ([PayloadTons] DECIMAL, [Axles] INT32);

-- Polymorphic array field
CREATE COLLECTION [App].[Items] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [Components] ARRAY OBJECT ([Type] STRING, [Power] INT32)
    WITH SUBTYPE [battery] ([Capacity] INT32)
    WITH SUBTYPE [motor] ([Torque] INT32)
);

-- With CHECK constraints (one or more, named or unnamed)
CREATE COLLECTION [App].[Items] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Name] STRING,
  [Qty] INT32
)
WITH CHECK [non_negative_qty] ([Qty] >= 0)
WITH CHECK (LENGTH([Name]) > 0);

-- With pinned foreign-key references (temporal target only;
-- ON REMOVE Cascade rejected — see "Pinned references" below)
CREATE COLLECTION [App].[Orders] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [PriceList] GUID PINNED REFERENCES [App].[PriceLists] ON REMOVE Restrict
);
```

Every collection has exactly one top-level stored primary key. Its type must be `INT8`, `INT16`, `INT32`, `INT64`, `GUID`, `STRING`, or `BYTES`; `DATETIME` and the remaining scalar types are not legal primary-key types. `AUTOINCREMENT` is limited to `INT32` and `INT64`, while `SEQUENTIAL` is limited to `GUID`.

### Defaults at creation

Declare a default at the end of a field definition, after its nullability and any REFERENCES modifiers:

```sql
CREATE COLLECTION [Core.Billing.Tariff].[Tariff]
WITH SCHEMA (
  [TariffId] INT32 PRIMARY KEY AUTOINCREMENT,
  [CreatedAt] DATETIME NOT NULL DEFAULT NOW,
  [Name] STRING NOT NULL DEFAULT 'Untitled'
);
```

`DEFAULT <literal>` and `DEFAULT NOW` work in CREATE COLLECTION and CREATE TYPE, including nested OBJECT fields, the objects of ARRAY OBJECT fields and of arrays of arrays at every level (`ARRAY ARRAY OBJECT (...)`), and subtype extensions. They persist the same default metadata as `ALTER FIELD ... SET DEFAULT`. An omitted field receives its default; an explicit value, including NULL, wins. Explicit NULL still fails on a NOT NULL field. DEFAULT does not change nullability. `NOW` is valid for DATETIME and resolves from the clock of each statement that takes it, not the creation statement's clock. Ordinary updates do not reset these fields, while every object, or list of objects, a write assigns as a field's value takes the defaults of the fields it leaves out, at any depth, as an insertion does ([defaults](dml.md#defaults)). `[Meta].[CreatedAt]` is separately maintained revision metadata and needs no schema declaration.

The existing SET DEFAULT restrictions also apply: literal values (including object, array, enum-member, and vector literals) or NOW only; no parameters, arbitrary expressions, PIN references, or DEFAULT NULL. Primary keys, computed fields, and discriminator fields cannot have defaults. Nested defaults apply inside present parents; a child default alone does not create a missing parent. A parent default such as `OBJECT ([Code] INT32 NOT NULL DEFAULT 7) DEFAULT ()` can supply that parent and its omitted children.

CREATE defaults apply to the fields later writes leave out of a value they build whole: an inserted document and every object, or list of objects, a later write assigns ([defaults](dml.md#defaults)), and a field a write assigns `DEFAULT`, but not the top-level fields a REPLACE leaves out. In contrast, an ALTER ADD field's own `DEFAULT` is a one-time backfill; use SET DEFAULT as well to retain a default after ALTER ADD. `DEFAULT NULL` is `FEATURE_NOT_SUPPORTED` in every one of these positions — CREATE COLLECTION, CREATE TYPE, ADD FIELD on a collection or type, and SET DEFAULT — since a field without a default already reads NULL.

### CHECK constraints

`WITH CHECK [name] (expr)` after the complete `WITH SCHEMA (...)` declaration declares a collection-level boolean predicate that every INSERT, UPDATE, REPLACE, and UPSERT must satisfy. Repeat the clause to attach more than one constraint; the name is optional but surfaces in violation messages, and the names of one collection's or type's CHECKs, its own and those its `EXTENDS` bases give it, are distinct regardless of case (a repeat is `DUPLICATE_NAME`), so `DROP CHECK [name]` names one constraint. Constraints are evaluated by the same expression engine that powers `WHERE`, so the full operator + scalar-function set (LENGTH, UPPER, COALESCE, comparison, AND/OR, …) is available. A CHECK fails only when its final result is FALSE; TRUE and UNKNOWN (NULL) pass. A NULL input can still produce FALSE, for example with `[F] IS NOT NULL`. Functions such as COALESCE and conditional expressions determine their result using their own NULL semantics; a NULL operand does not automatically make the whole constraint pass. `NOW` in a CHECK is the instant of the statement whose write it checks, the same instant every `NOW` of that statement reads, so `WITH CHECK ([Stamp] = NOW)` holds for a row the statement writes with `[Stamp]: NOW`, on a collection with `STORED` computed fields as on any other.

Inside a schema, `WITH CHECK` immediately following an inline `OBJECT (...)` belongs to that object definition. For `ARRAY OBJECT (...)`, it belongs to the element's object definition and runs independently for each element. OBJECTs and ARRAY OBJECT fields can nest recursively, including OBJECT inside OBJECT, OBJECT inside ARRAY OBJECT, and nested ARRAY OBJECT fields.

Field paths bind relative to the owning object, with no fallback to ancestor or collection fields. Names are local to each object definition: different objects and the collection may reuse the same CHECK name, but duplicate names within one object are rejected case-insensitively. Each object accepts at most 256 checks. Absent or NULL objects and empty arrays contain no object instances to check; field nullability rules still apply. Place object checks before field modifiers such as `NOT NULL` and `DEFAULT`.

```mql
CREATE COLLECTION [App].[Invoices] WITH SCHEMA (
    [Id] INT32 PRIMARY KEY,
    [EmailRecipients] ARRAY OBJECT (
        [Email] STRING NOT NULL,
        [Delivery] OBJECT (
            [Label] STRING,
            [Tags] ARRAY OBJECT ([Name] STRING NOT NULL)
                WITH CHECK [NameLength] (LENGTH([Name]) <= 30)
        ) WITH CHECK [LabelLength] (LENGTH([Label]) <= 50)
    ) WITH CHECK [EmailLength] (LENGTH([Email]) <= 100) NOT NULL DEFAULT []
) WITH CHECK [PositiveId] ([Id] > 0);
```

Here `EmailLength` checks each recipient, `LabelLength` checks that recipient's Delivery object, and `NameLength` checks each Delivery.Tags element. `PositiveId` checks the collection row. Inline object checks also retain their scope through `CREATE TYPE` and `ALTER ... ADD FIELD` declarations, printing, and database reopen.

The CHECKs of an object with subtypes ([field-level polymorphism](polymorphism.md#field-level-polymorphism-polymorphic-sub-documents)) check each value by the subtype it holds. They test it with `SUBTYPE IS [Name]`, `SUBTYPE IS NOT [Name]` and `SUBTYPE [NOT] IN ([A], [B])` and read the fields its subtypes add: a value reads a field by the declaration of its own subtype, and a field its subtype lacks reads NULL, so a rule for some subtypes names them. A subtype test names only subtypes the object declares; another name is `INVALID_DEFINITION` (`Unknown subtype 'X'. Known subtypes: …`) when the CHECK is declared, as it is in a collection CHECK's test of the collection's subtypes or of an array element's inside `MATCH`. While a CHECK names one of a field's subtypes, or reads a field only that subtype declares, `ALTER FIELD ... DROP SUBTYPE` refuses it ([a polymorphic field's subtypes](#field-subtypes)).

```mql
CREATE COLLECTION [Billing].[Tariffs] WITH SCHEMA (
    [Id] INT32 PRIMARY KEY,
    [Components] ARRAY OBJECT ([Amount] DECIMAL NOT NULL)
        WITH CHECK [Payer] (SUBTYPE NOT IN ([Subscription], [StationFee])
            OR ([PayerMode] = 1 AND [PayerId] IS NOT NULL)
            OR ([PayerMode] = 2 AND [PayerId] IS NULL))
        WITH SUBTYPE [Usage] ()
        WITH SUBTYPE [Subscription] ([PayerMode] INT16 NOT NULL, [PayerId] INT32)
        WITH SUBTYPE [StationFee] ([PayerMode] INT16 NOT NULL DEFAULT 2, [PayerId] INT32)
        NOT NULL DEFAULT []
);
```

`Payer` checks each Subscription and StationFee element by its own payer fields and passes every Usage element, which has none.

String size constraints use CHECK expressions; `STRING(50)`, `VARCHAR(50)`, and `CHAR(2)` are not field-type declarations in MQL. For example:

```sql
CREATE COLLECTION [App].[Labels] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Country] STRING NOT NULL,
  [Name] STRING NOT NULL
)
WITH CHECK [CountryLength] (LENGTH([Country]) = 2)
WITH CHECK [NameLength] (LENGTH([Name]) <= 50);
```

`LENGTH` counts UTF-16 code units, including trailing spaces. The first check requires exactly two units; the second permits zero through fifty. `NOT NULL` rejects NULL independently and does not reject the empty string.

One collection accepts at most 256 table-level CHECK constraints, the maximum representable by the persisted schema codec. Parsing and direct AST validation enforce the limit before reading or traversing another expression. When the collection uses `EXTENDS`, inherited type CHECKs and locally declared CHECKs share this 256-entry total.

CHECK expressions are self-contained schema metadata. Every field reference must resolve against its owning object schema or, for collection checks, the collection schema, the fields their subtypes add included. Query parameters are rejected, and a CHECK cannot reference a virtual computed column; stored computed values are available. Invalid persisted bindings fail executor construction with `CHECK_EXPRESSION_INVALID` rather than being interpreted as NULL.

Embedded hosts can publish metadata after executor initialization. The first named-type use, ALTER, computed query, or object-relative CHECK evaluation that observes invalid live metadata fails with `CHECK_EXPRESSION_INVALID` or `COMPUTED_EXPRESSION_INVALID` and quarantines the catalog domain; an empty collection does not postpone computed validation until a row is enumerated.

> **Persistence.** CHECK constraints are persisted in the catalog and survive engine restart with no operator intervention. `DROP COLLECTION` removes the persisted CHECK constraints alongside the collection.

A migration adds and drops the CHECKs of an existing collection, of its object fields, and of a named type; see [Add and drop CHECK constraints](#add-and-drop-check-constraints).

Typed columns (`GUID`, `DATETIME`) compare against string literals as you'd expect — the literal is read as a value of the column's type before comparison. The common "non-empty Guid" sentinel predicate is therefore safe to write directly:

```sql
WITH CHECK [owner_set] ([Owner] != '00000000-0000-0000-0000-000000000000')
WITH CHECK [created_after_2020] ([CreatedAt] >= '2020-01-01T00:00:00Z')
```

A string literal that names no value of the column's type (e.g. `'not-a-guid'` against a `GUID` column) refuses the definition with `INVALID_VALUE` before it is published, as the same comparison refuses a query ([a literal is read as a value of the type it is compared with](queries.md#comparison-value-notes)).

Foreign-key targets bind to internal collection identities. Renaming or moving either collection preserves the relationship without rewriting the referencing schema or documents. Reusing the former target name does not redirect the constraint. Metadata shows the target's current name. Dropping the target requires removing its referencing fields or collections in the same migration; commit rejects any remaining dangling identity, even when no referencing rows remain. Recreating a target under its former name requires a new FK declaration.

### Pinned references (`PINNED REFERENCES`)

A pinned FK records both the target document's id AND its revision, so the source row keeps pointing at the exact version it was written against even if the target later expires its `ValidTo` or rolls forward to a new version. Use `PIN(id, revision)` in INSERT / UPDATE / REPLACE values to write a pinned reference.

```sql
-- Pinned FK against a temporal price list
CREATE COLLECTION [App].[Orders] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [PriceList] GUID PINNED REFERENCES [App].[PriceLists] ON REMOVE Restrict
);

-- Inserting a pinned reference
INSERT INTO [App].[Orders] (
  [Id]: '...',
  [PriceList]: PIN('a1b2c3d4-...-...', 7)
);

-- Pinned-array FK: per line-item revision binding on an invoice
CREATE COLLECTION [App].[Invoices] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [TariffRefs] ARRAY GUID PINNED REFERENCES [App].[Tariffs]
    ON REMOVE Restrict
);

INSERT INTO [App].[Invoices] (
  [Id]: '...',
  [TariffRefs]: [PIN('t1-...-...', 3), PIN('t2-...-...', 7)]
);
```

`PINNED REFERENCES` requires a temporal target. Standard collections may advance a row's revision, but they do not retain addressable historical bodies. `ON REMOVE Cascade` is rejected at registration — pinning is intentional immutability; cascading deletion would defeat the point. For **scalar** pinned fields, `Restrict` (default) blocks `CANCEL VERSION AT` against any pinned target version, and `SetNull` (nullable fields only) clears the complete pin when the bound revision is cancelled. For **pinned-array** fields, `Restrict` blocks `CANCEL VERSION AT` when *any* element binds the cancelled revision and `SetNull` drops every element matching both id and revision (the surrounding elements and same-id bindings to other revisions survive).

These removal actions also apply when `REPLACE`, a covering temporal INSERT, or `CANCEL VERSION ... UPDATE PREVIOUS SET` removes the last retained fragment of the bound revision. A surviving front or tail keeps that revision addressable by `PIN`, even when the original physical segment was replaced. All reference changes commit or roll back with the parent mutation.

Scalar pinned fields participate in expressions as their declared id type: comparison, ordering, grouping, joins, and window `PARTITION BY` use the id half and do not split partitions by revision. The revision tail remains part of the stored pin and is validated as exact framing; it is not exposed as trailing scalar bytes. The two halves are field paths of their own: `[PriceList].[Id]` reads the identifier and `[PriceList].[Revision]` the bound revision as `INT64`, so `FROM [App].[Orders] WHERE [PriceList].[Revision] = 7 SELECT [Id], [PriceList].[Revision] AS [Rev]` filters by and projects the revision. Projecting the field itself returns the complete pin ([nested field access](queries.md#nested-field-access)).

Pinned references work in both base and polymorphic-extension positions — including collection-level subtype extensions (`WITH SUBTYPE [X] ([PinnedFk] GUID PINNED REFERENCES …)`) and field-level subtype extensions on a nested polymorphic Object. The pinned slot rides on the per-subtype extension binary; reads and cascade mutations route through the engine's envelope-aware FK plumbing.

```sql
-- Pinned FK in a collection-level subtype extension
CREATE COLLECTION [App].[Invoices] WITH SCHEMA (
  [Id] GUID PRIMARY KEY, [Kind] STRING
)
WITH SUBTYPE [Paid] (
  [TariffRef] GUID PINNED REFERENCES [App].[Tariffs] ON REMOVE Restrict,
  [PaidAmount] INT32
)
WITH SUBTYPE [Free] ([Note] STRING);

INSERT INTO [App].[Invoices] SUBTYPE [Paid] (
  [Id]: '...', [Kind]: 'Paid',
  [TariffRef]: PIN('a1b2-...-...', 1), [PaidAmount]: 500
);
```

A migration gives an existing pinned field a reference, changes its policy or drops it with [`ALTER FIELD … SET PINNED REFERENCES` and `DROP REFERENCES`](#change-a-reference); the field stays stored pinned throughout.



## Field types

| Type | Aliases | Size | Description |
|------|---------|------|-------------|
| `STRING` | | Variable | UTF-8 text |
| `INT8` | | 1 byte | Signed 8-bit integer |
| `INT16` | | 2 bytes | Signed 16-bit integer |
| `INT32` | `INT` | 4 bytes | Signed 32-bit integer |
| `INT64` | `LONG` | 8 bytes | Signed 64-bit integer |
| `BOOL` | | 1 byte | Boolean |
| `FLOAT32` | `SINGLE` | 4 bytes | 32-bit floating point (IEEE 754 single) |
| `FLOAT64` | `FLOAT`, `DOUBLE` | 8 bytes | 64-bit floating point (IEEE 754 double) |
| `VECTOR(n)` | | `4 × n` bytes | Fixed-dimension dense `FLOAT32` vector; `n` is 1–32,768 |
| `DECIMAL` | | 16 bytes | 128-bit decimal |
| `GUID` | | 16 bytes | UUID |
| `DATETIME` | | 10 bytes | DateTimeOffset (UTC ticks + offset) |
| `DATEONLY` | | 4 bytes | Date only (day number) |
| `TIMEONLY` | | 8 bytes | Time only (ticks since midnight) |
| `BYTES` | | Variable | Raw binary data |
| `OBJECT (...)` | | Variable | Nested object with sub-fields |
| `ARRAY type` | | Variable | Array of scalar type |
| `ARRAY OBJECT (...)` | | Variable | Array of objects |
| `ARRAY ARRAY type` | | Variable | Nested arrays; each ARRAY declares one element level |

Custom enum types can also be used as field types (see [Enums](enums.md)). Vector values, literals, and nearest-neighbour indexes are documented in [Vector values and nearest-neighbour search](vectors.md). An empty nested schema is written with its required delimiters: `OBJECT ()` or `ARRAY OBJECT ()`. Canonical output retains the empty parentheses, so the statement remains parseable when printed and replayed. An empty Object value is `()` and remains present, distinct from NULL. Empty Object elements retain their positions inside flat or nested arrays. ALTER accepts the same explicit `OBJECT ()` shape. Unrelated field changes preserve empty Object values, and `ADD FIELD [F] OBJECT () DEFAULT ()` backfills a present empty Object. Adding a child to an existing empty Object materializes the child's default in present parents; NULL parents remain NULL.

Nested `OBJECT` and `ARRAY` declarations share a **100-level parser guard**. The persisted schema separately enforces a **16-level recursive depth limit**; a declaration that parses can still exceed the schema storage bound and fail DDL validation.

`BYTES` literals use Base64 text. The empty literal `''` is a present, zero-length byte array and is distinct from `NULL`: it participates in equality, secondary indexes, joins, grouping, `COUNT([field])`, and `COUNT(DISTINCT [field])` as a non-null value.

Nested arrays use one recursively owned element schema per ARRAY keyword. Nested literals preserve empty arrays, scalar types, Object fields, and Object subtypes. Field membership, enum membership, and required fields are validated at every Object element; NULL array elements are invalid at every level.

## Field modifiers

Modifiers appear after the type in this order:

```
[FieldName] TYPE [NOT NULL | NULLABLE] [PRIMARY KEY] [AUTOINCREMENT | SEQUENTIAL]
  [AS (<expression>)]
  [[PINNED] REFERENCES [Ns].[Col] [ON REMOVE action] [MATCH FULL | MATCH SIMPLE]]
```

```sql
-- Nullability
[Name] STRING NOT NULL              -- required field
[Email] STRING NULLABLE             -- explicit nullable (the default for every type; the primary key is never nullable)

-- Primary key (one top-level stored field per collection)
[Id] GUID PRIMARY KEY
[Id] GUID PRIMARY KEY SEQUENTIAL     -- engine-generated UUIDv7 when omitted on INSERT
[Id] INT32 PRIMARY KEY AUTOINCREMENT
[Id] INT64 PRIMARY KEY AUTOINCREMENT
[Code] STRING PRIMARY KEY
[Digest] BYTES PRIMARY KEY

-- Foreign key references
[CustomerId] GUID REFERENCES [Store].[Customers]
[CustomerId] GUID REFERENCES [Store].[Customers] ON REMOVE Cascade
[TagIds] ARRAY GUID REFERENCES [Store].[Tags] ON REMOVE Cascade
[ParentId] INT32 REFERENCES [Store].[Parents] ON REMOVE Restrict
[RefId] GUID NULLABLE REFERENCES [Store].[Others] ON REMOVE SetNull
[CountryCode] STRING REFERENCES [Geo].[Countries]
[ContentHash] BYTES REFERENCES [Store].[Blobs]

-- MATCH FULL on an array-of-object FK (all-or-nothing per array value)
[Contributors] ARRAY OBJECT (
  [AuthorId] GUID REFERENCES [App].[Authors] MATCH FULL,
  [Role]     STRING
)
```

Primary-key types are `INT8`, `INT16`, `INT32`, `INT64`, `GUID`, `STRING`, and `BYTES`; another type is `INVALID_DEFINITION`. A key may also declare `REFERENCES` (a one-to-one relationship), or take one later with [`ALTER FIELD [Id] SET REFERENCES`](#change-a-reference), but never `PINNED REFERENCES`: a key is the identifier itself, not an identifier-and-revision pin, so `[Id] INT32 PRIMARY KEY PINNED REFERENCES [App].[Owners]` is `INVALID_DEFINITION`. Signed integer identities retain their native one-, two-, four-, or eight-byte widths. String identity uses invariant simple-lowercase Unicode without normalization: the stored value preserves its spelling, while lookup, uniqueness, joins, foreign keys, snapshots, and subscriptions compare the folded identity. Bytes identity uses exact bytes. `''` is a present empty String key and also the Base64 spelling of a present empty Bytes key; neither is null. `WHERE [Id] IS NULL` (or `= NULL`) therefore matches no primary-key rows, including temporal queries and mutation targets; it never selects an empty key.

String and Bytes identities remain on the row's home data page even when the document body spills. A key that cannot fit with the slot metadata is rejected before the write mutates storage. `AUTOINCREMENT` remains `INT32` / `INT64`-only and `SEQUENTIAL` remains Guid-only; a migration adds either to an existing key or drops it with [`ALTER FIELD [key] SET | DROP AUTOINCREMENT | SEQUENTIAL`](#change-the-key-generator).

Ordinary foreign keys support all seven primary-key types in scalar fields, packed arrays, and fields repeated inside array objects. String array references use invariant simple-lowercase identity; Bytes references use exact bytes. Empty String/Bytes values are present references and require a matching target. Null array-object fields are absent references. These rules also apply inside nested objects and polymorphic subtypes, through rollback and reopen. Object segments may appear before or between array dimensions. Packed reference arrays inside array objects follow the same validation and removal rules as root arrays; nullable scalar leaves retain their surrounding objects and unrelated keys.

```sql
[RelatedCodes] ARRAY STRING REFERENCES [App].[Products] ON REMOVE Cascade
[Attachments] ARRAY BYTES REFERENCES [App].[Blobs] ON REMOVE Restrict
[Contributors] ARRAY OBJECT (
    [Code] STRING REFERENCES [App].[People] ON REMOVE SetNull
)
```

`Cascade` on a packed array removes every matching identifier and retains the other elements; `SetNull` on an array-object leaf clears matching fields, or under `MATCH FULL` the field of every element of that value (below), while preserving the surrounding objects. A compatible array-element index can accelerate membership reads over an ordinary packed reference array. Malformed array framing or invalid UTF-8 in stored String reference arrays fails closed with `ARRAY_PAYLOAD_CORRUPT` before target lookup.

`PINNED REFERENCES` supports the same seven identifier types, including `STRING` and `BYTES`, in scalar fields, packed arrays, and array-object leaves. Use `PIN(id, revision)` for each present pin. Empty identifiers are valid; `NULL` denotes an absent scalar reference. Cancellation matches the identifier and exact revision, so `SetNull` retains other revisions of that identifier, except inside an array-object value under `MATCH FULL`, which it clears whole.

**`MATCH FULL` / `MATCH SIMPLE`** apply to a foreign key whose value can be *partially* null — in practice an `ARRAY OBJECT(...)` whose FK field is nullable, where some elements reference and others are null. `MATCH SIMPLE` (the default) validates each element independently: a null FK field skips, and a partially-null array is accepted as long as every non-null reference resolves. `MATCH FULL` makes the array **all-or-nothing**: within one array value, the FK field must be non-null in *every* element or null in *every* element; a mix is rejected with an `FK MATCH FULL violation` (an empty / all-null array is vacuously clean). Under `MATCH FULL` an `ON REMOVE SetNull` treats the row's array value as one reference, as SQL's `ON DELETE SET NULL` nulls every column of a `MATCH FULL` key: when any element names the removed target (for a pinned reference, the cancelled revision), it clears the FK field in every element of that value, including elements naming targets that still exist, so the row is never left partly referencing and the removal is never refused for it. The value is every occurrence the rule counts: each element's FK field across nested arrays, object segments and active subtype envelopes, limited to the occurrences that declare the same policy. A `DELETE`, the end of a temporal row, `CANCEL VERSION`, a statement removing several targets of one value, and wall-clock expiry all settle each row this way; to keep the other references, repoint them before the removal or declare `MATCH SIMPLE`, under which `SetNull` clears only the matching elements. `MATCH` is accepted by CREATE COLLECTION and ALTER ADD FIELD. For a single-column FK the two modes are identical (a single value is null or not), and scalar array elements cannot be null, so `MATCH FULL` only has teeth on array-of-object FKs. Counts follow the complete nested path, including object segments between arrays and active subtype envelopes; empty inner arrays contribute no leaf positions.

### `SEQUENTIAL` — timestamp-local Guid keys

`AUTOINCREMENT` is the engine-assigned-PK modifier for `INT32` / `INT64` keys (a monotone counter). `SEQUENTIAL` is the corresponding generation option for `GUID`: when an `INSERT` omits the PK, the engine generates an RFC 9562 UUIDv7. Its first 48 bits carry a millisecond timestamp, and Guid index keys use RFC network byte order. With a normally advancing clock, new timestamps therefore cluster near the current right-side key range instead of scattering like UUIDv4.

UUIDv7 is not a gap-free or strictly increasing sequence. The runtime fills the remaining uniqueness bits randomly, so values generated within the same millisecond have no guaranteed order. Clock rollback and explicitly supplied Guid values can also insert into an earlier range.

```sql
CREATE COLLECTION [App].[Events] WITH SCHEMA (
  [Id]   GUID PRIMARY KEY SEQUENTIAL,
  [Name] STRING
);
INSERT INTO [App].[Events] ([Name]: 'click');   -- [Id] is a generated UUIDv7
```

- Valid only on a `GUID` **primary key** (rejected on non-Guid types and non-PK fields; use `AUTOINCREMENT` for `INT32` / `INT64`).
- Mutually exclusive with `AUTOINCREMENT` and `AS (<expression>)`.
- Added to or dropped from an existing `GUID` key with [`ALTER FIELD [Id] SET | DROP SEQUENTIAL`](#change-the-key-generator).
- An explicitly supplied PK value wins — `SEQUENTIAL` only fills an omitted key, never overrides one the INSERT provides.
- Use `AUTOINCREMENT` when the application needs monotone numeric reservations; `SEQUENTIAL` promises timestamp locality, not sequence numbers.
- The key is still a 16-byte `Guid`; no on-disk row-format change.

## ALTER COLLECTION

Schema changes bind dependent plain views against the migration's resulting schema. Compatible changes succeed; an invalid direct or transitive view aborts the migration. Pending view replacements and removals participate in this check, which repeats at commit against current dependencies. See [view lifecycle rules](views.md#lifecycle-and-write-rules). [Trigger dependencies](triggers.md) also bind against the resulting schema, including bodies owned by other collections; invalid references abort the migration. [Materialized-view source changes](materialized-views.md#source-schema-changes) validate saved queries and rebuild affected maintained generations in the same migration.

Row-rewriting forms (`ADD FIELD`, `DROP FIELD`, `ADD SUBTYPE … FOR EXISTING DOCUMENTS`, `RENAME SUBTYPE`, `DROP SUBTYPE`, `MOVE FIELD`, `SET EXTENDS`, `DROP EXTENDS`) include the migration's prior writes and private appended rows. They may follow DML or an earlier rewrite of the same collection. The owner sees each new generation; competing writers remain fenced until commit or rollback. See [Migrations](migrations.md#multi-step-migration).

A field composed of a named type, or an `ARRAY` of one, holds the type's fields and CHECKs, which `ALTER TYPE` changes in every use at once. An action whose path reaches inside such a field (`[Ship].[Zip]`, `[Stops][].[Zip]`, or a CHECK of `[Ship]` itself), in the base schema, a subtype or a polymorphic field's subtype, is `FEATURE_NOT_SUPPORTED`; the field itself stays the collection's, and `SET TYPE OBJECT` or `ARRAY OBJECT` detaches it from its type ([composition](composite-types.md#composition--a-field-whose-type-is-a-composite-type)).

A field the collection inherits through `EXTENDS` is its declaring type's, field, nested fields, elements and CHECKs alike. Every action on an inherited field or on a path inside one — `ADD FIELD [Addr].[Floor]`, `DROP`, `RENAME` or `MOVE FIELD`, `ALTER FIELD` with `SET` or `DROP DEFAULT`, `NOT NULL`, `TYPE`, `COMPUTED`, `STORED`, `REFERENCES`, `AUTOINCREMENT` or `SEQUENTIAL`, `ALTER FIELD [Addr] ADD` or `DROP CHECK`, and `SET PRIMARY KEY` — is `FEATURE_NOT_SUPPORTED`, naming the type that declares the field; change it with [`ALTER TYPE`](composite-types.md#alter-type--evolve-a-type-propagating-to-every-user) on that type, which changes it in every collection extending the type ([inheritance](composite-types.md#extends--type-and-collection-inheritance)).

```sql
-- Add field with DEFAULT — existing rows are backfilled with the value
-- and the new field is NOT NULL. Omit DEFAULT to add the field as
-- nullable (existing rows read NULL for it; new inserts may set or
-- omit it). DEFAULT NOW (DATETIME fields only) backfills every existing
-- row with the ALTER-time timestamp; for an ongoing INSERT-time NOW
-- default, pair it with ALTER FIELD ... SET DEFAULT NOW. Parameter
-- references (@p) are rejected — a per-call value has no meaning as a
-- persistent backfill value.
ALTER COLLECTION [Store].[Products] ADD FIELD [Currency] STRING DEFAULT 'USD';
ALTER COLLECTION [Store].[Products] ADD FIELD [CreatedAt] DATETIME DEFAULT NOW;
ALTER COLLECTION [Store].[Products] ADD FIELD [Notes] STRING;

-- Add nested field
ALTER COLLECTION [Store].[Products] ADD FIELD [Details].[Color] STRING DEFAULT 'black';

-- Add a foreign key to an existing collection. Unlike CREATE inside a
-- migration (which tolerates forward references), the target MUST already
-- exist — ALTER runs against the live catalog. Without a DEFAULT the field
-- is nullable (existing rows read NULL); with a DEFAULT every existing row
-- is backfilled with that value, which is validated to resolve in the target.
ALTER COLLECTION [Store].[Products] ADD FIELD [SupplierId] GUID REFERENCES [Store].[Suppliers] ON REMOVE SetNull;
ALTER COLLECTION [Store].[Products] ADD FIELD [SupplierId] GUID REFERENCES [Store].[Suppliers] DEFAULT '…guid…';

-- Pinned FK (revision-bound) — target must be temporal. DEFAULT PIN here is
-- a one-time ALTER backfill validated against the bound revision; pinned
-- fields cannot retain a catalog default.
ALTER COLLECTION [App].[Invoices] ADD FIELD [TariffRef] GUID PINNED REFERENCES [App].[Tariffs] ON REMOVE Restrict;
ALTER COLLECTION [App].[Invoices] ADD FIELD [TariffRef] GUID PINNED REFERENCES [App].[Tariffs] DEFAULT PIN('…guid…', 3);

-- Add an ARRAY of a scalar element type (existing rows read NULL when no
-- DEFAULT is supplied; an array-literal DEFAULT backfills them). Array foreign keys are supported.
ALTER COLLECTION [Store].[Products] ADD FIELD [Tags] ARRAY STRING;
ALTER COLLECTION [Store].[Products] ADD FIELD [TagIds] ARRAY GUID REFERENCES [Store].[Tags] ON REMOVE Cascade;

-- Add an ARRAY OBJECT with a composite element schema. Existing rows
-- read NULL without a DEFAULT. Element fields may declare REFERENCES /
-- PINNED REFERENCES the same way CREATE COLLECTION would; the
-- in-session FK registration runs on the nested fields. The
-- polymorphic WITH SUBTYPE form is supported — discriminator auto-
-- pins to '_type'.
ALTER COLLECTION [App].[Invoices] ADD FIELD [Adjustments]
    ARRAY OBJECT ([Reason] STRING, [Amount] DECIMAL);
ALTER COLLECTION [App].[Posts] ADD FIELD [Contributors]
    ARRAY OBJECT ([AuthorId] GUID REFERENCES [App].[Authors], [Role] STRING);
ALTER COLLECTION [App].[Posts] ADD FIELD [Items]
    ARRAY OBJECT ([Tag] STRING)
    WITH SUBTYPE [text] ([Body] STRING)
    WITH SUBTYPE [image] ([Url] STRING, [Width] INT32);

-- Compose a named composite type, or an array of it. REFERENCES declared
-- by the type are enforced once the ALTER publishes; a composite DEFAULT
-- backfill must resolve every such reference.
ALTER COLLECTION [App].[Jobs] ADD FIELD [Next] [App].[Schedule];
ALTER COLLECTION [App].[Jobs] ADD FIELD [History] ARRAY [App].[Schedule];

-- Drop field
ALTER COLLECTION [Store].[Products] DROP FIELD [LegacyId];

-- Drop nested array element field
ALTER COLLECTION [Facility].[Hubs] DROP FIELD [Connectors][].[DeprecatedField];

-- A field of an array of arrays' innermost objects takes one [] per level; a DEFAULT
-- backfills every stored element, at every level, and a rename reaches them all.
ALTER COLLECTION [Facility].[Hubs] ADD FIELD [Bays][][].[Label] STRING DEFAULT 'none';
ALTER COLLECTION [Facility].[Hubs] RENAME FIELD [Bays][][].[Label] TO [Tag];

-- Rename field
ALTER COLLECTION [Store].[Products] RENAME FIELD [Rate] TO [BaseRate];

-- Rename a field inside a polymorphic subtype's extension schema. The SUBTYPE [value]
-- modifier scopes the rename to that subtype (an unscoped RENAME FIELD cannot reach a
-- subtype extension field). Row pages stay intact; affected array indexes rebuild.
ALTER COLLECTION [Fleet].[Vehicles] RENAME FIELD SUBTYPE [Car] [Doors] TO [DoorCount];

-- Drop a field, or a field nested in one of its objects, from one subtype's extension.
-- That subtype's stored versions lose the value; other subtypes keep their own field
-- of the same name, and an array index on the path keeps serving them.
ALTER COLLECTION [Fleet].[Vehicles] DROP FIELD SUBTYPE [Car] [Doors];
ALTER COLLECTION [Fleet].[Vehicles] DROP FIELD SUBTYPE [Car] [Engine].[Fuel];

-- Move a field among the fields of its own level: first, last, or before or after a
-- sibling named by its bare name. Every retained version, history included, is
-- rewritten in the new order.
ALTER COLLECTION [Store].[Products] MOVE FIELD [Sku] FIRST;
ALTER COLLECTION [Store].[Products] MOVE FIELD [Notes] AFTER [Name];
ALTER COLLECTION [Store].[Products] MOVE FIELD [Details].[Color] BEFORE [Size];
ALTER COLLECTION [Facility].[Hubs] MOVE FIELD [Connectors][].[Power] LAST;
ALTER COLLECTION [Fleet].[Vehicles] MOVE FIELD SUBTYPE [Car] [Doors] FIRST;

-- Pin or clear a column-level default on an existing field. Schema-only —
-- existing rows are not rewritten; the default fills the field where an
-- inserted document, or an object a later write assigns, leaves it out,
-- and where a write assigns it DEFAULT (not the top-level fields a REPLACE
-- leaves out). Explicit values (including NULL) override the default.
ALTER COLLECTION [Store].[Products] ALTER FIELD [Currency] SET DEFAULT 'USD';
ALTER COLLECTION [Store].[Products] ALTER FIELD [Qty] SET DEFAULT 0;
ALTER COLLECTION [Store].[Products] ALTER FIELD [CreatedAt] SET DEFAULT NOW;
ALTER COLLECTION [Store].[Products] ALTER FIELD [Currency] DROP DEFAULT;

-- SET NOT NULL validates every retained row version before publication.
-- DROP NOT NULL relaxes the constraint without rewriting values.
ALTER COLLECTION [Store].[Products] ALTER FIELD [Sku] SET NOT NULL;
ALTER COLLECTION [Store].[Products] ALTER FIELD [Sku] DROP NOT NULL;

-- Give the primary key a generator or take it away. Schema-only; an added
-- AUTOINCREMENT starts after the highest key any retained version holds.
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Id] SET AUTOINCREMENT;
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Id] DROP AUTOINCREMENT;
ALTER COLLECTION [Store].[Events] ALTER FIELD [Id] SET SEQUENTIAL;
ALTER COLLECTION [Store].[Events] ALTER FIELD [Id] DROP SEQUENTIAL;

-- Give an existing field a reference, replace its target and policy, or take it
-- away. Schema-only; every retained version must resolve in the target first.
ALTER COLLECTION [Store].[Products] ALTER FIELD [SupplierId] SET REFERENCES [Store].[Suppliers];
ALTER COLLECTION [Store].[Products] ALTER FIELD [SupplierId] SET REFERENCES [Store].[Suppliers] ON REMOVE SetNull;
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Lines][].[ProductId] SET REFERENCES [Store].[Products] MATCH FULL;
ALTER COLLECTION [App].[Invoices] ALTER FIELD [TariffRef] SET PINNED REFERENCES [App].[Tariffs];  -- a field stored pinned
ALTER COLLECTION [Store].[Products] ALTER FIELD [SupplierId] DROP REFERENCES;

-- SET TYPE changes a field's type and converts the value of every retained
-- version as CAST would; a value without a conversion refuses the statement.
ALTER COLLECTION [Store].[Products] ALTER FIELD [Qty] SET TYPE INT64;
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Status] SET TYPE [Store].[OrderStatus];  -- by member label or value
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Lines][].[Qty] SET TYPE DECIMAL;         -- every element
ALTER COLLECTION [Store].[Products] ALTER FIELD [Codes] SET TYPE ARRAY INT32;          -- element by element
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Ship] SET TYPE [Store].[Address];       -- field by field, by name
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Lines] SET TYPE ARRAY [Store].[Line];   -- every element, by name
ALTER COLLECTION [Store].[Products] ALTER FIELD [Embedding] SET TYPE VECTOR(768);      -- only while every version holds NULL
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Ship] SET TYPE OBJECT;                 -- detach the field from its named type
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Stops] SET TYPE ARRAY OBJECT;          -- detach an array's elements from their named type

-- The SUBTYPE [value] modifier scopes a DROP FIELD / RENAME FIELD / ALTER FIELD change
-- to a polymorphic subtype's extension schema (drop, rename, default, nullability,
-- computed expression, reference, or an object field's CHECK).
ALTER COLLECTION [Fleet].[Vehicles] ALTER FIELD SUBTYPE [Car] [Doors] SET NOT NULL;

-- A field path steps into a polymorphic field's subtype with SUBTYPE [value]; the
-- change reaches that subtype's fields and the values of that subtype only.
ALTER COLLECTION [App].[Items] ADD FIELD [Component].SUBTYPE [solar].[Panels] INT32 DEFAULT 1;
ALTER COLLECTION [App].[Items] RENAME FIELD [Component].SUBTYPE [solar].[Efficiency] TO [Yield];
ALTER COLLECTION [App].[Plants] ALTER FIELD [Components][].SUBTYPE [battery].[Capacity] SET TYPE INT64;
ALTER COLLECTION [Fleet].[Vehicles] ALTER FIELD SUBTYPE [Car] [Engine].SUBTYPE [diesel].[Cc] SET NOT NULL;

-- Add a subtype to a polymorphic field, or fields to a subtype it has, whose stored
-- values take their defaults; drop a subtype no retained version holds a value of.
ALTER COLLECTION [App].[Items] ALTER FIELD [Component] ADD SUBTYPE [hydro] ([Head] DECIMAL);
ALTER COLLECTION [App].[Items] ALTER FIELD [Component] ADD SUBTYPE [wind] ([Hub] INT32 DEFAULT 80);
ALTER COLLECTION [App].[Plants] ALTER FIELD [Components] DROP SUBTYPE [motor];

-- Add a new subtype. Existing rows retain their subtype;
-- INSERTs with the new SUBTYPE [value] (...) succeed after commit.
ALTER COLLECTION [Fleet].[Vehicles] ADD SUBTYPE [bike] ([Gears] INT32, [Electric] BOOL NOT NULL);

-- ADD SUBTYPE is additive: naming a subtype that already exists merges the
-- new fields onto it. Ordinary new fields read NULL in existing rows;
-- computed values follow their declared expression and storage mode.
ALTER COLLECTION [Fleet].[Vehicles] ADD SUBTYPE [bike] ([FrameSize] STRING);

-- A collection without subtypes becomes polymorphic by naming the subtype its
-- documents take: every retained version, history included, takes [item] with its
-- fields' defaults. Later ADD SUBTYPE statements add further subtypes.
ALTER COLLECTION [Store].[Products] ADD SUBTYPE [item] ([Weight] DECIMAL DEFAULT 0) FOR EXISTING DOCUMENTS;
ALTER COLLECTION [Store].[Products] ADD SUBTYPE [bundle] ([Parts] INT32);

-- Rename a subtype: it keeps its fields, and every retained version, history
-- included, stores the new name.
ALTER COLLECTION [Fleet].[Vehicles] RENAME SUBTYPE [bike] TO [bicycle];

-- Drop subtype (atomically deletes all documents of that subtype, FK-enforced)
ALTER COLLECTION [Fleet].[Vehicles] DROP SUBTYPE [truck];

-- Change the types the collection extends. The fields the new bases give come
-- first, in order; a field they still give keeps its values, one they no longer
-- give is dropped with its values, and one they newly give takes its default.
ALTER COLLECTION [Store].[Products] SET EXTENDS [Store].[Priced], [Store].[Audited];
ALTER COLLECTION [Store].[Products] DROP EXTENDS;

-- Rename collection
ALTER COLLECTION [Old].[Name] RENAME TO [New].[Name];
```

**`RENAME TO`** gives the collection a complete destination name; another namespace moves it there. The collection keeps its ID, documents, indexes, CHECKs and every view, materialized view, trigger and reference that reads it, all bound to its identity. Names compare without regard to letter case: a name that differs only in case respells the collection, and the catalog, the migration's later statements and other sessions after `COMMIT` read the new spelling, while the spelling it already has is the same name (`INVALID_STATEMENT`), as is a name that differs from it only in the namespace's letter case, since the namespace keeps its own spelling, which `ALTER NAMESPACE` changes. A name another collection, view or materialized view holds is `DUPLICATE_NAME`. The migration's later statements name the collection by its new name, a `REFRESH`, `ALTER MATERIALIZED VIEW` or `DROP MATERIALIZED VIEW` of the materialized view whose backing it is among them, and the name it leaves, like the name of a relation the migration drops, is free for another relation in the same migration: each statement acts on the relation the name holds at that point, and every definition keeps reading the relation it is bound to.

Numeric defaults use MQL's invariant spelling during catalog persistence and row backfill. In particular, the ASCII `-` in `DEFAULT -1` has the same meaning regardless of the server process's current culture.

`DROP SUBTYPE` publishes the reduced subtype schema only after every matching document delete succeeds in the same transaction. A row already removed by an earlier FK cascade in that transaction is skipped; any unrelated delete failure aborts the statement and preserves both the subtype schema and its rows. Subtype membership and the last-subtype restriction are checked against the schema protected by the operation's admission, including changes committed while it waited. Concurrently added subtypes and their rows are preserved. A live failure or cancellation while acquiring schema admission also settles the statement's standalone transaction, leaving no active transaction behind.

Nested `ADD FIELD` and `DROP FIELD` rewrite every existing `OBJECT` and `ARRAY OBJECT` value recursively. Unchanged sibling values and element order are preserved; an added leaf's `DEFAULT` is materialized in every existing object/array element, and a dropped leaf's bytes are removed from every element. If the dropped leaf declared `REFERENCES`, its forward and reverse FK registrations are removed in the same migration publication, so later parent deletes cannot enforce a constraint that no longer exists.

**Field DDL keeps partial-index filters in lockstep.** `RENAME FIELD` rewrites the persisted `WHERE` filter of every partial index that references the renamed field (key-field paths already followed the rename). `DROP FIELD` is rejected while a partial index's filter references the field — drop that index first, or recreate it with a filter that doesn't use the field. Indexes whose keys or INCLUDE paths name the dropped field or its descendants are dropped automatically. This includes UNIQUE indexes; remove the affected INCLUDE entries first with ALTER INDEX to retain their key constraint. See [indexes.md → Partial filters and field DDL](indexes.md).

Renaming a field to an existing sibling name is rejected before changing dependencies. The destination field keeps its indexes, INCLUDE paths, and partial-index filters after the migration aborts. Names compare without regard to letter case, so a sibling is any other field of the same object (or of the same subtype for `RENAME FIELD SUBTYPE`, or of the same type for `ALTER TYPE`) whose name differs only in case, and the field's own name never counts against it: `RENAME FIELD [Note] TO [NOTE]` respells the field, and its catalog entries, index key paths, partial-index filters and foreign-key path take the new spelling, which survives reopen. Naming the spelling the field already has changes nothing. The same rule applies to an enum member's label in `ALTER ENUM … RENAME VALUE`.

**`RENAME` / `DROP FIELD` is rejected while the collection's own CHECK constraint or computed column references the field** (`DEPENDENT_OBJECTS_EXIST`), wherever the declaration sits: a CHECK of every document, one of an object field, an array's object elements, an object nested in either, a subtype's field or a polymorphic field's subtype, each reading the fields of its own object. The schema rewrite does not rewrite CHECK / computed *expression text*, so a silent rename would leave the predicate resolving a now-missing field to `NULL` — which under ANSI three-valued logic "passes", disabling the constraint for every future row. Drop the dependent CHECK constraint (`DROP CHECK`, see [Add and drop CHECK constraints](#add-and-drop-check-constraints)) or computed column first, then rename or drop the field (and re-add the constraint against the new name). Renaming or dropping the computed column *itself* is allowed, and so is renaming or dropping the object that declares a CHECK, which goes with it, and a rename that changes only the letter case of the field's name: references resolve without regard to case, so they still name the field. The same refusal protects a field the [CreatedBy display](maintenance.md#createdby-display) reads when the collection is the policy's actor collection, until the migration restates the policy without `DISPLAY`. If a persisted CHECK or computed expression cannot be parsed, the ALTER also fails closed (`CHECK_EXPRESSION_INVALID` or `COMPUTED_EXPRESSION_INVALID`). Dependency analysis cannot safely treat unreadable metadata as though it referenced nothing. Every MQL executor also validates all persisted CHECK and computed expressions during construction. Invalid catalog metadata therefore fails before any statement executes or any change event can expose a degraded computed value.

**`SET DEFAULT` accepts only literal values and `NOW`** — parameter references (`@p`) are rejected at DDL time because parameter values are per-call and have no meaning in the later writes that take the default. `PIN(id, revision)` literals are also rejected. Programmatic value ASTs are structurally validated before canonical default printing; malformed or over-deep nested `PIN` values fail before recursion or persistence. `DEFAULT NULL` is rejected with the message that `DROP DEFAULT` is the canonical way to clear an existing default. `SET DEFAULT` cannot target a primary key, discriminator, computed/virtual field, or pinned foreign key. Those fields have authoritative identity/subtype/derived values or require an explicit `PIN(id, revision)`. Object and array defaults use their ordinary MQL literal syntax and are validated against the full nested schema. An enum default must name a member of the referenced enum. A VECTOR default uses `VECTOR[...]` with exactly the declared dimension and finite FLOAT32 components.

Default application — for an INSERT's document and for every object or list of objects a write assigns ([defaults](dml.md#defaults)) — recursively visits the base schema, the selected collection/field-level subtype extensions, nested `OBJECT` values, and each present `ARRAY OBJECT` element. It fills an omitted leaf only: explicit `NULL` remains explicit, and an absent/null parent object, absent array, or inactive subtype is never synthesized merely to host a child default. A default declared on the parent itself may supply that object or array. The same recursion is used after reopen, so persisted defaults do not depend on whether the field is top-level.

**`SET NOT NULL` / `DROP NOT NULL`** change a field's nullability. SET NOT NULL checks every retained row, including expired temporal versions, under the schema publication barrier. Named-type changes validate all consuming collections. Any violating row rejects the change before publication. A nullable absent parent does not require its children; present objects, array elements, and active subtype extensions must satisfy their required fields. Defaults, generated values, and merged updates are validated as complete candidate rows. `DROP NOT NULL` is always safe. The primary key cannot be made nullable. `SET NOT NULL` is rejected while a foreign key uses `ON REMOVE SetNull`, because that action must be able to store NULL in the source slot. The `SUBTYPE [value]` modifier scopes `RENAME FIELD` / `ALTER FIELD` (rename, default, nullability) to a polymorphic subtype's extension schema.

<a id="change-a-field-type"></a>

**`SET TYPE`** changes a field's type: a field of the document, of an object in it, of one subtype's extension (`ALTER FIELD SUBTYPE [value]`), or of every element of an `ARRAY OBJECT` (`[Lines][].[Qty]`). The value every retained version holds, history included, is converted as `CAST(value AS type)` converts it, so `'5'` becomes `5`, `5` becomes `'5'`, a `DECIMAL` becomes an integer truncated toward zero when the result is in the target's range, and `'2024-05-01T10:00:00Z'` becomes a `DATETIME`. Into an enum, a text value that names a member's label takes that member's value and any other value must CAST to one of its members' values; out of an enum, the stored member value is converted like a value of the enum's underlying type. The first value without a conversion refuses the statement with `INVALID_VALUE` naming it, and the collection stays as it was. Indexes on the field are rebuilt on the converted values, and a unique index whose converted keys collide refuses the change with `UNIQUE_VIOLATION`; on a temporal collection keys collide only between different documents whose versions overlap in valid time, so one document's history may convert to a shared key ([UNIQUE on a temporal collection](indexes.md#unique-on-a-temporal-collection)). Making a field an enum of its own underlying type, or that enum's field a plain one again, keeps the stored values and rewrites nothing; the values are still checked to be members.

An `ARRAY` converts element by element by the same rules, so an `ARRAY STRING` holding `['1', '2']` becomes an `ARRAY INT32` holding `[1, 2]`, and an `ARRAY` of an enum takes each element's label or value; an element without a conversion refuses the statement, since an array holds no NULL element. An object field becomes a named composite type, and an `ARRAY OBJECT` an `ARRAY` of one, field by field by name: each field the type declares takes the stored field of the same name, converted by these rules, and a field the stored object lacks takes the type's default, or NULL. A stored field the type does not declare must be NULL in every version, so no value is dropped: the first that holds one refuses the statement with `INVALID_VALUE` naming it. The type's `NOT NULL` fields (`DOCUMENT_PAYLOAD_INVALID`), CHECKs (`CHECK_VIOLATION`) and references (`FK_VIOLATION`) are proven against every converted version; the field's own CHECKs leave with its old shape, and the field belongs to the type afterwards, so `ALTER TYPE` reaches it. A polymorphic object, or a stored computed field the type declares and the stored object lacks, converts into no named type (`FEATURE_NOT_SUPPORTED`). A `VECTOR(n)` becomes a `VECTOR(m)` only while every version holds NULL, since a vector is neither truncated nor padded: the first present vector refuses the statement with `INVALID_VALUE`. The same holds for a change between a single value, an object, an array and a vector, such as `STRING` to `ARRAY STRING` or an object to a `STRING`.

`SET TYPE OBJECT` detaches a field of a named type from its type, and `SET TYPE ARRAY OBJECT` the elements of an `ARRAY` of a named type from theirs: the field keeps its fields, CHECKs, references and values, no version is rewritten, `ALTER TYPE` no longer reaches it, and its own fields (`[Ship].[Zip]`, `[Stops][].[Zip]`) then change with `ALTER COLLECTION`. On an inline object, or an `ARRAY OBJECT`, it changes nothing, and on any other field it is `FEATURE_NOT_SUPPORTED`.

The target is a scalar type, an enum, a named type, `VECTOR(n)`, an `ARRAY` of a scalar, an enum or a named type, `OBJECT` or `ARRAY OBJECT`; an inline object shape, or an `ARRAY` of arrays or vectors, is a `SYNTAX_ERROR` (an inline object's fields change one by one). A field whose type something else owns is refused: the primary key and auto-increment or sequential fields (`FEATURE_NOT_SUPPORTED`), a computed field, whose type is its expression's, and a reference or a field stored pinned, or an array of them, which holds the referenced key (`INVALID_DEFINITION`), the discriminator, a field the collection inherits from a named type and a field inside a field of a named type, which `ALTER TYPE ... ALTER FIELD ... SET TYPE` changes (`FEATURE_NOT_SUPPORTED`). A CHECK or computed field of the collection that reads the field, the [CreatedBy display](maintenance.md#createdby-display) when it reads the field of its actor collection, a partial index that filters on it, an expiry index that keys on it, an index on a path inside it that the new type does not declare or turns into an array, an object or a vector, and a [vector index](vectors.md#optional-vamana-index) that reads it refuse the change (`DEPENDENT_OBJECTS_EXIST`), and a default that is not a value of the new type refuses it (`INVALID_DEFINITION`): drop or change them first, in the same migration (a vector index before it).

<a id="move-a-field"></a>

**`MOVE FIELD`** moves one field among the fields of its own level: the document's top level, one subtype's own fields (`MOVE FIELD SUBTYPE [value]`), the fields of an object in it, or those of every element of an `ARRAY OBJECT` (`[Lines][].[Qty]`). `FIRST` and `LAST` put it at an end of the level, and `BEFORE [Sibling]` and `AFTER [Sibling]` beside another field of the level, named by its bare name. The order is the one `SELECT *`, `DESCRIBE`, `[Mnemic].[Collections]` and result schemas list and the one set operations match columns by; fields are still found by name, so the primary key, indexes, foreign keys, CHECKs and computed fields keep working. Every retained version, history included, is rewritten in the new order in the migration, and an aborted migration leaves the order and the rows as they were. A move to the place the field already has changes nothing and rewrites nothing.

The discriminator and the fields the collection inherits through `EXTENDS` stay first: moving one of them, or another field before them, is `FEATURE_NOT_SUPPORTED`, and so is moving a field inside a field of a named type, as every action reaching inside one is ([named types](composite-types.md#composition--a-field-whose-type-is-a-composite-type)); move such a field on its type with [`ALTER TYPE … MOVE FIELD`](composite-types.md#move-a-field). An unknown field or sibling is `UNKNOWN_FIELD`, a subtype the collection does not have `UNKNOWN_OBJECT`, and a field placed before or after itself `INVALID_DEFINITION`; a sibling written as a path is a `SYNTAX_ERROR`. A view, trigger or materialized view the new order breaks, such as a `UNION` of `SELECT *` over collections whose fields no longer line up, refuses the move. A remote typed client whose registration declares the fields in another order fails closed until it declares the new one ([mapping classes](../Clients/DotNet/collections.md#new-and-existing-collections)).

**`SET EXTENDS`** replaces the types the collection extends with the listed ones, in order, and **`DROP EXTENDS`** leaves it extending none, exactly as [`ALTER TYPE … SET EXTENDS`](composite-types.md#change-the-types-a-type-extends) changes a collection that extends the altered type. The collection's fields become its discriminator, the fields the new bases give in their resolved order, then its own fields, and its CHECKs its own followed by the new bases'. Inherited fields are matched by name. A field the new bases still give keeps its values; when they declare it with another type, its values convert as [`SET TYPE`](#change-a-field-type) converts them, refused as `SET TYPE` refuses them, and a field the old or the new declaration computes, or that only its fields declare, has no such conversion and is `FEATURE_NOT_SUPPORTED`. A field they no longer give is dropped from every retained version, history included, under the rules of `DROP FIELD`: the collection's own CHECK or computed field and a partial index filter that reads it refuse the change with `DEPENDENT_OBJECTS_EXIST`, and a view or trigger that reads it with the error naming it (`UNKNOWN_FIELD`). A field they newly give is added with its declared default filled into the documents that exist; a required field without a default is `DOCUMENT_PAYLOAD_INVALID` while documents lack it. Every retained version must meet the new bases' CHECKs (`CHECK_VIOLATION`) and resolve their references (`FK_VIOLATION`), which are enforced from the statement on. The collection then extends the new bases: `ALTER TYPE` on one of them reaches it, `DROP TYPE` refuses while it extends one, and its inherited fields change only through `ALTER TYPE`. The bases it already extends, in the same order, change nothing.

A base that does not exist is `UNKNOWN_OBJECT`, a base listed twice `INVALID_STATEMENT`, and a field or CHECK name the new bases share with the collection's own fields, a subtype's own fields or its own CHECKs, or two unrelated bases give, is `DUPLICATE_NAME`, as at `CREATE COLLECTION`; more than 1,024 fields or 256 CHECKs is `LIMIT_EXCEEDED`. The new bases publish with the schema they give, survive reopen and roll back with an aborted migration, together with the fields and values.

**`ADD SUBTYPE`** is the symmetric pair of `DROP SUBTYPE` and is **additive**: when the discriminator value is new it registers the variant; when the value already exists, the supplied fields are merged onto that subtype's extension schema (this is how you add a field to an existing collection subtype — `ADD FIELD` takes no collection `SUBTYPE` scope; a polymorphic field's subtype takes one through its [path](#field-subtypes)). A supplied field whose name already exists in the subtype is rejected (`ADD SUBTYPE` adds fields, it never redefines one). Either way it is a catalog-only mutation — existing rows are not rewritten; they keep their subtype tag and read any newly-merged field as null until rewritten. The supplied fields go through the same validators `CREATE COLLECTION WITH SUBTYPE` uses (reserved-name check, duplicate-field-name, computed-column mutual-exclusion rules), and any foreign keys flow through the same FK registration: the pinned target must be temporal with a PK shape of Int8 / Int16 / Int32 / Int64 / Guid / String / Bytes, `ON REMOVE Cascade` is rejected on pinned, `SetNull` is valid only on a nullable scalar field, and the field type must match the target's PK type. Like `ADD FIELD … REFERENCES`, the FK target must **already exist** at ALTER time (no forward reference — `ALTER` runs against the live catalog). On a collection without subtypes a plain `ADD SUBTYPE` is `WRONG_OBJECT_TYPE`, since its documents have no subtype to keep; `FOR EXISTING DOCUMENTS` makes it polymorphic. Inside a `BEGIN MIGRATION` block the subtype-list change rides the migration transaction (restored on `ABORT MIGRATION`, WAL-enrolled at `COMMIT MIGRATION`).

<a id="field-subtypes"></a>

**A path into a polymorphic field's subtype.** A polymorphic `OBJECT` field, or the elements of a polymorphic `ARRAY OBJECT`, holds each value's own subtype ([field-level polymorphism](polymorphism.md#field-level-polymorphism-polymorphic-sub-documents)). A DDL field path steps into the fields one subtype adds with a `SUBTYPE [value]` segment after the field: `[Component].SUBTYPE [solar].[Efficiency]`, `[Components][].SUBTYPE [battery].[Capacity]`, deeper as the fields nest (`[A].SUBTYPE [x].[B][].SUBTYPE [y].[C]`), and inside a collection subtype's scope (`ALTER FIELD SUBTYPE [Car] [Engine].SUBTYPE [diesel].[Cc]`). `ADD FIELD`, `DROP FIELD`, `RENAME FIELD`, `MOVE FIELD` and every `ALTER FIELD` form take such a path, in `ALTER COLLECTION` and in `ALTER TYPE` for a type's polymorphic field; `SET PRIMARY KEY` and the key generators do not, since the key is a top-level field. The change reaches that subtype's fields and the values of that subtype only, under the rules of the same action on any other field: `ADD FIELD` gives every stored value of the subtype the field's default, or NULL, `DROP FIELD` removes it from those values, and both rewrite every retained version, history included; `RENAME FIELD` changes only the name and leaves the stored bodies as they are, which still read after reopen, while an array index on the path keeps it as long as another subtype declares it, rebuilt from the values that still hold it, and follows the rename of its last declaration, rebuilt from every declaration of the new path; `MOVE FIELD` orders the subtype's fields among themselves, its `BEFORE` or `AFTER` sibling one of them; `SET TYPE` converts that subtype's values only, so another subtype may keep a field of the same name and another type. The default, nullability, computed mode, reference and an object's CHECK change as on any field. A name stays distinct from the subtype's other fields and from the fields every value of the polymorphic field has (`DUPLICATE_NAME`); another subtype of the same field may share it. A subtype the field does not have is `UNKNOWN_OBJECT`, a `SUBTYPE` segment after a field without subtypes `WRONG_OBJECT_TYPE`, and after an array without its element marker (`[Components].SUBTYPE …`) `TYPE_MISMATCH`, as any step into an array's elements without it is. A subtype named in a path holds no `[` or `]` (`SYNTAX_ERROR`); the segment never ends a path. A path through a field the collection inherits or a field composed of a named type is refused as any other path into one is (`FEATURE_NOT_SUPPORTED`), and a B-tree index, or a partial index filter, on a subtype's field of a polymorphic field stays refused, while an [array index](indexes.md#array-element-gin-indexes) orders by one inside a list or by a list one declares.

**`ALTER FIELD [path] ADD SUBTYPE [value] (field_definitions)`** gives a polymorphic field another subtype, which no stored value has yet, so no version is rewritten; naming a subtype the field has adds the fields to it, as the collection's own `ADD SUBTYPE` merges, and every retained version is rewritten, each stored value of the subtype taking each added field's declared default, or NULL. A field the subtype, or every value of the field, already has is `DUPLICATE_NAME`, and a reference among the fields must resolve for every version (`FK_VIOLATION`). **`ALTER FIELD [path] DROP SUBTYPE [value]`** removes a subtype from the field. It never removes data: while any retained version, history included, holds a value or element of the subtype, the statement is `DEPENDENT_OBJECTS_EXIST`, since those values depend on the subtype as a field depends on the enum it stores; change them to another subtype first, with an `UPDATE` earlier in the same migration, and on a temporal collection, whose history keeps them, the subtype stays. The proof reads every retained version under the migration's schema barrier and no version is rewritten. The last subtype of a field is `INVALID_DEFINITION`: every value has one, so drop the field instead, and so is a subtype that alone declares a field a CHECK or computed field reads, which would read nothing once it goes, or that a CHECK's subtype test names, which would name no subtype. Both take the field's path, the polymorphic `ARRAY OBJECT` field for its elements; a field without subtypes is `WRONG_OBJECT_TYPE`, a subtype the field does not have `UNKNOWN_OBJECT`, and `FOR EXISTING DOCUMENTS` belongs to a collection's `ADD SUBTYPE` only (`SYNTAX_ERROR`). An aborted migration restores the field's subtypes and every value.

<a id="make-a-collection-polymorphic"></a>

**`ADD SUBTYPE [value] (...) FOR EXISTING DOCUMENTS`** makes a collection without subtypes polymorphic: the discriminator `_type`, a required `STRING`, becomes its first field, before any field it inherits through `EXTENDS`, and `[value]` its one subtype, which every document takes. Nothing gives documents a subtype by convention: the clause names it, and a plain `ADD SUBTYPE` on such a collection is refused. Every retained version, history included, is rewritten into the subtype's envelope in the migration, its subtype fields holding what an `INSERT` naming none of them stores: each field's declared default, NULL elsewhere. A required subtype field without a default is `DOCUMENT_PAYLOAD_INVALID` while the collection holds any version; on an empty collection it simply becomes polymorphic. The fields may be an empty list `()`. A subtype field whose name a base field has, or a base field already named `_type`, is `DUPLICATE_NAME`, and a reference among the subtype's fields must resolve for every version (`FK_VIOLATION`). Every index is rebuilt and the collection's statistics are dropped. From the statement on, `INSERT` names a subtype, `SELECT *` carries the `_type` column and the envelope ([polymorphism](polymorphism.md#querying-by-subtype)), and further `ADD SUBTYPE` statements add subtypes; views, materialized views and triggers over the collection are validated against it as after any `ALTER COLLECTION`. An aborted migration leaves the collection without subtypes, with its rows and indexes. On a collection that is polymorphic already the clause is `WRONG_OBJECT_TYPE`: its documents have their subtypes.

<a id="rename-a-subtype"></a>

**`RENAME SUBTYPE [old] TO [new]`** gives a subtype another name. The subtype keeps its fields, and every retained version of it, history included, is rewritten in the migration to store the new name as its discriminator, so `SUBTYPE IS [new]`, `SELECT SUBTYPE`, `AS OF` reads and an index on `[_type]` find the documents by it, and the old name is unknown from the statement on (`UNKNOWN_OBJECT`, as any unknown subtype is); the migration's later statements, such as `ADD SUBTYPE [new] (…)` or `RENAME FIELD SUBTYPE [new] …`, name the subtype by its new name. Every index is rebuilt and the collection's statistics are dropped. Names compare without regard to letter case: naming the name the subtype already has changes nothing, and a change of letter case alone respells the subtype and the name its documents store. A collection without subtypes is `WRONG_OBJECT_TYPE`, a subtype it does not have `UNKNOWN_OBJECT`, and the name of another of its subtypes `DUPLICATE_NAME`. The rename does not rewrite a stored definition's text, so while one names the subtype — the collection's CHECK or computed field (`SUBTYPE IS [old]`), a partial index's filter, the [CreatedBy display](maintenance.md#createdby-display) of the collection as actor collection, or a view, materialized view or trigger over the collection that tests the subtype, compares `[_type]` with its name or inserts `SUBTYPE [old]`, and a trigger on the collection itself whose statements or `WHEN` condition compare `NEW.[_type]` or `OLD.[_type]` with it — it is `DEPENDENT_OBJECTS_EXIST`; drop or replace the definition first. An aborted migration restores the old name in the catalog and in every version.

Foreign keys belong to the fields that declare them. Sibling subtypes may reuse a logical path with different targets, key types, array dimensions, removal actions, or pin modes; another subtype may leave the path unconstrained. Validation, reverse lookup, sealing, and removal use the active subtype's field declaration. Identical policies at the same path coalesce, including `MATCH FULL`: its all-null-or-all-present rule counts only occurrences declaring that same policy. Renames, subtype additions/drops, migration rollback, and reopen preserve each surviving declaration.

**`ADD FIELD` with a foreign key** honors the same invariants as `CREATE COLLECTION`: a pinned target must be temporal with a PK shape of Int8 / Int16 / Int32 / Int64 / Guid / String / Bytes; `ON REMOVE Cascade` is rejected on pinned; `SetNull` is valid only on a nullable (no-`DEFAULT`) scalar field and is rejected on a non-pinned array FK; the field type must match the target's PK type. Two differences from `CREATE`: (1) the target must **already exist** (no forward reference — `ALTER` runs against the live catalog), and (2) a `DEFAULT` backfills every existing row and is validated to resolve against the target (a non-pinned default to a missing id, or a `PIN(id, rev)` default to a missing revision, is rejected at `ALTER`). A non-pinned FK's `DEFAULT` is a plain id literal. A pinned FK may use `DEFAULT PIN(id, revision)` only as this ALTER's validated one-time backfill; the pinned field retains no catalog default and each later write that stores it must supply an explicit PIN. References that enter through a named composite type (`ADD FIELD [F] [Ns].[Type]`, `ARRAY [Ns].[Type]`, or a type field in `ADD SUBTYPE`) are enforced from the same ALTER, and a composite DEFAULT backfill is validated against every target. `ADD FIELD` supports scalar arrays (including array FKs), `ARRAY OBJECT`, and recursive arrays such as `ARRAY ARRAY INT32` or `ARRAY ARRAY OBJECT (...)`. A supplied array or object DEFAULT backfills existing rows; without one, a newly added field reads NULL. Programmatic AST execution admits the same scalar, vector, object and array literals and `NOW`, plus one-time `PIN(id, revision)` backfills for pinned slots. Field references, subqueries, parameters, and other expression nodes are rejected before schema mutation rather than being silently treated as no default.

<a id="change-a-reference"></a>

**`SET REFERENCES` and `DROP REFERENCES`** give an existing field a reference, replace the one it has, or take it away. `ALTER FIELD [F] SET [PINNED] REFERENCES [Ns].[Target] [ON REMOVE Restrict|Cascade|SetNull] [MATCH FULL|SIMPLE]` takes the modifiers `ADD FIELD` declares; `ALTER FIELD [F] DROP REFERENCES` takes none. The field may be a top-level field, a field of an object (`[Ship].[DepotId]`) or of every element of an `ARRAY OBJECT` (`[Lines][].[ProductId]`), a field of one subtype's extension (`ALTER FIELD SUBTYPE [value] [F]`), or an array of keys (`[TagIds]`), whose element type is the key type. No document is rewritten. On a field that has a reference, `SET REFERENCES` replaces its target, removal action and match mode together; this is how `ON REMOVE` and `MATCH FULL` change, and an omitted `ON REMOVE` is the implicit `Restrict`. The statement the field already answers to changes nothing.

The reference is admitted the way a declared one is, with the same codes. The target must exist now (`COLLECTION_NOT_FOUND`); the field's type, or an array's element type, must be the target's key type (`TYPE_MISMATCH`); `SetNull` needs a nullable field that is no plain array of keys, and a pin a temporal target and no `Cascade` (`INVALID_DEFINITION`). A computed field and a discriminator take no reference (`INVALID_DEFINITION`). The primary key takes the one-to-one reference `CREATE COLLECTION` lets it declare, under the same rules: never `PINNED`, since a key is the identifier itself, and never `SetNull`, since a key is required (`INVALID_DEFINITION`); with `Cascade`, removing the target removes the document. Then every retained version, history included, must resolve in the target, `MATCH FULL` must hold for every array value, and a `Cascade` or `SetNull` must not be able to reach a sealed document: the first value that fails refuses the statement with `FK_VIOLATION`, and the migration rolls back to the reference the field had, enforcement included. The migration's own later statements enforce the new reference at once and other sessions once it commits; it survives reopen, and an aborted migration restores the reference the field had.

`PINNED` names how the field stores its values rather than changing it: a field stored pinned holds identifier-and-revision pins, which a reference change neither creates nor strips. `SET PINNED REFERENCES` is therefore valid only on a field stored pinned, which `DROP REFERENCES` leaves pinned, and plain `SET REFERENCES` only on a field that is not; the other combination is `FEATURE_NOT_SUPPORTED`. `DROP REFERENCES` is always allowed, whatever the target's documents rely on, and on a field without a reference it changes nothing. A field a type the collection extends declares, and a field inside a field of a named type, change with [`ALTER TYPE … ALTER FIELD … SET REFERENCES`](composite-types.md#change-a-fields-reference) (`FEATURE_NOT_SUPPORTED` here). An unknown field is `UNKNOWN_FIELD`, an unknown subtype `UNKNOWN_OBJECT`, and a path to an array's elements (`[Tags][]`) an `INVALID_STATEMENT`.

### Add and drop CHECK constraints

```sql
BEGIN MIGRATION 3;
ALTER COLLECTION [Store].[Products] ADD CHECK [PositivePrice] ([Price] > 0);
ALTER COLLECTION [Store].[Products] ADD CHECK (LENGTH([Name]) > 0);
ALTER COLLECTION [Store].[Products] DROP CHECK [LegacyRule];
ALTER COLLECTION [Store].[Products] DROP CHECK (LENGTH([Name]) > 0);
ALTER COLLECTION [Store].[Products] ALTER FIELD [Dimensions] ADD CHECK [Positive] ([Width] > 0 AND [Height] > 0);
ALTER COLLECTION [Store].[Orders] ALTER FIELD [Lines] ADD CHECK ([Qty] > 0);
ALTER COLLECTION [App].[Vehicles] ALTER FIELD SUBTYPE [Car] [Engine] DROP CHECK [Power];
COMMIT MIGRATION;
```

`ADD CHECK [name] (predicate)` adds a collection-level CHECK under the same rules as `WITH CHECK` at creation: the predicate binds against the collection's fields and its subtypes' extension fields, parameters and virtual computed fields are rejected (`INVALID_DEFINITION`), a name already used by another of the collection's CHECKs is `DUPLICATE_NAME`, and a 257th CHECK is `LIMIT_EXCEEDED`. With `ALTER FIELD [Object]` the CHECK belongs to that OBJECT field and binds against its fields; on an `ARRAY OBJECT` field it checks each element, and `SUBTYPE [value]` selects an object field of one subtype's extension. A field that is not an OBJECT or ARRAY OBJECT is `TYPE_MISMATCH`.

Before the migration publishes an added CHECK, every retained version of every document, including temporal history and the migration's own earlier writes, must satisfy it; one stored version that fails it refuses the statement with `CHECK_VIOLATION` and rolls the migration back. Only the added CHECK is evaluated, so existing CHECKs that read `NOW` do not re-judge older versions. From then on the CHECK applies to every write and survives reopen. A collection-level CHECK appears in `[Mnemic].[Constraints]`; a CHECK that belongs to an OBJECT or ARRAY OBJECT field is enforced but not listed there: the `Checks` of the field's entry in `[Mnemic].[Collections]`, or `[Mnemic].[Types]` for a type's, list it.

`DROP CHECK [name]` drops a CHECK by its name, compared without regard to case. `DROP CHECK (predicate)` drops the unnamed CHECKs whose predicate matches: the two match when they print the same expression, so spacing, keyword and function-name case, and redundant parentheses do not matter, while field names and literals must be the same. A named CHECK is dropped only by its name. Naming no CHECK of the collection (or of the object field) is `UNKNOWN_OBJECT`. Dropping a CHECK leaves existing rows as they are and releases the fields it referenced for `RENAME FIELD` and `DROP FIELD`.

A collection's CHECKs inherited through `EXTENDS`, which `[Mnemic].[Constraints]` lists with the type that declares them in `DeclaredBy`, the CHECKs of a field composed of a named type, and the object CHECKs of an inherited field belong to the type: dropping them on the collection, or changing a composed field's CHECKs, is `FEATURE_NOT_SUPPORTED`. Change them with [`ALTER TYPE ... ADD CHECK` / `DROP CHECK`](composite-types.md#alter-type--evolve-a-type-propagating-to-every-user), which updates every extender and composer in the same migration.

A CHECK constrains later writes and leaves stored values as they are, so the collection's statistics survive the change.

### Base fields in polymorphic collections

`ALTER COLLECTION … ADD FIELD` and `DROP FIELD` work on the base schema of polymorphic collections, including nested base-object paths. Rewrites migrate the base portion of every current and historical envelope while preserving its subtype extension. Ordinary defaults and stored computed values are backfilled under the same migration; indexes and rollback share that outcome.

A base field cannot reuse an active subtype extension's name. The discriminator and primary key remain mandatory. Drop or rename is rejected when a subtype's computed expression depends on that base field, just as for base computed fields.

```sql
BEGIN MIGRATION 2;
ALTER COLLECTION [App].[Vehicles] ADD FIELD [Color] STRING DEFAULT 'Blue';
ALTER COLLECTION [App].[Vehicles] DROP FIELD [Description];
COMMIT MIGRATION;
```

Extending an existing collection subtype with `ADD SUBTYPE` recomputes stored values and rebuilds affected indexes when the collection has stored expressions. This includes base consumers of an extension field that previously read NULL. Rows, indexes, and schemas commit or roll back together. `DROP SUBTYPE` and later schema edits must preserve valid computed and CHECK bindings in every remaining subtype: dropping the subtype that alone declares a field a computed field or CHECK reads is `INVALID_DEFINITION`.

## DROP COLLECTION

```sql
DROP COLLECTION [Store].[Archives];
```

Enforces FK constraints — blocks if referenced documents exist with Restrict action. Cascade and SetNull govern row removal; they do not remove the FK declaration. Before committing a dropped target, remove its referencing fields or collections so no schema retains the dropped collection ID; otherwise COMMIT MIGRATION fails with `DEPENDENT_OBJECTS_EXIST`. Dropping or renaming a collection that does not exist reports `COLLECTION_NOT_FOUND`.

## DESCRIBE COLLECTION

```sql
DESCRIBE COLLECTION [Store].[Products];
```

Returns the collection schema, field types, and metadata. Does not require migration context. Equivalent to a one-row SELECT against `[Mnemic].[Collections]` with `[Name]` filtered to the named collection; see [System virtual collections](system-collections.md).

## Computed columns

Add `STORED` after an expression to persist its value on every write, for example `[Total] DECIMAL AS ([Price] * [Qty]) STORED`. Stored columns support ordinary indexes, INCLUDE payloads, partial indexes, and CHECK references. `ALTER COLLECTION … ADD FIELD … AS (…) STORED` backfills existing rows. Stored expressions use immutable built-in functions and ordinary or stored fields; clock functions, custom functions, enum lookup functions, and virtual computed dependencies are rejected. A virtual dependency can be declared STORED.

Virtual computed columns derive their value from an expression evaluated on every read. The expression text is stored in the catalog and re-evaluated against each row at query time; the value is **not** stored on disk. Suitable for cheap derivations that should be expressed once on the schema instead of repeated in every SELECT.

```
[FieldName] TYPE AS (<expression>)
```

```sql
-- Per-row scalar derivation
CREATE COLLECTION [Shop].[Lines] WITH SCHEMA (
  [Id]    INT32 PRIMARY KEY,
  [Price] DECIMAL,
  [Qty]   INT32,
  [Total] DECIMAL AS ([Price] * [Qty])
);

-- String functions in the expression
CREATE COLLECTION [App].[People] WITH SCHEMA (
  [Id]      GUID PRIMARY KEY,
  [Name]    STRING,
  [Display] STRING AS (UPPER([Name]))
);

-- Add a computed column to an existing collection
BEGIN MIGRATION 7;
ALTER COLLECTION [Shop].[Lines] ADD FIELD [Total] DECIMAL AS ([Price] * [Qty]);
COMMIT MIGRATION;

-- Recursive positions bind to their containing object. The subtype
-- extension may read both base fields and fields in that same extension.
CREATE COLLECTION [App].[Bundles] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Price] OBJECT (
    [Net] DECIMAL,
    [Gross] DECIMAL AS ([Net] * 1.25)
  ),
  [Lines] ARRAY OBJECT (
    [Qty] INT32,
    [Each] DECIMAL,
    [LineTotal] DECIMAL AS ([Qty] * [Each])
  ) WITH SUBTYPE [Discounted] (
    [Discount] DECIMAL,
    [Payable] DECIMAL AS ([Each] - [Discount])
  )
);
```

INSERT, UPDATE, REPLACE, and UPSERT reject any user-supplied value for a computed field, including a nested, array-element, or active-subtype slot. Its value is derived from the expression and synthesized on reads, `RETURNING`, and change-event presentation images. The direct binary engine boundary enforces the same recursive physical rule: all virtual computed slots must be NULL before storage. Stored computed slots are always derived at the engine write boundary, even for raw binary callers. A non-NULL incoming slot is `DOCUMENT_PAYLOAD_INVALID`; observing one in a stored row is `DOCUMENT_PAYLOAD_CORRUPT`, not a value that query evaluation may preserve.

Computed expressions are bound to the exact schema generation used by each query. Dropping and recreating a collection under the same name immediately uses the recreated schema's expressions, even when its schema version matches the old collection; no reconnect or process restart is required. The first plan for every generation repeats semantic binding validation, so an embedded host cannot publish a late programmatic schema whose unknown dependency is silently read as NULL. Recursive evaluation wires both enum lookups: `ENUM_LABEL` by field enum id and `ENUM_VALUE` by enum name.

Computed results may be scalar, `OBJECT`, `ARRAY`, or `VECTOR(n)`. Structured results must preserve the declared shape in every value-producing arm: field order, names, child types/nullability, vector width, named-type and enum identity, reference targets, subtype layouts, and nested computed definitions. Copied references in nested fields and array elements must name the same target: bound collection IDs are authoritative, and unbound names compare case-insensitively. Field references, `CASE`, `COALESCE`/`IFNULL`, `NULLIF`, and vector literals retain those shapes. Incompatible arms reject at definition time. Polymorphic container paths use the active subtype; a missing extension value is NULL, and candidates with incompatible shapes cannot bind to one computed result. Reading a child of a virtual computed container evaluates the parent before filtering or projection.

Computed declarations use the canonical built-in type and shape contracts even when an embedded host replaces a scalar function. Virtual scalar results must still convert to the declared column type. A replacement for COALESCE, IFNULL, or NULLIF cannot promise its original structured argument shape; an affected virtual structured field fails safely without quarantining healthy schema metadata. Restoring the original implementation permits subsequent reads. STORED evaluation continues to use the original immutable implementations.

Current restrictions:

- A computed field may appear in the collection base, a nested `OBJECT`, an `ARRAY OBJECT` element, a field- or collection-level subtype extension, or a named composite type used through composition or `EXTENDS`.
- Collection-level base and active-subtype fields share a dependency graph. A field absent from the active subtype has typed NULL when another subtype owns its declaration; it never reads another row or subtype's data. Unknown fields, incompatible result types, and cycles in any active subtype reject before publication. Stored consumers require stored computed dependencies. Nested objects and array elements retain their containing-object scopes; nested subtype expressions may read their base plus their own extension.
- Computed columns have nullable schema metadata; `NOT NULL` is rejected, including ALTER COLLECTION / ALTER TYPE SET NOT NULL. This is a conservative declaration rule, not a claim that every NULL input produces NULL: COALESCE, IFNULL, and CASE can return non-NULL values.
- Computed columns cannot be `PRIMARY KEY`, `AUTOINCREMENT`, or `REFERENCES` (any form, including `PINNED`).
- A computed value is one value per row, so an array element path cannot be it or an operand of it (`AS ([Tags][])`, `AS (UPPER([Tags][]))`): the definition is rejected with `FEATURE_NOT_SUPPORTED`, as is a `CHECK` operand such as `UPPER([Tags][])`. An element predicate keeps its meaning in both: `[Urgent] BOOL AS ([Tags][] = 'urgent')`, `WITH CHECK ([Tags][] != 'bad' ALL)` ([element paths read no single value](queries.md#array-element-paths-read-no-single-value)).
- Computed columns may depend on other computed columns in their visible scope. Dependencies evaluate before their consumers, regardless of declaration order. Child objects are enriched before parent expressions, and subtype expressions can read computed base fields. Self-references and cycles reject at definition time.
- `WHERE` / `ORDER BY` / `GROUP BY` on a virtual computed field work but require read-time derivation. Use `STORED` with a secondary index when filter selectivity matters. Single-source queries without `LAST PER` retain native stored-field filters before derivation, including primary and secondary index access. AND permits independent stored conjuncts; OR moves only when both complete branches use stored values. Queries that do not touch any computed column keep engine-side pushdown unchanged.
- A write's `WHERE` (UPDATE, REPLACE, DELETE, SEAL, UNSEAL, CANCEL VERSION) that reads a virtual computed field matches the rows the same read finds: its candidate rows are derived before the predicate filters them, and the write then targets their primary keys, so `EXPLAIN` reports `DeferredPredicate`. A write `WHERE` over stored fields alone keeps its native write access.
- Aggregate arguments, aggregate-local `FILTER (WHERE ...)`, and aggregate `ORDER BY` keys also see the injected computed value. `SELECT * EXCLUDE` / `REPLACE` over a computed-column collection applies the modifier normally; the injection fast path never drops projection work or its validation. Qualifying any of these references with the source alias has identical semantics. A JOIN binds each source reference to its original collection occurrence before deciding which outer or inner values need derivation. A unique declared `[Body].[Derived]` source path therefore observes the same computed value as its alias-qualified spelling, including primary and additional equality keys. Completed aggregate aliases in HAVING and final ORDER BY retain their result scope; reusing a computed source field's name does not request evaluation of that unrelated field.
- Correlated `EXISTS` / scalar predicates can sit beside computed-column references. Equality correlations that qualify for decorrelation use the set-based SEMI / ANTI or aggregate-CTE path; other shapes evaluate per outer row after computed values are injected. Filtering still precedes grouping, total counting, projection, and `SKIP` / `LIMIT`. With a JOIN, the residual predicate sees the complete joined row after computed values are injected into its inputs.
- `JOIN` consumers see computed values on either side. A computed-bearing outer is enriched before joining. A physical inner is enriched and materialised in the bounded, spill-capable virtual-collection replay only when ON keys, residual filters, ordering, grouping, aggregation, or projection can observe one of its computed slots. The replay retains the statement snapshot, cancellation, and resource ownership. An inner whose computed slots are not referenced keeps the ordinary physical scan and index path. An explicit CTE remains equivalent when an application wants to name the materialisation boundary itself.
- Scalar/window projection dependency discovery walks complete window arguments, partitions, ordering keys, and FILTER predicates. A computed field referenced only inside `OVER (...)` is still injected before windowing.
- Field-to-field JOIN residuals inspect both operands. A computed outer field on the right-hand side triggers injection just as it does on the left.
- `CHECK` constraints cannot reference virtual computed columns (they evaluate at write time, before injection); inline the source expression instead.
- Replace an existing expression with `ALTER FIELD [path] SET COMPUTED (<expr>)`, which keeps the declared type and the virtual/STORED mode unless `STORED` follows the expression. The same statement makes a plain field computed, `SET STORED` and `DROP STORED` switch the mode, and `DROP COMPUTED` makes a computed field plain, keeping its values; see [Change a computed expression or mode](#change-a-computed-mode).

Persisted expression text is parsed and schema-bound at DDL time and again when an executor opens the catalog. An invalid persisted expression fails startup or the read that encounters it; it is never skipped and returned as a permanent NULL. Enrichment also validates each computed-bearing binary scope: malformed document/array/envelope layout, a missing plan binding, or a runtime evaluation fault aborts the read, RETURNING result, or event presentation instead of fabricating NULL. Ordinary expression NULL propagation still yields NULL, but a numeric value that cannot fit the declared width fails the read with `INVALID_VALUE` (`Computed result cannot be represented as Int32.`), so one such row fails every read that computes the field, `SELECT *` included; wrap the expression in `TRY(...)` to read NULL instead. Only ordinary syntax/semantic parse failures are converted into the documented validation or persisted-metadata codes. A direct or wrapped structural detector keeps its original code, and direct or wrapped process-memory exhaustion reaches the executor's terminal failure policy instead of being relabelled as invalid CHECK/default/computed metadata.

DDL-time validation includes a numeric narrowing check on the declared output type:

- A field reference whose declared numeric type is strictly wider than the declared output type (same family) is rejected with the offending source field name: `INT8 AS [Int64Field]` rejects. Add an explicit `CAST([Int64Field] AS INT8)` to acknowledge the runtime narrowing.
- A value-producing subtree whose precise integer `[min, max]` range is entirely outside the declared integer type's domain is rejected at that subnode: `INT8 AS (1000)`, `INT8 AS ([Int8X] + 1000)`, `INT8 AS (CASE WHEN ... THEN 1000 ELSE -1 END)`.

Safe boundary arithmetic (`INT32 AS ([X] + 1)` where `[X]` is already INT32) accepts — the source-fits check passes before the bounds check runs. Cross-family numerics (int ↔ decimal / float) stay permissive; the runtime narrows safely or throws on overflow.

For integer inputs, `MOD` and `%` use conservative remainder bounds that cover both signs and the largest divisor magnitude. Decimal, floating-point, or unbounded inputs leave the integer remainder range unknown. A varying divisor therefore does not make a representable shifted remainder a DDL-time error; the ordinary runtime width checks still apply to each computed value.

### Change a computed expression or mode

`ALTER COLLECTION ... ALTER FIELD [path] SET COMPUTED (<expr>)` replaces an existing computed expression while preserving its type, nullability, and virtual or STORED mode; `STORED` after the expression also makes a virtual field stored. Dotted OBJECT paths, ARRAY OBJECT paths with `[]`, and collection `SUBTYPE [value]` scopes are supported. The complete dependency graph and result shape are validated before publication. Virtual changes retain physical rows; STORED changes recompute current and historical rows and rebuild existing indexes. A failed expression, CHECK, or rewrite rolls back the schema, data, and indexes.

`ALTER TYPE [Ns].[Type] ALTER FIELD [Field] SET COMPUTED (<expr>)` applies the same change atomically to inherited and composed uses, including repeated object, array, and subtype occurrences. Alter an inherited field on its declaring type.

```sql
ALTER COLLECTION [App].[Items] ALTER FIELD [Total]
SET COMPUTED ([Price] * [Qty] - [Discount]);

ALTER COLLECTION [App].[Items] ALTER FIELD [Lines][].[Total]
SET COMPUTED ([Price] * [Qty]);
```

<a id="change-a-computed-mode"></a>

A field also moves between plain, virtual computed and stored computed, at the same paths and scopes. Every retained version, history included, ends with the value its new mode gives it:

| Statement | Field before | Field after | Stored values |
|---|---|---|---|
| `SET COMPUTED (<expr>)` | plain | virtual | cleared in every version; reads derive the value |
| `SET COMPUTED (<expr>) STORED` | plain or virtual | stored | every version stores the expression's value; indexes on the field are rebuilt from it |
| `SET STORED` | virtual | stored | every version stores the expression's value |
| `DROP STORED` | stored | virtual | cleared in every version; reads derive the value |
| `DROP COMPUTED` | stored | plain | kept as stored; only the schema changes |
| `DROP COMPUTED` | virtual | plain | every version stores the value a read of it presents at the statement |

```sql
ALTER COLLECTION [App].[Items] ALTER FIELD [Total] SET COMPUTED ([Price] * [Qty]) STORED;
ALTER COLLECTION [App].[Items] ALTER FIELD [Total] DROP STORED;
ALTER COLLECTION [App].[Items] ALTER FIELD [Total] SET STORED;
ALTER COLLECTION [App].[Vehicles] ALTER FIELD SUBTYPE [Car] [Score] DROP COMPUTED;
```

The result follows every rule a declared computed field does, and a change that breaks one is refused with that rule's code and leaves the collection as it was. The primary key, a discriminator, a field with `REFERENCES` or a pinned layout, a field with a `DEFAULT` (remove it first with `DROP DEFAULT`) and a `NOT NULL` field (`DROP NOT NULL` first) cannot become computed (`INVALID_DEFINITION`). The expression must bind in the field's scope, fit its declared type and form no cycle, and a stored one reads no virtual field and calls only immutable built-in functions (`INVALID_DEFINITION`). A field cannot become virtual while a CHECK reads it or a stored computed field depends on it (`INVALID_DEFINITION`), nor while an index keys on it, includes it, filters on it or expires by it, or a [vector index](vectors.md#optional-vamana-index) reads it (`DEPENDENT_OBJECTS_EXIST`; drop the index first). `SET STORED` and `DROP STORED` on a plain field are `INVALID_DEFINITION`. A field that already has the mode asked for, `SET STORED` on a stored field, `DROP STORED` on a virtual one or `DROP COMPUTED` on a plain one, changes nothing, as `DROP DEFAULT` on a field without a default does. An unknown field is `UNKNOWN_FIELD`, an unknown subtype `UNKNOWN_OBJECT`, a path to an array's elements (`[Tags][]`) `INVALID_STATEMENT`, and a field a type the collection extends declares, or one inside a field of a named type, changes with [`ALTER TYPE`](composite-types.md#change-a-fields-computed-mode) (`FEATURE_NOT_SUPPORTED`). A change of stored values drops the collection's statistics and rebuilds incremental materialized views over it as any data change does; an aborted migration restores the mode and every value.

Computed dependencies bind to their containing object scope. `Details.Total AS ([Rate] * 2)` protects `Details.Rate`, including within ARRAY OBJECT elements; it does not refer to a root `Rate` field. The dependency walk includes nested subtype extensions. Dropping a whole container removes its owned expressions, and renaming that container preserves their local sibling bindings. Expressions outside the container still protect every field they reference.

Nested Object CHECK constraints protect their local field dependencies too. A subtype-qualified field change checks base expressions and that subtype's expressions; an unrelated subtype's same-named field is a separate binding.

### Typed-client `[MnemicComputed]` attribute

The .NET typed client mirrors the MQL `AS (<expr>)` form. Annotate a property with `[MnemicComputed("<expr>")]` and the schema generator emits the matching `IsComputed = true` + `ComputedExpressionText`:

```csharp
public sealed class OrderLine {
    public Guid Id { get; set; }
    public decimal Price { get; set; }
    public int Qty { get; set; }

    [MnemicComputed("[Price] * [Qty]")]
    public decimal Total { get; set; }
}
```

For body types you don't control — or for a per-collection expression override — use the fluent equivalent on `CollectionBuilder`:

```csharp
options.Collection<OrderLine>(
    "[Shop].[Lines]",
    c => {
        c.HasKey(o => o.Id);
        c.Computed(o => o.Total, "[Price] * [Qty]");
    });
```

Fluent wins over the attribute on conflict (per-collection configuration is closer to the call site). The typed-client's INSERT / UPDATE / REPLACE path silently drops the computed property from the MQL literal — the engine populates it at read time.

## ON REMOVE actions

| Source relationship shape | `Restrict` | `Cascade` | `SetNull` |
|---|---|---|---|
| Scalar field | Blocks an explicit removal while referenced | Deletes or ends the containing source document | Clears the nullable source field |
| `ARRAY <key> REFERENCES` | Blocks if any element references the target | Removes matching target ids from the array | Rejected because packed scalar elements cannot be null |
| FK field inside `ARRAY OBJECT` | Blocks if any element references the target | Deletes or ends the containing source document | Clears the matching nested field in each affected element; under `MATCH FULL`, the field in every element of that value |
| Scalar `PINNED REFERENCES` | Blocks physical cancellation of the exact target revision | Rejected | Clears the complete nullable pin |
| `ARRAY <key> PINNED REFERENCES` | Blocks physical cancellation if any exact pair matches | Rejected | Removes elements matching the exact id and revision |
| Scalar `PINNED REFERENCES` inside array objects | Blocks physical cancellation if any exact pair matches | Rejected | Clears only matching nullable slots and preserves array boundaries and other revisions; under `MATCH FULL`, clears every slot of that value |

`Restrict` is the default; a migration changes a field's action with [`ALTER FIELD … SET REFERENCES … ON REMOVE`](#change-a-reference). Every `SetNull` declaration requires the FK field to be nullable, including a pinned array whose runtime effect removes elements. An ordinary temporal end retains its target version, so pinned actions do not fire merely because `ValidTo` arrives. They apply when `CANCEL VERSION` physically removes the bound revision. For ordinary references, an explicit temporal end applies the action across the target's resulting absence interval, including future-effective source versions. At wall-clock expiry, time cannot be rolled back: `Cascade` and `SetNull` run as integrity work, while `Restrict` is reported but cannot postpone the authored end.

A materialized view can be a reference target. A key that leaves the view when REFRESH, `ALTER MATERIALIZED VIEW`, or a source rebuild publishes a new generation is a removal: its action runs inside that migration, and `Restrict` rejects the replacement. See [references to a materialized view](materialized-views.md#references-to-a-materialized-view).

## ALTER NAMESPACE

```sql
ALTER NAMESPACE [OldName] RENAME TO [NewName];
```

Renames the namespace identity. Collections, types, enums, plain views and materialized views follow its new label. Their stable IDs, schema generations, stored definition payloads and document rows remain unchanged. Existing static dependencies continue to refer to the same objects. Other sessions see the new label after commit; the migration sees it immediately.

When the destination namespace already exists, the command merges the source members into it transactionally and rejects destination-name collisions (`DUPLICATE_NAME`); every incremental materialized view, among the members or reading one, keeps maintaining from its sources, within the migration and after it. A source namespace with no members is `UNKNOWN_OBJECT`. Namespaces are created implicitly. Trigger names remain globally scoped, while their target collections follow stable IDs.

The `[Mnemic]` namespace (and any `[Mnemic.*]` sub-namespace) is reserved for engine-internal use. `ALTER NAMESPACE [Mnemic] RENAME TO ...` is rejected with `"Cannot rename Mnemic namespace."`, and renaming any other namespace TO `[Mnemic]` / `[Mnemic.*]` is rejected with `"Cannot rename namespace to Mnemic."`. The same guard applies to `ALTER COLLECTION ... RENAME TO` and `DROP COLLECTION` against system collections.

Collection names embedding `__refresh_`, `__alter_`, or `__retired_` are also reserved — those segments name materialized-view transient backings, and a generation a refresh or alter replaced keeps its durable retirement mark until the drop that engine open completes ([materialized views](materialized-views.md)). `CREATE COLLECTION`, `CREATE VIEW`, `CREATE MATERIALIZED VIEW`, `ALTER COLLECTION ... RENAME TO`, `ALTER VIEW ... RENAME TO` and `ALTER NAMESPACE ... RENAME TO` (whose namespace becomes a segment of every relation name it holds) reject them with `RESERVED_NAME`.

Reusable types and enums can also move between namespaces with `ALTER TYPE` or `ALTER ENUM ... RENAME TO`. Their destination is a complete bracketed identity; see [type renames](composite-types.md#rename-or-move-a-type) and [enum renames](enums.md#rename-or-move-an-enum).

## See also

- [Polymorphism](polymorphism.md)
- [Enums](enums.md)
- [Indexes](indexes.md)
- [Migrations](migrations.md) — required wrapper for DDL

## Detailed syntax and execution

### Executable metadata initialization

Before the first statement, MQL asynchronously validates persisted CHECK and computed expressions, views, materialized views, and triggers as one metadata generation. Cancellation and statement deadlines cover this work; a cancelled attempt has no statement effects and leaves initialization retryable. Corrupt metadata fails closed before execution. Embedded hosts can explicitly await `MqlExecutor.InitializeAsync` before accepting requests; constructing an executor alone does not validate its catalogs.

`ALTER COLLECTION ... ADD FIELD` accepts `MATCH FULL` and `MATCH SIMPLE` on foreign keys, including a field inside an existing array of objects. Missing values backfill as NULL; DEFAULT backfills every occurrence and makes the new field NOT NULL. The match policy survives later nullability changes and reopen.

### Object and array defaults

Use `SET DEFAULT ([Field]: value)` for an object and `SET DEFAULT [value, ...]` for an array. Nested arrays, objects, and `SUBTYPE [Name] (...)` retain their ordinary literal shapes. `ALTER ... ADD FIELD ... DEFAULT` uses the same shapes for one-time backfill. Named-type defaults propagate to all consumers.

Default leaves are literals or NOW; parameter references, field references, subqueries, and arbitrary expressions are rejected. Schema validation checks field membership, subtypes, dimensions, nullability and enum values before publication, and checks persisted defaults again at executor startup. An explicit NULL wins over a default. DROP DEFAULT changes what later writes give the field without rewriting stored values.

Nested field evolution also updates composite default templates in the same migration. RENAME retains each supplied value under the new field name; DROP removes the dropped member; ADD with a backfill supplies that member in existing parent templates. Arrays and selected subtype objects retain their shape. The transformed template is validated before publication, and rollback restores both the prior schema and its default text.

A metadata change that makes an existing parent default invalid is rejected and rolled back. For example, before dropping a required child's default, replace or remove any parent template that relies on it. Before SET NOT NULL, replace a parent template that explicitly supplies NULL for that child.

When ADD backfills NOW into an existing parent template, it stores the admitted ALTER statement timestamp, matching the row backfill. NOW already declared in a SET DEFAULT template continues to resolve in each later write that takes it.



<a id="syntax-3-type-system"></a>

### Type system

| Keyword | CLR | Width |
|---------|-----|-------|
| `STRING` | string | variable |
| `INT8`, `INT16`, `INT32`, `INT64` (alias `LONG`) | sbyte / short / int / long | 1 / 2 / 4 / 8 |
| `BOOL` | bool | 1 |
| `FLOAT32` (alias `SINGLE`), `FLOAT64` (aliases `FLOAT`, `DOUBLE`) | float / double | 4 / 8 |
| `VECTOR(n)` | float[] | `4 × n`, `1 <= n <= 32768` |
| `DECIMAL` | decimal | 16 |
| `GUID` | Guid | 16 |
| `DATETIME` | DateTimeOffset | 10 (ticks + offset minutes) |
| `DATEONLY` | DateOnly | 4 |
| `TIMEONLY` | TimeOnly | 8 |
| `BYTES` | byte[] | variable |
| `OBJECT` | nested document | variable |
| `ARRAY` | list of element-schema | variable |

`BYTES` values are supplied as Base64 text. `''` decodes to a present zero-length byte array; it is not `NULL` and remains non-null through filters, secondary-index seeks, joins, grouping, and aggregates. Join equality preserves that distinction under every physical plan (left-deep or bushy): the empty value matches another empty value and never `NULL`, while a `NULL` key matches only another `NULL` key (see [JOIN](queries.md#join)). Correlated `MATCH` preserves it too, including base or extension fields on polymorphic array elements.

`VECTOR(n)` stores exactly `n` finite `FLOAT32` components. Use a `VECTOR[1, -0.5, 2]` literal or bind a `float[]`; wrong dimensions and NaN/infinity are rejected. Exact scalar functions are `COSINE_DISTANCE(a, b)`, `L2_DISTANCE(a, b)`, and `DOT_PRODUCT(a, b)`.

Primary keys: one top-level stored `INT8`, `INT16`, `INT32`, `INT64`, `GUID`, `STRING`, or `BYTES` field. Signed integer identities retain their native one-, two-, four-, or eight-byte widths. `AUTOINCREMENT` is limited to `INT32` / `INT64`; `SEQUENTIAL` is limited to `GUID`. String identity uses invariant simple-lowercase Unicode without normalization while preserving the inserted spelling. Bytes identity is exact. Empty String and Bytes keys are present values. Variable keys must fit with slot metadata on the home data page; their document bodies may spill, but their identity bytes do not.

<a id="syntax-create-collection"></a>

### CREATE COLLECTION

```sql
CREATE COLLECTION [Ns].[Collection]
  [(Temporal: TRUE | FALSE,                    -- each option at most once, any order;
    CreatedBy: TRUE | FALSE,                   -- an omitted option is FALSE; () is rejected
    SkipUnchangedUpdates: TRUE | FALSE,
    CacheCurrentVersion: TRUE | FALSE)]
  [EXTENDS [Ns].[Type] [, [Ns].[Type]]*]       -- after the options, before every WITH
WITH SCHEMA (
  [Id]      INT32 PRIMARY KEY [AUTOINCREMENT], -- also INT8, INT16, INT64, GUID, STRING, or BYTES; GUID may be SEQUENTIAL
  [Name]    STRING NOT NULL,
  [Notes]   STRING NULLABLE,
  [Price]   DECIMAL DEFAULT 0,
  [CreatedAt] DATETIME NOT NULL DEFAULT NOW,
  [Total]   DECIMAL AS ([Price] * [Qty]),       -- virtual computed column
  [RegionId] GUID REFERENCES [Geo].[Regions] [ON REMOVE Restrict|Cascade|SetNull],
  [Pinned]  ARRAY GUID PINNED REFERENCES [App].[Versions] ON REMOVE Restrict,
  [Status]  [App].[OrderStatus],            -- enum-typed
  [Tags]    ARRAY STRING,
  [Address] OBJECT ([City] STRING, [Zip] STRING NULLABLE),
  [Slots]   ARRAY OBJECT ([Power] INT32, [Connector] STRING)
)
[WITH SUBTYPE [A] ([X] INT32) WITH SUBTYPE [B] (...)]    -- polymorphic
[WITH CHECK [name] (predicate)]                          -- persisted row predicate
-- WITH SCHEMA, WITH SUBTYPE, and WITH CHECK may come in any order; WITH SCHEMA exactly once
```

`CREATE COLLECTION` and `CREATE TYPE` each accept at most 256 table-level `WITH CHECK` constraints. Parsing and direct AST validation enforce the persisted schema-codec limit before reading or traversing another expression. For `EXTENDS`, inherited type CHECKs and locally declared collection CHECKs share one 256-entry total. A CHECK rejects a row only when the final predicate result is FALSE; TRUE and UNKNOWN (NULL) pass. A NULL input can still yield FALSE, for example with `[F] IS NOT NULL`; it does not automatically satisfy the constraint.

`WITH SCHEMA` is the sole authoritative schema input. The exact-v1 `ExecuteMql` frame has no binary schema field; it carries flags, MQL text, and the mandatory parameter count plus parameters. Missing or trailing fields are rejected before statement execution. Every individual schema field list accepts at most 1,024 fields, including collection/type roots, nested OBJECTs, ARRAY OBJECT elements, and subtype extensions. Each collection-level or field-level subtype union accepts at most 1,024 subtypes.

Field modifiers: `PRIMARY KEY`, `AUTOINCREMENT`, `SEQUENTIAL`, `NOT NULL`, `NULLABLE`, `DEFAULT <literal>|NOW`, `AS (<expression>)` (virtual computed column), `[PINNED] REFERENCES ... [ON REMOVE Restrict|Cascade|SetNull] [MATCH FULL|MATCH SIMPLE]`.

Foreign keys belong to the fields that declare them. Sibling subtypes may reuse a logical path with different targets, key types, array dimensions, removal actions, or pin modes; another subtype may leave the path unconstrained. Validation, reverse lookup, sealing, and removal use the active subtype's field declaration. Identical policies at the same path coalesce, including `MATCH FULL`: its all-null-or-all-present rule counts only occurrences declaring that same policy. Renames, subtype additions/drops, migration rollback, and reopen preserve each surviving declaration.

Ordinary foreign keys support `INT8`, `INT16`, `INT32`, `INT64`, `GUID`, `STRING`, and `BYTES` targets in scalar fields, packed arrays, and array-object leaves, including nested fields and polymorphic subtypes. For example: `[Codes] ARRAY STRING REFERENCES [App].[Products] ON REMOVE Cascade`. String identity is invariant simple-lowercase; Bytes identity is exact. Empty values are present references requiring a target; nullable object leaves can represent absence. Cascade removes matching identifiers from a packed array; SetNull clears matching object leaves. Rollback, reopen, and array membership indexes retain those semantics. Stored array framing and String UTF-8 are validated completely before target lookup; corruption is `ARRAY_PAYLOAD_CORRUPT`. `PINNED REFERENCES` supports the same seven identifier types, including `STRING` and `BYTES`, in scalar fields, packed arrays, and array-object leaves. Use `PIN(id, revision)` for each present pin. Empty identifiers are valid; `NULL` denotes an absent scalar reference. Cancellation matches the identifier and exact revision, so `SetNull` retains other revisions of that identifier, except inside an array-object value under `MATCH FULL`, which it clears whole.

Each CHECK predicate is persisted and must be a self-contained schema expression: every field reference must resolve against the collection schema, query parameters are forbidden, and virtual computed columns cannot be referenced. Invalid persisted bindings fail executor initialization rather than evaluating as NULL.

**FK `MATCH FULL` / `MATCH SIMPLE`** are accepted on CREATE COLLECTION and `ALTER … ADD FIELD`. Only meaningful on an `ARRAY OBJECT(...)` FK whose FK field is nullable (the one shape where a *partial*-null reference set can occur). `MATCH SIMPLE` (default) validates each element independently — null FK fields skip, partially-null arrays are accepted if every non-null reference resolves. `MATCH FULL` is all-or-nothing per array value: every element references or every element is null; a mix is rejected (`FK MATCH FULL violation`; empty/all-null arrays are clean). An `ON REMOVE SetNull` never leaves a mix: when any element names the removed target or cancelled revision, it clears the FK field in every element of that value. For single-column FKs the two are identical, and scalar array elements can't be null, so `MATCH FULL` only has teeth on array-of-object FKs.

**FK removal actions.** `Restrict` is the default and blocks an explicit removal while a matching reference exists. `Cascade` deletes or ends the containing source document for a scalar FK or an FK nested inside `ARRAY OBJECT`; for `ARRAY <key> REFERENCES`, it removes matching ids from the packed array. `SetNull` clears a nullable scalar FK or the matching nested FK fields in `ARRAY OBJECT` (under `MATCH FULL`, every nested FK field of that value); it is rejected for an ordinary packed scalar array because an element cannot be null. Every `SetNull` declaration requires the declaring FK field to be nullable, including a pinned array. Pinned FKs reject `Cascade`: on physical version cancellation, `Restrict` blocks the exact `(id, revision)`, scalar `SetNull` clears the pin, and pinned-array `SetNull` removes exact matches. Scalar pins inside array objects validate every non-null leaf's retained revision. Their `Restrict` and `SetNull` actions match both key and revision across nested array dimensions and subtype envelopes; SetNull preserves the containing objects and array boundaries, and under `MATCH SIMPLE` pins to other keys or revisions. Ordinary temporal expiry retains pinned target revisions, so it does not fire pinned actions. Wall-clock expiry cannot be postponed by `Restrict`. A bounded temporal INSERT preserves existing front/tail identity coverage; a version's `ValidTo` alone is not a parent removal and does not veto a valid overlay through reverse Restrict references. UPDATE inherits the current end or clears it; it has no explicit new `VALID TO` clause. DELETE and CANCEL VERSION enforce actions on actual absence intervals. Use Cascade or SetNull when the relationship requires automatic cleanup on expiry; the rows that cleanup writes maintain the INCREMENTAL materialized views over their collection like any other write.

`AUTOINCREMENT` (INT32/INT64 PK only) is an engine-assigned monotone counter. `SEQUENTIAL` is the GUID-PK generation option: the engine creates an RFC 9562 UUIDv7 when an INSERT omits the key. Guid index keys use RFC network byte order, so with a normally advancing clock new timestamp prefixes cluster near the current right-side key range rather than scattering like UUIDv4. The random suffix means values created within one millisecond are not strictly ordered; clock rollback and explicit values can also insert earlier. `SEQUENTIAL` is valid only on a GUID primary key, is mutually exclusive with `AUTOINCREMENT` / `AS`, and an explicitly supplied PK still wins. Use `AUTOINCREMENT` when monotone numeric reservations are required.

**Stored computed columns** use `AS (<expr>) STORED`. The engine derives values after generated key assignment and every retry-local merge, before constraints, triggers, and indexes. ADD backfills existing rows. Stored fields support ordinary indexes, INCLUDE/partial indexes, CHECK references, and row-local incremental materialized-view projections. Stored DATETIME fields may define TTL/EXPIRY indexes. Expressions use immutable built-ins and ordinary/stored dependencies; reject clock/custom/enum-lookup functions and virtual dependencies (declare the dependency STORED). Raw binary writes also derive the value, and recovery replays the stored image.

Computed results may be scalar, `OBJECT`, `ARRAY`, or `VECTOR(n)`. Structured results must preserve the declared shape in every value-producing arm: field order, names, child types/nullability, vector width, named-type and enum identity, subtype layouts, and nested computed definitions. Field references, `CASE`, `COALESCE`/`IFNULL`, `NULLIF`, and vector literals retain those shapes. Incompatible arms reject at definition time. Polymorphic container paths use the active subtype; a missing extension value is NULL, and candidates with incompatible shapes cannot bind to one computed result. Reading a child of a virtual computed container evaluates the parent before filtering or projection.

Computed declarations use the canonical built-in type and shape contracts even when an embedded host replaces a scalar function. Virtual scalar results must still convert to the declared column type. A replacement for COALESCE, IFNULL, or NULLIF cannot promise its original structured argument shape; an affected virtual structured field fails safely without quarantining healthy schema metadata. Restoring the original implementation permits subsequent reads. STORED evaluation continues to use the original immutable implementations.

**Virtual computed columns** (`AS (<expr>)`) derive their value from an expression evaluated on every read. The text is stored in the catalog and re-evaluated against each row at query time — the value is **not** stored on disk. INSERT/UPDATE/REPLACE/UPSERT must NOT supply a value for a computed field, including a nested, array-element, or active-subtype slot. Values are also synthesized for `RETURNING` and change-event presentation images. Computed columns have nullable schema metadata even when an expression handles NULL inputs with COALESCE, IFNULL, or CASE. `NOT NULL` is rejected at declaration and by ALTER COLLECTION / ALTER TYPE SET NOT NULL. They cannot be `PRIMARY KEY`, `AUTOINCREMENT`, or `REFERENCES`. `WHERE` / `ORDER BY` / `GROUP BY` on a virtual computed field require read-time derivation; stored computed values support ordinary indexes. Correlated EXISTS / scalar predicates beside a computed reference run after per-row injection when they cannot be decorrelated; eligible equality shapes keep the set-based decorrelation path. Grouping, total count, projection, and paging consume the filtered injected rows in that order. With JOIN, the residual predicate binds the complete merged row and can reference either joined alias.

Aggregate arguments, aggregate `FILTER (WHERE ...)`, and aggregate-local `ORDER BY` keys use the same injected values. Wildcard EXCLUDE/REPLACE/RENAME modifiers remain effective and retain their normal schema checks on this path. Source-alias qualification does not change computed-reference discovery.

A physical JOIN inner is enriched through a statement-owned, spill-bounded virtual replay when ON, residual filtering, ordering/grouping/aggregation, or projection can observe one of its computed slots. If no JOIN consumer observes those slots, the ordinary physical scan and index path remains eligible. An explicit CTE is semantically equivalent but is not required for correctness.

Window arguments, partitions, window ordering keys, and window FILTER predicates all trigger read-time injection when they reference a computed field. Field-to-field JOIN residuals inspect both operands for computed outer paths.

A computed field may appear in the collection base, a nested `OBJECT`, an `ARRAY OBJECT` element, a field- or collection-level subtype extension, or a named composite type used through field composition or `EXTENDS`. Each expression binds relative to its containing object. Collection-level base and active-subtype fields share a dependency graph. A field absent from the active subtype has typed NULL when another subtype owns its declaration; it never reads another row or subtype's data. Unknown fields, incompatible result types, and cycles in any active subtype reject before publication. Stored consumers require stored computed dependencies. Nested objects and array elements retain their containing-object scopes; nested subtype expressions may read their base plus their own extension. Computed fields can depend on other computed fields in that scope, including nested computed values. Dependencies evaluate before consumers, regardless of declaration order; self-references and cycles reject at DDL and metadata-validation boundaries. Persisted expression text is parsed and schema-bound at DDL time and executor startup, and again before a generation's first cached plan is used; invalid text or a late programmatic schema binding fails closed instead of being skipped as NULL. Every computed-bearing binary scope is structurally validated before enrichment. Malformed document/array/envelope layout, a missing expression-plan binding, or a runtime evaluator failure aborts the query, RETURNING result, or event presentation rather than fabricating NULL. Recursive paths wire `ENUM_LABEL` through the field's enum id and `ENUM_VALUE` through the enum name. Intentional expression NULL propagation yields NULL; a numeric result that cannot fit its declared width fails the read with `INVALID_VALUE` ([integer arithmetic uses INT64](#pitfall-integer-arithmetic-uses-int64-and-explicit-recovery)).

DDL-time validation rejects clearly-overflowing numeric assignments: referencing a same-family numeric field wider than the declared output type (`INT8 AS [Int64Field]`) and value-producing subtrees whose `[min, max]` integer range is entirely outside the declared integer domain (`INT8 AS (1000)`, `INT8 AS ([Int8X] + 1000)`, `INT8 AS (CASE WHEN ... THEN 1000 ELSE -1 END)`). Safe boundary arithmetic (`INT32 AS ([X] + 1)` where `[X]` is INT32) accepts — the source-fits rule passes before the bounds rule runs. Wrap an intentional narrowing in `CAST` to acknowledge the runtime check.

`Temporal: TRUE` opens the temporal version chain (`ValidFrom` / `ValidTo` on every row, AS-OF queries, version cancellation). A temporal collection retains past and current versions; ordinary `DELETE` closes validity without reclaiming history. `CANCEL VERSION` physically removes only a future version. MQL has no automatic retention policy or general historical purge.

<a id="syntax-alter-collection"></a>

### ALTER COLLECTION

```sql
ALTER COLLECTION [Ns].[Coll] ADD FIELD [Name] TYPE [DEFAULT v];       -- DEFAULT backfills existing rows; field becomes NOT NULL
ALTER COLLECTION [Ns].[Coll] ADD FIELD [At] DATETIME DEFAULT NOW;     -- backfills the ALTER-time timestamp (DATETIME only; @p rejected)
ALTER COLLECTION [Ns].[Coll] ADD FIELD [Total] DECIMAL AS ([Price] * [Qty]); -- virtual computed
ALTER COLLECTION [Ns].[Coll] ADD FIELD [Ref] GUID REFERENCES [Ns].[T] [ON REMOVE Restrict|Cascade|SetNull] [DEFAULT v];
ALTER COLLECTION [Ns].[Coll] ADD FIELD [Ref] GUID PINNED REFERENCES [Ns].[T] [ON REMOVE Restrict|SetNull] [DEFAULT PIN(id, rev)]; -- DEFAULT is one-time backfill only
ALTER COLLECTION [Ns].[Coll] ADD FIELD [Tags] ARRAY STRING;           -- ARRAY of a scalar element type
ALTER COLLECTION [Ns].[Coll] ADD FIELD [TagIds] ARRAY GUID REFERENCES [Ns].[Tags] ON REMOVE Cascade; -- array FK
ALTER COLLECTION [Ns].[Coll] ADD FIELD [Items] ARRAY OBJECT ([Sku] STRING, [Qty] INT32); -- composite element schema
ALTER COLLECTION [Ns].[Coll] ADD FIELD [Items] ARRAY OBJECT ([AuthorId] GUID REFERENCES [Ns].[Authors], [Role] STRING); -- ARRAY OBJECT with nested element FK
ALTER COLLECTION [Ns].[Coll] ADD FIELD [Items] ARRAY OBJECT ([Tag] STRING)
    WITH SUBTYPE [text]  ([Body] STRING)
    WITH SUBTYPE [image] ([Url] STRING, [Width] INT32);               -- polymorphic ARRAY OBJECT (auto _type)
ALTER COLLECTION [Ns].[Coll] ADD FIELD [F] [Ns].[Type] [DEFAULT (...)]; -- named composite type (or ARRAY [Ns].[Type]); type REFERENCES enforced
ALTER COLLECTION [Ns].[Coll] DROP FIELD [Name];
ALTER COLLECTION [Ns].[Coll] RENAME FIELD [Old] TO [New];
ALTER COLLECTION [Ns].[Coll] DROP FIELD SUBTYPE [value] [Name];          -- drop a subtype extension field and its values
ALTER COLLECTION [Ns].[Coll] RENAME FIELD SUBTYPE [value] [Old] TO [New];  -- rename a subtype extension field
ALTER COLLECTION [Ns].[Coll] MOVE FIELD [SUBTYPE [value]] [F] FIRST | LAST | BEFORE [S] | AFTER [S]; -- rewrites every retained version in the new order
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [F] SET NOT NULL | DROP NOT NULL; -- SET validates every retained row
ALTER COLLECTION [Ns].[Coll] ALTER FIELD SUBTYPE [value] [F] SET NOT NULL; -- ... scoped to a subtype field
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [F] SET TYPE <scalar | enum | [Ns].[Type] | VECTOR(n) | ARRAY <scalar | enum | [Ns].[Type]>>; -- converts every retained value as CAST, element by element and field by field
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [F] SET TYPE OBJECT | ARRAY OBJECT; -- detaches a field, or an array's elements, from its named type
ALTER COLLECTION [Ns].[Coll] ADD SUBTYPE [value] ([F1] T1, [F2] T2 NOT NULL, ...);
ALTER COLLECTION [Ns].[Coll] ADD SUBTYPE [value] ([F1] T1 DEFAULT v, ...) FOR EXISTING DOCUMENTS; -- a collection without subtypes becomes polymorphic; every retained version takes [value]
ALTER COLLECTION [Ns].[Coll] RENAME SUBTYPE [old] TO [new];          -- every retained version stores the new name
ALTER COLLECTION [Ns].[Coll] DROP SUBTYPE [value];
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [Obj] ADD SUBTYPE [v] ([F1] T1 [DEFAULT d], ...); -- a polymorphic field's new subtype, or fields for one it has
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [Obj] DROP SUBTYPE [v];  -- refused while any retained version holds a value of [v]
-- Every field path [F] above may step into a polymorphic field's subtype: [Obj].SUBTYPE [v].[F], [Arr][].SUBTYPE [v].[F].
ALTER COLLECTION [Ns].[Coll] SET EXTENDS [Ns].[T1] [, [Ns].[T2] ...];  -- the inherited fields follow the new bases by name; kept ones keep their values
ALTER COLLECTION [Ns].[Coll] DROP EXTENDS;                             -- the inherited fields and their values leave
ALTER COLLECTION [Ns].[Coll] ADD CHECK [name] (predicate);             -- every retained version must pass first
ALTER COLLECTION [Ns].[Coll] DROP CHECK [name];                        -- or DROP CHECK (predicate) for an unnamed CHECK
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [Obj] ADD CHECK [name] (predicate); -- an OBJECT / ARRAY OBJECT field's CHECK
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [Obj] DROP CHECK [name] | (predicate);
ALTER COLLECTION [Ns].[Coll] RENAME TO [NewNs].[NewName];               -- a name that differs only in letter case respells it
ALTER COLLECTION [Ns].[Coll] SET CREATEDBY | TEMPORAL | SKIPUNCHANGEDUPDATES | CACHECURRENTVERSION TRUE|FALSE; -- a creation option, changed later
ALTER COLLECTION [Ns].[Coll] SET PRIMARY KEY [F];                        -- another field identifies the documents; every version is re-keyed
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [Key] SET | DROP AUTOINCREMENT | SEQUENTIAL; -- the primary key's generator; no row is rewritten
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [F] SET [PINNED] REFERENCES [Ns].[T] [ON REMOVE Restrict|Cascade|SetNull] [MATCH FULL|SIMPLE]; -- every retained version must resolve
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [F] DROP REFERENCES;  -- the field keeps its values and stored layout
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [F] SET COMPUTED (<expr>) [STORED]; -- a new expression; a plain field becomes computed, STORED makes it stored
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [F] SET STORED | DROP STORED;      -- a computed field's value is stored, or derived on reads
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [SUBTYPE [value]] [F] DROP COMPUTED;                 -- the field becomes plain and keeps its values

-- Pin / clear a column-level default on an existing field. Schema-only —
-- existing rows stay untouched; the default fills the field where an
-- inserted document, or an object a later write assigns, leaves it out,
-- and where a write assigns it DEFAULT (not the top-level fields a REPLACE
-- leaves out). Explicit values (including NULL) override.
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [Name] SET DEFAULT 'USD';
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [Qty]  SET DEFAULT 0;
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [At]   SET DEFAULT NOW;
ALTER COLLECTION [Ns].[Coll] ALTER FIELD [Name] DROP DEFAULT;
```

`ADD FIELD DEFAULT` accepts scalar literals or `NOW`; `PIN(id, rev)` is accepted only with `PINNED REFERENCES`. The same closed set is enforced for programmatic ASTs before any schema mutation—field-reference, subquery, list, and other expression nodes never collapse to a silent null default. Numeric defaults use invariant MQL spelling during persistence and row backfill; the ASCII `-` in `DEFAULT -1` is independent of server culture.

Row-rewriting forms (`ADD FIELD`, `DROP FIELD`, `ADD SUBTYPE … FOR EXISTING DOCUMENTS`, `RENAME SUBTYPE`, `DROP SUBTYPE`, `MOVE FIELD`, `SET EXTENDS`, `DROP EXTENDS`) can follow DML or earlier rewrites of the same collection in one migration. Their scans include the owner's dirty pages and private appended rows. The owner sees the new schema and rows immediately; commit or rollback settles the whole block. Competing owners remain fenced. Standalone raw-engine rewrites reject an open competing transaction touching the collection; settle that owner first. After a migration stages any query-visible field or subtype schema change, it owns that collection's schema+row+index generation through commit/rollback: its own later DML and reads (including FK probes) use the new metadata/layout, competing writers retry after settlement, committed readers cannot cross the private generation, and competing index/schema DDL waits or fails before shared- generation mutation. Subtype-only mutations advance both the collection and extension-schema versions so cached plans cannot retain old extension fields. Disjoint-collection changes remain concurrent.

`DROP SUBTYPE` publishes the reduced subtype schema only after every matching document delete succeeds in the same transaction. A row already removed by an earlier FK cascade in that transaction is skipped; any unrelated delete failure aborts the statement and preserves both the subtype schema and its rows.

Nested `ADD FIELD` / `DROP FIELD` recursively rewrite existing `OBJECT` and `ARRAY OBJECT` values. Array element order and unchanged sibling values are preserved. `ADD FIELD ... DEFAULT` materializes the default in every existing element; `DROP FIELD` removes the leaf payload and any FK registration rooted at that leaf in the same migration publication.

`ADD FIELD` supports `ARRAY` of a **scalar** element type (incl. array FKs) AND `ARRAY OBJECT (...)` with a composite element schema — the same parser primitives `CREATE COLLECTION` uses produce the nested element shape, so polymorphic `WITH SUBTYPE ...` and nested element-level `REFERENCES` are supported the same way. Existing rows read NULL, or an array-literal `DEFAULT` (`DEFAULT ['x', 'y']`) that backfills them. Recursive element declarations such as `ARRAY ARRAY INT32` and `ARRAY ARRAY OBJECT (...)` are accepted by CREATE and ALTER. Each nested literal is validated against its own element schema; empty arrays are preserved, and NULL elements are rejected at every level.

`ADD FIELD … REFERENCES` adds a foreign key to a live collection: the target must **already exist** (no forward reference, unlike `CREATE` in a migration), all `CREATE`-time FK invariants apply (pinned target temporal; PK shape Int8 / Int16 / Int32 / Int64 / Guid / String / Bytes; Cascade rejected on pinned; SetNull only on a nullable scalar / not on a non-pinned array; field type matches target PK), the field is nullable without a `DEFAULT`, and a `DEFAULT` backfills all existing rows and is validated to resolve in the target (pinned ⇒ one-time `DEFAULT PIN(id, rev)` backfill with no retained catalog default; non-pinned ⇒ id literal; an `ARRAY` FK ⇒ an array literal whose every element must resolve). `ALTER FIELD [F] SET [PINNED] REFERENCES …` gives an existing field a reference or replaces its target and policy once every retained version resolves, and `DROP REFERENCES` removes it, neither rewriting a row ([Change a field's reference](#change-a-reference)). `DROP / RENAME FIELD` cascades to FK / index registrations — including partial-index `WHERE` filters: `RENAME FIELD` rewrites the persisted filter of every partial index referencing the field, and `DROP FIELD` is **rejected** while any partial index's filter references the field (drop that index first; indexes depending on the field or its descendants through keys or INCLUDE are otherwise dropped automatically, including UNIQUE indexes). `RENAME FIELD SUBTYPE [value] [Old] TO [New]` renames a field inside a polymorphic subtype's extension schema — the only way to rename a subtype field (an unscoped `RENAME FIELD` can't reach one and errors with that guidance). It retains row pages and rebuilds affected array indexes from the new schema. A shared array path stays indexed for sibling declarations; an exclusive path follows the rename. It rejects an unknown field or a name that already exists in the subtype. `DROP FIELD SUBTYPE [value] [Name]` removes a field, or a field nested in one of its objects, from one subtype's extension and rewrites that subtype's current and historical versions without it; the base portion and every other subtype stay as they are. An array index on the path keeps serving the other declarations of it and retires with the last one. It rejects a subtype the collection does not have (`UNKNOWN_OBJECT`), a field the subtype does not declare (`UNKNOWN_FIELD`), a path through a field composed of a named type (change the type instead), and, as for base fields, a field the collection's own CHECK constraints or computed columns read (`DEPENDENT_OBJECTS_EXIST`). `ALTER FIELD [F] SET NOT NULL / DROP NOT NULL` flips nullability: `SET NOT NULL` checks every retained row before it publishes and refuses the change while any row lacks a value ([`SET NOT NULL` validates every retained row](#pitfall-set-not-null-validates-every-retained-row)); the PK cannot be made nullable. `ALTER FIELD [F] SET TYPE` converts every retained version's value with CAST semantics in one rewrite, an array element by element and an object into a named type field by field, and is refused for a value without a conversion or that the new type cannot hold, a field whose type something else owns, and a field a declaration or a vector index reads; `SET TYPE OBJECT` and `SET TYPE ARRAY OBJECT` detach a field, or an array's elements, from its named type; see [Change a field's type](#change-a-field-type). The `SUBTYPE [value]` modifier scopes `DROP FIELD` / `RENAME FIELD` / `ALTER FIELD` (drop, rename, type, default, nullability, computed expression and mode, reference, CHECK) to a subtype's extension schema. FK targets declared at `CREATE` are validated at `COMMIT MIGRATION` (deferred check; see Migrations).

**`SET DEFAULT` accepts only literal values and `NOW`** — parameter references (`@p`) are rejected at DDL time because parameter values are per-call and have no meaning in the later writes that take the default. `PIN(...)` literals are also rejected. `DEFAULT NULL` is rejected with the message that `DROP DEFAULT` is the canonical way to clear an existing default. `SET DEFAULT` cannot target a primary key, discriminator, computed/virtual field, or pinned foreign key. Object and array defaults use ordinary MQL literal syntax; an enum default must name a member of its enum. Default application recurses through base and selected subtype schemas, nested present `OBJECT` values, and present `ARRAY OBJECT` elements. It fills only omitted leaves: explicit `NULL` wins, and an absent/null parent, absent array, or inactive subtype is not synthesized merely for a child default. A default declared on the parent itself may supply an object or array. `SET NOT NULL` is rejected while the field's FK uses `ON REMOVE SetNull`, because that action requires a nullable source slot. Programmatic value ASTs are structurally validated before canonical default printing; malformed or over-deep nested `PIN` values fail before recursion or persistence.

<a id="syntax-drop-collection"></a>

### DROP COLLECTION

```sql
DROP COLLECTION [Ns].[Coll];
```

Honors FK Restrict — blocks if active references exist.

<a id="syntax-describe-collection"></a>

### DESCRIBE COLLECTION

Returns the collection's one `[Mnemic].[Collections]` row ([system collections](system-collections.md)): its options and its `Fields` entries, each field's type, nullability, default, computed expression and foreign-key target included. Indexes and CHECK constraints are not on that row; read them from `[Mnemic].[Indexes]` and `[Mnemic].[Constraints]`.

<a id="syntax-rename-namespace"></a>

### RENAME NAMESPACE

```sql
ALTER NAMESPACE [OldNs] RENAME TO [NewNs];
```

`[Mnemic]` namespace cannot be renamed. Other namespaces cannot be renamed *to* `[Mnemic]` / `[Mnemic.*]`.

Collection names embedding `__refresh_`, `__alter_`, or `__retired_` are reserved (materialized-view transient backings, whose replaced generations open drops through their durable retirement mark); CREATE COLLECTION, CREATE VIEW, CREATE MATERIALIZED VIEW, ALTER COLLECTION ... RENAME TO, ALTER VIEW ... RENAME TO and ALTER NAMESPACE ... RENAME TO reject them (`RESERVED_NAME`).

<a id="syntax-object-and-array-defaults"></a>

### Object and array defaults

Use `SET DEFAULT ([Field]: value)` for an object and `SET DEFAULT [value, ...]` for an array. Nested arrays, objects, and `SUBTYPE [Name] (...)` retain their ordinary literal shapes. `ALTER ... ADD FIELD ... DEFAULT` uses the same shapes for one-time backfill. Named-type defaults propagate to all consumers.

Default leaves are literals or NOW; parameter references, field references, subqueries, and arbitrary expressions are rejected. Schema validation checks field membership, subtypes, dimensions, nullability and enum values before publication, and checks persisted defaults again at executor startup. An explicit NULL wins over a default. DROP DEFAULT changes what later writes give the field without rewriting stored values.

Nested field evolution also updates composite default templates in the same migration. RENAME retains each supplied value under the new field name; DROP removes the dropped member; ADD with a backfill supplies that member in existing parent templates. Arrays and selected subtype objects retain their shape. The transformed template is validated before publication, and rollback restores both the prior schema and its default text.

When ADD backfills NOW into an existing parent template, it stores the admitted ALTER statement timestamp, matching the row backfill. NOW already declared in a SET DEFAULT template continues to resolve in each later write that takes it.

## Usage pitfalls


<a id="pitfall-unbalanced--after-with-schema-"></a>

### Unbalanced `)` after `WITH SCHEMA (...)`

A script statement ends only at a semicolon outside strings, brackets, parentheses, comments, and a trigger body ([multi-statement execution](syntax.md#multi-statement-execution)); line breaks and indentation never end one. A stray or missing `)` therefore keeps the parentheses open across the following statements, and the script is rejected as unbalanced rather than split at the line where the mistake is. Close every `(` and end every statement with `;`:

```sql
CREATE COLLECTION [a].[b] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY
);                                 -- the closing paren may start a line
```

<a id="pitfall-add-field-default-backfills-set-default-applies-at-insert"></a>

### `ADD FIELD DEFAULT` backfills; `SET DEFAULT` applies to later writes

CREATE field defaults and SET DEFAULT retain default metadata; neither changes nullability. `ALTER ... ADD FIELD [F] T DEFAULT v` rewrites every existing row with `v` (the field becomes NOT NULL) but does NOT register an ongoing default. `ALTER ... ALTER FIELD [F] SET DEFAULT v` is schema-only — existing rows untouched, the default fills the field where an inserted document and every object, or list of objects, a later write assigns ([defaults](dml.md#defaults)) leaves it out, and where a write assigns it `DEFAULT`, but not where a REPLACE leaves out a top-level field. `DEFAULT NOW` works on both (DATETIME fields only): in ADD FIELD it backfills the ALTER-time timestamp; in SET DEFAULT it re-resolves per INSERT. Want both behaviours? Run both statements.

<a id="pitfall-set-not-null-validates-every-retained-row"></a>

### `SET NOT NULL` validates every retained row

`ALTER ... ALTER FIELD [F] SET NOT NULL` checks every retained row, expired temporal versions included, before it publishes. A row without a value for `[F]` rejects the statement with `DOCUMENT_PAYLOAD_INVALID` (`Invalid document payload for collection '[App].[N]': NOT NULL field row.A has no value.`), which rolls the migration back. Fill the missing values first, for example with `UPDATE … SET ([F]: …) WHERE [F] IS NULL`. Once the field is NOT NULL, an explicit `NULL` is always rejected and an INSERT that omits `[F]` reports `NOT_NULL_VIOLATION` unless `[F]` has a default. `DROP NOT NULL` is always safe; the PK can't be made nullable. Scope to a subtype with `ALTER FIELD SUBTYPE [value] [F] SET NOT NULL`.

<a id="pitfall-implicit-auto-increment-vs-explicit-pk"></a>

### Implicit auto-increment vs explicit PK

If a collection's PK is `INT32 PRIMARY KEY AUTOINCREMENT`, omit the PK in the INSERT literal — the engine assigns. An explicit `[Id]: 0` also requests assignment; use `FORCE ID` to store literal zero or any other migration identity. The forced reservation never moves backward and is not rolled back while the process remains open, so a failed or rolled-back import can leave a gap. An unclean restart may forget such a reservation when neither its row nor a saved high-water mark made it durable.

<a id="pitfall-empty-string-and-bytes-array-references-are-not-null"></a>

### Empty String and Bytes array references are not null

Ordinary `ARRAY STRING REFERENCES ...` / `ARRAY BYTES REFERENCES ...` and String/Bytes reference fields inside `ARRAY OBJECT` are supported. Every `''` element is a present identifier requiring an empty-key target. A nullable object leaf set to `NULL` skips that relationship; an empty or null packed array has no relationships. String reference matching is invariant simple-lowercase and Bytes matching is exact, including Cascade/SetNull actions. Pins support the same key types and bind the exact revision; use `PIN('', revision)` for an empty identifier.

<a id="pitfall-string-and-bytes-primary-keys-stay-on-the-home-page"></a>

### String and Bytes primary keys stay on the home page

`STRING` primary-key equality is invariant-lowercase and preserves the inserted spelling; `BYTES` equality is exact. `''` is a valid empty identity for either type, not `NULL` (for Bytes it is empty Base64). Unlike a large document body, the variable-width identity never spills to overflow pages. An oversized key is therefore rejected before mutation even when the body itself could spill.

<a id="pitfall-integer-arithmetic-uses-int64-and-explicit-recovery"></a>

### Integer arithmetic uses INT64 and explicit recovery

Integer arithmetic produces INT64, so INT32 values of 2000000000 added together produce 4000000000. Overflow beyond INT64 raises an error. Declared computed slots and explicit casts also raise if the result cannot fit their target type. Use TRY or TRY_CAST to recover representability failures as NULL. See [scalar expressions](queries.md#select-projection) for expression semantics.

<a id="pitfall-alter-add-field-adds-array-of-scalar-array-object-and-fks"></a>

### `ALTER ADD FIELD` adds ARRAY-of-scalar, ARRAY OBJECT, and FKs

`ADD FIELD` accepts `ARRAY <scalar>` (incl. array FKs), `ARRAY OBJECT (...)` with a composite element schema (including polymorphic `WITH SUBTYPE` and nested element-level `REFERENCES`), and `[PINNED] REFERENCES`. Caveats vs `CREATE`:

- **Nested arrays retain a schema at each level.** `ARRAY ARRAY INT32` and deeper arrays of scalars or objects accept matching nested literals. Empty inner arrays remain present values. NULL elements remain invalid at every level; use a nullable field inside an object when absence must be represented. Use one `[]` per consecutive array level. `LENGTH([Matrix])` counts outer arrays; `LENGTH([Matrix][][] > 0)` counts matching scalar leaves. `MATCH FULL` foreign keys require all object leaves to reference a value or all to be NULL, across inner arrays and in either part of a polymorphic collection. Only leaves declaring that same target/action/pin/match policy contribute; siblings with a different relationship or an unconstrained field do not.
- **FK targets must already exist** — `ALTER` runs against the live catalog, so there's no forward-reference tolerance (unlike `CREATE` inside a migration, whose FK targets are checked at `COMMIT MIGRATION`). A `DEFAULT` backfills every existing row and is validated to resolve in the target; a pinned FK's `DEFAULT` must be a `PIN(id, revision)` literal, a non-pinned FK's a plain id, and an `ARRAY` field's an array literal; without a `DEFAULT` existing rows read NULL.

<a id="pitfall-fk-targets-are-validated-at-commit-migration-not-at-create"></a>

### FK targets are validated at `COMMIT MIGRATION`, not at `CREATE`

A `CREATE COLLECTION` may reference a collection created later in the same migration (forward references — needed for self / cyclic FKs). The targets, PK-type match, and pinned-temporal rule are checked at `COMMIT MIGRATION`; a bad target aborts and rolls back the whole migration (dropping the collections it created) with a collection+field-qualified error.

<a id="pitfall-drop-field-cascades-to-indexes--fks"></a>

### `DROP FIELD` cascades to indexes / FKs

When you drop a field used by an index key or INCLUDE path, or drop its parent Object, the dependent index is removed in the same migration. This includes UNIQUE indexes. Remove an INCLUDE dependency with ALTER INDEX before dropping the field if its key constraint should remain. FK source registrations are also removed. Abort restores the registrations; committed changes survive reopen.

<a id="pitfall-fk-references--on-remove-restrict-blocks-delete"></a>

### FK `REFERENCES ... ON REMOVE Restrict` blocks DELETE

The deleted parent stays alive — the engine throws `ForeignKeyViolationException`. Either `Cascade` (deletes referencing rows) or `SetNull` (only on nullable FK fields) to allow the delete.

<a id="pitfall-a-reference-and-a-concurrent-removal-cannot-both-commit"></a>

### A reference and a concurrent removal of its target cannot both commit

A write that references a target and a statement that deletes, ends, or cancels that target, running in transactions that cannot see each other, never both commit: whichever claims the target second fails with the transient `TX_CONFLICT_FWW`, at that statement or at `COMMIT`. Retry the whole transaction; the retry sees the other's outcome and then succeeds or raises the ordinary foreign-key error. With `EngineOptions.SnapshotIsolation` the same applies to a transaction whose snapshot is older than the other's commit. This holds under the default snapshot conflict mode, which otherwise does not prevent write skew.

<a id="pitfall-pinned-fk-requires-a-temporal-target"></a>

### Pinned FK requires a temporal target

Only collections declared `(Temporal: TRUE)` can be referenced by `PINNED REFERENCES`. Standard rows can advance their revision, but standard collections do not retain the addressable historical bodies an exact `(id, revision)` reference requires.

<a id="pitfall-scalar-pinned-expressions-use-the-id-half"></a>

### Scalar pinned expressions use the id half

A scalar pin is stored as `[id][revision:8 LE]`, but comparison, ordering, grouping, joins, and window `PARTITION BY` evaluate it as the declared id type. Rows pinned to different revisions of the same id therefore compare and partition together. The revision tail is exact storage framing, not trailing scalar value bytes.

<a id="pitfall-pinned-array-fk-every-element-needs-an-explicit-pin"></a>

### Pinned-array FK: every element needs an explicit `PIN(...)`

`ARRAY <type> PINNED REFERENCES` requires `[PIN(id, rev), PIN(id, rev), …]` per element. Bare ids (`['<guid>']`) are rejected — same rule as scalar pinned fields. Fixed keys use `[id][revision:8 LE]`; String and Bytes keys use `[id byte length:4 LE][id bytes][revision:8 LE]`, the same frame as a scalar pin. Empty identifiers remain present. `Restrict` blocks `CANCEL VERSION AT` when *any* element binds the cancelled revision; `SetNull` drops every matching element from the array. A pinned-FK array can be array-indexed for any supported identifier type; the physical index key is the id alone, so membership is revision-agnostic. `MATCH ([Id] = value AND [Revision] = revision)` requires both values on the same element. `LENGTH([Pins][].[Revision] = revision)` counts only matching revisions.

<a id="pitfall-compound-array-payloads-and-schema-changes"></a>

### Compound-array payloads and schema changes

A scalar compound-array key may also be included because other component postings do not implicitly carry its value. CREATE and ALTER INCLUDE share this rule; array traversal paths themselves are not scalar payloads. INCLUDE paths cannot repeat, and scalar B-tree keys remain implicitly covered.

Subtype changes preserve the full ordered key list. A shared path retains its name; its last declaration follows a scoped rename. Removing any component's last declaration retires the whole index. Rollback restores the complete original definition, identity, and generation. Independent component matches never replace an authored same-element MATCH check.

<a id="pitfall-computed-columns-cannot-be-pk--autoincrement--fk--not-null"></a>

### Computed columns cannot be PK / AUTOINCREMENT / FK / NOT NULL

Virtual computed columns (`[F] TYPE AS (<expr>)`) derive their value from an expression on every read, so they have no identity, no auto-increment counter to write into, no concrete value to anchor a foreign key to. Their schema metadata is nullable, but expressions such as COALESCE, IFNULL, and CASE can return non-NULL values from NULL inputs. The parser rejects:

- `AS` + `PRIMARY KEY` — computed columns can't identify a row.
- `AS` + `AUTOINCREMENT` — no slot to write the counter into.
- `AS` + `REFERENCES` (any form, including `PINNED`) — value moves with the source fields; FK target would chase a moving reference.
- `AS` + `NOT NULL` — computed fields use conservative nullable metadata; the declaration rule does not infer expression-specific non-null guarantees. ALTER COLLECTION / ALTER TYPE SET NOT NULL rejects computed fields too.
- `AS` + `DEFAULT` (ALTER ADD path) — pick one or the other.

`ALTER FIELD … SET COMPUTED` refuses to make such a field computed for the same reasons, and a field with a `DEFAULT` or `NOT NULL` until those are dropped ([Change a computed expression or mode](#change-a-computed-mode)).

The computed field itself may live at the collection root, in a nested Object, in an Array Object element, in a field- or collection-level subtype extension, or in a named type used through composition or `EXTENDS`. Its expression binds to the containing object. Collection-level base and active-subtype fields share a dependency graph. A field absent from the active subtype has typed NULL when another subtype owns its declaration; it never reads another row or subtype's data. Unknown fields, incompatible result types, and cycles in any active subtype reject before publication. Stored consumers require stored computed dependencies. Nested objects and array elements retain their containing-object scopes; nested subtype expressions may read their base plus their own extension. A computed field can reference another computed field in its visible scope, including nested computed values. Evaluation follows the dependency graph, not declaration order. Self-references and cycles are rejected at DDL and metadata-validation boundaries.

<a id="pitfall-structured-computed-results-preserve-the-complete-field-shape"></a>

### Structured computed results preserve the complete field shape

Computed results may be scalar, `OBJECT`, `ARRAY`, or `VECTOR(n)`. Structured results must preserve the declared shape in every value-producing arm: field order, names, child types/nullability, vector width, named-type and enum identity, subtype layouts, and nested computed definitions. Field references, `CASE`, `COALESCE`/`IFNULL`, `NULLIF`, and vector literals retain those shapes. Incompatible arms reject at definition time. Polymorphic container paths use the active subtype; a missing extension value is NULL, and candidates with incompatible shapes cannot bind to one computed result. Reading a child of a virtual computed container evaluates the parent before filtering or projection.

Computed declarations use the canonical built-in type and shape contracts even when an embedded host replaces a scalar function. Virtual scalar results must still convert to the declared column type. A replacement for COALESCE, IFNULL, or NULLIF cannot promise its original structured argument shape; an affected virtual structured field fails safely without quarantining healthy schema metadata. Restoring the original implementation permits subsequent reads. STORED evaluation continues to use the original immutable implementations.

<a id="pitfall-virtual-computed-fields-require-read-time-evaluation-stored-fields-can-use-indexes"></a>

### Virtual computed fields require read-time evaluation; STORED fields can use indexes

The engine has no index over a virtual computed value. Predicates against a computed column trigger a full materialise + per-row inject + post-row filter path — correct results, but every row goes through the executor. For selective queries on derived values, declare the computed field `STORED` and create a secondary index. Stored expressions require immutable built-in functions and ordinary/stored dependencies; clock/custom/enum-lookup functions and virtual computed dependencies are rejected. `CREATE INDEX` rejects a virtual computed column both as a key and in `INCLUDE`: it has no stored slot from which the physical tree could build an authoritative key or covering payload. Single-source queries without `LAST PER` can filter stored fields before virtual derivation and retain primary/secondary index access. AND permits independent stored conjuncts; OR requires both complete branches to be stored predicates. Virtual-field predicates still run after derivation. Queries that don't touch any computed field stay on the engine- pushdown fast path; the gate only fires when WHERE / ORDER BY / GROUP BY / SELECT / JOIN-ON actually references a computed column (or `SELECT *` / bare `FROM [c]` implicitly does). Aggregate arguments, aggregate-local FILTER predicates, and aggregate ORDER BY keys count as references too. `SELECT * EXCLUDE` / `REPLACE` still runs its projection and schema validation after computed values are injected. Source-alias-qualified computed paths trigger the same injection path.

Correlated EXISTS / scalar predicates are valid beside a computed-column reference. Equality shapes that qualify for decorrelation keep the set-based SEMI / ANTI or aggregate-CTE path. Other shapes evaluate per outer row only after computed slots are injected, then flow through grouping, total count, projection, and paging. With JOIN, residual correlation runs on the full merged row and resolves both joined aliases. Parameter-backed `SKIP` / `LIMIT` are real paging clauses and never take the trivial computed streaming pass-through.

<a id="pitfall-join-enriches-only-computed-bearing-sides-that-consumers-observe"></a>

### JOIN enriches only computed-bearing sides that consumers observe

A `JOIN` whose ON predicate (or post-join filter) references a computed column on the outer collection routes through a materialise-into-VirtualCteCollection path: the entire outer scan pays the per-row injection cost before joining. A physical computed-bearing inner uses the same statement-owned, admission-accounted, spill-bounded replay only when ON keys, residual filters, ordering/grouping/aggregation, or projection can observe one of its computed slots. When no consumer observes those slots, the inner keeps its normal physical/index path. An explicit `WITH [x] AS (FROM [ComputedCollection]) ... JOIN [x] ...` boundary remains equivalent, but is optional.

Window arguments, PARTITION BY, window-local ORDER BY, and window FILTER all participate in computed-field dependency discovery. A computed reference hidden inside `OVER (...)` is not a raw NULL slot. Field-to-field JOIN residuals inspect both operands, so putting the computed outer path on the right-hand side does not bypass injection.

<a id="pitfall-computed-columns-reject-clearly-overflowing-numeric-assignments"></a>

### Computed columns reject clearly-overflowing numeric assignments

DDL-time validation runs an abstract-interpretation pass on the parsed expression and rejects two narrowing shapes:

- **Source-fits**: an `AS` expression that references a field whose declared numeric type is strictly wider than the declared output type (same family). `[Y] INT8 AS ([Int64Field])` rejects with a message naming the wider source field. Add an explicit `CAST([Int64Field] AS INT8)` to acknowledge the runtime narrowing.
- **Literal-bounds**: any value-producing subtree whose precise integer `[min, max]` range is entirely outside the declared integer type's domain. `[Y] INT8 AS (1000)` rejects; `[Y] INT8 AS (CASE WHEN ... THEN 1000 ELSE -1 END)` rejects on the THEN branch even though the union range overlaps INT8.

Safe arithmetic that overshoots by one at the type boundary (`[Plus1] INT32 AS ([X] + 1)` where `[X]` is INT32) accepts — the source-fits rule green-lights it before the bounds rule runs. Cross-family numerics (int ↔ decimal / float) stay permissive because runtime `CoerceToSchemaType` narrows safely or throws.

<a id="pitfall-check-constraints-cannot-use-unknown-fields-parameters-or-virtual-computed-columns"></a>

### CHECK constraints cannot use unknown fields, parameters, or virtual computed columns

CHECK constraints evaluate at write time against the row binary BEFORE the read-seam injection populates computed slots. A CHECK that references a virtual computed column would always read NULL and silently fail or pass against user intent. The DDL parser rejects this with a message pointing at the workaround: inline the source expression. (`WITH CHECK ([Total] >= 0)` against a computed `[Total]` becomes `WITH CHECK (([Price] * [Qty]) >= 0)`.) Unknown field references and query parameters are rejected for the same reason: a persisted constraint must be fully bound to its schema and evaluable without per-call state. Stored computed columns are materialized before CHECK and may be referenced. Reopened invalid metadata fails with `CHECK_EXPRESSION_INVALID`.

An executor may outlive programmatic collection/type registration. Invalid CHECK or computed metadata published after executor initialization is therefore validated again at first use. Named-type use/ALTER, object-relative CHECK evaluation, and computed queries (including empty collections and subtype extensions) raise `CHECK_EXPRESSION_INVALID` or `COMPUTED_EXPRESSION_INVALID` and quarantine the catalog domain; do not retry them as ordinary validation errors.

<a id="pitfall-every-not-null-field-needs-a-value-at-every-nesting-level"></a>

### Every NOT NULL field needs a value, at every nesting level

An INSERT that omits a NOT NULL field reports `NOT_NULL_VIOLATION` unless a default or a generator supplies it: `Field 'B' is NOT NULL and cannot be omitted on '[App].[N]'.` The same holds inside every `Object` and `ARRAY OBJECT` element the INSERT supplies, and every one an UPDATE, REPLACE or `ON CONFLICT DO UPDATE` assigns — `Field 'E' is NOT NULL and cannot be omitted on '[App].[N].D'.` — while a sub-field's own default fills it when its parent object is present. A nullable parent that the INSERT omits is never synthesized to host a child default, so its required children are not checked. Generated values are an auto-increment or `SEQUENTIAL` primary key and computed fields. On polymorphic elements (`SUBTYPE [x] (...)`) the required fields are base AND extension fields; only the discriminator is exempt (the `SUBTYPE` clause supplies it). `SUBTYPE [x] ()` is only valid when every NOT NULL field of the element schema besides the discriminator has a default.

<a id="pitfall-explain-validates-projection-schemas-without-evaluating-query-functions"></a>

### EXPLAIN validates projection schemas without evaluating query functions

Missing nested wildcard targets, duplicate output names, incompatible set or recursive column layouts, and invalid NEAREST vectors are planning errors even when the source is empty. Derived queries and scalar subqueries are schema-bound without executing their expressions. During normal SELECT execution, discovering a scalar subquery's result type likewise makes no extra function calls: an empty outer query does not evaluate the inner merely to determine its columns. Static EXPLAIN cannot prove data-dependent cardinality or expression success; `EXPLAIN ANALYZE` executes and drains the statement to measure actual behavior.

Computed plan notes include nested and subtype fields. When virtual values require enrichment before filtering, the plan reports the storage access that actually runs, including any earlier stored-field seek. Stored computed values can still use ordinary indexes. Scalar residual filters preserve native conjuncts in the reported source access; their source may require full-row enrichment even when the final projection selects only stored fields.

<a id="pitfall-primary-key-must-name-a-field-present-in-the-projection"></a>

### `PRIMARY KEY` must name a field present in the projection

```mql
-- Inside a migration: [Id] is not in the SELECT.
CREATE MATERIALIZED VIEW [App].[X] WITH (REFRESH = MANUAL, PRIMARY KEY = [Id])
AS FROM [App].[Y] SELECT [Customer]   -- error
```

The PK clause is mandatory; auto-derivation from the query shape is intentionally not supported.

<a id="pitfall-set-computed-preserves-storage-mode-and-validates-all-dependants"></a>

### SET COMPUTED preserves storage mode and validates all dependants

On a computed field, `ALTER COLLECTION ... ALTER FIELD [path] SET COMPUTED (<expr>)` replaces the expression while preserving its type, nullability, and virtual or STORED mode; only `STORED` after the expression changes the mode, from virtual to stored, and a stored field becomes virtual with `DROP STORED`. On a plain field the statement makes the field computed, virtual unless `STORED` follows, and clears or recomputes every retained value ([Change a computed expression or mode](#change-a-computed-mode)). Dotted OBJECT paths, ARRAY OBJECT paths with `[]`, and collection `SUBTYPE [value]` scopes are supported. The complete dependency graph and result shape are validated before publication. Virtual changes retain physical rows; STORED changes recompute current and historical rows and rebuild existing indexes. A failed expression, CHECK, or rewrite rolls back the schema, data, and indexes.

`ALTER TYPE [Ns].[Type] ALTER FIELD [Field] SET COMPUTED (<expr>)` applies the same change atomically to inherited and composed uses, including repeated object, array, and subtype occurrences. Alter an inherited field on its declaring type.

<a id="pitfall-stored-computed-conflict-keys-are-derived-before-upsert-probes"></a>

### Stored computed conflict keys are derived before UPSERT probes

Stored computed fields can be UPSERT conflict targets when backed by a matching UNIQUE index. The probe uses the fully materialized proposed key even though the INSERT literal cannot assign that computed field. NULL key parts retain ordinary UNIQUE semantics and take the insert path. This applies to DO NOTHING and DO UPDATE.

SET expressions evaluate referenced virtual computed fields against the current row at each mutation retry; EXCLUDED reads the independent proposed image. Derived read values stay in a separate evaluation buffer, so unassigned virtual slots remain NULL in the stored mutation image.

<a id="pitfall-engine-slow-query-defaults-and-session-overrides-have-separate-scope"></a>

### Engine slow-query defaults and session overrides have separate scope

`ALTER ENGINE SET SLOWQUERY (Threshold = '100ms', RingCapacity = 1024)` changes and persists the database engine's shared defaults. It requires ServerAdmin and no caller transaction or migration. `SET slow_query_threshold` changes only the connection; an explicit override continues to win after a global change. Use `SET slow_query_threshold = DEFAULT` to inherit again. Global `Threshold = NONE` keeps shared history and session-selected events. A session override alone cannot allocate history, but any successful global configuration enables it.

<a id="pitfall-match-full-on-added-foreign-key-fields"></a>

### MATCH FULL on added foreign-key fields

ALTER ADD FIELD preserves MATCH FULL for existing array-object collections. A DEFAULT makes the added field NOT NULL; use ALTER FIELD ... DROP NOT NULL when nullable all-or-none references are intended. Existing NULL occurrences satisfy MATCH FULL, and non-null defaults must resolve before publication.

<a id="pitfall-composite-defaults-are-schema-bound-literals"></a>

### Composite defaults are schema-bound literals

Object/array SET DEFAULT supplies an omitted parent field; a child default alone does not synthesize a missing parent. Explicit NULL remains NULL. Nested default values must match field names, subtypes and scalar domains, and cannot contain parameters, field references or arbitrary expressions.

A metadata change that makes an existing parent default invalid is rejected and rolled back. For example, before dropping a required child's default, replace or remove any parent template that relies on it. Before SET NOT NULL, replace a parent template that explicitly supplies NULL for that child.
