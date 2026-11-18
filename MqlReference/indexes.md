# Indexes

> **In this section** — `CREATE INDEX` (single-field, compound, covering, partial / filtered), `DROP INDEX`, `ALTER INDEX ADD / DROP INCLUDE`.

Every collection has an automatic `pk` index; temporal collections also have `temporal` and `revision` indexes. These names are local to the collection. Renaming or moving a collection preserves index IDs, definitions, and roots.

For dense-vector nearest-neighbour acceleration, the optional Vamana plugin adds `CREATE VECTOR INDEX` and `DROP VECTOR INDEX`. Vector indexes have their own metric, graph-quality, and optional `FILTER_FIELDS` equality-label options and do not use the scalar B+-tree forms on this page. See [Vector values and nearest-neighbour search](vectors.md).

## CREATE INDEX

```sql
-- Single-field
CREATE INDEX ON [Store].[Products] ([Name]);
CREATE INDEX ON [App].[Orders] ([CustomerId]);
CREATE INDEX ON [App].[Orders] ([Shipping].[City]);   -- nested field path

-- Compound (two or more comma-separated fields)
CREATE INDEX ON [App].[Orders] ([Region], [Status]);
CREATE INDEX ON [App].[Sales]  ([Region], [Amount], [Id]);
CREATE INDEX ON [App].[Names]  ([Last, First], [TenantId]); -- comma inside one bracketed field

-- Covering (INCLUDE payload projected from the index after live-slot validation).
-- Single-key OR compound full-prefix equality both qualify; the planner
-- serves the lookup from leaf data when every projection / filter is in
-- (key ∪ include). EXPLAIN annotates the strategy row's Notes with
-- "Covered (slot-validated)." when the runtime would take that path.
CREATE INDEX ON [App].[Orders] ([CustomerId]) INCLUDE ([Status], [Total]);
CREATE INDEX ON [App].[Orders] ([Region], [Status]) INCLUDE ([Total]);  -- compound + covered

-- Partial / filtered (index maintains and serves only rows satisfying the predicate)
CREATE INDEX ON [App].[Orders] ([CustomerId]) WHERE [Status] = 'Open';
CREATE INDEX ON [App].[Orders] ([CustomerId]) WHERE [Status] = 'Open' AND [Region] = 'EU';
CREATE INDEX ON [App].[Orders] ([CustomerId]) WHERE [Amount] > 100;
CREATE INDEX ON [App].[Orders] ([CustomerId]) WHERE [Amount] >= 100 AND [Amount] < 1000;
CREATE INDEX ON [App].[Orders] ([CustomerId]) WHERE [Status] IN ('Open', 'Pending');
CREATE INDEX ON [App].[Orders] ([CustomerId]) WHERE [DeletedAt] IS NULL;           -- soft-delete sparse index
CREATE INDEX ON [App].[Users] ([Email]) WHERE [VerifiedAt] IS NOT NULL;            -- IS NOT NULL covers any strict probe

-- Partial + covering combined
CREATE INDEX ON [App].[Orders] ([CustomerId]) INCLUDE ([Total]) WHERE [Status] = 'Open';

-- Unique secondary index — duplicate non-null values rejected at INSERT/UPDATE
CREATE UNIQUE INDEX ON [App].[Customers] ([Email]);
CREATE UNIQUE INDEX ON [App].[Tenant_Users] ([TenantId], [UserName]);

-- Array-element inverted (GIN) index — the trailing [] marks an array field.
-- One index entry per distinct element, so an array-membership predicate
-- (WHERE [Field][] = value, the default ANY quantifier) is a seek, not a scan.
CREATE INDEX ON [App].[Posts] ([Tags][]);         -- ARRAY STRING field (scalar elements)
CREATE INDEX ON [App].[Nums]  ([Vals][]);         -- ARRAY INT64 field (scalar elements)
CREATE INDEX ON [App].[Orders] ([Items][].[Sku]); -- nested scalar in an ARRAY OBJECT element
```

A scalar `PINNED REFERENCES` field is indexable. Its comparison key is the referenced id, matching scalar pinned-field equality: rows bound to different revisions of the same id share one index key. If such a field is projected by a covering index, the payload retains the complete stored pin so the bound revision is preserved in the result.

Commas separate index fields only when they occur outside bracketed identifiers. `[Last, First]` is one field name, so indexes on `([A,B], [C])` and `([A], [B,C])` are distinct and can be created, queried, dropped, and reopened independently.

Every explicit index field-path list accepts at most 1,024 paths, matching the maximum schema width. This covers key lists in `CREATE INDEX`, `ALTER INDEX`, and `DROP INDEX`, plus `INCLUDE` lists in `CREATE INDEX` and `ALTER INDEX`. Text parsing rejects the next comma before parsing another path; canonical printing and direct execution apply the same bound to programmatic ASTs before joining or copying the caller-provided list.

### Array-element (GIN) indexes

`CREATE INDEX ON [C] ([Field][])` builds an inverted index over the elements of an array field; `CREATE INDEX ON [C] ([Items][].[Sub])` indexes a nested scalar sub-field within each object element of an `ARRAY OBJECT` field. Each document contributes one entry per **distinct** element value, so a membership query answers from the index. Arrays can also live inside nested Objects, including polymorphic Object extensions: `([Profile].[Tags][])` and `([Profile].[Items][].[Sku])` use the same forms. Each declaring subtype must supply an array with a stored scalar key domain, but Object and element layouts may differ. A null or absent containing Object contributes coverage without a scalar value:

```sql
FROM [App].[Posts] WHERE [Tags][] = 'urgent' SELECT [Id];               -- ANY element = 'urgent'
FROM [App].[Posts] WHERE [Tags][] IN ('urgent', 'blocker') SELECT [Id]; -- ANY element in the set
FROM [App].[Scores] WHERE [Vals][] >= 90 SELECT [Id];                     -- ANY element >= 90
FROM [App].[Orders] WHERE [Items][].[Sku] = 'A1' SELECT [Id];           -- ANY element's Sku = 'A1'
```

`EXPLAIN` of such a query reports `Strategy = ArrayMembership`, including when an accompanying scalar equality could also use a compound or secondary index: array membership has the same earlier precedence in EXPLAIN as at runtime. The `IN` form unions the per-value posting-list seeks and a range comparison scans the ordered element keys on the matching side of its bound. Each matching document is returned once, including when an `IN` union contains one document and the query sorts, aggregates, pages, or counts its result. Separate comparisons on the same array path are independent existentials: `[Vals][] > 5 AND [Vals][] < 10` can be satisfied by different elements. The planner may serve one side from the index and leave the other as a residual filter; it never collapses them into a same-element interval. Use `MATCH` when predicates must bind one object element. String elements are case-folded like scalar string keys (`'urgent'` matches a stored `'Urgent'`). The empty string is an ordinary element value — `IN ('', …)` and `= ''` seek its posting list like any other string. A listed value the element type cannot parse (e.g. `'banana'` in an `IN` list over `ARRAY INT64`) makes the whole predicate fall back to the authoritative scan path (`Strategy = FullScan`). An unparseable numeric arm is a non-match there, while parseable siblings retain their normal meaning. The planner never probes only the encodable subset and consumes the complete predicate.

`BYTES` is also a supported scalar element or nested scalar sub-field type. Its MQL literal is base64, and `''` denotes a present zero-length byte sequence. That empty value is indexed and matches `= ''`; it is distinct from a nullable nested `BYTES` field whose value is `NULL`, which contributes null-leaf coverage.

A **unique** array-element index reserves each distinct element value for one document:

```sql
CREATE UNIQUE INDEX ON [App].[Posts] ([Tags][]);
CREATE UNIQUE INDEX ON [App].[Orders] ([Items][].[Sku]);
```

