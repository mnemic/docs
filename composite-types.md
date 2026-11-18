# Composite types

> **In this section** — `CREATE TYPE`, using a type as a field (composition), `EXTENDS` type and collection inheritance, renaming/moving types, `DROP TYPE`.

A **composite type** is a reusable, named set of fields — like an [enum](enums.md), but a structured field set instead of a value set. Define it once with `CREATE TYPE` and use it as the type of a field (composition) or as a base of another type or collection (`EXTENDS`).

> **Migration-only DDL.** `CREATE TYPE`, `ALTER TYPE`, and `DROP TYPE` are structural changes — they must appear between `BEGIN MIGRATION` and `COMMIT MIGRATION` (see [Migrations](migrations.md)); run outside one they are rejected with a "requires a migration context" error. Wrapping them in the migration is what makes `ALTER TYPE`'s multi-collection propagation atomic and lets `SCRIPT MIGRATION` replay the type into a fresh database. The examples below elide the wrapper for brevity.

## CREATE TYPE

```sql
CREATE TYPE [App].[Schedule] AS (
  [StartsAt] DATETIME NOT NULL,
  [EndsAt]   DATETIME,
  [Cron]     STRING
);

CREATE TYPE [App].[Money] AS (
  [Amount]   DECIMAL NOT NULL,
  [Currency] STRING NOT NULL
) WITH CHECK ([Amount] >= 0);
```

Type names require brackets in declarations, field types, and EXTENDS bases: `[Money]` at the top level or `[App].[Money]` in a namespace. Bare names and partially bracketed qualified names are rejected.

The field list uses the same grammar as `CREATE COLLECTION ... WITH SCHEMA (...)` — every per-field modifier applies (`NOT NULL`, `DEFAULT`, computed `AS (...)`, nested `OBJECT` / `ARRAY`, `REFERENCES` foreign keys) — and a type may carry table-level `WITH CHECK` constraints. A type is **PK-less**: it cannot declare `PRIMARY KEY`, `AUTOINCREMENT`, or `SEQUENTIAL` — the collection that uses the type owns its primary key.

One type accepts at most 256 table-level CHECK constraints, matching the persisted schema codec. Parsing and direct AST validation enforce the limit before reading or traversing another expression. An `EXTENDS` collection's inherited and locally declared CHECKs share one 256-entry total.

