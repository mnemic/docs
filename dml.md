# DML — INSERT, UPDATE, REPLACE, DELETE, SEAL, UNSEAL, CANCEL VERSION

> **In this section** — every data-modifying statement, plus the concurrency-control modifiers (`EXPECTED REVISION`, `SKIP IF UNCHANGED`, `FORCE WRITE`).

## INSERT

```
INSERT INTO [Namespace].[Collection]
  [SUBTYPE [discriminator_value]]
  ([Field]: value | DEFAULT, ...)
  | ( ([Field]: value | DEFAULT, ...) [<row clauses>], ... )
  [FORCE ID]
  [VALID FROM <instant> [TO <instant>]]
  [ON CONFLICT [(target)] (DO NOTHING | DO UPDATE SET (...))]
  [CREATEDAT <instant>]
  [CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>]
  [RETURNING ...]
```

> **Clause order is fixed.** Every clause is optional, and those present follow the order above: `SUBTYPE`, the document or the row list, `FORCE ID`, `VALID`, `ON CONFLICT`, `CREATEDAT`, `CREATEDBY`, `RETURNING`. `<row clauses>` are `[VALID FROM <instant> [TO <instant>]] [CREATEDAT <instant>] [CREATEDBY ...]`, in that order. The parser rejects any other ordering, such as `VALID FROM` before `FORCE ID`. Each `<instant>` is an ISO-8601 string literal, `NOW`, an `@parameter` or an [instant expression](#instant-expressions-in-writes).

> **One INSERT lists up to 1,000 documents, in parentheses.** `INSERT INTO [Ns].[C] (([Id]: 1), ([Id]: 2))` inserts both; the bare `([Id]: 1), ([Id]: 2)` is a syntax error. Every clause after the list belongs to the statement and applies to each row that names no clause of its own ([multi-row INSERT](#multi-row-insert)). A `VALUES` list and `INSERT … SELECT` / `INSERT … FROM` are parse errors; to copy rows from a query, read them and insert them.

> **`NOW` is statement-pinned.** Every `NOW` inside one statement resolves to the same instant (the SQL Server / PostgreSQL `CURRENT_TIMESTAMP` convention): all fields of one INSERT store one timestamp, and an UPDATE / REPLACE matching many rows stamps every row with the same statement timestamp — not a per-row clock read.

```sql
-- Simple insert
INSERT INTO [Store].[Products] (
  [Id]: '550e8400-e29b-41d4-a716-446655440000',
  [Name]: 'Widget',
  [Price]: 29.99,
  [Active]: TRUE
);

-- With nested objects and arrays
INSERT INTO [Facility].[Hubs] (
  [Id]: '...',
  [Name]: 'Hub One',
  [Location]: ([City]: 'Boston', [Zip]: '02101'),
  [Connectors]: [
    ([Type]: 'CCS', [Power]: 150),
    ([Type]: 'CHAdeMO', [Power]: 50)
  ]
);

-- Temporal insert with validity period (temporal collections only).
-- VALID FROM/TO on non-temporal collections is rejected with an error.
-- A bound at '0001-01-01T00:00:00Z' is INVALID_VALUE: tick zero marks absent
-- or open-ended validity metadata (the same applies to DELETE ... AT,
-- CANCEL VERSION AT, and UPDATE PREVIOUS VALID TO).
INSERT INTO [HR].[Contracts] ([Id]: '...', [Salary]: 75000)
VALID FROM '2024-01-01T00:00:00Z' TO '2024-12-31T23:59:59Z';

-- Temporal insert starting now (open-ended)
INSERT INTO [HR].[Contracts] ([Id]: '...', [Salary]: 80000)
VALID FROM NOW;

-- Polymorphic insert (required on polymorphic collections)
INSERT INTO [Fleet].[Vehicles] SUBTYPE [car] (
  [Id]: '...', [Make]: 'Toyota', [Year]: 2024, [Doors]: 4, [Sunroof]: TRUE
);

-- Polymorphic insert with nested polymorphic sub-document
INSERT INTO [App].[Items] (
  [Id]: '...',
  [Component]: SUBTYPE [Solar] ([Power]: 100, [Efficiency]: 0.22)
);

-- Explicit PK on auto-increment collection (data migration)
INSERT INTO [HR].[Employees] ([Id]: 999, [Name]: 'Imported') FORCE ID;

-- Row-independent expressions in value positions (functions, CASE,
-- arithmetic, unary minus) are evaluated at insert time
INSERT INTO [Store].[Products] (
  [Id]: '...', [Name]: UPPER('widget'), [Stock]: - 5
);
```

### Multi-row INSERT

An INSERT may list several documents where the single-document form has one: a row list in parentheses, its rows separated by commas, `INSERT INTO [Ns].[C] ((…), (…))`. A list of one row is allowed. The statement has the outcome of one single-document INSERT per row, executed in row order, all in one transaction:

- **The statement's clauses apply to every row.** One `SUBTYPE` (a statement inserts one subtype; send a statement per subtype), one `FORCE ID`, one `ON CONFLICT` action and one RETURNING list cover all the rows, and the statement's validity window, `CREATEDAT` and `CREATEDBY` are every row's defaults; `NOW`, the default `CREATEDAT` and the default `VALID FROM` are the statement's one instant.
- **A row may name its own revision metadata.** After its document, inside the list, a row may name `VALID FROM <instant> [TO <instant>]`, `CREATEDAT <instant>` and `CREATEDBY ...`, spelled and ordered as the statement's clauses; each replaces the statement's for that row. The validity window is replaced as a whole, so a row naming `VALID FROM` alone is open-ended whatever the statement's `TO`. On an attributed collection a statement whose rows all name `CREATEDBY` needs none of its own; when it names none, a row naming none is refused as its single INSERT would be. Row `n` then behaves as its single INSERT with those clauses, so `DO UPDATE` writes the row's timestamp and actor and `DO NOTHING` keeps the existing row's. A row's clauses are resolved and checked when its document is admitted — a missing parameter, a value that names no instant, tick zero, an inverted window, a window on a standard collection, `VALID TO` beside `DO UPDATE`, an actor the collection's attribution refuses — so their refusal starts with `INSERT row <n>: `; those of a list's only row are the statement's. An instant that reads a field is `INVALID_STATEMENT`, as everywhere in a write.
- **Rows are written in document order, and each sees the earlier ones.** A later document may reference an earlier one through a foreign key. A later document that repeats an earlier one's primary key or UNIQUE value violates it (`UNIQUE_VIOLATION`); with `ON CONFLICT` it conflicts with the earlier row instead: `DO NOTHING` skips it and `DO UPDATE` updates the row the earlier document wrote, `EXCLUDED` reading the later document. On a temporal collection a later version of the same key splices the timeline the earlier one started.
- **Triggers and maintenance run per row.** Each row's BEFORE triggers, write, incremental view maintenance and AFTER triggers complete before the next row starts, as for a multi-row UPDATE ([triggers](triggers.md#timing-and-validation)).
- **RETURNING yields one row per document, in document order** — the inserted row, or for a conflict the row `DO NOTHING` found or `DO UPDATE` produced. A polymorphic `RETURNING *` lists every subtype's fields, as a multi-row UPDATE's does. Without RETURNING the affected count is the sum of the rows': `DO NOTHING` rows count zero.
- **All or nothing.** Every document is admitted (defaults, generated `SEQUENTIAL` keys, expressions, field, NOT NULL, enum and CHECK validation) before the first is written, so a statement refused at admission writes nothing; an auto-increment key is assigned at its row's write, as for a single INSERT. A refusal of any row rolls back every row of the statement, with its triggers' effects; inside `BEGIN TRANSACTION` it aborts the transaction, as every failed DML statement does ([transactions](transactions.md#a-failed-statement-aborts-the-transaction)). The error keeps the single statement's code and its message starts with `INSERT row <n>: `, `n` counting the documents from 1 ([errors](errors.md#multi-row-insert-refusals-name-their-row)).
- **Outside a transaction the statement is one implicit transaction.** It commits once, after its last row, with one durability barrier, and appends to the data page the session's previous statement left open, as single-row statements do. A transient conflict before the commit retries the whole statement; a row of an `ON CONFLICT` statement that loses its insert race to a concurrently committed row fails the statement with `UNIQUE_VIOLATION`, as an UPSERT inside a transaction does, instead of re-probing. A cancellation before the commit rolls every row back; one after it reports the committed statement.

```sql
-- Three orders, one round trip, one commit
INSERT INTO [Sales].[Orders] (
  ([Id]: 1, [Customer]: 'A', [Total]: 10),
  ([Id]: 2, [Customer]: 'B', [Total]: 20),
  ([Id]: @id3, [Customer]: @customer3, [Total]: @total3)
)
RETURNING [Id], [Meta].[CreatedAt];

-- Idempotent import: rows whose key already exists are skipped
INSERT INTO [App].[Items] (([Id]: 1, [Name]: 'a'), ([Id]: 2, [Name]: 'b'))
ON CONFLICT DO NOTHING;

-- Imported rows keep their source creation times and actors; the
-- last row takes the statement's actor and NOW
INSERT INTO [App].[Items] (
  ([Id]: 1, [Name]: 'a') CREATEDAT @created1,
  ([Id]: 2, [Name]: 'b') CREATEDAT '2024-06-20T06:28:08Z' CREATEDBY 22,
  ([Id]: 3, [Name]: 'c')
)
CREATEDBY 2;

-- Imported versions, each with its own validity window
INSERT INTO [Sales].[Prices] (
  ([Sku]: 'A1', [Price]: 10) VALID FROM '2024-01-01T00:00:00Z' TO '2024-07-01T00:00:00Z',
  ([Sku]: 'A1', [Price]: 12) VALID FROM '2024-07-01T00:00:00Z'
);
```

A statement lists at most 1,000 documents (`LIMIT_EXCEEDED` when it is parsed), binds at most 65,535 parameters and holds at most 65,536 tokens, as every statement does ([syntax](syntax.md#strings)), so a parameterized row clause such as `CREATEDAT @c0` costs its row one parameter and two tokens, and its documents' encoded bodies total at most 16 MiB (`LIMIT_EXCEEDED` before any row is written). Static `EXPLAIN INSERT` admits every row with the clauses it names and reports the count in `EstimatedRows` and `Notes`.

**Value expressions** — a value position accepts any row-independent expression (function calls, `CASE`, arithmetic, `@parameters`, `NOW`); it is evaluated once at insert time and the result stored. Field references (`[Other] + 1`) are rejected with an error — there is no existing row to read from on the insert path (use `UPDATE … SET` for row-derived values).

Assignment expressions may start with a literal or parameter, such as `([Qty]: 1 + @delta)`, without parentheses around the RHS. UPDATE also accepts row-derived expressions such as `([Name]: 'prefix' || [Name])`. The parsing rule applies to every SET clause; each verb retains its row-reference restrictions.

**Temporal behavior** — on temporal collections, INSERT writes the document as a version valid from `VALID FROM` (NOW when omitted) to `VALID TO` and splices the timeline around it, so no two versions of the document overlap: a version it partly covers is truncated or split, and the uncovered remainder keeps that version's body and revision; a version it fully covers is replaced. Without `VALID TO` the inserted version is open-ended and therefore replaces every version that starts after its `VALID FROM`; give `VALID TO` to overlay one interval and keep the later history, or use `UPDATE ... VALID FROM` to change fields of the version valid at an instant. An INSERT onto a key that already has a timeline is an UPDATE event for [triggers](triggers.md#temporal-collections), and `CREATEDBY COPY` copies from the version valid at the new `VALID FROM`. On non-temporal collections, `VALID FROM` / `VALID TO` is rejected with an error — temporal parameters are never silently discarded.

**Field validation** — all field names and types are validated against the collection schema before writing. Unknown fields are rejected. Type mismatches (e.g., document literal for a scalar field) are rejected. Explicit NULL on NOT NULL fields is rejected (including null parameter values), and a row-independent expression that evaluates to NULL is checked again after evaluation. A CHECK that references an auto-increment primary key runs after the engine assigns that key and before write admission, so it sees the exact stored candidate rather than the zero assignment sentinel. An INSERT gives each field it leaves out, at every depth, the default its declaration holds, as every write gives the fields left out of an object it assigns ([defaults](#defaults)), and an omitted required field must acquire a non-null value from that default or an engine-generated value. Validation covers the complete resulting row, including provided nested objects, each array element, and the active subtype extension. A nullable absent parent exempts its children. UPDATE validates the merged row; REPLACE validates the complete replacement. Binary client writes obey the same required-field boundary.

Document/array literal nesting is capped at 100 levels for both text and the public programmatic AST. Deeper values reject before validation, printing, or execution can recurse over them.

Each array literal contains at most 32,768 elements. Bracket contents are tokenized independently; this is the largest comma-separated scalar list that fits the 65,536-token inner lexer, so a longer literal is `LIMIT_EXCEEDED` for its token count. Direct AST validation and serialization reject larger counts before element traversal or capacity calculation.

**FORCE ID** — stores an explicit PK value on an `INT32` or `INT64` auto-increment collection. The PK must be present; zero and negative values are valid forced identities. Without `FORCE ID`, an omitted or explicit-zero PK requests engine assignment and any other explicit value is rejected with `INVALID_VALUE`, with or without `ON CONFLICT`: an upsert refuses it before its conflict probe, so the refusal does not depend on whether a row holds that key. On a non-auto-increment collection (including any Guid PK), `FORCE ID` is rejected because the collection already accepts its PK directly.

The omitted/zero generation request also applies with `ON CONFLICT`: a stored ID zero created with `FORCE ID` does not conflict with an ordinary generated insert. Adding `FORCE ID` instead requests conflict handling for that exact ID.

Once the write path accepts a forced value, the generator makes a monotone reservation beyond it when possible. A lower value never regresses the high-water mark, and forcing the PK type's ceiling exhausts later generated assignment: a generated insert past the ceiling is refused with `INVALID_VALUE` (PostgreSQL reports a sequence at its limit as class 22), and the counter keeps its value. Reservations are not rolled back: a later validation failure or transaction rollback can leave a gap. This is the same nontransactional identity-allocation policy used by ordinary auto-increment inserts and keeps a concurrent allocator from reusing an observed identity. `EXPLAIN INSERT … FORCE ID` reports the bypass and its monotone, saturating reservation.

The reservation is process-local until the forced row or generator high-water mark becomes durable. After an unclean restart, a failed or rolled-back reservation that left no durable evidence may be forgotten. `FORCE ID` is not a standalone durable sequence-reservation operation.

### INSERT … ON CONFLICT (UPSERT)

`ON CONFLICT` resolves a primary-key or `UNIQUE`-index collision instead of failing the INSERT.

```
INSERT INTO [Namespace].[Collection]
  ([Field]: value | DEFAULT, ...)
  ON CONFLICT [([Field], ...)] (DO NOTHING | DO UPDATE SET ([Field]: value | DEFAULT, ...))
  [CREATEDAT <instant>]
  [CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>]
  [RETURNING ...]
```

```sql
-- Idempotent insert — duplicate PK silently no-ops
INSERT INTO [App].[Items] ([Id]: 1, [Name]: 'Widget', [Qty]: 5)
ON CONFLICT DO NOTHING;

-- Upsert on PK — collision triggers an UPDATE with the supplied SET list
INSERT INTO [App].[Items] ([Id]: 1, [Name]: 'Widget', [Qty]: 5)
ON CONFLICT DO UPDATE SET ([Qty]: 5, [Name]: 'Widget');

-- Upsert on a secondary UNIQUE index — must match a registered UNIQUE
-- index exactly (single-column or compound). Auto-increment PK is fine
-- because the conflict probe uses the unique target rather than the PK.
INSERT INTO [Shop].[Users] ([Email]: 'a@x.com', [Name]: 'A')
ON CONFLICT ([Email]) DO UPDATE SET ([Name]: 'A');

-- Compound UNIQUE conflict target (matches a UNIQUE INDEX on (Tenant, Slot))
INSERT INTO [Shop].[Tickets] ([Tenant]: 't1', [Slot]: 'A', [Qty]: 1)
ON CONFLICT ([Tenant], [Slot]) DO UPDATE SET ([Qty]: 999);
```

The optional `(target)` clause names the conflict target. Omitting it implies the primary key. `DO NOTHING` skips the insert silently and reports zero affected rows. `DO UPDATE SET (...)` applies the listed assignments to the existing row and reports one affected row. The SET list is evaluated by the same expression engine as a standalone UPDATE, so RHS expressions referencing the existing row are permitted. `EXCLUDED.[Field]` reads the proposed INSERT value after defaults and row-independent expressions are evaluated. Combine both scopes, for example `SET ([Qty]: [Qty] + EXCLUDED.[Qty])`. Nested paths and schema-aware functions such as `ENUM_LABEL(EXCLUDED.[State])` retain the proposed field's type metadata. An unknown EXCLUDED path fails before probing, including on an insert miss. An array element path is no assigned value, in this SET list or an UPDATE's (`SET ([Name]: [Tags][])`, `SET ([Code]: EXCLUDED.[Tags][])`): it is rejected with `FEATURE_NOT_SUPPORTED` before any row is read ([element paths read no single value](queries.md#array-element-paths-read-no-single-value)). An explicit conflict target accepts at most 1,024 fields, matching the maximum stored-schema width. The parser rejects the next target before constructing it, and programmatic AST validation enforces the same bound before duplicate checks. The proposed INSERT document is still fully validated on a conflict hit, including unknown fields, NOT NULL/enum/CHECK rules, even for `DO NOTHING`. Defaults, generated primary-key placeholders, row-independent expressions, and NOW are materialized once before a secondary-target probe; the exact effective value probed is the value used by the eventual insert. A NULL unique target does not invent a conflict. Under contention, a losing insert re-probes the conflict target and resolves the declared action against the winner; a standalone statement re-probes a bounded number of times, after which it reports the last lost insert's `UNIQUE_VIOLATION`. A transient secondary-index ownership conflict during the probe or conflict action is retried by the same bounded standalone-statement policy, and one statement's re-probes and retries share that one bound. Each re-probe and retry counts in the server-side retry totals of [`[Mnemic].[Metrics]`](system-collections.md#virtual-collections), a re-probe as a conflict. `ON CONFLICT` therefore does not leak a `UNIQUE_VIOLATION` or `TX_CONFLICT_FWW` merely because another session committed between probe and action. `DO UPDATE` re-evaluates its SET expressions against the current row claimed by each engine retry. Explicit transactions retain their caller-owned retry boundary. If the target collection has read-time computed columns, the secondary-unique probe may be streaming; UPSERT consumes that stream before deciding hit versus miss. A streaming result is never interpreted as an empty probe, so the declared conflict action still applies to the existing row. Its schema generation is frozen before the stream is consumed, so row production cannot change how the matched primary key or conflict image is decoded.

**Conflict-target shapes.** Two are supported: the primary key (the default) and a registered `UNIQUE` secondary index. The target must align exactly with one of these — listing fields that match a *non-unique* secondary index, or fields with no backing index, both fail with `"ON CONFLICT (...): the index on this target is not UNIQUE."` / `"...: no index covers the target..."`. An array element path (`ON CONFLICT ([Tags][])`) is no conflict target (`FEATURE_NOT_SUPPORTED`): a UNIQUE array index keeps elements distinct across rows, but a conflict must resolve to one row. The unique-index requirement is structural: a non-unique target could match multiple rows and the conflict handler can't pick one deterministically.

Nested scalar keys are accepted, for example `ON CONFLICT ([Address].[Email])`; compact `[Address.Email]` normalizes to that spelling. Key values are resolved from the effective proposed document, including nested values and defaults. For a partial UNIQUE index, both the proposed row and the existing row must satisfy its WHERE predicate to conflict. An excluded row with equal key fields does not cause DO NOTHING or DO UPDATE; an excluded proposal is inserted normally.

**Polymorphic collections.** The proposed INSERT requires `SUBTYPE [value]`. On a miss, that subtype is inserted. On a conflict, DO UPDATE preserves the existing row's subtype and applies SET to its base and extension fields; EXCLUDED reads the proposed subtype's values even when the two subtypes differ. Assignments to fields absent from the existing subtype fail atomically. DO NOTHING with RETURNING captures the existing row, including its metadata and any requested extension field or envelope, in the conflict probe.

**Temporal collections.** A temporal conflict is a version valid at the proposed `VALID FROM` (NOW when omitted): both conflict-target shapes probe the document as of that instant, and `DO UPDATE` then behaves as `UPDATE ... VALID FROM` that instant — it supersedes the conflicting version, keeps its end, and leaves the earlier and later versions alone. When no version is valid at that instant, the INSERT runs instead, with the ordinary [temporal INSERT](#insert) splice. `VALID TO` cannot be combined with `DO UPDATE` and is rejected with `FEATURE_NOT_SUPPORTED` whichever outcome the probe would choose, because the updated version ends where the superseded one ends; `DO NOTHING` and a plain INSERT keep it.

**Conflict-target value validation.** The PK value in the literal is range- and integrality-checked against the PK type *before* the conflict probe runs: a fractional value (`[Id]: 1.5` on an integer PK) or an out-of-range value (`[Id]: 5000000000` on an `INT32` PK) fails the whole statement with an error — it is never truncated or wrapped into a probe that could match (and `DO UPDATE`) a different row. This matches the plain-INSERT coercion rules for integer fields.

## UPDATE

```
UPDATE [Namespace].[Collection]
  SET ([Field]: value | DEFAULT, ...)
  {WHERE expression | CONFIRM UPDATE ALL}
  [VALID FROM <instant>]
  [CLEAR VALID TO]
  [EXPECTED REVISION n|@param]
  [SKIP IF UNCHANGED | FORCE WRITE]
  [CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>]
  [RETURNING ...]
```

> `CLEAR VALID TO` makes the new version open-ended instead of ending where the version it supersedes ends, so, like an open-ended INSERT, it replaces every later version. `SKIP IF UNCHANGED` and `FORCE WRITE` are mutually exclusive; when both are omitted, the collection's `SkipUnchangedUpdates` default applies.

The SET list is admitted before any row is read: a field the collection does not declare (`UNKNOWN_FIELD`), the primary key or a polymorphic discriminator (`FEATURE_NOT_SUPPORTED`), a NULL for a NOT NULL field (`NOT_NULL_VIOLATION`) or an unknown enum label is refused whether or not `WHERE` matches a row, and static `EXPLAIN UPDATE` reports the same pair, as it does for INSERT's document, REPLACE's SET list, an `ON CONFLICT DO UPDATE` list and a `RETURNING` column.

**Partial update** — only fields in SET are modified; all other fields are preserved, including the difference between a present empty Object (`()`) and NULL. This also applies to fields retained by `ON CONFLICT DO UPDATE`, whether the assignments patch fixed-width fields or rebuild the document. An object, or a list of objects, that SET assigns is the field's whole new value, built with the [defaults](#defaults) of the fields it leaves out, as an INSERT builds one.

```sql
-- Non-temporal: modifies the document in-place
UPDATE [Store].[Products]
SET ([Price]: 39.99)
WHERE [Id] = '550e8400-e29b-41d4-a716-446655440000';

-- Temporal: creates a new version, previous version auto-ended
UPDATE [Billing].[Tariffs]
SET ([Name]: 'Premium')
WHERE [TariffId] = 1;

-- Temporal with explicit start time for the new version
UPDATE [Billing].[Tariffs]
SET ([Name]: 'Premium')
WHERE [TariffId] = 1
VALID FROM '2025-06-01T00:00:00Z';

-- Optimistic concurrency
UPDATE [App].[Items] SET ([Name]: 'Bob') WHERE [Id] = 1 EXPECTED REVISION 2;

-- Mass update (safety keyword required)
UPDATE [App].[Items] SET ([Status]: 'archived') CONFIRM UPDATE ALL;

-- Set field to null
UPDATE [App].[Items] SET ([Notes]: NULL) WHERE [Id] = 1;
```

```sql
-- Update a polymorphic sub-document (SUBTYPE required for polymorphic fields)
UPDATE [App].[Devices]
SET ([Component]: SUBTYPE [Solar] ([Manufacturer]: 'NewCorp', [Power]: 300))
WHERE [Id] = '...';

-- Update a polymorphic array field (replaces entire array)
UPDATE [App].[Plants]
SET ([Components]: [
  SUBTYPE [Solar] ([Manufacturer]: 'Sun', [Power]: 500),
  SUBTYPE [Wind] ([Manufacturer]: 'Air', [Speed]: 80)
])
WHERE [Id] = '...';
```

**Temporal behavior** — on temporal collections, UPDATE authors one new version and keeps the rest of the history. Its target instant is `VALID FROM`, or the statement's NOW when omitted. `WHERE` matches each document by its version valid at that instant, and that version is the one the update supersedes: SET expressions, CHECK constraints, `EXPECTED REVISION`, `SKIP IF UNCHANGED`, `CREATEDBY COPY` and a trigger's `OLD` all read it. The new version runs from the target instant to the superseded version's end, the superseded version is cut to end at the target instant, and every other version is left as it was. A document with no version valid at the target instant is not matched. With `VALID FROM` at instant `t`:

| `t` lies | Result |
|---|---|
| before the first version | no match, 0 rows |
| inside a past version `[p, q)` | that version becomes `[p, t)`; the new version is `[t, q)`, built from it; later versions stay |
| at a version's start | the new version replaces that version over its whole interval; pinned references to the replaced revision apply their removal action |
| inside the current version | the current version ends at `t`; the new version runs to the current version's end |
| inside a scheduled version `[s, e)` | that version becomes `[s, t)`; the new version is `[t, e)`, built from it |
| in a gap left by `DELETE ... AT` or `CANCEL VERSION` | no match, 0 rows; resume the timeline with `INSERT ... VALID FROM` |

With `CLEAR VALID TO` the new version is `[t, ∞)` from the version valid at `t` (no match when there is none) and replaces every later version. An UPDATE without `VALID FROM` is dated to its statement's NOW; when a later version took effect while the statement ran, as another statement's concurrent update does, the update supersedes that version at its own start instead, so the current state keeps both changes. Two scheduled UPDATEs therefore give the same timeline in either order; see [scheduled pricing](examples.md#scheduled-pricing). On non-temporal collections, UPDATE modifies in-place.

**Safety** — `UPDATE` requires either a `WHERE` clause or `CONFIRM UPDATE ALL`. Omitting both is a parse error. Primary key fields cannot be modified via SET.

**Atomicity** — multi-match UPDATE without an explicit `BEGIN TRANSACTION` is wrapped in an implicit engine transaction — a mid-iteration failure (FK re-validation, temporal timeline conflict) rolls back all earlier mutations. Single-match UPDATE keeps the per-doc mini-transaction path. Each SET merge is derived after the engine claims and verifies the row — on a temporal collection, the version it supersedes — and is re-evaluated on transient write retries. RHS field references use that original containing row, and referenced virtual computed fields are synthesized before evaluation. A declared `Body` Object retains its own descendants in this scope; reading a virtual descendant does not populate its stored virtual slot. Concurrent updates to disjoint fields therefore preserve both changes, and concurrent `[Count] + 1` updates do not retry stale whole-document bytes. CHECK/FK validation, `EXPECTED REVISION`, and `SKIP IF UNCHANGED` apply to that retry-local merged row for both standard and temporal collections. An exact-primary-key temporal UPDATE resolves both row existence and the superseded version inside that same retry attempt, so an eager temporal-index re-key cannot become a false zero-row result. A genuinely missing row still returns affected count zero without evaluating unrelated SET fields.

## Partial array mutations

UPDATE SET and ON CONFLICT DO UPDATE SET accept ordered `ALTER [name] (...)` entries for append, positional insertion/removal/replacement, predicate removal, object element updates, and nested array edits. See [Editing arrays](arrays.md#editing-arrays) for syntax and strict NULL/index semantics. These assignments use the same temporal validity, conflict, transaction, and RETURNING rules as other SET assignments.

## REPLACE

```
REPLACE [Namespace].[Collection]
  SET ([Field]: value | DEFAULT, ...)
  WHERE expression
  [EXPECTED REVISION n|@param]
  [SKIP IF UNCHANGED | FORCE WRITE]
  [CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>]
  [RETURNING ...]
```

**Full document replacement** — a top-level field not in SET becomes NULL without its default, so leaving out a NOT NULL field is refused with `NOT_NULL_VIOLATION` even when it declares a default (`Field 'Qty' is NOT NULL and cannot be omitted on '[App].[Items]'.`); `[Field]: DEFAULT` gives it the default ([DEFAULT](#default)), and an object SET assigns is built with the [defaults](#defaults) of the fields it leaves out. Always in-place (no new temporal version). Primary key is automatically preserved from the existing document.

On temporal collections the replacement revision is greater than every retained revision of that identity, including scheduled future versions and writes earlier in the same transaction. The validity interval is preserved. If its old revision has no remaining fragment, pinned references apply their declared `ON REMOVE` policy: `Restrict` rejects the replacement and `SetNull` clears only pins to that exact revision in the same transaction. `SKIP IF UNCHANGED` does not retire a revision or run those actions. The replacement cannot introduce a self-pin to a revision it removes. Pins created by BEFORE UPDATE triggers also receive the removal policy. A `Restrict` rejection rolls back both the replacement and those trigger writes.

```sql
-- Minimal replace (WHERE is mandatory)
REPLACE [App].[Items] SET ([Name]: 'Bob') WHERE [Id] = 1;

-- Optimistic concurrency — fails if the stored revision is not 2
REPLACE [App].[Items]
SET ([Name]: 'Bob', [Score]: 10)
WHERE [Id] = 1
EXPECTED REVISION 2;

-- Skip if unchanged (no write, no revision bump)
REPLACE [Store].[Products]
SET ([Name]: 'Widget', [Price]: 29.99, [Active]: TRUE)
WHERE [Id] = @id
SKIP IF UNCHANGED;

-- Force a write even if the collection's default is skip-unchanged
REPLACE [Store].[Products]
SET ([Name]: 'Widget', [Price]: 29.99, [Active]: TRUE)
WHERE [Id] = @id
FORCE WRITE;

-- Polymorphic REPLACE — base NOT NULL fields validated before query,
-- subtype extension fields validated per-document after reading discriminator
REPLACE [Fleet].[Vehicles]
SET ([Make]: 'Toyota', [Year]: 2024, [Doors]: 5, [Sunroof]: FALSE)
WHERE [Id] = @id;
```

**Safety** — `REPLACE` always requires a `WHERE` clause (no `CONFIRM REPLACE ALL`).

**Atomicity** — multi-match REPLACE without an explicit `BEGIN TRANSACTION` is wrapped in an implicit engine transaction — a per-subtype extension NOT NULL validation that passes for one match but fails for another (different discriminator) won't leave the first match already replaced on disk.

**UPDATE vs REPLACE** — `UPDATE` is a merge (preserves unmentioned fields). `REPLACE` is a full overwrite (unmentioned top-level fields become NULL, taking no default). On temporal collections, `UPDATE` creates a new version; `REPLACE` modifies in-place.

For collection-level subtypes, UPDATE, REPLACE, and UPSERT conflict assignments validate nested values against the target row's active extension. A same-named object in another subtype may have different children, enum values, or nullability. A literal valid for one declaration is admitted, then each matched row must accept it under its own subtype. Invalid active-subtype assignments fail atomically instead of dropping unknown children. A path into a polymorphic object's subtype fields (`[Part].[C]: …`) works the same way at every depth, inside a list's elements and a collection subtype's fields too: the statement is admitted when one subtype of the object declares the path and accepts the value, and each row's object takes the value under the subtype it holds — its type, its shape, its nullability and its default for `DEFAULT` — whatever another subtype declares under the same name.

### Subqueries in DML predicates

DML `WHERE` resolves source fields using the same declaration rules as a query. `[Body].[Code]` addresses a real `Body` field when the base schema or any subtype declares that root, including when the requested child is absent. Otherwise `Body` is the optional document prefix. Native comparisons and predicates that need subquery or scalar evaluation use the same rule; adding a scalar conjunct does not change which field a native comparison reads.

Static `EXPLAIN` validates a write's statement-level clauses (its document or SET list, conflict action and RETURNING list) as execution does before reading a row, and its extended write predicates and their view/subquery schemas without evaluating them. A `DeferredPredicate` plan leaves the write access path and cardinality unresolved until predicate execution. `EXPLAIN ANALYZE` runs the mutation through its ordinary validation and policy gates and reports the affected count. See [query introspection](queries.md#explain--query-plan-introspection).

`UPDATE`, `REPLACE`, `DELETE`, `SEAL`, `UNSEAL`, and `CANCEL VERSION` accept the same `IN (FROM …)`, scalar, `EXISTS`, and `NOT EXISTS` forms in `WHERE` as a query. Non-correlated subqueries execute once per statement. Correlated forms are evaluated through the query executor first, then lowered to the exact matched primary-key set before the write phase. A zero-row correlated result is an explicit no-match predicate and never degrades into an unfiltered write. Native comparison predicates use the ordinary DML path directly; scalar and negated predicates retain candidate-row evaluation and normal validation. If the target schema changes after values or candidate identities were prepared, the mutation fails before writing; retry the statement against the current schema.

Views referenced by these subqueries are inlined transitively, including view-on-view chains. Plugin read gates see both the subquery/view name and the underlying base collections before any mutation starts; the target write still passes through `IWriteGuard` independently.

```sql
DELETE FROM [Sales].[Orders]
WHERE [Id] IN (FROM [Sales].[ExpiredOrderIds] SELECT [Id]);

UPDATE [Sales].[Customers] SET ([Status]: 'active')
WHERE EXISTS (FROM [Sales].[Orders]
              WHERE [CustomerId] = [Sales].[Customers].[Id]);
```

## DELETE

```
DELETE FROM [Namespace].[Collection]
  {WHERE expression | CONFIRM DELETE ALL}
  [AT <instant>]
  [RETURNING ...]
```

```sql
-- Filtered delete
DELETE FROM [Store].[Products] WHERE [Active] = FALSE;

-- Delete all (safety keyword required)
DELETE FROM [Store].[Products] CONFIRM DELETE ALL;

-- Temporal: end document at a specific time
DELETE FROM [HR].[Contracts] WHERE [Id] = '...' AT '2025-06-01T00:00:00Z';

-- Temporal: end document now (explicit)
DELETE FROM [HR].[Contracts] WHERE [Id] = '...' AT NOW;
```

**Temporal behavior** — on temporal collections, DELETE ends the document at `AT` (now when omitted) and keeps its history: `WHERE` matches documents by their current version, the version valid at the end instant is closed there, and every version scheduled to start at or after it is cancelled as `CANCEL VERSION` cancels it — pinned references to a cancelled revision apply their removal action, and referential actions run once for the absence that begins at the end. A version that has already taken effect is never removed, so an `AT` at or before the current version's `ValidFrom` is rejected with `INVALID_VALUE`; so is an `AT` after the document's last version has ended, because DELETE only shortens a lifetime — extend one with `INSERT ... VALID FROM` or `UPDATE ... VALID FROM ... CLEAR VALID TO`. On non-temporal collections, `AT` is rejected with an error — temporal syntax is never silently discarded.

**AT preflight** — when `AT '<timestamp>'` is specified, every matching document's `ValidFrom` is checked up-front. If `AT` is at or before any match's `ValidFrom` (which would create an inverted validity window), the statement errors without mutating anything.

**Foreign keys** — removal actions compare complete target identifiers. String references use invariant simple-lowercase identity, including empty String identifiers. `Restrict`, `Cascade`, and `SetNull` apply identically with or without an index on the referencing field.

**Atomicity** — when `WHERE` matches more than one document and no explicit `BEGIN TRANSACTION` is in progress, the executor wraps every per-document delete in a single implicit engine transaction. A mid-iteration failure (FK Restrict on the Nth match, for example) rolls the wrapper back so earlier deletes never land on disk. Single-match DELETE and explicit-transaction DELETE keep the per-doc mini-transaction path. If an earlier FK cascade already removed another matched candidate, that confirmed row-absence signal is skipped. Any unrelated engine failure remains an error and is never reported as a successful lower affected count.

**Safety** — bare `DELETE FROM [col]` without WHERE or CONFIRM is a parse error.

## RETURNING

Every DML statement (INSERT / UPDATE / REPLACE / DELETE / SEAL / UNSEAL) accepts an optional `RETURNING` clause that emits the affected rows back as a result set instead of (or in addition to) the affected-count acknowledgement. `RETURNING *` returns every field; an explicit projection `RETURNING [F1], [F2]` returns only the named fields. A statement with `RETURNING` always returns a result set with that projection's schema; when WHERE matches nothing, or every matched identity is a no-op (an already sealed `SEAL` target), the result set is empty rather than an affected-count acknowledgement, so the result kind never depends on the data. An explicit `RETURNING` list accepts at most 1,024 fields, matching the maximum result-schema width. The limit is enforced before parsing or validating another field; `RETURNING *` remains the single-token all-fields form.

```sql
-- Insert and read the assigned auto-increment Id back
INSERT INTO [HR].[Employees] ([Name]: 'Imported') RETURNING *;

-- Update with a projected subset
UPDATE [App].[Items] SET ([Qty]: [Qty] + 1) WHERE [Id] = 1
RETURNING [Id], [Qty];

-- Capture deleted rows before they're gone (e.g. archive workflow)
DELETE FROM [App].[Items] WHERE [Status] = 'Trash'
RETURNING [Id], [Name];

-- REPLACE returns the post-replace row by default; same projection rules apply
REPLACE [App].[Items] SET ([Name]: 'Widget', [Qty]: 7) WHERE [Id] = 1 RETURNING *;

-- Seal metadata is the post-toggle state; no-op identities emit no row
SEAL [Archive].[Records] WHERE [Id] = @id RETURNING [Id], [Meta].[Sealed];
```

`RETURNING` is evaluated against the post-mutation document for INSERT / UPDATE / REPLACE and `ON CONFLICT DO UPDATE`, against the conflict-probe document for `ON CONFLICT DO NOTHING`, and against the pre-deletion document for DELETE. SEAL/UNSEAL return the unchanged selected body with post-toggle metadata. The result schema is the schema a SELECT projecting the same paths publishes: it carries the schema version of the written collection's generation, and a foreign-key column carries its target collection's identity and no name, which `[Mnemic].[Collections]` names as the target is called now, also after the target was renamed (`RETURNING *` included). A field the list names that the target does not declare is rejected before any row is written, with the diagnostic and code a read of the same path reports (`RETURNING on '[App].[Items]': Column 'Nope' does not exist. …`, `UNKNOWN_FIELD`). An array element path (`RETURNING [Items][]`) is rejected as it is in SELECT, before any row is written, with `FEATURE_NOT_SUPPORTED` ([element paths read no single value](queries.md#array-element-paths-read-no-single-value)). Each name is a path of the written row, relative to it: unlike in WHERE, a collection qualifier is not consumed, so `RETURNING [Items].[Code]` on `[App].[Items]` names a field `Items` (and is rejected when the target declares none), and `[Body]` is optional shorthand unless the target declares a `Body` field. Use `RETURNING [Meta].[Revision]` to capture the revision the engine assigned when you need it for a subsequent `EXPECTED REVISION` round-trip.

Polymorphic `RETURNING` is a flattened, single-schema result. An explicit extension projection, and every extension field retained by `RETURNING *`, must therefore have one structurally identical schema across the subtypes that declare that path. An incompatible same-named extension is rejected during RETURNING preflight, before the statement mutates any row. Base and extension fields retain the distinction between NULL and present empty Bytes or Objects (`()`). An extension absent from a row's active subtype remains NULL when that field is part of the result schema. Projected pinned references retain both the identifier and revision, including empty String or Bytes identifiers, in both base and extension fields.

Every DML statement pins one exact collection schema generation across field validation, source matching, mutation, and RETURNING snapshot capture. A concurrent rewriting `ALTER` waits for that boundary; it cannot publish between serialization and write admission. INSERT captures the exact accepted body after auto-increment assignment, each row's for a multi-row INSERT; UPSERT captures either the exact merged body or the row that won its conflict probe. Multi-row UPDATE/REPLACE captures each exact post-image inside the mutation path, and DELETE retains each exact pre-delete image. No RETURNING arm re-reads the primary key after the decisive probe or mutation, so a concurrent later writer cannot substitute its value or remove the reported row. Each snapshot also carries the exact virtual `Meta` values from that same slot (`Revision`, validity window, `CreatedAt`, `CreatedBy`, and `Sealed`); metadata is never reconstructed by rereading the collection. The snapshots are complete before the boundary is released, so delaying result enumeration until after a later ALTER still returns bytes and a result schema from the original statement generation.

Statement cancellation is rechecked after an UPSERT conflict probe and before `DO NOTHING` or `DO UPDATE` acts. A cancellation that wins in that interval returns the normal statement-cancelled error and cannot apply the update.

## Defaults

A field's declared default (`DEFAULT <literal>` or `DEFAULT NOW`, [declared with the field or by SET DEFAULT](ddl.md#defaults-at-creation)) is the value a write gives the field when it builds a value whole and leaves the field out. An INSERT builds its document whole, and every write builds whole each object, or list of objects, it assigns as a field's value: in an INSERT's document, an UPDATE's or an `ON CONFLICT DO UPDATE`'s SET list, a REPLACE's SET list, the SET list of `CANCEL VERSION … UPDATE PREVIOUS`, and the elements an [array edit](arrays.md#editing-arrays) adds or the objects its element `UPDATE … SET` assigns. Each field such a value leaves out takes its default at every depth: inside nested objects, in the objects of a list and of a list of lists at every level, and among the fields a polymorphic value's `SUBTYPE` adds. On a collection whose subtypes declare the same object differently, an UPDATE, a REPLACE and a conflict update build it with the declaration of each matched row's own subtype, and a path into a polymorphic object builds the value it assigns with the declaration of the subtype that object holds in each row. A nullable object the value leaves out is not built, so its children take no default; an explicit value, NULL included, takes precedence over a default; and a NOT NULL field left out is refused with `NOT_NULL_VIOLATION` only when it declares no default. `NOW` in a default is the statement's pinned time, as every `NOW` of the statement is. The defaults belong to the candidate the statement admits, so static `EXPLAIN`, a trigger body's validation at `CREATE TRIGGER` and execution judge the same document, array edits included, before a row is read, and a trigger's `NEW` holds them.

A SET list itself is not built whole. An UPDATE merges into the stored row, so the fields its SET list leaves out keep their stored values. A REPLACE overwrites the row in place and keeps no earlier version, so the top-level fields it leaves out become NULL without their defaults: a default given silently there would lose a forgotten value with no history to recover it from, and a `NOW` default would restamp its field on every replacement.

```sql
-- [Box] OBJECT ([A] INT32, [B] INT32 NOT NULL DEFAULT 9), [N] INT32 NULLABLE DEFAULT 5
UPDATE [App].[Items] SET ([Box]: ([A]: 2)) WHERE [Id] = 1;                -- [Box].[B] is 9, [N] keeps its value
REPLACE [App].[Items] SET ([Name]: 'b', [Box]: ([A]: 2)) WHERE [Id] = 1;  -- [Box].[B] is 9, [N] is NULL
```

### DEFAULT

`[Field]: DEFAULT` gives a field its declared default as the value a write assigns it: in an INSERT's document, the SET lists of UPDATE, REPLACE, `ON CONFLICT DO UPDATE` and `CANCEL VERSION … UPDATE PREVIOUS`, an array edit's element `UPDATE … SET` and the objects its values add, at the top level and at any depth of their object literals, a path assignment (`[Box].[B]: DEFAULT`) and a polymorphic value's `SUBTYPE` fields among them. On a collection whose subtypes declare a field differently, each matched row takes its own subtype's default, and a path into a polymorphic object (`[Part].[C]: DEFAULT`) takes the default of the subtype that object holds in each row. A field that declares no default takes NULL, so DEFAULT on a NOT NULL field without one is refused with `NOT_NULL_VIOLATION`, and a `NOW` default is the statement's pinned time. On a primary key the engine generates (`AUTOINCREMENT` or `SEQUENTIAL`), an INSERT's DEFAULT asks for the generator as leaving the key out does, and `FORCE ID`, which carries an explicit key, refuses it with `INVALID_STATEMENT` as it refuses a missing key. DEFAULT is how a REPLACE gives a top-level field its default.

DEFAULT is resolved when the write builds its value, before the statement admits it, so everything downstream reads the value it names: static `EXPLAIN` admits or refuses it as execution does, `EXCLUDED.[Field]` reads the default a proposed row's DEFAULT gave, `SKIP IF UNCHANGED` skips a write whose DEFAULT gives a field the value it already holds, a temporal REPLACE rewrites its version in place, and a trigger's `NEW` holds the default. A trigger body may write DEFAULT itself; its stored text keeps the keyword, which resolves each time the body runs. No parameter carries DEFAULT: a bound parameter is always a value, so DEFAULT is written in the statement text.

DEFAULT is a field's whole value and nothing else. As the primary key or a polymorphic discriminator of a SET list it is refused with `FEATURE_NOT_SUPPORTED`, as assigning those any value is, and on a computed field with `INVALID_STATEMENT`. Anywhere else — an operand (`DEFAULT + 1`), inside parentheses, a function argument, a `CASE` result, an array element, a predicate or `IN` value, a projection, `CREATEDBY`, an instant, a `PIN` half or a declared default — it is refused with `FEATURE_NOT_SUPPORTED` at its position: `DEFAULT is valid only as a field's whole value in a written document or SET list ([Field]: DEFAULT); it is not a value inside an expression, an array, a predicate or a declared default.` A bracketed `[DEFAULT]` after a field's colon is an array holding DEFAULT, refused as such; copy a field named DEFAULT with `([DEFAULT])`.

```sql
-- [Qty] INT32 NOT NULL DEFAULT 1, [Stamp] DATETIME NULLABLE DEFAULT NOW, [Box] OBJECT ([A] INT32, [B] INT32 NOT NULL DEFAULT 9)
UPDATE [App].[Items] SET ([Qty]: DEFAULT, [Stamp]: DEFAULT) WHERE [Id] = 1;
REPLACE [App].[Items] SET ([Name]: 'b', [Qty]: DEFAULT) WHERE [Id] = 1;
INSERT INTO [App].[Items] ([Id]: 2, [Name]: 'c', [Box]: ([A]: 1, [B]: DEFAULT));
UPDATE [App].[Items] SET ([Box].[B]: DEFAULT) WHERE [Id] = 2;
```

## Concurrency control

These three modifiers apply to UPDATE and REPLACE, after `WHERE`: `EXPECTED REVISION` first, then at most one of `SKIP IF UNCHANGED` and `FORCE WRITE`, which exclude each other. `EXPECTED REVISION` also applies to SEAL and UNSEAL; the other two modifiers do not. None applies to `UPDATE PREVIOUS` inside `CANCEL VERSION`, where each is a `SYNTAX_ERROR`.

### EXPECTED REVISION

Optimistic concurrency — fails with `User / REVISION_MISMATCH` if the document's current revision doesn't match (or its current version no longer exists). A WHERE clause that matches more than one document fails with `CARDINALITY_VIOLATION` before writing:

```sql
UPDATE [App].[Items] SET ([Name]: 'v2') WHERE [Id] = 1 EXPECTED REVISION 1;
REPLACE [App].[Items] SET ([Name]: 'v2') WHERE [Id] = 1 EXPECTED REVISION 1;
```

`EXPECTED REVISION` is per-document. The statement is rejected at execution time if `WHERE` matches more than one document — a single revision number can't be meaningfully asserted across a multi-row mutation, and a partial-match loop would either fail mid-iteration (leaving earlier documents already mutated) or silently require every match to share that revision. Use a PK-specific `WHERE` or drop `EXPECTED REVISION`.

The precondition applies to the document the `WHERE` clause matches: a `WHERE` clause that matches no document writes nothing and reports 0 affected rows rather than `REVISION_MISMATCH`, while a matched document whose current version is gone by the time the write claims it fails with `REVISION_MISMATCH`. With `FORCE WRITE`, which writes even an unchanged document, 0 affected rows therefore means the document was not there to match.

### SKIP IF UNCHANGED

Skip the write if the new data is binary-identical to existing data:

```sql
UPDATE [App].[Items] SET ([Name]: 'same') WHERE [Id] = 1 SKIP IF UNCHANGED;
REPLACE [App].[Items] SET ([Name]: 'same') WHERE [Id] = 1 SKIP IF UNCHANGED;
```

### FORCE WRITE

Force the write even when the collection has `SkipUnchangedUpdates` enabled:

```sql
UPDATE [App].[Items] SET ([Name]: 'same') WHERE [Id] = 1 FORCE WRITE;
REPLACE [App].[Items] SET ([Name]: 'same') WHERE [Id] = 1 FORCE WRITE;
```

`SKIP IF UNCHANGED` and `FORCE WRITE` are mutually exclusive. These apply to both non-temporal and temporal UPDATE, as well as REPLACE. On temporal collections, `SKIP IF UNCHANGED` prevents creating a new version when the data is identical to the version the update supersedes; without it, temporal UPDATE always creates a new version. `FORCE WRITE` overrides the collection's `SkipUnchangedUpdates` default. Note: on temporal collections, `CLEAR VALID TO` and explicit `VALID FROM` always create a new version even when skip-unchanged would normally suppress it — these clauses indicate intentional temporal metadata changes.

## Instant expressions in writes

Every timestamp a write takes — `VALID FROM` / `TO`, `CREATEDAT`, `DELETE ... AT`, `CANCEL VERSION AT`, `UPDATE PREVIOUS VALID TO` and `SEAL` / `UNSEAL ... AS OF` — is an `<instant>`: a timestamp string, `NOW`, an `@parameter`, or an instant expression over literals, parameters and `NOW` (scalar functions, `CAST`, arithmetic, concatenation, `CASE`). The expression is evaluated once per execution with the statement's single `NOW`, and its value is admitted as a timestamp parameter's is ([instant expressions](temporal.md#instant-expressions)): `NULL` and text that names no instant are `INVALID_VALUE`, any other non-`DATETIME` type is `TYPE_MISMATCH`. A write has no enclosing row, so an instant reads no field (`INVALID_STATEMENT`), and a validity bound at `0001-01-01T00:00:00Z` is refused with `INVALID_VALUE` however it is spelled.

```sql
-- A version scheduled from tomorrow at midnight UTC, for one week.
INSERT INTO [Sales].[Prices] ([Sku]: 'A1', [Price]: 12.5)
VALID FROM DATE_TRUNC('DAY', DATEADD('DAY', 1, NOW)) TO DATE_TRUNC('DAY', DATEADD('DAY', 8, NOW));

-- A price change effective on the first of next month.
UPDATE [Sales].[Prices] SET ([Price]: 13) WHERE [Sku] = 'A1'
VALID FROM DATEADD('MONTH', 1, DATE_TRUNC('MONTH', NOW));
```

## CREATEDAT creation timestamps

`[Meta].[CreatedAt]` is a non-null UTC `DATETIME` on each authored revision, including standard collections. INSERT, UPDATE, and REPLACE default it to the statement-pinned `NOW`. Every row written by one statement receives the same instant, unless a row of a [multi-row INSERT](#multi-row-insert) names its own. It does not express commit order; use revision/concurrency contracts for ordering rather than assuming wall-clock timestamps are unique or increasing.

INSERT and INSERT ... ON CONFLICT accept `CREATEDAT <instant>` (a timestamp string, `NOW`, an `@parameter` or an [instant expression](#instant-expressions-in-writes)). Place it after the validity and ON CONFLICT clauses, before CREATEDBY and RETURNING, where it stamps every row of the statement; a row of a parenthesized row list may name its own after its document, which replaces the statement's for that row ([multi-row INSERT](#multi-row-insert)). Omission uses NOW. Explicit timestamps normalize to UTC; NULL, missing/null parameters, invalid timestamps, an expression whose value is no instant, `0001-01-01T00:00:00Z` (tick zero, which marks absent temporal metadata, refused with `INVALID_VALUE` however it is spelled, as for a validity bound), and duplicate clauses are rejected. Both insert and conflict-update paths use the supplied timestamp. DO NOTHING preserves the existing value. A conflict update writes a revision even when only the supplied timestamp differs from the existing row.

```mql
INSERT INTO [App].[Items] ([Id]: 1, [Name]: 'Imported')
ON CONFLICT DO UPDATE SET ([Name]: 'Imported')
CREATEDAT @originalCreatedAt CREATEDBY @actor
RETURNING [Meta].[CreatedAt];
```

`FORCE WRITE` refreshes the timestamp on an UPDATE/REPLACE even when its body is unchanged. A skipped write preserves it. Timeline fragments, validity closure, predecessor restoration during cancellation, sealing, schema rewrites, physical relocation, backups, and replay preserve the authored timestamp. Ordinary SET cannot assign the virtual metadata field. An application field recording the original creation of the document remains a separate concern.

The typed client's `InsertAsync` and `InsertVersionAsync` accept the optional `createdAt` argument and emit the same clause. `Document<TKey,TDocument>.CreatedAt` returns the engine timestamp through both embedded and remote clients. `InsertAsync` sends one `INSERT … RETURNING` that captures the primary key and the stored `[Meta].[CreatedAt]`, `[Meta].[Revision]`, `[Meta].[ValidFrom]` and `[Meta].[ValidTo]`, so the document it returns carries the metadata a later read of that document returns, through both clients. `InsertManyAsync` sends its bodies as [multi-row INSERT](#multi-row-insert) statements with the same RETURNING list, and `InsertManyCountAsync` without it.

## CREATEDBY attribution

Physical deletion of an Internal-policy actor checks a persisted reverse index and validates candidate source metadata. An uncommitted policy change uses a source scan until the migration builds its replacement index.

Collections opt into attribution with `(CreatedBy: TRUE)` at creation or `ALTER COLLECTION [Namespace].[Collection] SET CREATEDBY TRUE` in a migration. The default is FALSE, independently of Temporal. Enabling requires a database EXTERNAL or COLLECTION policy; see [policy configuration](maintenance.md#set-createdby).

On enabled collections, INSERT, UPDATE, REPLACE, both upsert branches, temporal version insertion, and CANCEL VERSION with UPDATE PREVIOUS SET require one of:

- `CREATEDBY <literal|@parameter>`: supply a non-empty actor.
- `CREATEDBY COPY`: copy the mutation's exact selected existing row or predecessor.
- `CREATEDBY COPY OR <literal|@parameter>`: use the supplied fallback when that source is missing or unattributed.

Put the clause after other mutation clauses and before RETURNING; a row of a [multi-row INSERT](#multi-row-insert) may name its own after its document, which replaces the statement's for that row, and a multi-row INSERT whose statement names none leaves each row to name its own, refusing the first row that names none as that row. NULL, empty actors, and omission are rejected on enabled collections with `CREATEDBY_REQUIRED`, even when a predicate matches no rows, whether the actor is a literal or a parameter's value. Disabled collections reject any CREATEDBY clause (`CREATEDBY_DISABLED`). Under an EXTERNAL policy the actor is text or bytes; a value of another type is `TYPE_MISMATCH`. Their new authored writes have no actor; old retained versions remain unchanged. An actor change, including clearing on a disabled collection, is a mutation even when the body is unchanged. DO NOTHING does not resolve an actor or change metadata.

COPY resolves under the write's row claims on each retry, never from a preliminary read or an unrelated historical version. Strict COPY fails when no actor exists to copy. COPY OR never falls back from an invalid copied actor: both copied and supplied actors pass current policy checks. External accepts bytes or strict UTF-8 text; COLLECTION uses its target's native primary-key type and requires a currently valid actor. The actor collection can opt in like any user collection; seed initial actors while it is disabled. Trigger bodies supply their own intent. DELETE, sealing, plain cancellation, validity-only restoration, engine cascades, and physical maintenance preserve their existing metadata rules. Rolling back an INSERT removes its attribution reference with the inserted row; no separate attribution cleanup statement is needed. Physical deletion removes that source row's reference in the same transaction; rolling back the deletion restores both the row and its reference. UPDATE and REPLACE move attribution references with their row replacement; rollback restores the previous attribution. Creating a temporal version retains earlier versions' attribution references. Soft temporal DELETE ends validity and keeps those stored references. Canceling a future version physically removes that version's reference. A cancellation that rewrites its predecessor body applies that authored write's explicit attribution; validity-only restoration preserves its original actor. Timeline splits retain attribution on surviving fragments; fully removed versions no longer contribute attribution references. Physical row relocation during compaction or a schema rewrite preserves attribution and its reference. Dropping a source collection removes references from all its retained versions; aborting the migration restores them. Other source collections keep their references. Aborting a schema rewrite restores the original references, including when source rows were inserted earlier in the migration. Attribution maintenance uses that migration's transaction; another transaction cannot mutate its unpublished attribution index.

```mql
INSERT INTO [App].[Items] ([Id]: 1, [Name]: 'Example')
CREATEDBY @actor RETURNING [Id], [Meta].[CreatedBy];

UPDATE [App].[Items] SET ([Name]: 'Changed') WHERE [Id] = 1
CREATEDBY @actor;

REPLACE [App].[Items] SET ([Name]: 'Keep actor') WHERE [Id] = 1
CREATEDBY COPY;

INSERT INTO [App].[Items] ([Id]: 2, [Name]: 'Example')
ON CONFLICT DO UPDATE SET ([Name]: 'Example')
CREATEDBY COPY OR @actor;
```

`Meta.CreatedBy` is nullable BYTES in SELECT and explicit RETURNING projections. The result schema marks it, and every column that passes its bytes through unchanged (an alias, a view's or materialized view's column, a CTE's), as [CreatedBy attribution](queries.md#createdby-attribution-columns), so a client decodes those columns by the database's policy rather than showing base64; a value an expression computes is unmarked. Byte parameters preserve exact content; BYTES filter literals use base64. The .NET `collection.WithCreatedBy(value)`, `WithCopiedCreatedBy()`, and `WithCopiedCreatedBy(fallbackActor)` handles emit the corresponding clause across insert, update, replacement, upsert, and version insertion. The original handle remains unchanged. `CollectionBuilder<T>.EnableCreatedBy()` opts in at creation; `collection.CreatedByEnabled` exposes the setting. `Document<TKey,TDocument>.CreatedBy` returns the metadata bytes.

## CANCEL VERSION

```
CANCEL VERSION AT <instant> FROM [Namespace].[Collection]
  WHERE expression
  [UPDATE PREVIOUS [SET (field: value, ...)] [VALID TO <instant>|NULL]]
  [CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>] -- with SET only
```

```sql
-- Cancel a scheduled future version (simple case — prior stays truncated)
CANCEL VERSION AT '2030-01-01T00:00:00Z' FROM [Sales].[Subscriptions]
WHERE [Id] = @id;

-- Cancel + restore prior to open-ended (the common "I scheduled this by mistake" path)
CANCEL VERSION AT '2030-01-01T00:00:00Z' FROM [Sales].[Subscriptions]
WHERE [Id] = @id
UPDATE PREVIOUS VALID TO NULL;

-- Cancel + update prior data AND extend to open-ended in one atomic step
CANCEL VERSION AT '2030-05-01T00:00:00Z' FROM [Sales].[Subscriptions]
WHERE [Id] = @id
UPDATE PREVIOUS SET ([Tier]: 'Gold') VALID TO NULL;

-- Cancel + set a different bounded ValidTo on the prior
CANCEL VERSION AT '2030-05-01T00:00:00Z' FROM [Sales].[Subscriptions]
WHERE [Id] = @id
UPDATE PREVIOUS VALID TO '2031-01-01T00:00:00Z';
```

**Semantics** — physically removes the temporal version whose `ValidFrom` equals the `AT` timestamp. This is the only MQL path that physically deletes temporal data — allowed because a future version (`ValidFrom > now`) has never been observed as current by any `AS OF` query. Past and current versions are never physically deletable; use `DELETE ... AT` to end active versions instead.

**Pinned-FK interaction** — a future version *can* be referenced by a `PINNED REFERENCES` field (scalar `PIN(id, rev)` or any element of a `ARRAY … PINNED REFERENCES … [PIN(...), …]` field). When such a binding exists at the moment of cancellation, the FK's `ON REMOVE` action takes over:

- **`Restrict` (default)** — `CANCEL VERSION AT` is rejected with a `pinned FK still references revision …` error. Applies whether the binding lives in a scalar pinned slot or in any element of a pinned array.
- **`SetNull`** — scalar pinned slots are nulled (id + revision tail zeroed). Pinned-array elements are dropped per element: only the element matching `(targetId, cancelledRevision)` is removed; same-id elements bound to other revisions survive. A pinned field inside array objects is cleared in the elements matching the pair, or under `MATCH FULL` in every element of that row's value ([ON REMOVE actions](ddl.md#on-remove-actions)).
- **`Cascade`** — rejected at registration on pinned FKs.

Scalar pin checks match the complete target identifier and the cancelled revision, with the same result whether the source field is indexed or scanned. A Bytes identifier that prefixes another identifier is a different target; String identifiers retain their case-insensitive identity, including empty identifiers. Pins to another identifier or revision remain unchanged.

**Future-only guard** — `AT` must be strictly in the future relative to `now` at execution time. A past or current timestamp is rejected with a clear error pointing at `DELETE ... AT` as the correct path for ending active versions.

**Temporal-only** — non-temporal collections reject `CANCEL VERSION` outright — there are no versions to cancel.

**`UPDATE PREVIOUS`** — optional clause that modifies the version whose `ValidTo` equals the canceled version's `ValidFrom` — i.e., the neighbor `AdjustTimeline` truncated when the future version was inserted. Cancellation alone does NOT restore the prior's original open-endedness; the prior stays truncated unless you specify `UPDATE PREVIOUS VALID TO NULL` (or a new timestamp). `SET` applies partial-merge updates to the prior's data (same semantics as regular `UPDATE SET`). At least one of `SET` or `VALID TO` must appear when `UPDATE PREVIOUS` is used — an empty clause is a parse error.

When `SET` rewrites the predecessor body, the statement accepts the same trailing CREATEDBY clause and collection requirements as UPDATE. The copy source is that exact predecessor. Validity-only restoration accepts no attribution clause.

```mql
-- For a collection with CreatedBy enabled, explicitly keep the predecessor's actor.
CANCEL VERSION AT '2030-05-01T00:00:00Z' FROM [Sales].[Subscriptions]
WHERE [Id] = @id
UPDATE PREVIOUS SET ([Tier]: 'Gold') VALID TO NULL
CREATEDBY COPY;
```

When `SET` rewrites the predecessor body, the replacement receives a fresh revision greater than every revision in that PK's live timeline as the cancellation begins. It never reuses the canceled target's revision or a later sibling's revision, even when versions were inserted out of validity order. If that correction removes the predecessor's final fragment, its pinned references apply `Restrict`/`SetNull` together with those of the canceled target. Any rejection rolls back the entire cancellation and predecessor correction.

Predecessor SET expressions read the claimed predecessor in each write attempt, so concurrent corrections are preserved and retries keep the same statement clock and function snapshot. Assignments receive the ordinary field, type, enum, and required-value validation. Unknown fields reject the cancellation. Stored computed values are rebuilt from the final predecessor, including nested and subtype fields; CHECK constraints see those values and foreign keys validate at the predecessor's ValidFrom. The stored body, secondary indexes, and mutation log share that final image. A failed candidate leaves every target and predecessor unchanged after rollback, including multi-target cancellations.

`UPDATE PREVIOUS` requires a version active immediately before the canceled version's `ValidFrom`. If the target is the first version, or a timeline gap leaves no predecessor, the statement errors with `OBJECT_NOT_IN_PREREQUISITE_STATE` before either half mutates: the future version is not canceled. Run the same `CANCEL VERSION` without `UPDATE PREVIOUS` when cancellation should proceed without a predecessor.

A restored bounded `VALID TO` must be strictly after the preceding version's `ValidFrom`. If another version follows the canceled target, the restored end must be no later than that successor's `ValidFrom`; `VALID TO NULL` is rejected in that case because it would create an overlapping timeline. Both bounds are `INVALID_VALUE`. A restored end that widens the predecessor can make it overlap another document's version of a `UNIQUE` key, which fails with `UNIQUE_VIOLATION` ([UNIQUE on a temporal collection](indexes.md#unique-on-a-temporal-collection)). Every rejection leaves both the target and predecessor unchanged.

**Atomicity** — cancellation and any `UPDATE PREVIOUS` changes live in a single transaction. No observable state exists where the future version is gone but the prior hasn't been restored yet. Incremental materialized views over the collection are recomputed in that transaction too, although the cancellation fires no triggers ([materialized views](materialized-views.md#maintenance-execution-classes)).

**`WHERE` is mandatory** — `CANCEL VERSION` never ranges across a whole collection implicitly. There's no `CONFIRM CANCEL ALL` equivalent — the future-only guard already bounds scope.

## SEAL and UNSEAL

`SEAL` is a reversible workflow guard for historical or otherwise finalized records. It is not a transaction lock and not compliance/WORM storage: ordinary writers with `Write` permission may run `UNSEAL`, and schema administrators may still alter or drop the collection.

```mql
SEAL|UNSEAL [Namespace].[Collection]
  [AS OF <instant>]
  {WHERE expression | CONFIRM SEAL|UNSEAL ALL}
  [EXPECTED REVISION integer|@parameter]
  [RETURNING *|[Field], ...]
```

```mql
SEAL [Archive].[Records]
WHERE [Id] = @id
EXPECTED REVISION @revision
RETURNING [Id], [Meta].[Sealed];

UNSEAL [Archive].[Records] WHERE [Id] = @id;
```

The selected version identifies the whole logical identity. Standard rows store one seal flag; temporal collections apply the same state atomically to every retained version, including past and scheduled versions. `AS OF` is accepted only for temporal collections and chooses which retained version is matched. Without it, matching uses statement time. The flag is identity control metadata: later `AS OF` reads report the current seal state, while a snapshot transaction that started before the toggle retains its earlier view.

While sealed, every logical row mutation is rejected with `DOCUMENT_SEALED`, including otherwise-unchanged UPDATE/REPLACE, temporal version insertion or validity changes, cancellation, UPSERT's update branch, triggers, incremental-view writes, plugin/replay writes, and FK Cascade or SetNull. `ON CONFLICT DO NOTHING` remains valid because it performs no row mutation. Scheduled temporal activation and expiry still follow their stored intervals; sealing neither rewrites nor suspends time. `UNSEAL` is the ordinary mutation exception.

The guard also applies when earlier statements created or replaced the version and sealed it inside the same transaction. Later INSERT, UPDATE, and UPSERT statements cannot bypass that private seal.

A multi-row toggle is atomic. Without `WHERE`, the matching confirmation must be written exactly as `CONFIRM SEAL ALL` or `CONFIRM UNSEAL ALL`. An identity already in the requested state is a no-op: affected count zero for that identity, no `RETURNING` row, and no event. `EXPECTED REVISION` requires at most one matched identity and checks the selected version before this no-op decision; sealing does not increment the document revision.

Before sealing, Mnemic rejects a temporal FK Cascade/SetNull relationship whose known target timeline contains an absence that would later mutate the sealed identity. Adding such a relationship to existing sealed rows through `ALTER COLLECTION ... ADD FIELD ... REFERENCES` is rejected atomically as well. Parent edits still revalidate at the claimed write boundary.

Plain views are read-only for every DML form, including `SEAL` and `UNSEAL`; target the underlying collection explicitly. Both statements require ordinary `Write` permission and may be used in trigger bodies, but toggles do not themselves fire INSERT/UPDATE/DELETE triggers. They do maintain incremental materialized views that read row metadata such as `[Meta].[Sealed]` ([materialized views](materialized-views.md#lifecycle-and-dependency-rules)).

## See also

- [Document literal syntax](document-literals.md)
- [Temporal queries](temporal.md) — read-side `AS OF` / `IN BETWEEN`
- [Transactions](transactions.md) — atomic multi-statement mutations
- [Polymorphism](polymorphism.md) — `SUBTYPE` clause on INSERT

## Detailed syntax and execution

Stored computed fields can be UPSERT conflict targets when backed by a matching UNIQUE index. The probe uses the fully materialized proposed key even though the INSERT literal cannot assign that computed field. NULL key parts retain ordinary UNIQUE semantics and take the insert path. This applies to DO NOTHING and DO UPDATE.

UPSERT validates proposed validity windows before choosing either conflict action; non-temporal collections reject VALID clauses even on a conflict hit, and `VALID TO` with `DO UPDATE` is rejected on temporal collections.

SET expressions, including those inside replacement Object and nested Array values, read the original containing row at each mutation retry. Referenced virtual computed fields are evaluated before use; EXCLUDED reads the independent proposed image throughout an UPSERT conflict assignment. All sibling assignments share that original-row scope, so assignment order does not affect expression results. Derived read values stay in a separate evaluation buffer, so unassigned virtual slots remain NULL in the stored mutation image.

RETURNING binds field paths against the statement's captured base and subtype declarations. A declared `Body` field is an ordinary stored root: for example, `RETURNING [Body].[Code]` validates that child and emits the complete `Body` Object, including its siblings, just as SELECT does. A missing child is rejected before mutation and cannot borrow an unrelated root named `Code`.

When no source declaration defines `Body`, RETURNING accepts it as an optional qualifier. For example, `RETURNING [Body].[Extra]` then captures the same extension value as `RETURNING [Extra]`, including UPSERT DO NOTHING through a secondary unique key.



<a id="syntax-insert"></a>

### INSERT

```sql
INSERT INTO [Ns].[Collection]
  [SUBTYPE [discriminator]]             -- polymorphic only
  ([Field]: value | DEFAULT, ...)       -- or a row list: ((doc) [row clauses], ...)
  [FORCE ID]                            -- explicit PK on auto-increment
  [VALID FROM <instant> [TO <instant>]] -- temporal only
  [ON CONFLICT [(target)] (DO NOTHING | DO UPDATE SET (...))]
  [CREATEDAT <instant>]                 -- authored-revision timestamp
  [CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>]
  [RETURNING * | [F1], [F2]]
-- a row list holds up to 1,000 rows in parentheses; a row may name
-- [VALID FROM <instant> [TO <instant>]] [CREATEDAT <instant>]
-- [CREATEDBY ...] after its document, replacing the statement's;
-- the statement's clauses keep this order and apply to every row
```

Field validation: every field is checked against the resolved schema before writing. Unknown fields and type mismatches are rejected. Explicit `NULL` on a `NOT NULL` field is rejected, and row-independent expressions are checked again after evaluation. A CHECK over an auto-increment PK runs after assignment but before write admission, against the exact stored candidate.

Value positions accept row-independent expressions — function calls, `CASE`, arithmetic, `@parameters`, `NOW`, a space-separated unary minus (`[Qty]: - 5`) — evaluated once at insert time. Field references (`[Other] + 1`) are rejected: there is no existing row to read from on the insert path (use `UPDATE … SET` for row-derived values).

Auto-increment: omit the PK, or provide zero, to let the engine assign. `FORCE ID` is the explicit data-migration override for `INT32` / `INT64` auto-increment collections; it requires a PK and stores that value verbatim, including zero or a negative value. The generator reserves beyond the forced identity when possible, never regresses, and becomes exhausted at the PK type's ceiling. The reservation is not rolled back if later validation or the transaction fails, so gaps are expected. `EXPLAIN INSERT … FORCE ID` surfaces the bypass and its monotone, saturating reservation in `Notes`. The reservation is process-local until its row or the generator high-water mark becomes durable; an unclean restart may forget a failed or rolled-back reservation that left no durable evidence. `FORCE ID` is not a standalone durable sequence-reservation operation.

<a id="syntax-statement-attribution"></a>

### Statement attribution

Collections opt into attribution with `(CreatedBy: TRUE)` at creation or `ALTER COLLECTION [Namespace].[Collection] SET CREATEDBY TRUE` in a migration. The default is FALSE, independently of Temporal. Enabling requires a database EXTERNAL or COLLECTION policy; see [policy configuration](maintenance.md#set-createdby).

On enabled collections, INSERT, UPDATE, REPLACE, both upsert branches, temporal version insertion, and CANCEL VERSION with UPDATE PREVIOUS SET require one of:

- `CREATEDBY <literal|@parameter>`: supply a non-empty actor.
- `CREATEDBY COPY`: copy the mutation's exact selected existing row or predecessor.
- `CREATEDBY COPY OR <literal|@parameter>`: use the supplied fallback when that source is missing or unattributed.

Put the clause after other mutation clauses and before RETURNING; a row of a [multi-row INSERT](#multi-row-insert) may name its own after its document, which replaces the statement's for that row, and a multi-row INSERT whose statement names none leaves each row to name its own, refusing the first row that names none as that row. NULL, empty actors, and omission are rejected on enabled collections with `CREATEDBY_REQUIRED`, even when a predicate matches no rows, whether the actor is a literal or a parameter's value. Disabled collections reject any CREATEDBY clause (`CREATEDBY_DISABLED`). Under an EXTERNAL policy the actor is text or bytes; a value of another type is `TYPE_MISMATCH`. Their new authored writes have no actor; old retained versions remain unchanged. An actor change, including clearing on a disabled collection, is a mutation even when the body is unchanged. DO NOTHING does not resolve an actor or change metadata.

COPY resolves under the write's row claims on each retry, never from a preliminary read or an unrelated historical version. Strict COPY fails when no actor exists to copy. COPY OR never falls back from an invalid copied actor: both copied and supplied actors pass current policy checks. External accepts bytes or strict UTF-8 text; COLLECTION uses its target's native primary-key type and requires a currently valid actor. The actor collection can opt in like any user collection; seed initial actors while it is disabled. Trigger bodies supply their own intent. DELETE, sealing, plain cancellation, validity-only restoration, engine cascades, and physical maintenance preserve their existing metadata rules.

`Meta.CreatedBy` uses ordinary BYTES equality and membership: parameters contain bytes and literal filters use base64. ORDER BY compares unsigned bytes without text decoding. Range operators (`<`, `<=`, `>`, `>=`) use the same byte order, with NULL operands never matching. Text operators require STRING. NULL or empty CREATEDBY inputs are rejected; unattributed retained metadata reads as NULL. `SET CREATEDBY` changes policy only inside a migration. Policy stays private until commit and is discarded on abort. Internal target identity survives collection and namespace rename; reconfigure before dropping the target.

**`ON CONFLICT (target)` shapes.** Two probe forms:

An explicit conflict target accepts at most 1,024 fields, matching the maximum stored-schema width. Programmatic ASTs carry the same limit.

1. **PK target** (or implicit, no target): `ON CONFLICT DO NOTHING` or `ON CONFLICT ([Id]) DO UPDATE SET (...)`. Probed via `GetAsync`.
2. **Secondary `UNIQUE` index target** (single-column or compound): `ON CONFLICT ([Email]) DO ...` or `ON CONFLICT ([Tenant], [Slot]) DO ...`. The target list must align exactly with a registered `UNIQUE` index. Non-unique or missing-index targets are rejected up-front — a non-unique target could match multiple rows and the conflict handler can't pick one deterministically. On a temporal collection the probe arbitrates at the statement's `NOW`: it matches the document whose current version holds the key, and `DO UPDATE` writes a new version of that document. A document that holds the key only in a scheduled future version, or in history the new version would overlap, is not a conflict target, so the miss insert can still fail with `UNIQUE_VIOLATION` ([UNIQUE on a temporal collection](indexes.md#unique-on-a-temporal-collection)).

```sql
-- Secondary UNIQUE target (auto-increment PK may be omitted)
INSERT INTO [Shop].[Users] ([Email]: 'a@x.com', [Name]: 'A')
ON CONFLICT ([Email]) DO UPDATE SET ([Name]: 'A');

-- Compound UNIQUE target
INSERT INTO [Shop].[Tickets] ([Tenant]: 't1', [Slot]: 'A', [Qty]: 1)
ON CONFLICT ([Tenant], [Slot]) DO UPDATE SET ([Qty]: 999);
```

Polymorphic INSERTs require `SUBTYPE` and support `ON CONFLICT`. A miss inserts the proposed subtype; DO UPDATE preserves the conflicting row's subtype and updates its base/extension fields. Assigning a field absent from that subtype fails atomically. DO NOTHING RETURNING captures requested extension values in the same probe as the existing key and metadata. Probe/insert races are resolved by a bounded re-probe: a concurrent winner is handled as the declared conflict instead of leaking `UNIQUE_VIOLATION`. Standalone secondary-target statements also retry transient index-ownership conflicts during the probe or conflict action instead of leaking `TX_CONFLICT_FWW`; explicit transactions keep retry policy with their caller. `DO UPDATE` re-evaluates SET expressions against the retry-local current row. `EXCLUDED.[Field]` reads the effective proposed INSERT value, including defaults and expression results. It supports nested paths and retains enum metadata for `ENUM_LABEL(EXCLUDED.[State])`. `[Qty] + EXCLUDED.[Qty]` combines the current and proposed values. The proposed row keeps its own subtype. Unknown EXCLUDED paths, including those inside replacement Object and nested Array values, fail before either conflict outcome. Nested SET expressions share the original containing row's scope with scalar assignments; sibling SET assignments do not see each other's new values. The complete proposed INSERT remains subject to unknown-field, NOT NULL, enum, CHECK, and validity-window validation even when the probe chooses `DO NOTHING` / `DO UPDATE`. Non-temporal collections reject VALID clauses on either outcome. On a temporal collection the probe and `DO UPDATE` both use the proposed `VALID FROM`, and `VALID TO` with `DO UPDATE` is rejected. Defaults, generated-key placeholders, expressions, and statement-pinned NOW are materialized once before a secondary-index probe and reused by the miss insert; a NULL unique target is never treated as a conflict. Collections with read-time computed columns can make that probe streaming; the executor consumes the stream before classifying hit versus miss, so an existing unique key still selects the declared action. The probe schema is frozen before streaming begins, so row production cannot retarget primary-key or conflict-image decoding mid-probe.

<a id="syntax-update"></a>

### UPDATE

```sql
UPDATE [Ns].[Collection]
SET ([Field]: value, [Field2]: [Field2] + 1, [Field3]: DEFAULT)
WHERE expr | CONFIRM UPDATE ALL
[VALID FROM <instant>]
[CLEAR VALID TO]
[EXPECTED REVISION n | @rev]
[SKIP IF UNCHANGED | FORCE WRITE]
[CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>]
[RETURNING ...]
```

Mandatory either `WHERE` or `CONFIRM UPDATE ALL`. PK fields cannot be modified. Set RHS accepts full expressions (read pre-update row). On a subtyped collection the expression resolves fields across the whole document — base and extension fields can reference each other. The merge is atomic with the engine's claim of the row — on a temporal collection, the version valid at `VALID FROM` (NOW when omitted), which `WHERE` also reads — and re-runs after a transient conflict, so disjoint concurrent changes and increments are not lost. For an exact-primary-key temporal UPDATE, row existence and that version are resolved inside that same retry attempt; concurrent temporal-index re-keying cannot become a false zero-row result. A genuinely missing row returns affected count zero without evaluating unrelated SET fields.

<a id="syntax-replace"></a>

### REPLACE

```sql
REPLACE [Ns].[Collection]
SET ([Field]: value | DEFAULT, ...)
WHERE expr
[EXPECTED REVISION ...]
[SKIP IF UNCHANGED | FORCE WRITE]
[CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>]
[RETURNING ...]
```

Full document replacement — a top-level field not in SET becomes NULL, taking no default, so a NOT NULL field must be set, if only to `DEFAULT`; an object SET assigns takes its [defaults](#defaults).

<a id="syntax-delete"></a>

### DELETE

```sql
DELETE FROM [Ns].[Collection]
WHERE expr | CONFIRM DELETE ALL
[AT <instant>]                          -- temporal: schedule expiry
[RETURNING ...]
```

`AT` is **temporal-collection-only** and ends the document at that instant: a future `AT` schedules the end, so the document stays current until then, and every version scheduled at or after `AT` is cancelled. Without `AT` the document ends now and its scheduled versions are cancelled. History before the end is kept. On non-temporal collections, `AT` is rejected.

Multi-target DELETE is atomic. A candidate already removed by an earlier FK cascade is skipped only when the engine returns its typed row-absence signal; unrelated key misses and all other engine failures remain statement errors and are never converted into a successful lower affected count.

<a id="syntax-seal--unseal"></a>

### SEAL / UNSEAL

```mql
SEAL|UNSEAL [Ns].[Collection]
[AS OF <instant>]                       -- temporal only
WHERE expr | CONFIRM SEAL|UNSEAL ALL
[EXPECTED REVISION n | @rev]
[RETURNING * | [F1], [Meta].[Sealed]]
```

These contextual statement-head commands toggle a reversible identity-level workflow guard. They are not transaction locks or WORM protection: an ordinary writer can explicitly `UNSEAL`, and privileged DDL/physical maintenance remains available. `seal` and `unseal` remain usable as identifiers elsewhere.

The selected standard row or temporal `AS OF` version identifies the logical identity. A temporal toggle updates every retained past/current/future version atomically without changing body bytes, revision, or validity. Later historical reads expose the identity's current `[Meta].[Sealed]` state; a snapshot transaction retains the state visible when it began. Scheduled activation and expiry continue normally.

A sealed identity rejects all logical mutations inside the engine's claimed write boundary: unchanged UPDATE/REPLACE, DELETE, temporal insertion/validity editing/cancellation, UPSERT's update branch, trigger/IVM/plugin/replay writes, and FK Cascade/SetNull. `ON CONFLICT DO NOTHING` is allowed. `UNSEAL` is the ordinary exception. A multi-match operation is atomic.

An already-desired state is a zero-affected no-op with no RETURNING row or event. `EXPECTED REVISION` permits at most one match and is checked first. Before sealing, temporal Cascade/SetNull FKs must have known continuous target coverage for every retained source version; an ALTER that adds an unsafe FK to existing sealed rows is rejected and rolled back.

`SEAL`/`UNSEAL` require `Write` permission and may appear in trigger bodies, but toggles fire no INSERT/UPDATE/DELETE triggers and no IVM. Plain views are read-only for both statements; target the underlying collection explicitly.

<a id="syntax-cancel-version-temporal-only"></a>

### CANCEL VERSION (temporal only)

```sql
CANCEL VERSION AT <instant>
FROM [Namespace].[Collection]
WHERE expression
[UPDATE PREVIOUS [SET (field: value, ...)] [VALID TO <instant> | NULL]]
```

Same FK-enforcement rules as DELETE (Restrict / Cascade / SetNull). Pinned references apply their permitted `Restrict` or `SetNull` action only where they bind the exact target identifier and cancelled revision, including through indexes; a `SetNull` that matches inside an array-object value under `MATCH FULL` clears that whole value. `UPDATE PREVIOUS` is atomic with the cancellation and requires a version active immediately before the target's `ValidFrom`; if none exists, the statement errors before mutation and leaves the future version intact. A predecessor body rewritten by `SET` receives a fresh revision above the whole live PK timeline; it never reuses the canceled target's identity or a later sibling's revision when validity order differs from insertion order. A bounded restored `VALID TO` must be after that predecessor's `ValidFrom`. When a later version follows the canceled target, the restored end cannot pass the successor's `ValidFrom`, and `VALID TO NULL` is rejected because it would overlap that successor. These failures leave the timeline unchanged. A restored end that widens the predecessor, with or without `SET`, is checked against every `UNIQUE` index over the widened interval: another document's version holding one of the predecessor's keys there fails the statement with `UNIQUE_VIOLATION` ([UNIQUE on a temporal collection](indexes.md#unique-on-a-temporal-collection)).

<a id="syntax-concurrency-control"></a>

### Concurrency control

- `EXPECTED REVISION n|@rev` — optimistic concurrency. Throws on mismatch; also applies to SEAL/UNSEAL before their no-op check.
- `SKIP IF UNCHANGED` — no-op if SET payload equals current.
- `FORCE WRITE` — bypass the collection's `SkipUnchangedUpdates` default.

`EXPECTED REVISION` comes first and combines with either of the other two; `SKIP IF UNCHANGED` and `FORCE WRITE` exclude each other ([concurrency control](#concurrency-control)).

<a id="syntax-returning"></a>

### RETURNING

An explicit `RETURNING` list accepts at most 1,024 fields, matching the maximum result-schema width. `RETURNING *` remains the single-token all-fields form.

`RETURNING *` or `RETURNING [F1], [F2]`. On INSERT returns the post-insert row (incl. assigned auto-increment PK). On UPDATE / REPLACE and `ON CONFLICT DO UPDATE` returns post-update. On DELETE returns the row immediately before deletion. On `INSERT … ON CONFLICT DO NOTHING` returns the row observed by the conflict probe. SEAL/UNSEAL return the unchanged selected body with post-toggle metadata; no-op identities emit no row.

DML pins one exact collection schema generation through validation, source matching, mutation, and RETURNING snapshot capture. Rewriting ALTER cannot publish inside that boundary. INSERT retains the accepted body after assigned-PK substitution; UPSERT retains its exact merged body or conflict-probe row; UPDATE/REPLACE retains each exact post-image; DELETE retains each exact pre-delete image. No arm re-reads the primary key after the decisive operation, so a concurrent writer cannot substitute or remove the returned row. Explicit `Meta.*` projections come from the same exact slot snapshot, including the engine-assigned `Revision`, temporal validity window, and `Sealed` state. Delayed enumeration after a later ALTER still uses the statement's original bytes and result schema. UPSERT also rechecks cancellation after its conflict probe and before either conflict action, so cancellation cannot race into `DO UPDATE`. On a polymorphic collection, an explicit extension projection and every extension field retained by `RETURNING *` must have a structurally identical schema across the subtypes that declare it. Ambiguity is rejected before any mutation.

## Usage pitfalls


<a id="pitfall-update--delete-without-where-refuses-to-run"></a>

### `UPDATE` / `DELETE` without WHERE refuses to run

Mass updates require `CONFIRM UPDATE ALL` (and `CONFIRM DELETE ALL` for DELETE). The parser rejects bare `UPDATE [c].[c] SET (...)` — this is a safety feature.

<a id="pitfall-seal-is-reversible-workflow-protection-not-a-lock-or-worm-policy"></a>

### `SEAL` is reversible workflow protection, not a lock or WORM policy

A sealed identity rejects logical mutation with `DOCUMENT_SEALED`, including an otherwise-unchanged UPDATE/REPLACE, temporal timeline edits, UPSERT updates, triggers, IVM, plugin/replay writes, and FK Cascade/SetNull. `ON CONFLICT DO NOTHING` is allowed because it performs no mutation. An ordinary writer can run `UNSEAL`, and DDL/physical maintenance remains privileged and available. Use the exact bulk confirmations `CONFIRM SEAL ALL` / `CONFIRM UNSEAL ALL`. An `EXPECTED REVISION` check runs even when the requested state is already set.

For temporal collections, `AS OF` chooses a version to identify the whole identity; all retained versions share one state. `[Meta].[Sealed]` on a historical body reports the state visible to the transaction snapshot. Sealing does not pause activation or expiry. A future Cascade/SetNull timeline gap blocks sealing, and adding such an FK to existing sealed rows aborts the migration.

<a id="pitfall-on-conflict-rejects-fractional--out-of-range-pk-values-up-front"></a>

### `ON CONFLICT` rejects fractional / out-of-range PK values up front

The conflict-target PK value is validated against the PK type *before* the probe: `[Id]: 1.5` on an integer PK or `[Id]: 5000000000` on an `INT32` PK errors the whole statement. It is never truncated or wrapped into a probe key — pre-validation, a wrapped value could have matched (and `DO UPDATE`d) a different row than the statement names. Same integrality/range rules as plain-INSERT integer coercion.

<a id="pitfall-on-conflict-validates-and-probes-the-effective-insert-candidate"></a>

### `ON CONFLICT` validates and probes the effective INSERT candidate

A conflict action does not excuse an invalid proposed insert: unknown fields, NOT NULL/enum/CHECK failures still reject `DO NOTHING` and `DO UPDATE` before the stored winner is touched. Secondary-target defaults, expressions, generated key placeholders, and NOW are evaluated once before probing and the miss path inserts that same candidate. A NULL unique target is not a conflict. `EXCLUDED.[Field]` reads that candidate inside DO UPDATE SET. Ordinary field references read the existing row. Both scopes retain their own subtype layouts: a proposed `SUBTYPE [B]` does not change an existing `[A]` row's subtype. SET can only assign fields present in the existing row; an unknown proposed path fails before probing. DO NOTHING RETURNING captures the existing row's requested extension values in the conflict probe. If read-time computed columns make the secondary probe streaming, UPSERT drains it before deciding there was no match; `Rows == null` is not an empty probe.
