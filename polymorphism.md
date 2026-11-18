# Polymorphism

> **In this section** — collection-level polymorphism (one collection, several subtypes), field-level polymorphism (polymorphic Object and ARRAY OBJECT fields), the `SUBTYPE` keyword for queries.

## Collection-level polymorphism

A polymorphic collection stores multiple subtypes sharing a base schema. Declare with `WITH SUBTYPE` clauses:

```sql
CREATE COLLECTION [Fleet].[Vehicles]
WITH SCHEMA ([Id] GUID PRIMARY KEY, [Make] STRING, [Year] INT32)
WITH SUBTYPE [car] ([Doors] INT32, [Sunroof] BOOL)
WITH SUBTYPE [truck] ([PayloadTons] DECIMAL, [Axles] INT32);
```

A collection created without subtypes becomes polymorphic when a migration names the subtype its documents take; every document it holds, history included, takes that subtype, and further subtypes follow as for any polymorphic collection ([make a collection polymorphic](ddl.md#make-a-collection-polymorphic)):

```sql
ALTER COLLECTION [Fleet].[Bikes] ADD SUBTYPE [bike] ([Gears] INT32 DEFAULT 1) FOR EXISTING DOCUMENTS;
ALTER COLLECTION [Fleet].[Bikes] ADD SUBTYPE [scooter] ([RangeKm] INT32);
```

`RENAME SUBTYPE [old] TO [new]` renames a subtype, rewriting the name every stored version holds; it is refused while a CHECK, computed field, partial index, view, materialized view or trigger names the subtype ([rename a subtype](ddl.md#rename-a-subtype)).

INSERT requires the `SUBTYPE` clause on polymorphic collections (omitting it is an error):

```sql
INSERT INTO [Fleet].[Vehicles] SUBTYPE [car] (
  [Id]: '...', [Make]: 'Toyota', [Year]: 2024, [Doors]: 4, [Sunroof]: TRUE
);
```

UPDATE and DELETE work transparently — base and extension fields are handled automatically:

```sql
-- UPDATE a base field (applies to any subtype)
UPDATE [Fleet].[Vehicles] SET ([Year]: 2025) WHERE [Id] = @id;

-- UPDATE an extension field (must match the document's subtype)
UPDATE [Fleet].[Vehicles] SET ([Doors]: 5) WHERE [Id] = @id AND SUBTYPE IS [car];

-- Expression SET values resolve fields across the whole document —
-- base and extension fields can reference each other
UPDATE [Fleet].[Vehicles] SET ([Year]: [Axles] + 2020) WHERE SUBTYPE IS [truck];

-- DELETE is uniform across subtypes
DELETE FROM [Fleet].[Vehicles] WHERE SUBTYPE IS [truck] AND [Year] < 2015;
```

### Querying by subtype

Subtype names are schema identifiers — like collection and field names they are written in square brackets and matched case-insensitively. In WHERE the `SUBTYPE` keyword forms a type test with `IS`, `IS NOT`, and `IN`; `@parameters` are accepted wherever a name is:

```sql
-- Filter by subtype
FROM [Fleet].[Vehicles] WHERE SUBTYPE IS [car];
FROM [Fleet].[Vehicles] WHERE SUBTYPE IS NOT [car];
FROM [Fleet].[Vehicles] WHERE SUBTYPE IN ([car], [truck]);
FROM [Fleet].[Vehicles] WHERE SUBTYPE IS @t;

-- Include subtype in results (alongside base-schema fields)
FROM [Fleet].[Vehicles] SELECT SUBTYPE, [Make], [Year];

-- Sort by subtype
FROM [Fleet].[Vehicles] ORDER BY SUBTYPE ASC;

-- Combine with other filters
FROM [Fleet].[Vehicles] WHERE SUBTYPE IS [car] AND [Year] > 2020;
```

A polymorphic collection's row (`FROM [Fleet].[Vehicles]`, `SELECT *`) is its base fields with `Meta` and a nullable `Envelope` that keeps each row's subtype fields; the result carries the subtype mapping, so a derived table or CTE that imports the row reads and sorts by `[Doors]` through it (`WITH [c] AS (FROM [Fleet].[Vehicles]) FROM [c] ORDER BY [Doors]`). A qualified or modified wildcard (`SELECT [v].*`, `SELECT * EXCLUDE ([Make])`) instead expands the declared data fields with every subtype's extension fields as nullable columns, which a modifier may name; `ORDER BY` may name an extension field the wildcard leaves out (`ORDER BY [Doors] SELECT * EXCLUDE ([Doors])`), as it may an excluded base field. An item that names no declared field is rejected with a diagnostic listing the row's columns, and static schema discovery publishes exactly the schema execution returns for each form. As a JOIN input the collection contributes its base fields, each extension field its subtypes declare alike as a nullable engine-managed column, and its envelope to the joined row, on either side of the join, whether or not the collection has computed fields (a virtual computed column holds its evaluated value) and whether its rows reach the join as stored, with the `Meta` a clause reads (`WHERE [v].[Meta].[Revision] > 0`), or through a derived table or CTE that imports its row (`WITH [c] AS (FROM [Fleet].[Vehicles]) FROM [c] [v] JOIN ...`); a row that keeps its envelope keeps its own columns, `Meta` among them, before the extension columns. The column holds the field's value under the field's full declaration, as the row's base columns hold theirs, differing only in being nullable and engine-managed, so a subtype field that is a foreign key keeps its target, removal action and match mode on that column, a pinned reference its pinned layout, a field with a default its default and a computed column its expression and whether it is stored; the declaration describes the value, and no result column enforces or writes it, and `[v].*` over it, as over a derived table, CTE or APPLY body that keeps its row, expands its base fields and each extension field its subtypes declare alike on every route; a field its subtypes declare differently is no column of the joined row, so the wildcard has no slot for it and a reference to it is rejected wherever one fixed output schema needs it, while a leaf they declare alike beneath it stays readable.

A SELECT item or `RETURNING` path that names the envelope (`FROM [Fleet].[Vehicles] SELECT [Id], [Envelope]`, `UPDATE ... RETURNING [Id], [Envelope]`) returns the column the row publishes: nullable virtual `Envelope` Bytes holding the row's envelope, every computed field of its subtype evaluated, as `SELECT *` returns it. Every route returns that column, an import, a set operation, a JOIN's nested or flat projection, a row a nested query filters and a lateral body included, as does every write and upsert arm. Such a result is not a polymorphic row and carries no subtype mapping: its envelope is an opaque Bytes column, which a relation importing it reads as one and reads no extension field through, and set-operation branches combine two collections' envelope columns as Bytes columns. Only the polymorphic row itself (`FROM`, `SELECT *`, and a set operation or import of them) carries the mapping beside the envelope.

The scan of a stored polymorphic collection evaluates its clauses over the stored row, which carries no envelope, so `[Envelope]` in a WHERE (over a single source, a join, a nested query, a lateral body or a write, which EXPLAIN of the write reports too), or named as a single source's ORDER BY key, plain GROUP BY or DISTINCT key, aggregate operand or aggregate FILTER operand, is rejected with `UNKNOWN_FIELD` as a field the collection does not declare (`FROM [Fleet].[Vehicles] WHERE [Envelope] IS NOT NULL` fails with `Query filter field 'Envelope' does not exist in the collection schema.`), whether or not the collection has computed fields. A clause over a completed projection reads the column the projection presents (`ORDER BY [E] SELECT [Envelope] AS [E]`), and a derived table or CTE importing the row reads it in every clause.

An unknown subtype name is an **error**, not an empty result — `WHERE SUBTYPE IS [carr]` fails with `Unknown subtype 'carr'. Known subtypes: car, truck.`, the one message every statement that names a subtype reports (INSERT SUBTYPE, a filter over the collection or a composed input, a partial index) (unknown *fields* in WHERE deliberately follow SQL NULL semantics, but a typo'd subtype name can never match anything, so it surfaces loudly). Parameter-bound names are validated the same way at execution time. CTE and joined inputs retain this validation in `WHERE` and aggregate `FILTER`, including when the source is empty. `SUBTYPE IS NULL` is rejected: every document in a polymorphic collection has a subtype, because INSERT requires the SUBTYPE clause.

`SELECT SUBTYPE` projects the name with the spelling **declared in the schema**, regardless of how the INSERT cased it — matching is case-insensitive, storage is canonical. The column is named `_type`, for `SELECT SUBTYPE` and for `GROUP BY SUBTYPE … SELECT SUBTYPE` alike, and a plain row (`FROM [Fleet].[Vehicles]`, `SELECT *`) carries the same `_type` column first. `SUBTYPE` takes no alias — `SELECT SUBTYPE AS [Kind]` is a parse error — so rename the column through its field name instead: `SELECT [_type] AS [Kind], [Make]`, or `GROUP BY SUBTYPE SELECT [_type] AS [Kind], COUNT(*) AS [N]`.

### Same extension name in more than one subtype

Sibling subtypes may reuse an extension-field path with different schemas at collection level or inside a polymorphic Object / ARRAY OBJECT. Row-wise `WHERE`, `MATCH`, `CHECK`, UPDATE-expression, window-aggregate `FILTER`, and correlated-outer-reference evaluation resolve both the value type and schema metadata (including enum identity) from each document or element's active subtype at every nesting depth. Array ANY/ALL, `IN`, `LENGTH(predicate)`, and correlated `MATCH` use that same per-element rule. `MATCH` considers every subtype declaration of the outer path, so a scalar declaration in one subtype cannot hide an array declaration in another. Bound predicate parameters follow the same rule: admission accepts a value compatible with at least one candidate declaration, then each row resolves its active declaration and treats an incompatible value as a nonmatch. Subtype declaration order never determines whether the parameter is accepted. Literal admission is identical. A scalar literal must convert to at least one active type, and every non-NULL member of an `IN` list must convert to at least one active type (members may target different subtypes). The rule also applies to values materialized from an `IN (FROM …)` subquery. A value accepted by no candidate is an explicit query error rather than a silent empty result. Each row then compares by its own declaration, with the literal read as a value of that declaration's type: a row whose declaration refuses the literal (a number against a DATEONLY sibling of an INT32 declaration) does not match, for every operator, on the native WHERE route and in a CHECK constraint alike, and admission never depends on the order the subtypes are declared in. A nested query's comparison with an enclosing row's value follows the same rule: each row compares the value by its own declaration, a row whose declaration has no order with the value does not match, and an ordering no declaration has with the value is a `TYPE_MISMATCH`.

A scalar expression in a row-wise predicate — a `WHERE` expression and a window-aggregate `FILTER` expression, such as a `CASE` condition, a function or arithmetic — reads a collection-level extension path whose subtypes declare different types by each row's own declaration too. With `[V] INT32` in one subtype and `[V] STRING` in another, `WHERE [V] + 1 = 6` selects the INT32 rows holding 5 and `WHERE UPPER([V]) = 'ABC'` the STRING rows holding `'abc'`; a row whose declaration does not type the expression (a STRING `[V]` under `+ 1`) does not match, so `NOT ([V] + 1 = 6)` selects it, as a native comparison its declaration refuses does. The statement and `EXPLAIN` admit the expression when one subtype's declarations type it, and refuse it with that typing's error when none does (`[V] - 'x'` is a `TYPE_MISMATCH` before rows are read). A failure on a row whose declaration types the expression stays the statement's error. A JOIN `ON` expression and a non-window aggregate `FILTER` or `HAVING` expression keep one declaration per path, by the JOIN, grouping and aggregate rule below.

An operator that must flatten the leaf into one fixed result schema or consume one comparison domain cannot guess which declaration owns the bytes. Computed or window projection, `ORDER BY`, `JOIN`, `GROUP BY`, aggregates, `NEAREST`, and DML `RETURNING` therefore require structurally identical schemas wherever a referenced extension path is repeated. An incompatible path is rejected before rows are decoded or a mutation begins. Plain dotted projection of a nested polymorphic field is the safe exception: it preserves the containing Object and its subtype envelope instead of flattening the ambiguous leaf.

The same rejection applies after a containing Object passes through a derived query or CTE, to computed JOIN outputs and wildcard replacements, and during `EXPLAIN`. Empty input and subtype declaration order do not change admission. Computed DISTINCT validates the input projection before deduplicating values. A row-wise window FILTER does not relax compatibility for other projected values.

A scalar or window expression may read a compatible leaf beneath differently shaped parents. For example, if one subtype declares `[Payload] STRING` and another declares `[Payload] OBJECT ([Amount] INT32)`, selecting `[Payload].[Amount] AS [Amount]` yields the integer for the Object subtype and NULL for the String subtype. Declaration order does not affect decoding or result type. Direct queries, CTE consumers, scalar-subquery type binding, and stored-view schema validation use this leaf contract. Each projection admits up to 1,024 distinct extension inputs in its private working row; these inputs do not consume the public output-column allowance, and no field, whatever it is named, reads one: an extension field selected beside them returns its own value.

Foreign keys belong to the subtype field that declares them ([ALTER COLLECTION](ddl.md#alter-collection)): sibling subtypes may reuse one extension path with different targets, key types or removal actions, or leave it unconstrained, and validation, reverse lookup, sealing and removal follow each document's own declaration ([example](examples.md#example-independent-relationships-at-a-shared-subtype-path)). Declarations that differ only in their reference are still different schemas, so the flattening operators above, a plain `SELECT [Owner]` included, refuse the shared path with `TYPE_MISMATCH`; give the fields distinct names where one result column must carry them.

## Field-level polymorphism (polymorphic sub-documents)

Object and array fields can themselves be polymorphic:

```sql
-- Polymorphic object field
CREATE COLLECTION [App].[Items] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [Component] OBJECT ([Type] STRING, [Power] INT32)
    WITH SUBTYPE [solar] ([Efficiency] DECIMAL)
    WITH SUBTYPE [wind] ([BladeCount] INT32)
);

-- Polymorphic array field
CREATE COLLECTION [App].[Items] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [Components] ARRAY OBJECT ([Type] STRING, [Power] INT32)
    WITH SUBTYPE [battery] ([Capacity] INT32)
    WITH SUBTYPE [motor] ([Torque] INT32)
);
```

In document literals, use nested `SUBTYPE` syntax:

```sql
-- Insert with polymorphic sub-document
INSERT INTO [App].[Items] (
  [Id]: '...',
  [Component]: SUBTYPE [solar] ([Power]: 100, [Efficiency]: 0.22)
);

-- Insert with polymorphic array elements
INSERT INTO [App].[Items] (
  [Id]: '...',
  [Components]: [
    SUBTYPE [battery] ([Power]: 50, [Capacity]: 5000),
    SUBTYPE [motor] ([Power]: 200, [Torque]: 300)
  ]
);
```

### Filtering and sorting on polymorphic extension fields

Works on both Object and ARRAY OBJECT polymorphic fields:

```sql
-- Filter on an Object polymorphic extension field
FROM [App].[Items] WHERE [Component].[Efficiency] > 0.2;

-- Filter on an ARRAY OBJECT polymorphic extension field (ANY element)
FROM [App].[Plants] WHERE [Components][].[Capacity] > 4000;

-- MATCH on an ARRAY OBJECT polymorphic extension with subtype constraint —
-- SUBTYPE IS works inside MATCH bodies too, tested per element against
-- the element's declared subtypes.
FROM [App].[Plants]
WHERE [Components][] MATCH (SUBTYPE IS [battery] AND [Capacity] > 4000);

-- Order / project on extension fields. A plain dotted item returns the whole
-- Component object; an alias projects the leaf (NULL where the subtype lacks it).
FROM [App].[Items] ORDER BY [Component].[Power] DESC;
FROM [App].[Items] SELECT [Id], [Component].[Power] AS [Power], [Component].[Efficiency] AS [Efficiency];
```

`SELECT SUBTYPE` reads the collection-level discriminator ([querying by subtype](#querying-by-subtype)); over a collection that is polymorphic only in its fields, such as `[App].[Items]`, it is `UNKNOWN_FIELD`.

Binary layout for polymorphic values: `[base_len:4][base_binary][ext_binary]` — shared base schema followed by per-subtype extension schema.

### Pinned FK on extension fields

`PINNED REFERENCES` is allowed in subtype extension field lists, both at collection level and inside polymorphic Object sub-documents. The pinned slot's extended `[id][rev:8]` width rides on the extension binary; reads and cascade mutations are envelope-aware.

```sql
-- Collection-level subtype extension: only 'Paid' invoices pin a tariff
CREATE COLLECTION [App].[Invoices] WITH SCHEMA (
  [Id] GUID PRIMARY KEY, [Kind] STRING
)
WITH SUBTYPE [Paid] ([TariffRef] GUID PINNED REFERENCES [App].[Tariffs] ON REMOVE Restrict)
WITH SUBTYPE [Free] ([Note] STRING);
```

### Changing a polymorphic field's subtypes

A field path reaches the fields one subtype of a polymorphic field adds through a `SUBTYPE [value]` segment after the field, and `ALTER FIELD` on the field adds or drops a subtype ([a path into a polymorphic field's subtype](ddl.md#field-subtypes)):

```sql
-- One subtype's fields, and the values of that subtype only
ALTER COLLECTION [App].[Items] ADD FIELD [Component].SUBTYPE [wind].[Hub] INT32 DEFAULT 80;
ALTER COLLECTION [App].[Items] RENAME FIELD [Component].SUBTYPE [solar].[Efficiency] TO [Yield];
ALTER COLLECTION [App].[Plants] ALTER FIELD [Components][].SUBTYPE [battery].[Capacity] SET TYPE INT64;

-- Another subtype for the field, fields for a subtype it has, or a subtype dropped
ALTER COLLECTION [App].[Items] ALTER FIELD [Component] ADD SUBTYPE [hydro] ([Head] DECIMAL);
ALTER COLLECTION [App].[Plants] ALTER FIELD [Components] ADD SUBTYPE [battery] ([Cells] INT32 DEFAULT 1);
ALTER COLLECTION [App].[Plants] ALTER FIELD [Components] DROP SUBTYPE [motor];
```

Another subtype of the field may share a field's name, with its own type; a subtype's field never shadows a field every value has. Fields added to a subtype the field has give its stored values their defaults. `DROP SUBTYPE` never removes data: it is refused (`DEPENDENT_OBJECTS_EXIST`) while any retained version, history included, holds a value of the subtype, and the last subtype of a field stays (`INVALID_DEFINITION`), as does one that alone declares a field a CHECK or computed field reads, or that a CHECK's subtype test names. The field's own CHECKs test each value's subtype and read the fields it adds ([object CHECKs](ddl.md#check-constraints)). `ALTER TYPE` reaches a type's polymorphic field the same way, in every use ([composite types](composite-types.md#change-a-polymorphic-fields-subtypes)).

## See also

- [Schema definition (DDL)](ddl.md)
- [DML](dml.md) — `SUBTYPE` clause on INSERT, polymorphic UPDATE / REPLACE
- [Document literals](document-literals.md) — nested `SUBTYPE` syntax
- [Array queries](arrays.md) — discriminator inside `MATCH`

## Detailed syntax and execution


<a id="syntax-12-polymorphism"></a>

### Polymorphism

Array paths resolve through the active subtype's schema. Sibling subtypes may use different element types or array dimensions at the same logical field name. Write each `[]` needed by the shape you intend to traverse; a row with another shape does not contribute matching leaves to that path. Scalar element predicates, correlated `MATCH`, and predicate `LENGTH` keep their field types and layouts bound to the active array owner.


Two flavours:

- **Collection-level**: declared at `CREATE COLLECTION` via `WITH SUBTYPE [A] (...)`. Inserts must carry `SUBTYPE [value]`.
- **Field-level (sub-document)**: an OBJECT field with `WITH SUBTYPE` blocks inside its definition. Insert literals carry `SUBTYPE [value] (...)` on the field assignment. DDL field paths reach one subtype's fields as `[F].SUBTYPE [value].[X]`, and `ALTER FIELD [F] ADD SUBTYPE` / `DROP SUBTYPE` change the field's subtypes.

Subtype names are bracketed identifiers on every surface (DDL, INSERT, literals, WHERE). In WHERE / MATCH the `SUBTYPE` keyword forms a type test: `SUBTYPE IS [X]` / `IS NOT [X]` / `IN ([A], [B])` — unknown names error instead of matching nothing; matching is case-insensitive and `SELECT SUBTYPE` projects the schema-declared spelling.

See [`polymorphism.md`](polymorphism.md).

## Usage pitfalls


<a id="pitfall-polymorphic-insert-requires-subtype"></a>

### Polymorphic INSERT requires `SUBTYPE`

`INSERT INTO [Fleet].[Vehicles] (...)` on a polymorphic collection without `SUBTYPE [X]` is rejected. The discriminator is mandatory.

<a id="pitfall-subtype-names-are-bracketed-identifiers-not-strings"></a>

### Subtype names are bracketed identifiers, not strings

`SUBTYPE 'car'` (quoted) is a parse error on every surface — DDL, INSERT, document literals. Write `SUBTYPE [car]`. In WHERE the test is `SUBTYPE IS [car]` / `IS NOT` / `IN ([car], [truck])` — the comparison form `SUBTYPE = 'car'` (or `= [car]`) is rejected with guidance. `SUBTYPE IS NULL` is also rejected: polymorphic INSERT makes the subtype mandatory, so no document can lack one.

<a id="pitfall-unknown-subtype-names-error--they-dont-return-zero-rows"></a>

### Unknown subtype names ERROR — they don't return zero rows

`WHERE SUBTYPE IS [carr]` against a schema that declares `[car]` fails with `Unknown subtype 'carr'. Known subtypes: car, truck.` — unlike unknown *fields*, which deliberately follow SQL NULL semantics. The validation runs after parameter binding, so `SUBTYPE IS @t` with a typo'd value errors per execution; it also applies in MATCH bodies and partial-index `WHERE` clauses.

<a id="pitfall-same-named-subtype-objects-retain-their-own-write-validation"></a>

### Same-named subtype objects retain their own write validation

UPDATE, REPLACE, and UPSERT conflict assignments validate each matched row's active extension, including nested field names, enum values, and nullability. A literal valid for one subtype does not authorize dropping unknown children when another matched subtype has a differently shaped object. Such a mismatch fails the statement atomically.

<a id="pitfall-polymorphic-base-fields-can-evolve"></a>

### Polymorphic base fields can evolve

ADD/DROP FIELD rebuild the base part of polymorphic row envelopes and preserve subtype data, including historical temporal versions. Base additions cannot shadow extension names; discriminator fields cannot be dropped. A computed field declared in a subtype can depend on a base field and protects that field against DROP/RENAME until the dependency is removed.