Type fields can declare `DEFAULT <literal>` or `DEFAULT NOW`, with the same [semantics and restrictions as collection fields](ddl.md#defaults-at-creation). Defaults survive EXTENDS and field-of-type composition, including nested and subtype fields. For example, `CREATE TYPE [App].[Audit] AS ([At] DATETIME NOT NULL DEFAULT NOW)` assigns At when a consuming insert, or an object a write assigns, omits that field, not when the type is created.

A computed field declared by a named type works through both `EXTENDS` and field-of-type composition. Inherited fields are synthesized at the collection root; composed fields are synthesized inside their containing Object (including each `ARRAY [Type]` element and active polymorphic extension). Expressions bind relative to that containing object.

Type-level `WITH CHECK` constraints likewise work through both forms. `EXTENDS` folds them into the collection-level CHECK set. Composition copies the exact named predicates onto the Object field and evaluates them relative to that object on every INSERT, UPDATE, REPLACE, and UPSERT. NULL/UNKNOWN retains the ordinary CHECK-passes semantics.

Type names are namespace-scoped (`[Namespace].[Name]`) or bracketed top-level (`[Name]`), the same as enums.

## Composition — a field whose type is a composite type

```sql
CREATE COLLECTION [App].[Jobs] WITH SCHEMA (
  [Id]    INT32 PRIMARY KEY,
  [Sched] [App].[Schedule]            -- field-of-type
);

INSERT INTO [App].[Jobs]
  ([Id]: 1, [Sched]: ([StartsAt]: '2026-01-01T00:00:00Z', [Cron]: '0 0 * * *'));
```

`[Sched]` is an `OBJECT` whose nested fields are the type's. It reads, writes, and filters exactly like any nested object field — `WHERE [Sched].[Cron] = '0 0 * * *'`. Use `ARRAY [App].[Schedule]` for a list of the type.

The fields inside `[Sched]`, the elements of an `ARRAY` of the type and their CHECKs stay the type's: `ALTER TYPE` changes them in every use at once, and every `ALTER COLLECTION` action that reaches inside the field from one use — `ADD FIELD [Sched].[Note] STRING`, `DROP` or `RENAME FIELD [History][].[Cron]`, `ALTER FIELD [Sched].[Cron] SET DEFAULT`, `SET NOT NULL`, `SET TYPE`, `SET COMPUTED`, `SET REFERENCES`, `ALTER FIELD [Sched] ADD CHECK`, `MOVE FIELD`, in the base schema or a subtype — is `FEATURE_NOT_SUPPORTED`. The use's own field stays the collection's: it can be renamed, dropped, made `NOT NULL` or given a default, and `ALTER FIELD [Sched] SET TYPE OBJECT` (`ARRAY OBJECT` for a list) detaches it, after which its fields change with `ALTER COLLECTION`.

An existing collection composes a type the same way inside a migration: `ALTER COLLECTION [App].[Jobs] ADD FIELD [Next] [App].[Schedule]`, `ADD FIELD [History] ARRAY [App].[Schedule]`, or `ADD SUBTYPE [S] ([Sched] [App].[Schedule])`. Without a DEFAULT existing rows read NULL; a composite `DEFAULT (...)` backfills them.

`REFERENCES` declared by a type are enforced on every consumer from the statement that publishes the consumer's schema — `CREATE COLLECTION`, `ADD FIELD`, `ADD SUBTYPE`, or `ALTER TYPE` propagation — exactly as after a reopen. This covers nested types, `ARRAY [Type]` elements, subtype fields and `EXTENDS`. Dangling writes are rejected, `ON REMOVE` actions apply to target removal, and a composite DEFAULT backfill whose references do not resolve rejects the `ALTER`.

## EXTENDS — type and collection inheritance

```sql
CREATE TYPE [App].[Priced] EXTENDS [App].[Money] AS ([Sku] STRING);
CREATE TYPE [App].[TimedPrice] EXTENDS [App].[Priced], [App].[Schedule] AS ();
```

Types and collections accept an ordered comma-separated list of up to 32 direct bases. Types retain the same PK-less rules; `AS ()` is valid when the type adds no local fields. Inherited fields and CHECK constraints precede local declarations. A shared ancestor reached through several bases contributes each declaration once, in first-encounter order. Unrelated bases declaring the same field name or named CHECK conflict, even when their definitions are identical. Names are compared case-insensitively, and local declarations cannot shadow inherited ones: a type that declares a field its bases give it is refused with `DUPLICATE_NAME`, as a collection is. Inheritance is acyclic and limited to 64 type levels (a 65th is `LIMIT_EXCEEDED`), 1,024 resolved fields and 256 resolved CHECK constraints. `SET EXTENDS` that would give a type, or a collection or type inheriting it, a CHECK named like one it declares is refused with `DUPLICATE_NAME`, as `ADD CHECK` is.


```sql
CREATE COLLECTION [App].[Pricing] EXTENDS [App].[Money] WITH SCHEMA (
  [Id]    INT32 PRIMARY KEY AUTOINCREMENT,
  [Sku]   STRING NOT NULL
);
```

`EXTENDS [Type]` flattens the type's fields into the collection as **top-level** fields. `[App].[Pricing]` has `[Amount]`, `[Currency]` (from the type) plus `[Id]`, `[Sku]` — and the type's `WITH CHECK ([Amount] >= 0)` applies to the collection. Inherited fields are addressed as ordinary top-level fields:

```sql
INSERT INTO [App].[Pricing] ([Sku]: 'A1', [Amount]: 9.99, [Currency]: 'USD');
FROM [App].[Pricing] WHERE [Currency] = 'USD';
```

`EXTENDS` composes with `WITH SUBTYPE`: the resolved type fields become the collection's shared base fields.

An existing collection changes the types it extends inside a migration:

```sql
ALTER COLLECTION [App].[Pricing] SET EXTENDS [App].[Money], [App].[Schedule];
ALTER COLLECTION [App].[Pricing] DROP EXTENDS;
```

The collection changes as one extending a type does when [`ALTER TYPE … SET EXTENDS`](#change-the-types-a-type-extends) changes that type's bases: its fields become its discriminator, the new bases' fields and then its own, matched by name, so an inherited field the new bases still give keeps its values, one they no longer give leaves with its values, and one they newly give is filled with its declared default; the new bases' CHECKs must hold for every retained version, their references are enforced, and every rule of [`SET EXTENDS`](ddl.md#alter-collection) applies. From then on `ALTER TYPE` on a new base reaches the collection and `DROP TYPE` refuses while the collection extends it.

Inherited fields stay their declaring type's, with their nested fields, elements and CHECKs: `ALTER TYPE` changes them in every collection that extends the type, and every `ALTER COLLECTION` action on an inherited field or on a path inside one — `ADD FIELD` into an inherited object, `DROP`, `RENAME` or `MOVE FIELD [Currency]`, `ALTER FIELD [Currency]` with any `SET` or `DROP`, `ADD` or `DROP CHECK` on an inherited object, `SET PRIMARY KEY [Currency]` — is `FEATURE_NOT_SUPPORTED`, naming the type that declares the field ([ALTER COLLECTION](ddl.md#alter-collection)).

A collection's option list comes before `EXTENDS`, and both come before the `WITH` clauses ([CREATE COLLECTION](ddl.md#create-collection)); `EXTENDS [App].[Money] (Temporal: TRUE)` is a parse error:

```sql
CREATE COLLECTION [App].[PriceHistory] (Temporal: TRUE) EXTENDS [App].[Money], [App].[Schedule] WITH SCHEMA (
  [Id]    INT32 PRIMARY KEY,
  [Sku]   STRING NOT NULL
);
```

## ALTER TYPE — evolve a type, propagating to every user

```sql
ALTER TYPE [App].[Money] ADD FIELD [Note] STRING;
ALTER TYPE [App].[Money] DROP FIELD [Note];
ALTER TYPE [App].[Money] RENAME FIELD [Currency] TO [Ccy];
```

`ALTER TYPE` applies the change to the type **and** to every use in the same migration, including derived type definitions and types that compose the changed type. An `EXTENDS` user gets the change on its top-level fields; a field-of-type user gets it inside the nested object; an `ARRAY [Type]` user gets it inside every array element. `ADD FIELD` adds a nullable field without `DEFAULT`. Supplying `DEFAULT` makes the added field NOT NULL and backfills existing rows in current users; it is not retained as a default. The type definition, existing users, and future users retain the same nullability. Scalar backfills must fit the declared type or enum even when the type has no consumers; invalid literals reject before registry publication. CREATE TYPE and ADD validate foreign-key declarations before publication, including SetNull nullability, key-type compatibility with an existing target, and pinned-target temporality. Validation includes object and array subtype fields. Forward targets for types created or altered in the migration must exist and match at COMMIT MIGRATION, even without consumers. A failed check rolls back the migration; types dropped before commit no longer require their former targets. If propagation fails on any collection (e.g. a `NOT NULL` conflict), the whole migration aborts. Supported actions: `ADD FIELD`, `DROP FIELD`, `RENAME FIELD`, `MOVE FIELD`, `ADD CHECK`, `DROP CHECK`, `SET EXTENDS`, `DROP EXTENDS`, and `ALTER FIELD` with `SET TYPE`, `SET COMPUTED`, `SET STORED`, `DROP STORED`, `DROP COMPUTED`, `SET NOT NULL`, `DROP NOT NULL`, `SET DEFAULT`, `DROP DEFAULT`, `SET [PINNED] REFERENCES`, `DROP REFERENCES`, `ADD CHECK`, or `DROP CHECK`; `SET` or `DROP AUTOINCREMENT` and `SEQUENTIAL` are a `SYNTAX_ERROR`, since a type has no primary key to generate. Nested field paths address owned OBJECT fields and ARRAY OBJECT elements, for example `[Details].[Count]` or `[Items][].[Count]`. All field actions above use the containing object scope; RENAME takes a sibling leaf name (`TO [Saved]`). Paths cannot cross a composed named type: alter that declaring type instead. Inherited parent fields likewise belong to their declaring base type. Nested changes preserve parent defaults and propagate to existing and future consumers; abort restores the previous field graph and stored values.

Alter an inherited field on the type that declares it. A derived type cannot rename or drop it locally. Descendant name conflicts are checked before any consumer rewrite, and registry publication covers the complete changed closure. The same collision checks cover unused type definitions, including nested object and array subtype scopes; adding or renaming a base field cannot shadow an extension. The proposed type shape is validated before propagation, even when the type currently has no users. A DROP or RENAME of a field a CHECK or computed field reads is `DEPENDENT_OBJECTS_EXIST`, as in a collection, wherever the declaration sits: on the type or an object field in it at any depth, its array elements and a polymorphic field's subtypes included, on a type that extends it, on a type with a field composed of one of them, or on a collection that uses them. An ADD or any other change that leaves a computed expression or CHECK bound to an unknown field is `INVALID_DEFINITION`. Rejecting an incompatible edit rolls back the migration, including earlier writes, and leaves the engine available. It does not mark an otherwise valid stored CHECK as catalog corruption.

Propagation includes collection-level and field-level subtype extensions, including Object array elements. All uses in one collection share one physical rewrite for ADD, DROP or MOVE FIELD; RENAME records a layout translation without rewriting rows. Subtype envelopes retain their own field layouts, and sibling subtypes that do not use the changed type retain their values. Current and historical versions, indexes, and type definitions change atomically. `DROP TYPE` detects Object, Array, and subtype-extension uses recursively.

## Rename or move a type

```mql
ALTER TYPE [Core.Billing].[Tariff]
    RENAME TO [Core.Billing].[TariffDefinition];
ALTER TYPE [Core.Billing].[TariffDefinition]
    RENAME TO [Core.Pricing].[Tariff];
ALTER TYPE [Core.Pricing].[Tariff] RENAME TO [Tariff];
```

`RENAME TO` takes the complete destination identity. `[Tariff]` moves the type to the top level; it does not inherit the source namespace. A destination namespace needs no separate creation. Both names require brackets. The source must exist and the destination must be free of both types and enums. Reserved `[Mnemic]` and `[Mnemic.*]` namespaces are rejected. Identical names are a no-op; a casing-only change updates the stored spelling.

The stable type ID, fields, defaults, CHECKs, inheritance and nested consumers remain intact. The migration owner can use the new name immediately; other sessions see it after commit. Abort restores the original identity. The new name appears in `[Mnemic].[Types]`, spelled as MQL writes it (`[Billing].[Tariff]`, `[Tariff]` at the top level), and in generated schema declarations. External queries must use the new name; historical migration text is unchanged.

Commit validates the merged catalog by identity, definition generation and destination claims. Migration admission is exclusive; removed or incompatible dependencies and colliding destinations are rejected. See [migration concurrency](migrations.md) and [enum renames](enums.md#rename-or-move-an-enum) for persisted static expression bindings.

## DROP TYPE

```sql
DROP TYPE [App].[Schedule];
```

`DROP TYPE` is refused while a collection or another named type references it, through composition or an `EXTENDS` base. Drop or alter the referencing users first; a derived type protects its base even when no collection uses that derived type. Any other stored definition that names the type (a view, materialized view, trigger, CHECK constraint or computed column) also refuses it, with `DEPENDENT_OBJECTS_EXIST` naming that definition.

## Notes & limits

- A type's fields are **flattened into the using schema at definition time**; the using collection's on-disk schema is self-contained (no lookup needed to read it).
- A type may be nested inside another type or inherited through multiple bases.
- Introspect via `FROM [Mnemic].[Types]`: `Name` (spelled as MQL writes it, `[App].[Money]` or `[Money]`, as every type name a catalog lists is), `FieldCount`, `BaseTypes` (ordered direct base names as ARRAY STRING), `Fields` (the resolved shape, in the field entries `[Mnemic].[Collections]` uses, with each field's default, computed expression and reference), and `Checks` (the type's CHECK constraints, those it inherits first, each with `DeclaredBy`, the base type that declares it, which is null for the type's own).
- A collection's bases are listed in order in the `BaseTypes` of its `[Mnemic].[Collections]` row; its inherited fields come first among its `Fields` entries.

See also: [DDL](ddl.md) · [Enums](enums.md) · [Polymorphism](polymorphism.md).

`ALTER TYPE ... ALTER FIELD [Field] SET COMPUTED (<expr>)` replaces a computed expression throughout inherited and composed uses in one migration. Its type and virtual/STORED mode are preserved unless `STORED` follows the expression ([Change a field's computed mode](#change-a-fields-computed-mode)). Each affected collection uses one bounded rewrite for stored values, including historical rows; virtual changes update metadata. Expression and CHECK failures preserve the prior type and all users. Affected INCREMENTAL materialized views and their maintained descendants are rebuilt after all type consumers have changed, even when only computed values change. The migration owner sees the new values immediately; commit persists them together and abort restores the prior source and backing values. MANUAL snapshots retain their rows until REFRESH.

### Change field nullability

`ALTER TYPE [Ns].[Type] ALTER FIELD [Field] SET NOT NULL` and `DROP NOT NULL` propagate to every inherited and composed consumer in the migration, including array-element uses and composition inside subtype extensions. They change metadata without rewriting rows. SET NOT NULL first validates every retained non-deleted version and rejects missing required values; absent nullable parents remain allowed. DROP NOT NULL relaxes subsequent write validation. SET NOT NULL rejects ON REMOVE SetNull foreign keys, even when the type has no consumers. Alter an inherited field on its declaring base type.

### Change defaults on named-type fields

`ALTER TYPE [Ns].[Type] ALTER FIELD [Field] SET DEFAULT <literal-or-NOW>` and `DROP DEFAULT` propagate through derived types and all collection consumers in the same migration. Defaults use the same validation as collection defaults, including for a type with no consumers. SET DEFAULT changes only schema metadata: existing rows remain unchanged, the fields an inserted document and every object, or list of objects, a later write assigns ([defaults](dml.md#defaults)) leaves out receive the default, and explicit NULL remains NULL. Nested defaults require an existing parent object or array element. DROP DEFAULT removes the default without changing stored values. Both operations roll back with the migration and persist on reopen.

### Add and drop type CHECKs

```sql
ALTER TYPE [App].[Money] ADD CHECK [NonNegative] ([Amount] >= 0);
ALTER TYPE [App].[Money] DROP CHECK [NonNegative];
ALTER TYPE [App].[Money] DROP CHECK ([Amount] >= 0);              -- an unnamed CHECK, by its predicate
ALTER TYPE [App].[Rate] ALTER FIELD [Value] ADD CHECK ([Amount] > 0);  -- an OBJECT field's CHECK
```

`ADD CHECK` and `DROP CHECK` change the type's own CHECKs with the rules of the collection forms in [Add and drop CHECK constraints](ddl.md#add-and-drop-check-constraints): the predicate binds against the type's fields, a CHECK is dropped by its name or, unnamed, by a predicate that prints the same, and a CHECK the type inherits belongs to its declaring base type (`FEATURE_NOT_SUPPORTED`). With `ALTER FIELD [Object]` the CHECK belongs to an owned OBJECT or ARRAY OBJECT field of the type.

The change reaches every user in the same migration: each type that extends the type, each collection that extends it (after the collection's own CHECKs, so its own CHECKs keep their names and positions), and each field composed of it, directly or through a derived type. An added CHECK must hold for every retained version in every such collection before the migration publishes it, or the statement is refused with `CHECK_VIOLATION` and the migration rolls back. A name that a derived type or extending collection already uses is `DUPLICATE_NAME`, and an addition that would give one of them a 257th CHECK is `LIMIT_EXCEEDED`. A collection cannot drop or change the CHECKs it receives this way; see [Add and drop CHECK constraints](ddl.md#add-and-drop-check-constraints).

### Change a field's type

```sql
ALTER TYPE [App].[Money] ALTER FIELD [Amount] SET TYPE DECIMAL;
ALTER TYPE [App].[Order] ALTER FIELD [Ship] SET TYPE [App].[Address];   -- an object becomes a named type, field by field
ALTER TYPE [App].[Order] ALTER FIELD [Tags] SET TYPE ARRAY INT32;       -- every element converts
ALTER TYPE [App].[Order] ALTER FIELD [Ship] SET TYPE OBJECT;            -- detach the field from its named type
ALTER TYPE [App].[Route] ALTER FIELD [Stops] SET TYPE ARRAY OBJECT;     -- detach an array's elements from their named type
```

`ALTER FIELD ... SET TYPE` changes a field the type declares and converts its values in every use in the same migration, by the rules of the collection form in [Change a field's type](ddl.md#change-a-field-type): types that extend it, collections that extend it or a type derived from it, fields composed of it, and every element of an array of it. The field may become a scalar, an enum, a `VECTOR(n)`, an `ARRAY` or a named type, with every use's values converted element by element and field by field and the new type's `NOT NULL` fields, CHECKs and references proven against every use's retained versions; `SET TYPE OBJECT` detaches a field composed of a named type, and `SET TYPE ARRAY OBJECT` the elements of an array of one, in the type and in every use, keeping their fields and values without a rewrite. A field the type inherits belongs to its declaring base type (`FEATURE_NOT_SUPPORTED`). A field may not hold the type itself, a type extending it, or a type with such a field at any depth, since the type would contain itself; `SET TYPE` and `ADD FIELD` refuse it (`INVALID_DEFINITION`). A CHECK or computed field that reads the field refuses the change (`DEPENDENT_OBJECTS_EXIST`), whether it belongs to the type, a type derived from it, a type with a field composed of one of them, or a collection that uses them, and so does a partial, expiry or vector index of a using collection, or an index on a path the new type no longer declares. A stored value of any use without a conversion refuses it (`INVALID_VALUE`) and leaves the type and every use as they were.

### Change a field's reference

```sql
ALTER TYPE [App].[Line] ALTER FIELD [ProductId] SET REFERENCES [App].[Products] ON REMOVE SetNull;
ALTER TYPE [App].[Line] ALTER FIELD [ProductId] DROP REFERENCES;
```

`ALTER FIELD ... SET [PINNED] REFERENCES [Ns].[Target] [ON REMOVE ...] [MATCH FULL|SIMPLE]` and `DROP REFERENCES` change the reference of a field the type declares, or of a field of an object or array element in it, on the type and on every use in the same migration: types that extend or compose it, collections that extend it, and fields composed of it, in base schemas, arrays and subtype extensions alike. The rules and codes are those of the collection form in [Change a field's reference](ddl.md#change-a-reference): the target must exist now, even when the type has no users; the field's type must be the target's key type; `PINNED` names the layout the field stores; and a computed field or a discriminator takes no reference. No row is rewritten. Each using collection then proves the new reference against every version it retains, so one dangling value in any use refuses the statement with `FK_VIOLATION` and rolls the migration back, the type included. Setting the reference the field has, or dropping one it lacks, changes nothing. A field the type inherits belongs to its declaring base type (`FEATURE_NOT_SUPPORTED`).

### Change a field's computed mode

```sql
ALTER TYPE [App].[Line] ALTER FIELD [Total] SET COMPUTED ([Qty] * [Each]) STORED;
ALTER TYPE [App].[Line] ALTER FIELD [Total] DROP STORED;
ALTER TYPE [App].[Line] ALTER FIELD [Total] DROP COMPUTED;
```

`ALTER FIELD ... SET COMPUTED (<expr>) [STORED]`, `SET STORED`, `DROP STORED` and `DROP COMPUTED` move a field the type declares, or a field of an object or array element in it, between plain, virtual computed and stored computed, on the type and on every use in the same migration: types that extend or compose it, collections that extend it, and fields composed of it, in base schemas, arrays and subtype extensions alike. The rules, codes and stored values are those of the collection form in [Change a computed expression or mode](ddl.md#change-a-computed-mode), checked on the type even when it has no users and then on each use: a CHECK of the type, of an object in it, of a type extending or composing it or of a using collection that reads the field, and a stored computed field that depends on it, refuse making it virtual (`INVALID_DEFINITION`), as an index of a using collection that reads it does (`DEPENDENT_OBJECTS_EXIST`). Each use rewrites its retained versions when their stored values change, and a use that refuses rolls the migration back, the type included. A field that already has the mode asked for changes nothing; a field the type inherits belongs to its declaring base type (`FEATURE_NOT_SUPPORTED`).

### Move a field

```sql
ALTER TYPE [App].[Money] MOVE FIELD [Currency] FIRST;
ALTER TYPE [App].[Rate] MOVE FIELD [Value].[Amount] LAST;
```

`MOVE FIELD` moves a field the type declares among the fields of its own level, by the rules of the collection form in [Move a field](ddl.md#move-a-field), and every use follows in the same migration: types that extend it list its fields in the new order before their own, collections that extend it list them first in that order, and fields composed of it, and every element of an array of it, hold them in that order. Every collection the type reaches is rewritten, history included. The fields the type inherits keep their place before its own: moving one, or one of its own before them, is `FEATURE_NOT_SUPPORTED`; move it on its declaring type. A field inside a field composed of another named type moves on that type.

### Change the types a type extends

```sql
ALTER TYPE [App].[Price] SET EXTENDS [App].[Money], [App].[Audit];
ALTER TYPE [App].[Price] DROP EXTENDS;
```

`SET EXTENDS` replaces the type's direct bases, in order, and `DROP EXTENDS` leaves it extending none. The type keeps its own fields and CHECKs after the new inherited ones, and every user follows in the same migration: types that extend it, collections that extend it, and fields composed of it, directly or through a derived type. Inherited fields are matched by name. An inherited field the new bases still give keeps its values; when they declare it with another type, the values convert as [`SET TYPE`](ddl.md#change-a-field-type) converts them, and a field the old or the new declaration computes, or that only its fields declare, is `FEATURE_NOT_SUPPORTED`. A field they no longer give is dropped from every user with its values and the old bases' CHECKs, under the rules of `DROP FIELD`: a collection's own CHECK, computed field or partial index filter that reads it refuses the change (`DEPENDENT_OBJECTS_EXIST`), and a view or trigger that reads it refuses it with the error naming it (`UNKNOWN_FIELD`). A field they newly give is added with its declared default filled into existing documents, as `ADD FIELD ... DEFAULT` fills them; a required field without a default is refused like `SET NOT NULL` while documents lack it (`DOCUMENT_PAYLOAD_INVALID`). The new inherited CHECKs must hold for every retained version (`CHECK_VIOLATION`).

A base that is the type or extends it is `INVALID_DEFINITION`, an unknown base `UNKNOWN_OBJECT`, and a base that would give the type a field or CHECK name it declares itself `DUPLICATE_NAME`; the type's own CHECKs and computed fields must still bind against the new fields.

### Change a polymorphic field's subtypes

```sql
ALTER TYPE [App].[Part] ADD FIELD [Detail].SUBTYPE [gear].[Teeth] INT32 DEFAULT 12;
ALTER TYPE [App].[Part] RENAME FIELD [Detail].SUBTYPE [gear].[Teeth] TO [Cogs];
ALTER TYPE [App].[Part] ALTER FIELD [Detail].SUBTYPE [gear].[Cogs] SET TYPE INT64;
ALTER TYPE [App].[Part] ALTER FIELD [Detail] ADD SUBTYPE [spring] ([Rate] DECIMAL);
ALTER TYPE [App].[Part] ALTER FIELD [Detail] DROP SUBTYPE [spring];
```

A polymorphic field the type declares is reached as in a collection ([a path into a polymorphic field's subtype](ddl.md#field-subtypes)): a `SUBTYPE [value]` segment in a field path changes one subtype's fields, and `ALTER FIELD ... ADD SUBTYPE` and `DROP SUBTYPE` the field's subtypes, on the type and on every use in the same migration, with the rules and codes of the collection form. Each use's stored values of the subtype take an added field's default, a rename leaves them as they are, and `DROP SUBTYPE` is refused while any use's retained version holds a value of the subtype (`DEPENDENT_OBJECTS_EXIST`) or while a CHECK or computed field of the type, of a type composing it or of a use reads a field only the subtype declares (`INVALID_DEFINITION`).

## Detailed syntax and execution


<a id="syntax-create--drop-type--named-composite-types"></a>

### CREATE / ALTER / DROP TYPE — named composite types

A **composite type** is a reusable, named set of fields (an enum is a value set; a type is a field set). The `AS (...)` field list uses the same grammar as `WITH SCHEMA (...)` — all per-field modifiers + table-level `WITH CHECK` apply — but a type is **PK-less** (no `PRIMARY KEY` / `AUTOINCREMENT` / `SEQUENTIAL`). Migration-only DDL.

```sql
CREATE TYPE [App].[Money] AS (
  [Amount] DECIMAL NOT NULL,
  [Currency] STRING NOT NULL
) WITH CHECK ([Amount] >= 0);

-- Composition: a field whose type is the composite type (stored as an OBJECT).
CREATE COLLECTION [App].[Sales] WITH SCHEMA ([Id] INT32 PRIMARY KEY, [Price] [App].[Money]);
INSERT INTO [App].[Sales] ([Id]: 1, [Price]: ([Amount]: 9.99, [Currency]: 'USD'));
-- ... WHERE [Price].[Currency] = 'USD'   (addressed like any nested object)

-- Inheritance: EXTENDS flattens the type's fields to TOP-LEVEL collection fields.
CREATE COLLECTION [App].[Invoices] EXTENDS [App].[Money]
  WITH SCHEMA ([Id] INT32 PRIMARY KEY);
INSERT INTO [App].[Invoices] ([Id]: 1, [Amount]: 50, [Currency]: 'EUR');  -- inherited fields top-level

-- ALTER TYPE propagates the change to the type AND every collection using it (same migration).
ALTER TYPE [App].[Money] ADD FIELD [Note] STRING;        -- adds a nullable field everywhere
ALTER TYPE [App].[Money] DROP FIELD [Note];
ALTER TYPE [App].[Money] RENAME FIELD [Currency] TO [Ccy];
ALTER TYPE [App].[Money] MOVE FIELD [Currency] FIRST;             -- reorders the type and every user, rewriting their rows
ALTER TYPE [App].[Money] ALTER FIELD [Amount] SET TYPE DECIMAL;   -- converts every user's stored values as CAST
ALTER TYPE [App].[Order] ALTER FIELD [Ship] SET TYPE OBJECT;       -- detaches the field from its named type in the type and every user
ALTER TYPE [App].[Line] ALTER FIELD [ProductId] SET REFERENCES [App].[Products];  -- every user's retained versions must resolve
ALTER TYPE [App].[Line] ALTER FIELD [ProductId] DROP REFERENCES;
ALTER TYPE [App].[Line] ALTER FIELD [Total] SET COMPUTED ([Qty] * [Each]) STORED;  -- a plain field becomes computed in every user, its values recomputed
ALTER TYPE [App].[Line] ALTER FIELD [Total] DROP COMPUTED;      -- every user keeps the field's values as a plain field
ALTER TYPE [App].[Money] ADD CHECK [NonNegative] ([Amount] >= 0);  -- validated against every user's stored versions
ALTER TYPE [App].[Money] DROP CHECK [NonNegative];
ALTER TYPE [App].[Price] SET EXTENDS [App].[Money], [App].[Audit];   -- replaces the bases; users gain and lose inherited fields
ALTER TYPE [App].[Price] DROP EXTENDS;
ALTER TYPE [App].[Part] ALTER FIELD [Detail].SUBTYPE [gear].[Cogs] SET NOT NULL;     -- a field of one subtype of a polymorphic field, in every use
ALTER TYPE [App].[Part] ALTER FIELD [Detail] ADD SUBTYPE [spring] ([Rate] DECIMAL); -- DROP SUBTYPE is refused while a use holds a value of it
ALTER COLLECTION [App].[Invoices] SET EXTENDS [App].[Price];          -- a collection replaces its own bases the same way
ALTER COLLECTION [App].[Invoices] DROP EXTENDS;

DROP TYPE [App].[Money];   -- errors while any field-of-type or EXTENDS base still references it
```

Type names require the bracketed top-level (`[Money]`) or namespace-qualified (`[App].[Money]`) form. The same rule applies to declarations and references.

Introspect via `FROM [Mnemic].[Types]`. Types and collections support ordered multiple bases: `CREATE TYPE [App].[B] EXTENDS [App].[A], [App].[C] AS ()`. Shared ancestors contribute fields and CHECKs once; unrelated same-name fields or named CHECKs conflict, and local fields cannot shadow inherited fields. Limits: 32 direct bases, 64 type levels, 1,024 resolved fields, 256 resolved CHECKs. `ALTER TYPE` updates derived and composing definitions atomically with their collection users; alter inherited fields at their declaring type. `ALTER TYPE … SET EXTENDS` / `DROP EXTENDS` replaces a type's bases and `ALTER COLLECTION … SET EXTENDS` / `DROP EXTENDS` a collection's, matching inherited fields by name. `DROP TYPE` also refuses references from other named types without collection users. A type may be *nested* in another type as a field-of-type. ALTER TYPE field actions accept owned nested OBJECT paths and ARRAY OBJECT paths such as `[Details].[Count]` and `[Items][].[Count]`. RENAME uses a sibling leaf (`TO [Saved]`), preserving the parent path. A path cannot cross another named type or modify an inherited parent: change its declaring type. Nested changes preserve parent defaults, computed bindings, and rollback across current and future consumers. Nested base fields cannot shadow subtype extension names, even when the type has no consumers. `ALTER TYPE ADD FIELD` adds a nullable field without DEFAULT. With DEFAULT, it adds a NOT NULL field and backfills existing users, without persisting a default. The registry and existing/future users retain the same nullability. Scalar ADD backfills are validated against the declared type or enum even when no collection uses the type. Foreign-key CREATE TYPE and ADD validation also checks SetNull nullability, existing target key types, and pinned-target temporality before publication, including object and array subtype fields. COMMIT MIGRATION checks forward targets for created/altered types, including unused types. Missing or incompatible targets abort; dropped types are skipped.

Type computed expressions and CHECK predicates are schema-bound at CREATE, ALTER, and executor startup: unknown fields, parameters, computed dependency cycles, and edits that invalidate existing bindings fail closed. Computed fields and type-level CHECKs work through both `EXTENDS` and field-of-type composition, including `ARRAY [Type]` and active subtype envelopes. A composed CHECK is evaluated relative to its containing object on INSERT, UPDATE, REPLACE, and UPSERT.

`ALTER TYPE` propagates through `EXTENDS`, Object, nested Object, Array, and collection-level or field-level subtype extensions. All uses in one collection share one ADD, DROP or MOVE FIELD rewrite. RENAME records a layout translation and preserves stored rows. The type registry, consumer schemas, historical rows, and indexes share the migration's atomic outcome. Unaffected subtype branches keep their values. `DROP TYPE` finds Object, Array, and subtype uses recursively.

`REMOVE VALUE` fails closed: if the in-use probe errors or the `REPLACE` document rewrite fails anywhere (e.g. a CHECK constraint on a referencing field rejects the replacement value), the ALTER errors and the enum definition is untouched. The rewrite handles top-level, nested-object at any depth (ancestors are reassembled bottom-up), plain-array, array-object, and polymorphic fields. Wherever a `[base_len][base][ext]` envelope is involved — a collection-level row, a polymorphic sub-document, or a polymorphic ARRAY OBJECT element — it patches only the half that declares the field (base-resident and extension-resident enums both work) and preserves the other half byte-identical. A post-rewrite probe must prove the old value is absent before the enum definition changes. Before the first probe, structural-write barriers close and drain every referencing collection; they remain closed through rewrite, re-probe, and the migration's enum-registry overlay publication or completed rollback. Returning from `ALTER ENUM` does not open them. The migration transaction alone bypasses them for replacement writes, so concurrent external DML cannot land after the proof and before publish; retry refused DML after the migration commits or aborts. Explicit transactions retain their gate admission until commit or rollback. If the same migration already wrote a referencing collection, `REMOVE VALUE` fails; commit that DML in an earlier migration and retry.

[`ALTER ENUM ... SET VALUE` and `SET TYPE`](enums.md#alter-enum) reach named types as well: each field of the enum in a type and in every use takes the new type, every use's stored values are converted with the collection's, and a default naming the revalued member, in the type and in its uses, names its new value.

<a id="syntax-named-type-field-nullability"></a>

### Named-type field nullability

ALTER TYPE accepts `ALTER FIELD [Field] SET NOT NULL` and `DROP NOT NULL` inside a migration. Both propagate atomically to inherited/composed consumers. Existing NULL rows remain readable; new writes obey the new metadata. SET NOT NULL rejects ON REMOVE SetNull foreign keys. Change inherited fields at their declaring base type.

<a id="syntax-change-defaults-on-named-type-fields"></a>

### Change defaults on named-type fields

`ALTER TYPE [Ns].[Type] ALTER FIELD [Field] SET DEFAULT <literal-or-NOW>` and `DROP DEFAULT` propagate through derived types and all collection consumers in the same migration. Defaults use the same validation as collection defaults, including for a type with no consumers. SET DEFAULT changes only schema metadata: existing rows remain unchanged, the fields an inserted document and every object, or list of objects, a later write assigns ([defaults](dml.md#defaults)) leaves out receive the default, and explicit NULL remains NULL. Nested defaults require an existing parent object or array element. DROP DEFAULT removes the default without changing stored values. Both operations roll back with the migration and persist on reopen.

## Usage pitfalls


<a id="pitfall-composite-types-create-type-are-pk-less-flattened-and-migration-scoped"></a>

### Composite types (`CREATE TYPE`) are PK-less, flattened, and migration-scoped

A `CREATE TYPE` body cannot declare `PRIMARY KEY` / `AUTOINCREMENT` / `SEQUENTIAL` — the using collection owns its PK. A type's fields are **flattened into the using schema at definition time**, two ways: as a field-of-type (`[F] [Ns].[Type]`, stored as an `OBJECT` — address it like a nested object, `[F].[Sub]`) or via `EXTENDS` (fields become **top-level** collection fields — address them bare). An `EXTENDS` collection field that collides with an inherited field name is rejected. `DROP TYPE` errors while any collection references the type. `ALTER TYPE ADD/DROP/RENAME FIELD` propagates to every using collection in one migration (`ADD FIELD` adds a nullable field without DEFAULT, or a NOT NULL field with DEFAULT). Introspect with `FROM [Mnemic].[Types]`. Types and collections support multiple `EXTENDS` bases. Shared ancestors are flattened once; unrelated same-name fields or named CHECKs conflict. Alter an inherited field at its declaring type; change a collection's bases with `ALTER COLLECTION … SET EXTENDS` / `DROP EXTENDS`. DROP also checks references from other named types, even when they have no collection users.

Computed expressions and CHECK predicates on a type are validated at CREATE, after every ALTER, and again at executor startup. An ALTER that drops or renames a referenced field is rejected even when the type has no users. Computed-bearing or checked types work through both `EXTENDS` and field-of-type composition. A composed CHECK is evaluated relative to that Object on every INSERT, UPDATE, REPLACE, and UPSERT; nested and Array-composed computed values are synthesized recursively.

`ALTER TYPE` propagates through EXTENDS, nested Objects, Arrays, and collection- or field-level subtype extensions. Multiple uses in one collection share one ADD/DROP rewrite; RENAME records a layout translation. Historical versions and unaffected subtype branches retain their values. All consumer schemas, indexes, and type definitions commit or roll back together. `DROP TYPE` detects Object, Array, and subtype uses recursively.

<a id="pitfall-type-nullability-changes-validate-retained-rows"></a>

### Type nullability changes validate retained rows

ALTER TYPE SET NOT NULL rejects missing required values in any retained version, including temporal history, like ALTER COLLECTION. It does not backfill values. All dependent consumers change together; a rejected change or abort preserves their previous schemas and type definition.

<a id="pitfall-nested-type-evolution-respects-declaring-ownership"></a>

### Nested type evolution respects declaring ownership

ALTER TYPE can address owned OBJECT and ARRAY OBJECT fields with dotted paths and `[]` traversal. RENAME takes only the new sibling name. A composed named type is a separate owner; change that type instead of descending through it. An inherited parent is changed through its declaring base type. Computed expressions bind to the containing object, including stored replacements. ADD and RENAME cannot create a name collision with a subtype extension, even in unused types. DROP and RENAME must preserve CHECK dependencies.

<a id="pitfall-named-type-add-defaults-preserve-nullability"></a>

### Named-type ADD defaults preserve nullability

ALTER TYPE ADD FIELD with DEFAULT makes the field NOT NULL in the type registry and in existing and future consumers. The literal backfills existing rows only; use SET DEFAULT separately for a default, or DROP NOT NULL to permit NULL. Unused types still validate scalar backfills: non-numeric text, fractions, overflow, and NOW on an integer field reject before the new field is published. A DEFAULT makes the field NOT NULL, so ON REMOVE SetNull is invalid even when no collection uses the type. Existing target key types must match the FK type. Pinned references require temporal targets; these checks also traverse subtype fields. CREATE TYPE validates the resolved field graph by the same rules, including inherited and composed fields, before publishing an unused declaration. A forward FK target must resolve at COMMIT MIGRATION for every created/altered type still present, including unused types. Missing or incompatible targets roll back the migration; dropping the type removes its target requirement.

<a id="pitfall-named-type-default-changes-do-not-backfill"></a>

### Named-type default changes do not backfill

ALTER TYPE ... ALTER FIELD ... SET DEFAULT applies to omitted fields on later INSERTs, including nested objects and array elements whose parent exists, to the fields an object or list of objects a later write assigns leaves out, and to a field a write assigns `DEFAULT` ([defaults](dml.md#defaults)). Explicit NULL remains NULL. DROP DEFAULT removes that behavior. Validation also applies to types with no consumers; migration abort restores every consumer and the type definition.