Repeated canonical values within the same document are allowed (`'urgent'` and `'Urgent'` share one key). Another document containing any claimed value fails with `UNIQUE_VIOLATION`. Null, missing, and empty arrays and null nested leaves claim no key; empty strings and byte sequences do. Pinned references claim the identifier across all revisions. Updates, replacements, and upserts retain a row's own keys and release removed keys; a failed statement rolls back all of its element changes. Partial uniqueness applies only to documents matching the index filter. Uniqueness is enforced during offline or online creation and is preserved through reopen and rebuild. Online builds can reject duplicates in concurrent writes before the index becomes query-visible. On a temporal collection each element belongs to one document at every instant of valid time: a document's versions share their elements, and another document may hold an element only in versions that overlap none of them ([UNIQUE on a temporal collection](#unique-on-a-temporal-collection)).

Array indexes cannot be `ON CONFLICT` targets because distinct incoming elements could identify different existing documents. An upsert targeting the primary key or an ordinary unique index still enforces array uniqueness.

A **partial** array-element index is supported — a `WHERE` filter (a predicate on any non-array field) restricts which documents contribute their elements:

```sql
CREATE INDEX ON [App].[Posts] ([Tags][]) WHERE [Published] = true;
```

The planner uses a partial array index only when the query's `WHERE` *implies* the filter (e.g. the query also constrains `[Published] = true`); otherwise it falls back to a full scan rather than missing unindexed documents — the same rule as scalar partial indexes.

Arrays of **foreign keys** are indexable too — `WHERE [TagIds][] = id` membership over an `ARRAY <FK>` field. A pinned FK array stores `[id][revision]` per element; the index keys on the id, so membership is revision-agnostic (it finds every document pinning that id at any revision).

Scope: the indexed value must be a **scalar** — either the elements of a scalar array (`[Field][]`) or a scalar sub-field of an `ARRAY OBJECT` element (`[Items][].[Sub]`), with either array allowed inside Object containers. The nested leaf must be stored; a virtual computed leaf is synthesized only after materialization and is rejected. Stored foreign-key leaves are supported, including paths through nested objects such as `[Items][].[Sub].[Ref]`. Pins index their identifier; explicit `.Id` and `.Revision` paths can be indexed separately. All seven primary-key types are supported. Reference removal, exact-revision cancellation, rollback, and reopen maintain the same postings. Polymorphic elements and nested subtype-extension paths are supported when every subtype declaring the leaf supplies a stored scalar type. Types may differ; each type owns a separate ordered domain. The active subtype supplies its value; a subtype omitting the leaf contributes no scalar-value posting.

Compound array indexes accept an ordered list of stored scalar paths and explicit array leaf paths. Postings are factored by path, so independent arrays do not cause Cartesian expansion. UNIQUE compares independent value domains: two rows conflict only when every corresponding domain shares a non-null value. On a temporal collection they must also be versions of different documents whose validity intervals overlap. A component with no non-null values owns no value tuple. Shared array ancestors do not imply same-element correlation. Query MATCH predicates retain their ordinary row-level correlation semantics. Equality, IN, ranges, negative comparisons, explicit NULL, ALL, and runtime IN-subquery probes combine scalar and array components. Each predicate unions typed values and coverage candidates; a bounded, spill-capable merge intersects complete row pointers across predicates. Separate predicates on the same array remain independent. ALL includes empty and null coverage before full row rechecks. Runtime membership and types whose physical ordering differs from comparison semantics evaluate distinct stored values through the ordinary scalar evaluator. These postings do not provide scalar compound-key ordering.

If a typed decoder rejects a visited array-posting key during runtime membership or a compound scalar comparison, the query fails with `ARRAY_KEY_CORRUPT` (`StructuralCorruption`). The health record identifies the supplying index, and the engine quarantines ordinary MQL. Membership-callback errors retain their original classification.

Array indexes accept stored scalar base-row `INCLUDE` fields. Each distinct value posting and coverage posting carries the row's payload; included values do not affect element ordering or uniqueness. `ALTER INDEX ... ADD INCLUDE` and `DROP INCLUDE` preserve the array kind and work both offline and online. A single positive ANY equality on a homogeneous scalar domain can use a covered projection when every projected field is included and ordinary covered-plan eligibility holds. The visible source row must reproduce the element key and every payload value before reconstruction. The element key cannot reconstruct the complete array. Other query shapes retain ordinary row evaluation.

Repeated explicit boundaries are supported, including `[Matrix][][]` and `[Groups][].[Values][]`. One row contributes one posting per distinct leaf value across all inner arrays. Null, missing, or empty containers contribute no scalar-value postings. Pinned arrays support all seven identifier types, including String and Bytes, and index the identifier independently of its revision. The index accelerates the ANY-membership shapes (equality, `IN`, and ranges), `!=`, `NOT IN`, explicit `= NULL` / `!= NULL`, and the `ALL` quantifier. Negative and universal plans use value and coverage candidates with a residual predicate; null leaves remain distinct from null or empty array containers. Null-bearing lists preserve scalar NULL policy: IN with a NULL member matches a NULL leaf; NOT IN excludes that leaf. Present leaves compare against the present members. Non-null list members must be fully probeable; runtime subquery lists scan the ordered array index, probing the existing typed membership set once per distinct value. This preserves its spill budget and keeps the full row predicate as a residual.

Different declared scalar types occupy separate ordered key domains. Probes are normalized per domain and candidates are unioned with a full predicate recheck. UNIQUE ownership is per declared type and canonical value: for example, INT32 7 and INT64 7 have separate ownership, while duplicate INT32 7 values conflict.

Collection subtype extensions can own the array or its Object containers. The active collection subtype supplies the complete source layout; an omitted array contributes no scalar-value posting. A scoped rename keeps a shared index at the old path for remaining declarations and rebuilds its entries. When the renamed subtype was the last declaration, the index follows the new path. Dropping the last declaring subtype retires the index. If the destination path is already indexed, its existing index retains its identity and rebuilds to include the renamed rows. These changes roll back with the schema; adding a stored scalar domain preserves the index and its identity; malformed container shapes and virtual leaves are rejected before publication.

Secondary indexes accelerate equality and equality-prefix + trailing-range queries on the indexed field list. B+ tree structure, maintained on writes.

## UNIQUE INDEX

`CREATE UNIQUE INDEX` rejects duplicate values on the indexed field list. Compound unique indexes require the full tuple to be unique across rows; partial-tuple matches are permitted (an existing `(TenantId=1, UserName='alice')` does not block a new `(TenantId=2, UserName='alice')`). Backfill on `CREATE` rejects existing duplicates with a clear error before the index is materialised — surfaced with error code `UNIQUE_VIOLATION`, the same classification as a duplicate INSERT against a live unique index. UPDATE in-place against the same row's PK is tolerated (the previous-pointer row-identity check distinguishes a row updating its own indexed value from a foreign collision). The check considers the complete equal-key owner set: the temporary old+replacement overlap is allowed, but any third-row owner raises `UNIQUE_VIOLATION` before the replacement is added, even if damaged catalog/index state already contains the old pointer.

```sql
-- Reject duplicate emails
CREATE UNIQUE INDEX ON [App].[Customers] ([Email]);

-- Compound uniqueness
CREATE UNIQUE INDEX ON [App].[Tenant_Users] ([TenantId], [UserName]);

-- Unique + covering / partial combine the same way as non-unique forms
CREATE UNIQUE INDEX ON [App].[Customers] ([Email]) INCLUDE ([Name]);
CREATE UNIQUE INDEX ON [App].[Customers] ([Email]) WHERE [Active] = TRUE;
```

### UNIQUE on a temporal collection

On a temporal collection a unique index holds at every instant of valid time, like SQL:2011 `UNIQUE (..., period WITHOUT OVERLAPS)`: two different documents may not hold equal non-null keys in versions whose half-open `[ValidFrom, ValidTo)` intervals overlap, and an open end overlaps every later start. The versions of one document never overlap each other, so they always share their keys. Every retained version keeps its own index entry for `AS OF` reads, so the check reads the validity of each version holding the key rather than counting entries.

| Statement | Outcome |
|---|---|
| `UPDATE` keeping the value, at `NOW` or `VALID FROM` a past instant | Accepted: the new version and the truncated one belong to one document. |
| `UPDATE` to a value another document holds over part of `[ValidFrom, end)` | `UNIQUE_VIOLATION`. A backdated version is checked over its whole interval, including history. |
| A value another document holds only before or after the new interval | Accepted — the value is reused after `DELETE ... AT T` from `VALID FROM T` on, and refused for any earlier start. |
| A scheduled future version | Enforced now, in both directions: neither it nor a current version may overlap another document's version of the key. |
| `INSERT` on an existing primary key, a bounded version that splits one, `REPLACE` | Accepted for the document's own history; checked against other documents over the written interval. |
| `CANCEL VERSION ... UPDATE PREVIOUS [SET (...)] VALID TO` widening the predecessor | `UNIQUE_VIOLATION` when the widened interval overlaps another document's version of the key ([CANCEL VERSION](dml.md#cancel-version)). |

Compound keys apply the rule to the whole tuple. A unique array index applies it to each element ([unique array elements](#array-element-gin-indexes)), and a compound array index conflicts only for a row that intersects every component domain and overlaps in valid time. Building, rebuilding, or rewriting the index (`CREATE`, `ALTER INDEX ... INCLUDE`, recovery rebuild, `ALTER FIELD ... SET TYPE`, `SET PRIMARY KEY`) validates every retained version and fails with `UNIQUE_VIOLATION` only when two documents overlap. `SET TEMPORAL FALSE` keeps one current version per document, so it cannot introduce a duplicate; `SET TEMPORAL TRUE` turns each document into one open version. Readers resolve a key at one instant, so a unique index still identifies at most one row `AS OF` any time; history reads (`IN BETWEEN`, all versions) can return several versions with one key.

## Compound-index matching rules

Ordinary secondary indexes omit rows with a NULL or missing key component. An unconstrained compound tail is eligible only when every field in that tail and all its object parents are `NOT NULL`. A required leaf under a nullable object can be missing and therefore requires another complete access path. A non-null equality or range on the component itself excludes those missing values.

- Full-prefix equality (`WHERE [A] = x AND [B] = y` on a `(A, B)` index) → an exact seek, EXPLAIN strategy `CompoundEquality`.
- Partial-prefix equality (`WHERE [A] = x` on a `(A, B)` index) → prefix range scan, `CompoundRange`.
- Equality-prefix + trailing range (`WHERE [A] = x AND [B] > y` on a `(A, B)` index) → prefix-bounded range scan on the trailing field, `CompoundRange`.
- Predicate order in the WHERE clause is irrelevant — the matcher looks up each index field by name.

Repeated lower or upper predicates on the trailing range field use the tightest bound regardless of their order in WHERE. Exclusive bounds win equal-value ties; the remaining predicates still undergo ordinary residual evaluation.

<a id="syntax-compound-skip-scans"></a>

## Compound skip scans

A compound index can serve a selective later field even when earlier fields lack equality predicates. For an index on `([Tenant], [Kind], [Score])`, both `WHERE [Score] = 277` and `WHERE [Tenant] = 1 AND [Score] = 277` can use `CompoundSkipScan`. Run ANALYZE after seeding so the chooser can compare the estimated distinct prefixes and matching rows against other indexes and a heap scan. An earlier range can also precede a later constrained field.

All indexed fields must be non-null for qualifying rows. Unbound fields require a NOT NULL path (including object parents) or an explicit IS NOT NULL predicate; partial indexes require the query to imply their filter. Temporal/snapshot and transaction-private index membership retain their visibility-aware paths.

EXPLAIN names the complete selected field list and reports estimated searches and the search allowance in Notes. Execution bounds both repeated searches and candidate bitmap memory. If either allowance is exhausted, it falls back to the heap before publishing rows. The page-ordered candidates do not satisfy ORDER BY and are read from the heap even if the index has INCLUDE columns.

## Partial / filtered indexes

Bare numeric literals follow the [exact Decimal literal rules](syntax.md#numbers) at declaration and reopen. Quoted numeric text retains ordinary comparison coercion, including values outside Decimal's exact precision.

Partial indexes maintain only rows satisfying the `WHERE` predicate. The planner uses the index only when the query's WHERE *logically implies* the index's WHERE on a per-predicate basis. Queries that don't imply the predicate fall through to `FullScan`. This includes an `ORDER BY`-only query: it must sort the complete collection and cannot use a partial index that omits rows; `EXPLAIN` applies the same implication gate and does not advertise that index scan.

### Supported index-predicate → query-predicate implications

| Index predicate shape       | Query-predicate shapes that imply it                                               |
|-----------------------------|------------------------------------------------------------------------------------|
| `F = x`                     | `F = x`                                                                            |
| `F > x`                     | `F = v` (v > x), `F > v` (v ≥ x), `F >= v` (v > x)                                 |
| `F >= x`                    | `F = v` (v ≥ x), `F > v` (v ≥ x), `F >= v` (v ≥ x)                                 |
| `F < x` / `F <= x`          | mirror of the GT / GTE cases                                                       |
| `F IN (S)`                  | `F = v` (v ∈ S), `F IN (T)` with T ⊆ S                                             |
| `F != x` (non-null x) | `F IS NULL`, equality/IN excluding x, the same inequality, NOT IN containing x, or a range/string pattern excluding x |
| `F NOT IN (S)` | equality/IN disjoint from S, NOT IN with a superset of S, or a range/string pattern excluding every member; IS NULL only when S excludes NULL |
| `F STARTSWITH p` | Equality/IN values and stronger prefixes starting with p |
| `F CONTAINS p` | Equality/IN values, prefixes, or required substrings containing p |
| `F [NOT] LIKE p` | Equality/IN values satisfying p; contained prefix, substring, or LIKE/NOT LIKE languages (including ESCAPE) |
| `F IS NULL` | `F IS NULL` or `F IN (NULL)` |
| `F IS NOT NULL` or `F NOT IN (NULL)` | `F IS NOT NULL`, or any null-rejecting predicate on F (ranges, text predicates, equality/IN without NULL, NOT IN containing NULL) |

The `F IS NOT NULL` row is the **strictness rule** (informed by PostgreSQL's strong-implication and `clause_is_strict_for` machinery, then narrowed for MQL semantics): any predicate whose evaluator rejects null fields guarantees the matched row has a non-null F, so the `IS NOT NULL` index covers it. `!=` / `NOT IN` are deliberately excluded because Mnemic's `FilterEvaluator` treats `null != 'x'` and `null NOT IN (S)` as matches (MQL deviates from SQL three-valued logic here so `WHERE F != 'x'` doesn't silently drop nulls), so those queries can include null rows and don't imply `IS NOT NULL`. Without the strictness rule a sparse-column partial index would be effectively dead.

STRING queries using STARTSWITH, CONTAINS, LIKE, or NOT LIKE can imply a non-null `!=` or `NOT IN` filter when no forbidden literal satisfies the query. This uses the ordinary case fold and LIKE matcher, including ESCAPE and negative pattern polarity. The proof examines the declared literals, not current rows.

A non-null `!=` filter includes null rows. For example, `WHERE [State] != 20` includes missing/null State values and all values other than 20. `NOT IN (20, 30)` excludes both values while retaining null rows. A NULL-bearing list also excludes NULL rows; `NOT IN (NULL)` selects every present value. A query that rejects NULL can therefore use this partial index just as it can an `IS NOT NULL` index. An empty string is a real list member, distinct from NULL. A unique partial index enforces uniqueness across that entire subset. Equality, finite membership, and range proofs compare the excluded value in the field's logical domain; failed literal coercion never proves inequality.

Value comparison is type-aware: `"10"` vs `"10.0"` compare equal on a Float64 field, `'Open'` vs `'OPEN'` compare equal on a String field, and date / time literals compare on their stored logical values (UTC ticks for DateTimeOffset, day numbers for DateOnly, ticks for TimeOnly). Integer-valued decimal spellings accepted by the evaluator are ordered numerically as well: on INT32, `20.0` is 20, not a string that sorts after `100`. If either literal cannot be coerced to the field type, implication is not proven and the planner falls back to `FullScan`; it never guesses using lexical byte order for a typed value.

`IN` implication canonicalizes the partial predicate once and probes each query member in expected linear time. Empty values are real members: a partial filter such as `WHERE [Status] IN ('', 'Open')` can serve `WHERE [Status] = ''`; the empty string is not dropped while decoding the persisted list.

String prefix and substring predicates use the evaluator's case folding, including Unicode. Equality and IN must satisfy the declared pattern; a stronger required prefix/substring must contain the weaker one in the required position. These positive predicates reject null fields. An empty pattern matches every non-null string, including the empty string, and is implied by IS NOT NULL or another null-rejecting string predicate.

LIKE implication compares pattern languages over invariant-lowercased Unicode scalars. Equality and every non-null IN member use the production matcher. Prefix, substring, LIKE, and NOT LIKE queries use a bounded automata proof. LIKE/NOT LIKE queries can also imply STARTSWITH/CONTAINS index filters, so different operator spellings do not prevent index use. Proofs support reversed containment for two negative patterns. IS NOT NULL implies a universal pattern such as LIKE '%'. No proof samples existing rows. Identical patterns bypass exploration. Otherwise each pattern is limited to 1,024 UTF-8 bytes and 256 tokens for proof construction, with at most 4,096 reachable state pairs and 65,536 transitions. Exceeding a proof bound selects a complete access path; it does not reject the index or change membership.

### Partial-index WHERE grammar

- Boolean combinations with `NOT`, `AND`, `OR`, and parentheses. Precedence is NOT, then AND, then OR. Predicates admit at most 1,024 scalar atoms, 4,096 compiled instructions, and 64 nested parentheses/NOT levels.
- Per scalar atom: `field op literal` with `op` in `=`, `!=`, `<`, `<=`, `>`, `>=`; or `field [NOT] IN (literal, literal, …)`; or `field IS NULL` / `field IS NOT NULL`.
- Range operators (`<`, `<=`, `>`, `>=`) require an order-preserving field type — numeric, date / time, or string. Equality, inequality, and `IN` work on any scalar type. `IS NULL` / `IS NOT NULL` work on any scalar type.
- `field STARTSWITH literal` and `field CONTAINS literal` require a STRING field and a non-null literal.
- Scalar base-schema fields only — no array traversal (`[Items]…`), no composite types, virtual computed fields, or polymorphic-extension fields (including extensions on a nested Object). Computed values are synthesized after a row is materialized, while partial index maintenance evaluates the predicate directly over stored bytes.
- Literal right-hand-sides only — parameter references (`@name`) and `NOW` are rejected because the predicate must be pure and time-invariant so it can be persisted and re-evaluated at insert time. A `NOW` predicate would drift with the clock, and the planner's partial-index implication check would then believe the index covers rows it no longer does.
- `field [NOT] LIKE literal [ESCAPE 'c']` requires a STRING field and a non-null pattern. ESCAPE has the same meaning as in ordinary queries. Both operators reject null fields; NOT LIKE is a complement over non-null strings only. Invalid patterns or escape characters are rejected.
- `MATCH` remains outside the admitted partial-index predicate grammar.

Membership requires the complete predicate to be TRUE. Equality uses MQL's two-valued NULL rules: `NOT ([State] = 20)` includes null State values. Ordered and text comparisons retain UNKNOWN under NOT and exclude those rows. Negated STARTSWITH/CONTAINS preserve literal wildcard characters when mapped to escaped NOT LIKE for source lookup and implication. `NOT ([State] IN (20))` is identical to `[State] NOT IN (20)`, including NULL. IN and NOT IN are exact complements for every list.

Every OR query branch must imply the index predicate. A shared equality can supply one seek, with the complete OR condition checked before SKIP/LIMIT. ANALYZE statistics rank eligible shared equality indexes; EXPLAIN reports the chosen index and its estimate, or a cheaper full scan. Bounded proof work may select a full scan; it never changes index membership.

Predicate compilation consults the target's live schema for both scalar and array-element partial indexes. An ordinary unsupported shape returns the validation errors above, but structural corruption found during compilation (including through an aggregate/inner wrapper) fails closed with its stable code and quarantines its fault domain; it is never relabelled as an invalid filter.

Use `EXPLAIN <query>` to see whether the planner picks your partial index — matches land on `SecondaryEquality` / `SecondaryRange` / `CompoundEquality` strategies with a `PartialIndex: WHERE …` entry in the `Notes` column.

### Partial filters and field DDL

The persisted `WHERE` filter stays in lockstep with the schema:

- `ALTER COLLECTION … RENAME FIELD` rewrites the filter of every partial index that references the renamed field (the same way key-field paths already follow a rename), so the index keeps working across the rename and across reopens.
- `ALTER COLLECTION … DROP FIELD` is **rejected** while any partial index's filter references the field — drop that index first, or recreate it with a filter that doesn't use the field. Otherwise indexes referencing the dropped field or its descendants through keys or INCLUDE are dropped automatically, including UNIQUE indexes. The same dependency rule applies when ALTER TYPE removes a field from a collection's named type. Use `ALTER INDEX ... DROP INCLUDE` first to preserve an index whose only dependency is its included payload. Aborting the migration restores its complete definition and payload.

Every persisted `Ready` definition is revalidated against the current schema before reopen publishes it. If its filter cannot be parsed, or its key/INCLUDE shape is no longer a supported stored-value extraction (possible through out-of-band catalog mutation or older incompatible metadata), the index is **quarantined**: it is not registered for queries or writes. Serving a partial, empty, or NULL-filled tree as complete could silently drop or fabricate query results, so queries instead fall back to scans. The catalog row remains visible with `Quarantined` status; the stale tree is detached, and its pages stay allocated rather than being freed while their reachability is unproven. Recreate the index with `CREATE INDEX` (which replaces the quarantined row) or remove it with `DROP INDEX`.

Array indexes with an incompatible key format (`arrayidx_` or untyped `array1_`) instead refuse database open with `ARRAY_INDEX_FORMAT_UNSUPPORTED`, preserving their constraints. The database must be recreated in the current format; these indexes are not silently quarantined or upgraded.

## CREATE INDEX rejects

- Non-existent field paths — every listed field must be declared in the collection schema.
- Duplicate field paths in a single compound declaration (e.g. `([Region], [Region])`).
- Ordinary scalar-index paths that traverse *into* an `ARRAY` field's elements. Use the array-index forms `([Tags][])` for scalar elements or `([Items][].[Price])` for a scalar field of an `ARRAY OBJECT` element; see "Array-element (GIN) indexes" above.
- Paths whose final field is itself a composite type (whole `Object`, or a whole `Array` named *without* the `[]` element marker) — a scalar index on one would be maintained on every write but only matchable by byte-for-byte identical composite binaries. Index a scalar sub-field (e.g., `[Shipping].[City]` rather than `[Shipping]`), or use `([Field][])` for an array-element index.
- Polymorphic subtype-extension paths in ordinary scalar indexes or `INCLUDE`, including extensions on a nested Object. These definitions require one fixed schema layout. Fixed base fields inside a polymorphic Object remain indexable. Array-element indexes support distinct stored scalar domains across subtypes.
- Virtual computed columns as either index keys or `INCLUDE` values. They are synthesized at read time and have no stored slot. Their predicates evaluate after derivation; eligible stored-field conjuncts can still narrow candidates through ordinary indexes before derivation.
- Duplicate `CREATE INDEX` on the same field list. Use `DROP INDEX` first if you want to recreate.
- `WHERE` clauses that don't match the partial-index grammar above.

## DROP INDEX

```sql
DROP INDEX ON [Store].[Products] ([Name]);
DROP INDEX ON [App].[Orders] ([CustomerId]);
DROP INDEX ON [App].[Orders] ([Region], [Status]);
```

The field list must match the `CREATE INDEX` exactly (same order, same length). Swapping the order (`DROP INDEX ON [t] ([Status], [Region])` against an index created as `([Region], [Status])`) is rejected — typos don't silently succeed.

Auto-created indexes (PK index, temporal index) cannot be dropped.

## ALTER INDEX ADD / DROP INCLUDE

Change an existing secondary index's covered-column set in place — no `DROP INDEX` + `CREATE INDEX` round-trip.

```sql
ALTER INDEX ON [Store].[Orders] ([Customer]) ADD INCLUDE ([Status], [Total]);
ALTER INDEX ON [Store].[Orders] ([Customer]) DROP INCLUDE ([Total]);
```

- The `(Key)` list must identify an existing secondary index exactly (same fields, same order).
- `ADD INCLUDE` rejects fields already in the set, scalar B-tree key fields, and any path that a fresh `CREATE INDEX … INCLUDE` would reject (non-existent fields, composite paths, array-traversing paths, polymorphic-extension paths, virtual computed columns).
- A scalar key component in a compound array index may also appear in `INCLUDE`. This stores its value with postings for every component; array traversal paths themselves remain ineligible as scalar payloads. Duplicate INCLUDE paths are rejected case-insensitively by both MQL and the engine.
- `DROP INCLUDE` rejects fields not currently in the set. Dropping every included column is legal — the index becomes non-covering.
- A replacement is built beside the live index and published only when complete; a build failure leaves the old index authoritative. `ADD INCLUDE` reads the collection because the new values are absent from the old index. An offline `DROP INCLUDE` can project retained values from the old ordered leaf payload instead. The operation preserves the original key order, uniqueness, and partial-index predicate. After publication, failure to reclaim displaced pages does not undo the replacement; inspect `[Mnemic].[Health]` for a reported structural fault.
- Ordinary offline leaf-projection preparation or replacement-construction failure releases its temporary sort files and memory admission before returning the error. The original index remains authoritative, and the operation can be retried.
- An offline `ALTER INDEX` may run inside a migration, including after DML or row-rewriting DDL on the same collection: its rebuild reads the migration's own uncommitted rows (see [Rewrites and indexes share the migration's rows](#pitfall-rewrites-and-indexes-share-the-migrations-rows)). The new include set publishes at `COMMIT MIGRATION`; abort restores the prior one. Online `ALTER INDEX` is standalone.

## Online index build — `WITH (ONLINE = ON)`

By default `CREATE INDEX` / `ALTER INDEX … INCLUDE` hold the collection's write lock for the whole backfill — concurrent writers wait until the build finishes. Append `WITH (ONLINE = ON)` to allow concurrent writes during the long backfill: live `INSERT` / `UPDATE` / `REPLACE` / `DELETE` maintain the building index while a concurrent scan reconciles the pre-existing rows, then the finished tree is swapped in atomically. Writers can pause briefly at the registration and publication cuts.

```sql
CREATE INDEX ON [App].[Orders] ([CustomerId]) WITH (ONLINE = ON);
CREATE UNIQUE INDEX ON [App].[Users] ([Email]) WITH (ONLINE = ON);
CREATE INDEX ON [App].[Orders] ([CustomerId]) INCLUDE ([Total]) WHERE [Status] = 'Open' WITH (ONLINE = ON);
ALTER INDEX ON [App].[Orders] ([CustomerId]) ADD INCLUDE ([Status]) WITH (ONLINE = ON);
```

- `WITH (ONLINE = OFF)` (or omitting the clause) is the default offline build.
- **Online index DDL is standalone — it runs OUTSIDE a migration and is rejected inside one** (like PostgreSQL `CREATE INDEX CONCURRENTLY`, which cannot run in a transaction block). An online build commits independently; it cannot ride a migration's all-or-nothing apply. The refusal ends the open migration, as any failed statement does ([migrations](migrations.md#a-failed-statement-ends-the-migration)). Wrap only *offline* index DDL in `BEGIN MIGRATION` / `COMMIT MIGRATION`.
- Applies to **B-tree** indexes (single-field, compound, unique, partial, covering) **and array-element (`[Field][]`, GIN) indexes**, including unique and partial forms. Array indexes also support covering payloads and `ALTER INDEX ... INCLUDE ... WITH (ONLINE = ON)`.
- Same correctness as an offline build: the finished index contains exactly what a build over the final data would. A `CREATE UNIQUE INDEX … WITH (ONLINE = ON)` still aborts if the data contains duplicates.
- Reconciliation tracks concurrent changes under a per-build limit derived from `EngineOptions.MaxInMemoryStreamingState` (estimated 256 bytes per distinct pointer, at least one pointer). Exhaustion reports `ResourcePressure / ONLINE_BUILD_CAPACITY_EXCEEDED` for the build and triggering write; it never publishes an incomplete index. Increase the host budget, retry with less concurrent churn, or build offline.
- During a unique online build, live DML is already checked against the hidden building tree. A conflicting write can therefore receive `UNIQUE_VIOLATION` before the index becomes query-visible.
- A row whose `DELETE` or `UPDATE` rolls back during the build keeps its entry, which the build restores after its scan. A write that commits a second row with that row's value before then is accepted, and a unique build then fails with `UNIQUE_VIOLATION` when it publishes, as for a duplicate in the data.
- On a temporal collection the online build checks each retained version against the validity of the versions already in the hidden tree, and a live write that widens a version (`CANCEL VERSION ... UPDATE PREVIOUS VALID TO`) is checked against it too ([UNIQUE on a temporal collection](#unique-on-a-temporal-collection)). The build fails with `UNIQUE_VIOLATION` when it observes two documents holding a key over overlapping intervals, even if a concurrent write later removes the overlap.
- A backfill row that needs an index page another transaction still owns, or a unique key another transaction is removing, waits for that transaction to finish, then runs again. The wait is bounded by the lock-acquisition deadline of the `IndexMutationContext.TransactionOwnership` site (`EngineOptions.LockAcquireTimeoutMs` unless `LockAcquireTimeoutOverridesMs` names the site); when it expires, the build fails with `LOCK_TIMEOUT` and retires, and can simply be run again. A committed write gives its pages back before the backfill can meet them, so only a transaction that kept a page past its own commit makes a build wait; `[Mnemic].[IndexBuilds]` shows the waits and their time ([system collections](system-collections.md)).
- If a backfill row and its compensation both fail, the build fails instead of retrying that row as ordinary contention. Ordinary rollback attempts every callback before returning the combined failure; a terminal compensation failure stops later cleanup.
- Once the finished tree is swapped in, it remains authoritative even if retirement of the displaced tree reports a page-reclamation failure. Do not retry the DDL as though publication failed; inspect `[Mnemic].[Health]` for a structural fault and recover according to that domain.
- Trade-off: an online build is slower than offline (per-row insert instead of bulk-load) in exchange for writer availability during the long scan. Operators see online builds on the `mnemic.index.online_builds` meter.

## Expiry indexes

```sql
-- Delete sessions once CreatedAt is seven elapsed days old.
BEGIN MIGRATION 1;
CREATE INDEX ON [App].[Sessions] ([CreatedAt]) WITH TTL '7 DAYS';
COMMIT MIGRATION;

-- Or use an absolute deadline, with a standalone online build.
CREATE INDEX ON [App].[Jobs] ([ExpiresAt]) WHERE [Temporary] = TRUE
WITH EXPIRY WITH (ONLINE = ON);
```

`WITH TTL 'duration'` and `WITH EXPIRY` attach an automatic deletion policy to one stored scalar `DATETIME` key. `EXPIRY` means zero duration: the indexed timestamp is the deadline. `TTL` accepts a non-negative decimal duration followed by a unit: `ticks`, `ms`, `s`, `m`, `h`, or `d`, or the corresponding English singular or plural word (`m` is minutes, never months). Units are case-insensitive and may follow the number directly or after spaces, so `'30d'`, `'30 days'`, and `'30 DAYS'` are the same lifetime. Values must be exactly representable in signed 64-bit 100-nanosecond ticks, without rounding during parsing or unit conversion. Redundant zeros are accepted; the duration string is limited to 128 characters. Months, years, negative values, and sub-tick values are rejected. NULL never expires. Stored computed DATETIME fields are eligible. Compound, virtual computed, engine-virtual, and array-element keys cannot define an expiry policy. Partial filters, INCLUDE payloads, and uniqueness retain their ordinary index meaning.

The MQL runtime runs one background worker per engine, by default once per second, attempting up to 128 candidates per index in each pass. Deletion is eventual. Each candidate's current timestamp, partial membership, schema, and policy identity are rechecked under the write claim. Refreshing a deadline or dropping its policy prevents a stale candidate from being deleted. Rejected rows do not block later candidates. Ordinary DELETE rules apply, including BEFORE/AFTER triggers, foreign keys, sealing, and materialized views. On temporal collections expiry closes the current version and retains its history.

The policy persists through reopen, rename, and INCLUDE rebuilds; migration abort restores the prior policy, including indexes removed by a collection or named-type field drop. DROP INDEX removes it. To change a policy, drop and recreate the index. `[Mnemic].[Indexes].[ExpireAfterTicks]` exposes the duration; NULL means no policy. Engine-only hosts must initialize the MQL runtime to run expiry. Hosts may configure `EngineOptions.ExpirySweepInterval`; NULL disables automatic sweeps. Shutdown cancels and joins pending expiry work.

Expiry and online options follow any INCLUDE / WHERE clauses. CREATE INDEX accepts either option order, with at most one expiry option and one online option. Offline creation remains migration-scoped; online creation is standalone.

## Covered-path observability

`[Mnemic].[Metrics].CoveredIndexSeeks` counts the engine-wide subset of `IndexSeeks` whose result rows are reconstructed from a secondary index's `(key ∪ include)` payload. Before returning a row, the engine pins the authoritative live slot and verifies the encoded key plus every INCLUDE null bit and value; inline rows require no full-document copy, while spilled rows require their overflow body for validation. The counter attributes leaf-derived result construction, not skipped document-page reads. A flat zero against rising `IndexSeeks` on a workload that declared `INCLUDE` columns means projections aren't actually a subset of `(key ∪ include)`; `EXPLAIN`'s `Notes` annotates per-query.

`EXPLAIN` surfaces "Covered (slot-validated)." in the `Notes` column on qualifying secondary equality, secondary range, index-order, compound equality, and compound range strategy rows whenever the runtime would serve the lookup from the covered path. PostgreSQL's index-only scan is useful precedent for storing projection payload in an index, but Mnemic's note does not mean `Heap Fetches: 0`: authoritative-slot validation is mandatory for every candidate.

Covered results are **value-faithful**: projected key fields return the STORED value, byte-identical to a heap read, even for types whose index key canonicalizes for comparison (string case, DateTimeOffset offsets, decimal trailing scale, float negative zero). `WHERE [Name] = 'BOB' SELECT [Name]` returns `Bob` as stored — never the folded comparison form — regardless of which plan served the query.

String range bounds use the same folded, terminated key encoding as equality. A qualifying INCLUDE range can reconstruct stored string casing while retaining authoritative slot validation; case folding does not change projected values.

A malformed scalar leaf key encountered by a covered range or index-order scan fails with `COVERED_KEY_CORRUPT` (`StructuralCorruption`). Malformed compound keys use `COVERED_COMPOUND_KEY_CORRUPT`. The affected index appears in `[Mnemic].[Health]`, and the engine quarantines ordinary MQL. These failures retain their classification during lazy result consumption and do not fall back to a heap scan. Rebuild the damaged index or restore a verified backup before resuming the engine.

Covered scalar and compound ranges, including index-order scans, prepare their leaf candidates before yielding reconstructed rows. Their retained candidate budget is `EngineOptions.MaxInMemoryStreamingState` (64 MiB by default), including copied key/INCLUDE arrays and allocated candidate capacity. Exceeding it returns `COVERED_STATE_LIMIT` (`ResourcePressure`) without quarantining the engine. Narrow the range, use an eligible unique-index LIMIT window, or raise that host option. An invisible candidate can require an uncapped rescan; the same budget still applies. This bound is local to covered range preparation and is not a process-memory limit.

## See also

- [Basic queries](queries.md) — `EXPLAIN` and the planner's strategy enum
- [Migrations](migrations.md) — `CREATE INDEX` ordering rules in migrations

## Detailed syntax and execution

After `ANALYZE`, compound-array probes compete by estimated posting and heap work with scalar indexes and a full scan. A single selective component may be cheaper than intersecting every matching component. `EXPLAIN` and execution use the same selected probes; full predicates still recheck each candidate.

Subtype-local renames rebind each affected compound-array component while preserving component order, uniqueness, INCLUDE fields, and logical index ID. Shared paths keep their names; the last declaration follows its rename. Removing a component's last declaration retires the entire index. Migration rollback restores the original complete definition and generation.


<a id="syntax-create-index"></a>

### CREATE INDEX

```sql
CREATE INDEX        ON [Ns].[Coll] ([Field]);
CREATE UNIQUE INDEX ON [Ns].[Coll] ([Field]);
CREATE INDEX        ON [Ns].[Coll] ([Field1], [Field2]);               -- compound
CREATE INDEX        ON [Ns].[Coll] ([Last, First], [TenantId]);        -- comma inside one bracketed field
CREATE INDEX        ON [Ns].[Coll] ([Field]) INCLUDE ([F1], [F2]);     -- covering (single-key or compound full-prefix)
CREATE INDEX        ON [Ns].[Coll] ([A], [B]) INCLUDE ([F1]);          -- covering on a compound index
CREATE INDEX        ON [Ns].[Coll] ([F]) WHERE [F] IS NOT NULL;        -- partial (sparse-column canonical shape)
CREATE INDEX        ON [Ns].[Coll] ([F]) WHERE [F] IS NULL;             -- partial (soft-delete sentinel)
CREATE INDEX        ON [Ns].[Coll] ([F]) WHERE [F] > 0;                 -- partial (range bound)
CREATE INDEX        ON [Ns].[Coll] ([F]) WHERE [F] IN ('Open', 'Pending');
CREATE INDEX        ON [Ns].[Coll] ([F]) INCLUDE ([G]) WHERE [F] > 0;   -- partial + covering
CREATE INDEX        ON [Ns].[Coll] ([Tags][]);                          -- array-element (GIN) index
CREATE UNIQUE INDEX ON [Ns].[Coll] ([Tags][]);                          -- each element belongs to one document
CREATE INDEX        ON [Ns].[Coll] ([F]) WITH (ONLINE = ON);            -- online build (write lock released)
CREATE UNIQUE INDEX ON [Ns].[Coll] ([F]) INCLUDE ([G]) WHERE [F] > 0 WITH (ONLINE = ON);
```

The optional managed Vamana vector provider adds an independent DDL form:

```mql
CREATE VECTOR INDEX [embedding_idx]
ON [Ns].[Coll] ([Embedding]) USING VAMANA
[WHERE predicate]
WITH (METRIC = COSINE, MAX_DEGREE = 64, BUILD_LIST = 100, ALPHA = 1.2,
      FILTER_FIELDS = ([TenantId]));

DROP VECTOR INDEX [embedding_idx] ON [Ns].[Coll];
```

Vector-index DDL commits independently and cannot run while a transaction or migration is active; settle the open scope first. DROP commits when the catalog definition is absent. Later page/delta cleanup failure is degraded health, and startup or a same-incarnation, same-name CREATE retries that reclamation. Index identity follows the collection incarnation rather than its mutable name, so rename plus old-name recreation can use the same vector-index name on both incarnations without replacement. Pending mutations are fenced by incarnation too; work on the recreated collection does not disable ANN on the renamed original.

`METRIC` is `COSINE`, `L2`, or `DOT`; defaults are cosine, degree 64, build list 100, and alpha 1.2. `MAX_DEGREE` is 2–512, `BUILD_LIST` is at least the degree and at most 1,000,000, and `ALPHA` is 1–4. Each option may appear at most once. `FILTER_FIELDS` accepts one through four stored scalar paths. A query can use that index only when its top-level conjunctive `WHERE` contains equality for every listed path; traversal is label-aware and core rechecks the full predicate. A partial vector index is selected only when its canonical predicate exactly equals the query predicate; an unfiltered index may serve filtered queries because core filtering and underfill fallback remain authoritative. This DDL is supplied by the plugin, not the scalar B+-tree index grammar. Execution repeats the same option validation, so a direct programmatic AST cannot bypass metric/range checks, finite `ALPHA`, `BUILD_LIST >= MAX_DEGREE`, or case-insensitive `FILTER_FIELDS` uniqueness. One rule owner reports the same code whether an option arrives as text or in a built statement: an option value outside its range is `INVALID_VALUE`, more than four `FILTER_FIELDS` is `LIMIT_EXCEEDED`, and a repeated filter field is `DUPLICATE_NAME`.

The provider bounds mutation capture, which retains the identities a transaction writes only to collections a vector index covers: writes to other collections count against no capture bound. An overflow quarantines the current engine with `PLUGIN_RECOVERY_REQUIRED`, withholds its clean-shutdown marker, and rebuilds from authoritative rows after reopen; it never permits a clean open to trust a graph that may have missed a committed source mutation. Post-commit reconciliation and delta-persistence failures use the same non-resumable recovery path. Consolidation runs on a background worker once an index's update overlay reaches its threshold, so a write commit does not wait for it unless the overlay reaches twice that threshold; a consolidation that fails before it publishes leaves the overlay and its durable deltas authoritative, reports a degraded `VAMANA_CONSOLIDATION_DEFERRED` fault and is retried by a later commit, while one that fails after publishing uses the recovery path.

Commas separate index fields only outside bracketed identifiers. `[Last, First]` is one field, so `([A,B], [C])` and `([A], [B,C])` identify distinct indexes through create, drop, and reopen.

Every explicit index field-path list accepts at most 1,024 paths, matching the maximum schema width. This covers key lists in `CREATE INDEX`, `ALTER INDEX`, and `DROP INDEX`, plus `INCLUDE` lists in `CREATE INDEX` and `ALTER INDEX`. Text parsing rejects the next comma before parsing another path; canonical printing and direct execution apply the same bound to programmatic ASTs before joining or copying the caller-provided list.

Ordinary key and `INCLUDE` paths must resolve to stored base-schema scalar fields. Whole objects/arrays, array-traversing scalar paths, polymorphic extension fields (including nested Object extensions), and virtual computed columns are rejected. Subtype extensions may omit a path or reuse its name with another type, so they have no single schema-wide physical encoding. A computed value is synthesized only after reading its source row when declared virtual, so a virtual value cannot be a physical key or covering payload. Stored computed values support ordinary key, covering, and predicate access paths. Use the special `([Field][])` / `([Items][].[Sub])` forms for supported array-element indexes.

A scalar `PINNED REFERENCES` field is a valid B-tree key. Its comparison key is the referenced id, so bindings to different revisions compare and seek as the same scalar value. A covering index that projects the pinned key retains the complete stored pin in its payload and therefore preserves the revision.

Every persisted `Ready` index definition is revalidated against the current schema before reopen publishes it. An unparseable filter or unsupported key/INCLUDE extraction is quarantined and detached from queries and writes, and its pages stay allocated; scans remain authoritative until `CREATE INDEX` replaces the quarantined row or `DROP INDEX` removes it.

An `UPDATE` or `REPLACE` that keeps a row's key in an index moves the row's entry to its new version in one step: the key belongs to the row throughout, so the write checks no uniqueness for it, and a concurrent write of that key by another row fails with `UNIQUE_VIOLATION` at once instead of waiting for the update's outcome. When an update changes an array-element index's elements but keeps some of them, the old row temporarily retains each kept element while its replacement's entry is inserted. The engine accepts that self-overlap only after the complete equal-key owner set contains no pointer other than the old and replacement rows; any foreign owner raises `UNIQUE_VIOLATION` before mutation. On a temporal collection the owner set holds every retained version of the key: the incoming version may join its own document's versions and versions of other documents whose validity does not overlap its own, and any other document's overlapping version raises `UNIQUE_VIOLATION` before mutation ([UNIQUE on a temporal collection](#unique-on-a-temporal-collection)). A version an uncommitted transaction is writing holds its key over its interval until that transaction ends, so a concurrent write of the key by another document over an overlapping interval fails without waiting. A write that meets another transaction's pending removal of its key — a `DELETE`, an `UPDATE` or `REPLACE` that moves the value away, or a temporal write that ends or removes the version holding it — waits for that transaction instead: it succeeds once the removal commits and fails with `UNIQUE_VIOLATION` once it rolls back. The wait is bounded by the lock-acquisition deadline of the `IndexMutationContext.RemovalClaim` site (`LOCK_TIMEOUT`), and explicit transactions that each wait for the other's removal end with `DEADLOCK` for one of them.

**Online build (`WITH (ONLINE = ON)`).** Allows concurrent DML during the long backfill — live writes maintain the building index while a concurrent scan reconciles existing rows, then the tree is swapped in atomically. Writers can pause briefly at the registration and publication cuts. The trailing clause follows any `INCLUDE` / `WHERE`. `WITH (ONLINE = OFF)` / omitting it is the default offline build. **Online index DDL is standalone: run it OUTSIDE a migration — it is rejected inside one, and the refusal ends the migration** (it commits independently, like PG CIC which can't run in a transaction block). Works for B-tree indexes (scalar, compound, unique, partial, covering) and array-element (`[Field][]`) indexes, including `ALTER INDEX ... INCLUDE ... WITH (ONLINE = ON)`. Slower than offline (per-row insert vs bulk-load) in exchange for write availability during the scan; correctness is identical. A `UNIQUE` online build still aborts on duplicate data, and live DML is checked against the hidden building tree: a conflicting write can receive `UNIQUE_VIOLATION` before the index becomes query-visible.

**Array-element (GIN) index.** `([Field][])` indexes an array of scalar (fixed-width, string, or bytes) elements; `([Items][].[Sub])` indexes a stored scalar sub-field within each `ARRAY OBJECT` element. Either array can be inside nested Objects, including polymorphic Object extensions: `([Profile].[Tags][])` and `([Profile].[Items][].[Sku])` retain the same membership semantics. Every subtype declaring the array must provide a stored scalar key domain; container and element offsets may differ. Null or absent containers add no scalar-value key. One entry per distinct element value, so the ANY-membership shapes hit the index instead of a full scan (all `EXPLAIN` → `Strategy = ArrayMembership`): `WHERE [Field][] = value` is a seek; `WHERE [Field][] IN (a, b)` unions the per-value seeks; and a one-sided `WHERE [Field][] >= lo` range-scans the ordered element keys — same three shapes on the nested form (`WHERE [Items][].[Sub] = value`). IN/range union + dedup (each matching doc once). Two comparisons on one array path remain independent existentials: `[Vals][] > 5 AND [Vals][] < 10` may use different elements, so the planner serves one compatible range side and leaves the other as a residual instead of collapsing both into a same-element interval. String elements lowercase-normalize like scalar string keys. The empty string is an ordinary element value (`IN ('', …)` / `= ''` seek its posting list); a listed value the element type can't parse (e.g. `'banana'` over `ARRAY INT64`) drops the complete predicate to the authoritative full scan, where an unparseable numeric arm is a nonmatch and parseable siblings still match. A **partial** `WHERE` filter is supported (`CREATE INDEX ON [C] ([Tags][]) WHERE [Published] = true`) — the planner uses it only when the query implies the filter, like scalar partial indexes.

For `BYTES`, literals are base64 and `''` is a present zero-length byte sequence. It is indexed and matches normally; a nullable nested `BYTES` leaf set to `NULL` contributes no scalar-value posting, so null and empty remain distinct with or without the index. Virtual computed element leaves are rejected because no physical value exists for maintenance or backfill.

Compound array indexes accept an ordered list of stored scalar paths and explicit array leaf paths. Postings are factored by path, so independent arrays do not cause Cartesian expansion. UNIQUE compares independent value domains: two rows conflict only when every corresponding domain shares a non-null value. On a temporal collection they must also be versions of different documents whose validity intervals overlap. A component with no non-null values owns no value tuple. Shared array ancestors do not imply same-element correlation. Query MATCH predicates retain their ordinary row-level correlation semantics. Equality, IN, ranges, negative comparisons, explicit NULL, ALL, and runtime IN-subquery probes combine scalar and array components. Each predicate unions typed values and coverage candidates; a bounded, spill-capable merge intersects complete row pointers across predicates. Separate predicates on the same array remain independent. ALL includes empty and null coverage before full row rechecks. Runtime membership and types whose physical ordering differs from comparison semantics evaluate distinct stored values through the ordinary scalar evaluator. These postings do not provide scalar compound-key ordering.

**Foreign-key** arrays are indexable (`WHERE [TagIds][] = id`; pinned FK arrays key on the id, so membership is revision-agnostic, including String/Bytes pins). Stored foreign-key leaves inside object elements are also indexable, including `[Items][].[Sub].[Ref]` through nested objects. Pins index the identifier; explicit `.Id` and `.Revision` paths can be indexed separately. This supports all seven identifier types, partial filters, uniqueness, reference actions, rollback, and reopen. Polymorphic elements and nested subtype-extension paths are indexable when every declaring subtype supplies a stored scalar type. Different declared types occupy separate ordered domains; probes normalize per domain and retain a full residual check. UNIQUE ownership is per declared type and canonical value (INT32 7 and INT64 7 have separate ownership). Each element uses its active subtype layout; a missing subtype field contributes no scalar-value posting.

Array indexes accept stored scalar base-row `INCLUDE` fields. Each distinct value posting and coverage posting carries the row's payload; included values do not affect element ordering or uniqueness. `ALTER INDEX ... ADD INCLUDE` and `DROP INCLUDE` preserve the array kind and work both offline and online. A single positive ANY equality on a homogeneous scalar domain can use a covered projection when every projected field is included and ordinary covered-plan eligibility holds. The visible source row must reproduce the element key and every payload value before reconstruction. The element key cannot reconstruct the complete array. Other query shapes retain ordinary row evaluation.

Repeated explicit boundaries are supported, including `[Matrix][][]` and `[Groups][].[Values][]`. One row contributes one posting per distinct leaf value across all inner arrays. Null, missing, or empty containers contribute no scalar-value postings. Collection subtype extensions can own the array or its Object containers. The active collection subtype supplies the complete source layout; an omitted array contributes no scalar-value posting. A scoped rename keeps a shared index at the old path for remaining declarations and rebuilds its entries. When the renamed subtype was the last declaration, the index follows the new path. Dropping the last declaring subtype retires the index. If the destination path is already indexed, its existing index retains its identity and rebuilds to include the renamed rows. These changes roll back with the schema; adding a stored scalar domain preserves index identity; invalid containers and virtual leaves are rejected before publication.

`ALL`, `!=`, `NOT IN`, and explicit `= NULL` / `!= NULL` can use the array index. These plans union value and coverage candidates and retain an authoritative residual predicate. Coverage preserves empty-array ALL semantics and distinguishes null scalar leaves from absent array containers. Lists must be fully probeable; null-bearing lists use the same scalar NULL policy as heap evaluation. IN matches a NULL leaf when the list contains NULL; NOT IN excludes it. Runtime subquery lists scan indexed values and probe the existing typed membership set once per distinct value, retaining its spill budget and the full row predicate as a residual.

**Unique array elements.** `CREATE UNIQUE INDEX ON [C] ([Tags][])` (or the stored object-element scalar form) permits repeated canonical values within a document but rejects any value owned by another document with `UNIQUE_VIOLATION`. Case folding, decimal normalization, signed-zero equality, and pinned-id comparison apply before de-duplication. Null/missing/empty arrays and null nested leaves claim no scalar-value ownership; empty strings and bytes do. Partial uniqueness applies only to matching documents. Updates/replacements/upserts retain their own keys, release removed keys, and roll back every element on failure. Offline and online builds, reopen, and rebuild preserve uniqueness. Hidden online builds can reject conflicting writes before publication. On a temporal collection ownership is per instant of valid time, as for scalar unique indexes. An array index cannot be an `ON CONFLICT` target: different elements could identify different documents; primary-key or ordinary unique-index upserts still enforce its constraint.

**Partial-index implication.** The planner uses a partial index only when the query's WHERE *logically implies* the index's WHERE on a per-predicate basis with the complete Boolean structure preserved. Partial predicates support `NOT`, `AND`, `OR`, and parentheses over `= / != / < / <= / > / >= / IN / NOT IN / STARTSWITH / CONTAINS / LIKE / NOT LIKE / IS NULL / IS NOT NULL`. Limits are 1,024 scalar atoms, 4,096 compiled instructions, and 64 nested parentheses/NOT levels. Membership requires TRUE; UNKNOWN stays UNKNOWN under NOT. Every OR branch must be covered. Shared equalities can drive a seek while the full OR predicate remains a residual before pagination. ANALYZE enables value-aware ranking of shared equality indexes; EXPLAIN follows the same choice. `IS NOT NULL` indexes are matched by the **strictness rule** — any null-rejecting predicate (`=`, range, `IN`, `STARTSWITH`, `CONTAINS`, `LIKE`, `NOT LIKE`) on the same field implies it (informed by PostgreSQL's strong- implication and `clause_is_strict_for` machinery, then narrowed for MQL). Note: `!=` / `NOT IN` do NOT imply `IS NOT NULL` in MQL because both can match a null field. A non-null `!= x` partial filter includes null rows. It is implied by IS NULL, the same inequality, equality/IN excluding x, NOT IN containing x, or a range that excludes x. STRING pattern queries (STARTSWITH, CONTAINS, LIKE, NOT LIKE) also imply inequality/exclusion sets when the production matcher rejects every forbidden literal. ESCAPE and Unicode folding retain ordinary query semantics. Failed literal coercion never proves inequality. `NOT IN (S)` excludes each member, including NULL when listed. A NULL-bearing NOT IN query implies IS NOT NULL. Membership proofs use disjoint sets; negative membership uses reversed set containment. Empty strings remain real members. STRING fields also support STARTSWITH and CONTAINS with non-null literals. Proofs use the evaluator's case folding: equality/IN must satisfy the pattern; stronger prefixes/substrings must contain the weaker one in the required position. Empty patterns match every non-null string and are implied by IS NOT NULL. LIKE and NOT LIKE also support non-null patterns with optional ESCAPE on STRING fields. Both reject null fields. Equality/IN use the production matcher; prefix, substring, and pattern queries use Unicode scalar language containment, reversing containment for negative patterns. The same proof also matches LIKE/NOT LIKE queries to STARTSWITH/CONTAINS index filters when their whole language fits; overlapping matches alone do not suffice. Identical patterns are recognized directly. Other proofs use at most 1,024 UTF-8 bytes and 256 tokens per pattern, 4,096 reachable state pairs, and 65,536 transitions; exhaustion selects a complete access path without changing membership. A partial filter's `MATCH`, parameter refs, array paths, composite fields, virtual computed fields, and polymorphic extension fields (including nested Object extensions) are rejected at DDL; `OR`, `NOT` and parentheses are accepted ([partial-index WHERE grammar](#partial-index-where-grammar)). Implication compares literals in the field's evaluator domain, including integer-valued decimal spellings (`20.0` on INT32), Boolean/Guid alternate spellings, and the engine's date/time coercions. An uncoercible typed literal makes the proof unknown and forces a full scan; typed comparisons never fall back to lexical UTF-8 order because that could incorrectly treat `"20.0"` as greater than `"100"` and omit rows from a partial-index query. `IN` implication uses the evaluator's cached canonical set (expected linear time) and preserves empty members, so `IN ('', 'Open')` covers equality on the empty string as well as `Open`. Partial maintenance evaluates stored row bytes, so a computed value synthesized after materialization cannot gate physical membership. An `ORDER BY`-only query cannot imply a partial filter and sorts the complete collection instead of scanning a tree that omits rows. `EXPLAIN` applies the same gate. Secondary field paths are case-insensitive after an optional `Body.` prefix is stripped, including range-scan direction and LIMIT early-exit attribution.

Scalar key components of a compound array index may also appear in INCLUDE, which copies their value onto every component posting. Scalar B-tree keys are already covered. Array traversal paths cannot be scalar payloads, and INCLUDE paths must be distinct case-insensitively.

Subtype-local renames rebind each array component and preserve component order. Shared paths keep their names; the last declaration follows its scoped rename. Removing any component's last declaration retires the whole index. Rollback restores its complete ordered keys, INCLUDE fields, identity, and generation.

<a id="syntax-drop--alter-index"></a>

### DROP / ALTER INDEX

```sql
DROP INDEX  ON [Ns].[Coll] ([Field]);
DROP INDEX  ON [Ns].[Coll] ([F1], [F2]);                       -- compound
ALTER INDEX ON [Ns].[Coll] ([Field]) ADD  INCLUDE ([NewIncl]);  -- add to covering payload
ALTER INDEX ON [Ns].[Coll] ([Field]) DROP INCLUDE ([Incl]);     -- drop from covering payload (empty = non-covering)
ALTER INDEX ON [Ns].[Coll] ([Field]) ADD  INCLUDE ([NewIncl]) WITH (ONLINE = ON);  -- online rebuild (standalone, not in a migration)
```

`ALTER INDEX … (ADD|DROP) INCLUDE` is the only mutation that doesn't require a `DROP INDEX` + `CREATE INDEX` round-trip — key fields are preserved, only the per-pointer payload is rewritten. A replacement is published only after a complete rebuild, so failure leaves the old index authoritative. The rebuild preserves the original UNIQUE flag and partial-filter `WHERE` clause (the replacement contains entries only for rows that satisfy the persisted predicate).

Covered fast path qualifies on **single-key OR compound full-prefix equality** when every projection lives in `(key ∪ include)` — the single-column range / ORDER-BY path also qualifies for reversible order-preserving key types. The runtime reconstructs rows from leaf data only after verifying the encoded key and every INCLUDE value against the authoritative live slot. Inline rows avoid a full-document copy; spilled rows still require their overflow body for validation. This is leaf-derived reconstruction, not a promise that document-page reads were skipped. `EXPLAIN` annotates the row's `Notes` with `"Covered (slot-validated)."` when it would fire, and `[Mnemic].[Metrics].CoveredIndexSeeks` counts engine-wide covered hits. Excluded shapes (the runtime falls through to the slot-read path): JOIN, GROUP BY / aggregate, OR-groups, `Meta.*` projection, temporal `AS OF` / `IN BETWEEN`, unsupported range-key types, and partial-prefix equality. Covered results are value-faithful: projected key fields return the STORED value (string casing, DateTimeOffset offset, decimal scale, float sign preserved), never the canonicalized comparison-key form — plan choice never changes result bytes.

<a id="syntax-17-expiry-policies"></a>

### Expiry policies

`CREATE INDEX ON [Ns].[Collection] ([Timestamp]) WITH TTL '7 DAYS'` attaches an elapsed lifetime to one stored scalar DATETIME key. `WITH EXPIRY` uses the timestamp itself as the deadline (zero duration). NULL never expires. A TTL is a non-negative decimal followed by `ticks`, `ms`, `s`, `m` (minutes), `h`, `d`, or the corresponding English singular/plural unit, in any case and with or without spaces before it (`'30d'`, `'30 DAYS'`). It must fit exactly in signed 64-bit 100-nanosecond ticks without rounding during parsing or unit conversion. Redundant zeros are accepted, and the string is limited to 128 characters. Months, years, and sub-tick durations are rejected.

Partial WHERE, INCLUDE, and UNIQUE keep their ordinary semantics. Compound, array-element and virtual keys cannot define an expiry policy. A stored computed DATETIME field can define expiry using the materialized value. The expiry and online options follow INCLUDE / WHERE and can appear in either order, once each. Offline creation requires a migration; online creation is standalone. DROP INDEX removes the policy; changing it requires drop/recreate. The policy survives reopen, rename, and INCLUDE rebuilds; migration abort restores the previous definition, including expiry when a collection or named-type field drop removed the index.

One background worker per MQL engine attempts up to 128 candidates per index per pass, by default every second. It rechecks current membership, timestamp, schema, and policy identity under the write claim before ordinary DELETE. Deletion is eventual; rejected rows do not starve later candidates. BEFORE/ AFTER triggers, foreign keys, sealing, and materialized views still apply. Temporal expiry closes the current version and preserves history. The worker uses an independent database-owned MQL executor; the persisted policy authorizes maintenance without retaining the DDL author's session. Engine-only hosts must initialize MQL to run expiry. `EngineOptions.ExpirySweepInterval = null` disables automatic sweeps; shutdown cancels and joins them. Inspect nullable `[Mnemic].[Indexes].[ExpireAfterTicks]` for the persisted duration.

Bracket reserved names such as `[Expiry]` and `[Ttl]` when using them as identifiers.

`TRIGGER`, `BEFORE`, `AFTER`, `FOR`, `EACH`, `ROW` are in use by the [trigger DDL](triggers.md#syntax-create--drop-trigger). Both timings execute transactionally with immutable row images and failure-based veto.

## Usage pitfalls

<a id="pitfall-partial-index-predicates-must-be-pure-and-time-invariant"></a>

### Partial-index predicates must be pure and time-invariant

A `CREATE INDEX … WHERE <predicate>` filter is persisted in the catalog and re-evaluated at insert time, and the planner only uses the index when a query *implies* the filter. So the predicate must be pure: parameter references (`@name`) **and** `NOW` are rejected — a `NOW` filter would drift with the clock and desync the planner's implication check from what's actually indexed. Use only literal right-hand sides. Compilation uses the live target schema for scalar and array-element partial indexes. Unsupported shapes return validation errors, but any direct or wrapped structural-corruption proof fails closed with its stable code and quarantines its fault domain instead of becoming an "invalid partial filter" message.

Boolean partial filters preserve UNKNOWN under NOT. `NOT ([State] = 20)` and `NOT ([State] IN (20))` include null State values because equality, and `IN` as a disjunction of equalities, is two-valued, exactly as `NOT IN (20)` does; `NOT ([State] > 20)` excludes them because an ordered comparison with NULL stays UNKNOWN under NOT. Parentheses preserve grouping; NOT binds above AND, which binds above OR. Proof failure selects a complete access path.

A partial filter `[State] != 20` includes null/missing State values. A UNIQUE index with this filter therefore also constrains those rows. `IS NOT NULL` does not imply the filter because State could still equal 20. Equivalent literal spellings use the field's logical domain for containment proofs. String patterns may imply != or NOT IN when every forbidden value fails the query pattern. For example, LIKE 'Open%' implies != 'Closed', while NOT LIKE 'Open%' does not. ESCAPE and case folding apply to this check. A partial filter `NOT IN (NULL, 'Closed')` includes present values other than Closed. Without the NULL member, NOT IN also admits NULL rows. Empty-string members remain distinct from NULL.

Partial-index STARTSWITH and CONTAINS require STRING fields and reject null operands. Empty patterns include every non-null string, including empty strings. A query with a broader prefix or substring does not imply a narrower filter, even when the current rows happen to fit both.

Negated STARTSWITH/CONTAINS can use escaped NOT LIKE proofs. Literal `%`, `_`, and escape markers in the prefix/substring remain literal, including bound query parameters. Null query patterns and null fields do not become matches.

LIKE/NOT LIKE partial filters preserve ESCAPE through rename and reopen. NOT LIKE rejects null fields even when its pattern does not match them. A query must imply the pattern over all possible values; identical text is sufficient but not necessary. A LIKE query can also imply a STARTSWITH or CONTAINS filter; the two operators need not match. Complex containment proofs can reach the bounded planner work limit and select a full scan; membership still uses the ordinary LIKE matcher. Check EXPLAIN for the chosen path.

Predicate fields must be stored base-schema scalars. Array traversal, composites, polymorphic extension fields (including nested Object extensions), and virtual computed fields are rejected. Subtype extensions may omit the path or reuse its name with a different type, so no one physical predicate domain is valid across every row; the maintenance gate reads persisted row bytes and cannot safely derive a computed value before deciding physical index membership.

<a id="pitfall-online-index-ddl-is-standalone--do-not-wrap-it-in-a-migration"></a>

### Online index DDL is standalone — do NOT wrap it in a migration

Index DDL normally **requires** a migration (`CREATE INDEX` outside `BEGIN MIGRATION` / `COMMIT MIGRATION` errors "requires a migration context"). `WITH (ONLINE = ON)` **inverts** that rule: an online build commits independently and is **rejected inside a migration**, which the refusal ends (like PostgreSQL `CREATE INDEX CONCURRENTLY`, which can't run in a transaction block). So:

```sql
-- offline: MUST be in a migration
BEGIN MIGRATION 3;
CREATE INDEX ON [App].[Orders] ([CustomerId]);
COMMIT MIGRATION;

-- online: MUST be standalone (no migration wrapper)
CREATE INDEX ON [App].[Orders] ([CustomerId]) WITH (ONLINE = ON);
```

Online reconciliation has a per-build distinct-pointer limit derived from `EngineOptions.MaxInMemoryStreamingState` at an estimated 256 bytes per pointer (minimum one). Exceeding it reports `ResourcePressure / ONLINE_BUILD_CAPACITY_EXCEEDED` for the build and triggering write; increase the host budget, reduce concurrent churn, or use an offline build.

`WITH (ONLINE = ON)` works for B-tree and array-element (`[Field][]`) indexes, including `ALTER INDEX ... INCLUDE ... WITH (ONLINE = ON)`. The clause goes last, after any `INCLUDE` / `WHERE`.

Online means writers remain available during the long backfill, not that they can never wait: registration and publication each require a brief drained cut. During a unique online build, live DML is already checked against the hidden building tree, so a conflicting write can receive `UNIQUE_VIOLATION` before queries are able to use the index.

The finished tree is authoritative once the atomic swap succeeds. A later failure while reclaiming the displaced tree cannot make the DDL unpublished; the engine contains the unreachable-page leak and exposes any structural proof through `[Mnemic].[Health]` for its exact domain. Inspect health and follow the domain-specific recovery path—do not retry the DDL as though the old tree were still live.

<a id="pitfall-rewrites-and-indexes-share-the-migrations-rows"></a>

### Rewrites and indexes share the migration's rows

Rewriting `ALTER COLLECTION` and offline `CREATE INDEX` / `ALTER INDEX` can follow DML or earlier rewrites of the same collection in one migration. Scans include dirty pages and private appended rows, including large spilled bodies. The owner sees each new schema and its rows immediately; commit or abort settles the whole block. Competing owners remain fenced until settlement. Growing rows may allocate new pages; rebuilt indexes use those private page identities until commit, and abort restores the original rows and indexes.

```sql
BEGIN MIGRATION 4;
INSERT INTO [App].[Docs] ([Id]: 2, [Name]: 'x');
ALTER COLLECTION [App].[Docs] ADD FIELD [Score] INT32 DEFAULT 7;
UPDATE [App].[Docs] SET ([Score]: 9) WHERE [Id] = 2;
CREATE INDEX ON [App].[Docs] ([Score]);
COMMIT MIGRATION;
```

Standalone raw-engine rewrites still reject a collection with an open competing transaction. Settle that owner before rewriting its rows.

After any query-visible field or subtype schema change is staged, the collection's schema+row+index generation stays fenced until that migration commits or rolls back. The owning migration may issue DML, queries, and FK probes afterward in the new metadata/layout without waiting on its own fence; other writers receive a transient structural-write error and should retry after the migration settles. Committed readers wait/fail bounded rather than observing the private generation. Competing index or schema DDL likewise waits or fails before touching the shared generation. Subtype-only mutations advance both collection and extension-schema versions so cached plans cannot retain old extension fields. A second migration trying to mutate the same collection fails before mutation; disjoint-collection rewrites remain concurrent. Queries submitted after the fence is already held wait before plan lowering and bind the newly published generation. A lazy result created before the ALTER but not enumerated until afterward fails closed instead of reusing its old descriptor against new row bytes, with the retryable `Transient` / `SCHEMA_GENERATION_CHANGED` pair (see [errors](errors.md#classified-error-surface)).

DML statements use the complementary rule: they admit before pinning one exact schema generation and retain it through validation, matching, mutation, and RETURNING snapshot capture. A rewriting ALTER therefore waits rather than publishing between serialization and storage. INSERT captures its accepted body, UPSERT captures its merged body or conflict-probe row, UPDATE / REPLACE captures exact post-images, and DELETE captures exact pre-delete images. RETURNING never re-reads the primary key after that decisive operation, so a concurrent writer cannot substitute or remove the reported row. Explicit `Meta.*` fields come from that same slot snapshot, not a later read. It may be enumerated after a later ALTER because the row snapshots and projection schema were already captured under the statement generation.

<a id="pitfall-index-pointers-never-override-where-truth"></a>

### Index pointers never override WHERE truth

Index entries select candidate rows; they are not authoritative query results. The engine rechecks every predicate an access path consumed after materializing the row, in both buffered and streaming execution. A direct PK read also compares the resolved slot key with the requested key. Body-less temporal/FK existence and pinned-revision probes compare the slot-header PK before trusting validity or revision metadata, so a pointer to another live row cannot satisfy a reference. Covering-index equality and range plans also verify the encoded leaf key plus every INCLUDE value against the authoritative live slot before returning a leaf-projected row. A disagreement raises `INDEX_ROW_PREDICATE_MISMATCH`, quarantines the affected index or collection fault domain, and returns no row. An index pointer whose page/slot cannot name a live heap row raises `INDEX_POINTER_INVALID` once exact pending insert state and concurrent write/relocation races have been excluded. Temporal invisibility and a locator known to belong to an unsettled eager insert may be skipped; an impossible or stable tombstoned locator never becomes an empty or truncated successful result. Do not retry or interpret it as an empty result: restore or rebuild from a known-good source before resuming the affected collection.

<a id="pitfall-and-and-index-usage"></a>

### `AND` and index usage

Compound-array intersections can lose to a scan when a small `MaxInMemoryStreamingState` budget makes spill I/O expensive. A single exact component probe streams directly; verify the selected path with `EXPLAIN`.

Separate scalar indexes can combine as `BitmapAnd` when fresh statistics show that intersection costs less than an individual path or heap scan. EXPLAIN lists its input indexes. A compound prefix cannot omit an unconstrained nullable tail: rows with any NULL component are absent from that index. ANALYZE records full-collection component statistics separately from compound correlation. Full compound equalities use the requested tuple's measured frequency, so skew can favor a full scan even with an applicable index. A full-scan scalar equality conjunction counts equivalent predicates on one field once and estimates zero for conflicting values. Remaining independent fields use their actual probe frequencies; independence is an assumption, not a bound on the actual result. String ranges use folded, terminated keys, preserving strict prefixes. Bitmap capacity exhaustion safely selects the heap before any output escapes; page order still requires sorting for a user `ORDER BY`.

<a id="pitfall-or-and-index-usage"></a>

### `OR` and index usage

Same-field equality ORs rewrite to `IN`, and an `IN` list on a field with a single-column B-tree index is served as a union of exact seeks (`SecondaryInList`): one seek per distinct member key, a row read once however many members name it, and the complete predicate rechecked on every row. Without statistics it follows every equality, range and index-ordered path in the rule order, so an `ORDER BY` the same index serves keeps its ordered scan; with fresh statistics it is cost-ranked on the sum of its members' equality estimates and loses to a sequential scan when it would read most rows. It does not apply to a list with a NULL member (a scalar index holds no NULL key), to `NOT IN`, to a member that names no key of the field (a fractional number for an integer field, text that is no number), to a partial index the query does not imply, to an array-element index, or to a temporal collection; those lists read the collection. A cross-field OR where EVERY branch is a single equality on an indexed field (PK or non-partial single-column secondary) is served by a BitmapOr index merge — each side seeks its index and the pointer sets union before one heap fetch, unless fresh statistics show that the merged heap reads cost more than one sequential pass; that shape uses `FullScan`. EXPLAIN consumes the same cost verdict, so it never advertises a rejected bitmap/equality/range path. A cost-selected `FullScan` still reports the rejected candidate's filtered-output `EstimatedRows`; scan cost reading the whole heap is a separate quantity. Its `Notes` distinguishes a bitmap-only rejection from a global scan verdict. `BitmapOr` sums value-aware per-side row estimates and caps the union at the collection row count. For wider OR lists, proven repeated secondary probes within the runtime cache's entry and key-byte limits contribute once to both estimated rows and index work. Every authored branch still undergoes validation. Uncached duplicates remain conservatively priced, as do inputs from the first primary-key branch or unknown probe evidence onward. Any missing or stale input statistics still make the union estimate unknown. Any other OR shape (range predicates inside OR, multi-predicate branches, unindexed fields, partial indexes) falls back to a full scan — verify with `EXPLAIN`.

An `ORDER BY`-only query also never scans a partial index: without a WHERE clause that implies the index filter, doing so would silently drop every row excluded from that tree. Expect a complete scan plus sort in that shape.

<a id="pitfall-field-copies-and-compound-skip-scans"></a>

### Field copies and compound skip scans

- `[Target]: [Source]` copies the original row's field in UPDATE SET. Use `[Target]: ([TRUE])` for a field whose name can look like an array value; `[TRUE]` alone is still a one-element array. Parenthesized expressions and nested documents are distinct: a nested document starts with `[Field]:` or is empty `()`.
- A non-leading compound predicate can select `CompoundSkipScan` after ANALYZE. Unbound components must exclude null, including through nullable object parents. Partial-index implication still applies. EXPLAIN names the full index and search allowance. Exhausting searches or candidate bitmap memory falls back to the heap before output; this path makes no ORDER BY or covering promise. `COMPOUND_KEY_CORRUPT` fails closed on invalid search-key framing, component counts, or typed encodings (including malformed UTF-8 String keys) and quarantines the affected index before any row is published.

### Temporary sort-file lifetime

A drained external-sort reader transfers temporary-file deletion to the operating system. An observer may briefly retain the pending filename; this does not fail an otherwise successful query. Unread runs and failed closes retain explicit cleanup ownership, and actual cleanup failures remain errors.
