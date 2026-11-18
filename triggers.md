# Triggers

Triggers run an ordered MQL body automatically when a row is inserted, updated, or deleted in a target collection. Their primary use today is **derived-data maintenance**: an audit table populated on every write, a counter table kept in sync, a denormalised projection refreshed atomically with its source.

The trigger body executes **inside the same transaction** as the source write. If the body throws, the source write rolls back. If the source write rolls back, the trigger's effects roll back too. Caller cancellation propagates through every nested trigger statement; cancellation stops the nested work and rolls back both effects rather than leaving a trigger body running after the source statement unwinds.

## DDL

Trigger DDL is schema DDL and must run inside `BEGIN MIGRATION` / `COMMIT MIGRATION`. The catalog and executable registry publish together at commit; an abort publishes neither. Until commit, writes in the defining migration use the previously committed trigger set.

```mql
CREATE TRIGGER [name] ON [Ns].[Coll]
    BEFORE | AFTER {INSERT | UPDATE | DELETE} [OR {INSERT | UPDATE | DELETE} ...]
    [FOR EACH ROW]
    [WHEN (<condition>)]
    AS BEGIN <statement> [; <statement> ...] [;] END
```

```mql
DROP TRIGGER [name] ON [Ns].[Coll];
```

```mql
FROM [Mnemic].[Triggers];   -- catalog of every trigger in the database
```

### Example

This example assumes `[Sales].[Orders]` already exists.

```mql
BEGIN MIGRATION 1;

CREATE COLLECTION [Sales].[Audit] WITH SCHEMA (
    [Id] INT64 PRIMARY KEY AUTOINCREMENT,
    [Note] STRING,
    [At] DATETIME
);

CREATE TRIGGER [audit_orders] ON [Sales].[Orders] AFTER INSERT AS BEGIN
    INSERT INTO [Sales].[Audit] (
        [Note]: NEW.[CustomerName],
        [At]: NOW
    )
END;

COMMIT MIGRATION;
```

After this trigger is registered, every `INSERT INTO [Sales].[Orders]` also writes one row to `[Sales].[Audit]` — atomically, in the same transaction. The audit PK is `AUTOINCREMENT` so the body needs no key generator (there is no `NEWID()`-style GUID function in MQL), and `NOW` is the bare keyword — it resolves once per statement, so the audit timestamp equals every other `NOW` the firing statement resolved.

## Conditional execution

`WHEN (<condition>)` follows the optional `FOR EACH ROW` and precedes `AS BEGIN`. It is a scalar BOOL condition over the event's read-only OLD/NEW images; only TRUE executes the body. FALSE and NULL skip it. Available images, stored paths, function arity, and the result type are validated before persistence and again on reopen. Bare field names, caller parameters, subqueries, and window functions are rejected (`WHEN (ROW_NUMBER() OVER () = 1)` reports `SYNTAX_ERROR` `A window function (OVER at position …) is not allowed in this expression; window functions are valid only as SELECT projections.`); qualify row values as `NEW.[Value]` or `OLD.[Value]`. Scalar functions and NOW use the source statement's function snapshot and timestamp.

The condition is compiled once with the definition and evaluated immediately before that trigger's body at its BEFORE or AFTER boundary. It does not query the source collection again or replace an image. An evaluation error or cancellation rolls back the source mutation and any earlier trigger effects. A skipped condition does not increment the fired-body counter. Persisted condition text and identity bindings survive rename and reopen. The nullable public `WhenText` renders current enum names and the original OLD/NEW image references.

## Timing and validation

`BEFORE` runs after the candidate image is resolved, before changing the source row or its indexes. Source queries in the body see the old stored row; INSERT's NEW image includes the assigned identity and resolved defaults. `AFTER` sees the applied row, after incremental materialized-view maintenance. Both timings use the same transaction and cancellation scope, and each timing runs matching triggers in case-insensitive name order.

A statement that writes several rows fires per row, in its row order: a multi-row UPDATE, REPLACE or DELETE in the order it matched its rows, and a [multi-row INSERT](dml.md#multi-row-insert) in document order. Each row's BEFORE triggers, write, incremental maintenance and AFTER triggers complete before the next row starts, so a later row's triggers see the earlier rows and their trigger effects. A trigger body may itself be a multi-row INSERT; its rows run in the firing transaction.

Images are immutable. A BEFORE body can veto by failing but cannot replace NEW. A later rewrite from an AFTER body runs BEFORE validation again for its new candidate. A recursive write to the same pending source row from a BEFORE body is rejected, including indirect writes through other triggers or cascades. Other rows may be written normally. BEFORE and AFTER have separate recursion visitation so one does not suppress the other; both share the cascade depth bound.

Schema, CHECK, attribution, and outgoing FK validation may precede BEFORE; uniqueness is enforced during the source apply. DELETE runs BEFORE before its referential cascade/restrict processing. All body effects roll back if any later validation or write fails. Missing rows and updates suppressed by `SkipUnchangedUpdates` do not fire row triggers.

A body's writes are checked like any other write, even when a foreign key's `Cascade` or `SetNull` action fired the trigger: a body write that references a missing row fails with `FK_VIOLATION`, and the source statement rolls back with the body. Only the removal action's own rewrite of a referencing row skips rechecking that row's other references.

## Row images

`BEFORE` and `AFTER` trigger bodies can read the source row through read-only pseudo-documents:

| Event | Available images |
|---|---|
| `INSERT` | `NEW` |
| `UPDATE` | `OLD`, `NEW` |
| `DELETE` | `OLD` |

Use ordinary bracketed field paths, including nested paths:

```mql
CREATE TRIGGER [audit_status] ON [Sales].[Orders] AFTER UPDATE AS BEGIN
    INSERT INTO [Sales].[StatusAudit] (
        [OrderId]: NEW.[Id],
        [OldStatus]: OLD.[Status],
        [NewStatus]: NEW.[Status],
        [Region]: NEW.[Shipping].[Region]
    )
END;
```

The aliases are values, not assignment targets. An unavailable alias, an unknown field path, or an attempt to write through `OLD` or `NEW` fails when the trigger is created. Byte values are copied defensively. An empty Object `()` remains present through OLD/NEW copies and satisfies `WHEN (NEW.[ObjectField] IS NOT NULL)`. NULL values, NULL parent Objects, and fields absent from the active subtype remain NULL. A scalar pinned reference reads as its identifier, as in every expression, so an image copy of it fits an identifier field; a pinned field accepts only `PIN(id, revision)` and rejects the bare identifier. Row images also work for temporal collections; a later version exposes the prior version through `OLD` and the inserted version through `NEW`.

## Body restrictions

Trigger bodies can contain only DML and read queries. `CREATE TRIGGER` rejects:

- Transaction control: `BEGIN TRANSACTION` / `COMMIT` / `ROLLBACK` / `JOIN TRANSACTION` (would corrupt the source's transaction).
- Migration control: `BEGIN MIGRATION` / `COMMIT MIGRATION` / `MERGE` / `SCRIPT`.
- DDL of any kind: `CREATE` / `ALTER` / `DROP` for collections, indexes, enums, triggers, views, mat views, databases, users, roles.
- Maintenance: `BACKUP` / `RESTORE` / `DEFRAGMENT` / `SHRINK` / `RECLAIM ORPHAN PAGES` / `ANALYZE` / `SET CREATEDBY`.
- Subscriptions, statement-cancellation, `LIST`, `DESCRIBE`, `EXPLAIN`.

The error message identifies which shape is disallowed.

`SEAL` and `UNSEAL` are admitted DML inside a trigger body and execute in the firing transaction. A seal toggle does not itself produce an INSERT, UPDATE, or DELETE trigger event, so it cannot recursively fire row triggers on its collection. It does maintain incremental materialized views that read row metadata ([materialized views](materialized-views.md#lifecycle-and-dependency-rules)), whose backing triggers then fire for the rows that change.

BEFORE and AFTER bodies obey the target row's seal state, including when fired by an enum rewrite that preserves a sealed source row. The internal source rewrite allowance does not authorize trigger mutations. A rejected body rolls back the rewrite and its migration.

## Temporal collections

Triggers fire on temporal collections for all three events, with one semantic shift: the event is dispatched on **document existence**, not on the MQL verb. INSERT, UPDATE, and UPSERT on a temporal collection all create a new version through the same engine path, so the verb is not recoverable at the firing seam — and at the document level, a new version of an existing document *is* an update:

- The **first version** of a document fires `BEFORE INSERT` and `AFTER INSERT`.
- Any **later version onto an existing timeline** fires `BEFORE UPDATE` and `AFTER UPDATE` — including a literal `INSERT` statement whose PK already has a timeline (temporal INSERT-on-existing creates a new version rather than erroring), and whether or not a version is valid at the new version's start.
- `OLD` is the version the write replaces: the version valid at the new version's start, which an `UPDATE ... VALID FROM` supersedes and merges. An INSERT into a gap of the timeline reads the nearest earlier version, or, before the first version, the nearest later one.
- `DELETE` — including `DELETE ... AT` with a future timestamp — fires `BEFORE DELETE` and `AFTER DELETE` **at write time**: the trigger observes the statement, not the later expiry tick. `OLD` is the version the delete ends; the scheduled versions it cancels fire no triggers.
- `CANCEL VERSION` does not fire triggers (temporal cancellation is a timeline edit, not a row event — see the EXPLAIN note below).

Static `EXPLAIN INSERT` on a temporal collection annotates **both** the INSERT and UPDATE trigger lists at both timings, since either can fire depending on whether the PK already has a timeline.

## Limitations

Both timings execute inside the source transaction. These body and identity rules keep nested execution within that transaction:
- **Bounded ordered bodies.** A body contains 1–256 DML or read statements, separated by semicolons, with an optional final semicolon. Empty statements reject. Every result is consumed before the next statement starts; failure rolls back all earlier body effects and the source write.
- **Combined events.** `INSERT OR UPDATE`, `UPDATE OR DELETE`, and all three events share one body. Repeated events reject. Image references must be available for every selected event: NEW for INSERT/UPDATE, OLD for UPDATE/DELETE.
- **Globally-unique trigger names per database.** Postgres-style per-table scoping (`CREATE TRIGGER same_name ON different_table`) is not supported.
- **`[Mnemic]` / `[Mnemic.*]` namespace is rejected.** No triggers on catalog collections — would create re-entrant write loops on engine open and migration replay.
- **Collection lifecycle owns its triggers.** `DROP COLLECTION` deletes the target's trigger catalog rows in the same migration.
- **A trigger protects what its body names.** `DROP COLLECTION`, `DROP VIEW` (with or without `CASCADE`), `DROP MATERIALIZED VIEW`, `DROP ENUM` and `DROP TYPE` of an object that the body or WHEN condition of a trigger on another target reads, writes or names are refused at the statement with `DEPENDENT_OBJECTS_EXIST` naming the trigger (`DROP COLLECTION '[App].[Rates]' rejected: trigger 'price_log' reads or writes it. Drop or replace the trigger first.`), and the refusal ends the migration. Drop or replace the trigger earlier in the same migration, then drop the object. Collection and namespace renames preserve target and body identities without changing catalog rows. Static body references continue to address their original objects even when an old name is reused, so a drop by a name is judged by the triggers bound to the relation the name holds at that statement, and a body write's WHERE that qualifies a path with its target's name follows the rename as a read query's qualifiers do. Rendered definition text uses current names.
- **Schema changes validate trigger dependencies.** Compatible `ALTER COLLECTION` changes preserve triggers. Checks cover OLD/NEW and WHEN, read queries, body assignment targets and expressions, predicates, and RETURNING, including collections used by a trigger owned elsewhere. A body write's WHERE and RETURNING are checked as they bind when the body runs, so such a change rejects a RETURNING list exactly when the write would reject it: the check protects the fields the write reads under the [RETURNING name rule](dml.md#returning), and an explicit list names the first path the write rejects (`Unknown field '…' in RETURNING.`), a `RETURNING *` the write's own diagnostic. `CREATE TRIGGER` binds the body the same way (see Validation rules). Invalid dependencies name the trigger and abort the migration; drop or replace the trigger earlier in the same migration to let the change commit. Earlier trigger drops/replacements participate; commit rechecks current definitions and rejects conflicting source changes. Validation does not execute bodies or require their runtime parameter values.
- **A persisted trigger whose target vanished out-of-band is fatal.** An embedded host bypass or inconsistent restore can evade the DDL guard above. The next MQL executor initialization raises `TRIGGER_TARGET_MISSING` as catalog corruption instead of keeping the trigger dormant, because the orphaned row shows the catalog no longer matches its own DDL history.
- **Persisted timing and event values are an exact domain.** The catalog accepts `Timing = 0` (`BEFORE`) or `1` (`AFTER`) and a nonempty combination of event bits `1`, `2`, and `4` (`INSERT`, `UPDATE`, `DELETE`). Unknown timing values, zero events, and unknown event bits fail initialization with `TRIGGER_TIMING_INVALID` or `TRIGGER_EVENT_INVALID`; they are never treated as dormant triggers.

**Firing order.** When several triggers share the same collection, timing, and event, they fire in **trigger-name order** (case-insensitive, ordinal) — the same rule PostgreSQL uses. The order is deterministic and identical before and after a database restart, so an order-dependent side effect (e.g. two audit inserts) is stable. It does **not** follow creation order; name your triggers to control it.

Internal incremental-view maintenance for the source mutation completes before its user `AFTER` triggers run. A trigger that reads the incremental backing therefore sees the mutation's derived row in the shared transaction. If the trigger writes the same source row again, that nested mutation is later and its maintenance result remains authoritative after the outer hook completes. A trigger or maintenance failure still rolls back the complete source, backing, and trigger-effect transaction.

## Validation rules

`CREATE TRIGGER` rejects:

- a duplicate trigger name (must be globally unique per database);
- an unknown target collection (must already exist at CREATE time);
- a target collection in the `[Mnemic]` / `[Mnemic.*]` namespace;
- an alias unavailable for the selected event;
- an unknown or virtual row-image field, including a child of a virtual computed container;
- a body that does not bind as a later schema change revalidates it: a field a read query, write predicate, assignment expression or RETURNING list names that its collection does not declare, a write that leaves a NOT NULL field unset, or an array element path where one value per row is read (RETURNING, a SET value, a read's column, key or operand; [element paths read no single value](queries.md#array-element-paths-read-no-single-value)). `CREATE TRIGGER [t] ON [App].[Orders] AFTER UPDATE AS BEGIN UPDATE [App].[Items] SET ([Qty]: [Nope] + 1) WHERE [Id] = 1 END` reports `UNKNOWN_FIELD` `Invalid definition for trigger 't': Unknown field 'Nope' in expression.` Binding reads catalog metadata only; it does not execute the body or need its runtime values.

`DROP TRIGGER` rejects:

- an unknown trigger name;
- a mismatched `ON` clause (the trigger is registered ON one collection, the DROP names a different one — guards against typos).

Programmatically constructed `CREATE` and `DROP TRIGGER` statements must use the exact canonical trigger and `[Ns].[Collection]` identities accepted by parsed MQL. Malformed UTF-16 and non-canonical names fail before any collection or catalog lookup, so they cannot alias a different persisted trigger.

## Persistence

Triggers persist to the catalog collection `[Mnemic].[Triggers]`. Public queries render names and executable text from retained identities:

| Field           | Type            | Notes |
|-----------------|-----------------|-------|
| `Id`            | `GUID`          | primary key, synthesized at CREATE |
| `Name`          | `STRING`        | the trigger's globally unique name. UNIQUE secondary index (case-insensitive) — duplicates are rejected at the catalog level. |
| `CollectionId`  | `INT32`         | physical target identity; zero for a materialized-view target |
| `MaterializedViewId` | `GUID`     | logical materialized target identity; empty for an ordinary collection |
| `CollectionName`| `STRING`        | target name resolved from the caller's catalog snapshot |
| `Timing`        | `INT8`          | `0 = Before`, `1 = After`; every other value is invalid |
| `Event`         | `INT8`          | nonempty bitmask of `1 = Insert`, `2 = Update`, `4 = Delete`; zero and unknown bits are invalid |
| `BodyText`      | `STRING`        | body with current static-reference names and original OLD/NEW references |
| `WhenText`      | `STRING NULLABLE` | optional condition with current static enum names |
| `CreatedAt`     | `DATETIMEOFFSET`| UTC timestamp at CREATE time |

Internally, persisted source syntax is accompanied by structural body and WHEN bindings. Reopen restores those static dependencies by ID, including materialized targets whose physical backing has been replaced. Dynamic enum-name expressions remain runtime name lookups. Historical migration text remains unchanged.

Trigger definitions survive engine restart. Every `MqlExecutor` construction ensures the catalog has been scanned, validates targets and the exact timing/event domains, and re-parses each `BodyText` into an executable AST before any statement can run. Invalid persisted metadata surfaces as catalog corruption rather than silently disabling a trigger. Malformed required strings, unsafe/non-DML bodies, missing targets, and duplicate persisted names all fail closed before publication; Mnemic never chooses or deletes a conflicting executable definition automatically.

The transactional guarantee applies to writes executed through MQL. An engine-internal write with no active MQL session has no authorization or statement context in which to execute a stored body; if it reaches the trigger hook, matching triggers are skipped and the omission is logged.

## Comparison with `IMutationLogSubscriber`

Both observe writes; they solve different problems:

| | **Triggers** | **`IMutationLogSubscriber`** |
|---|---|---|
| Surface | MQL DDL (`CREATE TRIGGER`) | Plugin (C# in a separate assembly) |
| When | Inside the writing transaction | Inline after one mutation attempt is applied, before commit outcome |
| Consistency | Strictly transactional | Best-effort attempt observation; rollback phantoms, retry duplicates, and subscriber-local misses are possible |
| Body | MQL statement | C# code |
| Persistence | `[Mnemic].[Triggers]` survives restart | Host/plugin registration; not catalog-persisted |
| Typical use | Audit / counter / derived rows that *must* match the source | Fast in-process observation that tolerates speculative or missed events |

The two coexist. Use a trigger when the side-effect must commit or roll back atomically with the source. Use an `IMutationLogSubscriber` only when inline observation is required and its best-effort delivery model is acceptable. The callback has no operating-system thread affinity and is not a durable CDC/outbox surface; external lossless delivery needs independently durable capture.

## Observability

Two OpenTelemetry-compatible meter instruments published on the `Mnemic.Engine` meter:

| Instrument | Tags | What it measures |
|---|---|---|
| `mnemic.trigger.fired_count` (`Counter<long>`) | `trigger`, `collection` | One increment per BEFORE or AFTER body attempt. |
| `mnemic.trigger.body_us` (`Histogram<double>`) | `trigger` | Wall-clock microseconds spent awaiting the trigger body's nested statement. |

Failed and cancelled body attempts contribute to both instruments, so `body_us.sum / fired_count.sum` describes the same population. When neither a duration listener nor an analysis collector is active, the trigger path skips the duration clock reads and emits no duration observation.

Static `EXPLAIN` of `INSERT` / `UPDATE` / `REPLACE` / `DELETE` / `UPSERT` annotates the `Notes` column with the matching BEFORE and AFTER triggers' names when the target collection has any registered:

```
EXPLAIN INSERT INTO [Sales].[Orders] ([Id]: 1, [Total]: 10)
-- Notes column: "Fires 1 AFTER INSERT trigger(s): [audit_orders]"
```

UPSERT (`INSERT ... ON CONFLICT`) lists both event sets — the INSERT triggers at both timings (no-conflict branch) and, when `DO UPDATE` is specified, also the UPDATE triggers at both timings (conflict branch). CANCEL VERSION is intentionally not annotated (temporal cancellation doesn't fire triggers).

A successful `EXPLAIN ANALYZE` result adds one `TriggerFire(<name> ON <collection>)` stage per body it ran, with elapsed microseconds and rows produced.

## See also

- [System collections](system-collections.md) — the catalog views, including `[Mnemic].[Triggers]`.

## Detailed syntax and execution

Trigger OLD/NEW references can read stored computed fields. Both timings observe the already materialized value, including generated identities and stored subtype dependencies. An extension field absent from the active subtype reads NULL. Image paths require compatible declarations and durable containing fields; a child of a virtual computed container is not stored image data. The same binding checks run before trigger catalog hydration publishes executable bodies.



<a id="syntax-create--drop-trigger"></a>

### CREATE / DROP TRIGGER

```sql
CREATE TRIGGER [name] ON [Ns].[Coll]
    BEFORE | AFTER {INSERT | UPDATE | DELETE} [OR {INSERT | UPDATE | DELETE} ...]
    [FOR EACH ROW]
    [WHEN (<condition>)]
    AS BEGIN <statement> [; <statement> ...] [;] END;

DROP TRIGGER [name] ON [Ns].[Coll];

FROM [Mnemic].[Triggers];   -- catalog
```

Both statements are schema DDL and require `BEGIN MIGRATION` / `COMMIT MIGRATION`. Catalog and executable-registry publication occurs at commit; until then, writes in the defining migration use the previous committed trigger set.

**Row execution:**

`WHEN (<condition>)` follows the optional `FOR EACH ROW` and precedes `AS BEGIN`. It is a scalar BOOL condition over the event's read-only OLD/NEW images; only TRUE executes the body. FALSE and NULL skip it. Available images, stored paths, function arity, and the result type are validated before persistence and again on reopen. Bare field names, caller parameters, subqueries, and window functions are rejected; qualify row values as `NEW.[Value]` or `OLD.[Value]`. Scalar functions and NOW use the source statement's function snapshot and timestamp.

The condition is compiled once with the definition and evaluated immediately before that trigger's body at its BEFORE or AFTER boundary. It does not query the source collection again or replace an image. An evaluation error or cancellation rolls back the source mutation and any earlier trigger effects. A skipped condition does not increment the fired-body counter. Persisted condition syntax and identity bindings survive rename; public `WhenText` renders current enum names and OLD/NEW references.

- `BEFORE` runs after resolving the candidate image, before the source row and its indexes change. `AFTER` runs after apply and IVM maintenance. Both run in name order within their timing and join the source transaction. Body failure vetoes the statement; NEW cannot be replaced. Source validation may precede BEFORE, uniqueness follows during apply, and DELETE's FK processing follows BEFORE. A later failure rolls back every body effect. Missing rows and suppressed unchanged updates do not fire.
- A BEFORE body cannot recursively write its own pending source row; the entire statement fails with `TRIGGERED_DATA_CHANGE_VIOLATION` to prevent a stale outer write. Other-row chains work within the shared depth bound. BEFORE, AFTER, foreign-key removal actions and incremental maintenance each track their own visits, so a row a foreign key's removal check visited, referencing rows or not, still fires its AFTER DELETE triggers once.
- Both timings expose read-only row images: INSERT has `NEW.[path]`, UPDATE has `OLD.[path]` and `NEW.[path]`, and DELETE has `OLD.[path]`. Nested bracketed paths are supported. Unavailable aliases (`NEW is not available in a DELETE trigger at position …`) and unknown or virtual paths fail at CREATE; image values cannot be assignment targets: `SET (NEW.[Status]: 'x')` reports `SYNTAX_ERROR` `NEW.[Status] is a read-only row image and cannot be assigned (position …).`
- Bodies contain 1–256 DML or read statements in source order, separated by `;`. A final `;` is optional; empty statements reject. Each result is consumed before the next statement starts, and any failure rolls back the full body and source.
- Combine distinct events with `OR`, such as `INSERT OR UPDATE`. Each selected event fires the body once. Image references must be available for every event: NEW for INSERT/UPDATE, OLD for UPDATE/DELETE.
- Globally-unique trigger names per database (no per-collection scoping).
- `[Mnemic]` / `[Mnemic.*]` collections cannot be the trigger target.

`DROP COLLECTION` deletes every trigger owned by the target in the same migration. Dropping a collection, view, materialized view, enum or named type that another target's trigger body or WHEN names is refused with `DEPENDENT_OBJECTS_EXIST` naming the trigger; drop or replace the trigger first. Collection and namespace renames preserve catalog rows and retained target/body identities. Public text renders names from the current snapshot; reusing a former name cannot redirect a static body reference. Schema ALTER COLLECTION permits compatible changes after binding affected OLD/NEW and WHEN paths, read queries, mutation targets and expressions, predicates, conflict updates, and RETURNING. A trigger also protects collections referenced by its body. Invalid dependencies name the trigger and abort the migration. Pending definitions/removals/retargeting participate; commit rechecks current definitions and source generations. Binding does not execute a body or require invocation parameter values.

`DROP TRIGGER` requires the same `ON [Coll]` clause used at CREATE — mismatch is rejected, not silently accepted. Programmatic CREATE / DROP nodes must carry the exact canonical trigger and `[Ns].[Collection]` identities; malformed UTF-16 and non-canonical names fail before collection or catalog lookup and cannot alias another persisted trigger.

Trigger bodies fire **inside the source's transaction** — a body throw rolls back the source write. Foreign-key cascades, trigger bodies and incremental view maintenance that one write sets off share a nesting bound of 100 levels; the next level fails the statement with `LIMIT_EXCEEDED` and rolls back the source write.

Bodies may execute `SEAL` / `UNSEAL`. A seal toggle has its own mutation/change kind but does not recursively produce INSERT/UPDATE/DELETE trigger events on its collection; incremental views that read row metadata are maintained for it.

Internal incremental-view maintenance for a mutation completes before its user `AFTER` triggers run. A trigger can therefore read that mutation's derived row inside the shared transaction. If it writes the same source row again, the nested write is later and its incremental-maintenance result remains authoritative.

The guarantee applies to MQL-executed writes. Engine-internal work with no active MQL statement cannot synthesize a privileged session for a stored body; matching triggers are skipped and the omission is logged.

This transactional behavior differs from the plugin `IMutationLogSubscriber` surface. That callback observes an applied mutation attempt inline before commit outcome, has no thread-affinity guarantee, and does not provide at-least-once or committed delivery. Ordinary subscriber failure is isolated without retry or write veto. Use MQL triggers when derived state must commit or roll back atomically with its source.

**Temporal collections.** Triggers fire on temporal collections too, but the event dispatches on **document existence**, not the MQL verb (INSERT / UPDATE / UPSERT all create a new version through one engine path): the FIRST version fires INSERT triggers at both timings; any later version onto an existing timeline fires UPDATE triggers at both timings — including a literal `INSERT` whose PK already has a timeline, whether or not a version is valid at its start. `OLD` is the version valid at the new version's start, else the nearest earlier version, else the nearest later one. `DELETE` (including a future-dated `DELETE ... AT`) fires BEFORE and AFTER DELETE at write time, not at the expiry tick. `CANCEL VERSION` fires no triggers.

**Observability.** Static `EXPLAIN` of `INSERT` / `UPDATE` / `REPLACE` / `DELETE` / `UPSERT` annotates the `Notes` column with the names of BEFORE and AFTER triggers that can fire on the matching event (`Fires N AFTER INSERT trigger(s): [t1], [t2]`). UPSERT lists both INSERT and (on `DO UPDATE`) UPDATE triggers; INSERT on a temporal collection also lists both (an existing timeline makes it fire UPDATE triggers); CANCEL VERSION is intentionally not annotated. Two meter instruments on the `Mnemic.Engine` meter — `mnemic.trigger.fired_count` (per fire) and `mnemic.trigger.body_us` (per-fire wall time) — surface runtime trigger cost. Failed and cancelled bodies contribute to both instruments; when neither duration telemetry nor analysis is active, duration clock reads are skipped. `EXPLAIN ANALYZE` reports successful trigger-body stages with elapsed time and produced rows.

See [`triggers.md`](triggers.md).

## Usage pitfalls


<a id="pitfall-alter-enum-remove-value--replace-can-be-blocked-by-check-constraints-and-triggers"></a>

### `ALTER ENUM REMOVE VALUE … REPLACE` can be blocked by CHECK constraints and triggers

The `REPLACE` document rewrite runs as ordinary UPDATEs/replaces, so everything that gates a normal write gates the rewrite too: a CHECK constraint on a referencing field that rejects the replacement value, or an `AFTER UPDATE` trigger body that throws, makes the whole ALTER fail — fail-closed, the enum definition is untouched and the in-migration rewrite rolls back. Retire the conflicting CHECK ([`DROP CHECK`](ddl.md#add-and-drop-check-constraints)) first, or pick a replacement value the constraints accept. A blocking trigger must be dropped in an earlier migration that has committed: writes inside a migration, the rewrite included, fire the committed trigger set, so a `DROP TRIGGER` earlier in the same migration does not stop the trigger from firing on the rewrite.

Additionally, after the rewrite a **post-condition re-probe** verifies the old value is gone everywhere. This includes nested and array enums in both the base and exact subtype-extension halves of a collection-level polymorphic `[base_len][base][ext]` envelope. The other half of the envelope is preserved during each rewrite. If a malformed or unsupported row cannot be migrated, the re-probe makes the ALTER fail naming the collection and leaves the enum definition unchanged.

Array-bearing enum paths are validated before replacement. Truncated or trailing array framing, invalid object offset tables, and malformed polymorphic element envelopes/discriminators raise `ARRAY_PAYLOAD_CORRUPT`; no corrupt element is skipped or carried through, and the migration leaves the enum catalog unchanged.

<a id="pitfall-triggers-on-temporal-collections-dispatch-on-document-existence-not-the-mql-verb"></a>

### Triggers on temporal collections dispatch on document existence, not the MQL verb

On a temporal collection, INSERT / UPDATE / UPSERT all create a new version through one engine path, so the trigger event keys on whether the document already had a timeline: the FIRST version fires INSERT at both timings; any later version fires UPDATE at both timings — **including a literal `INSERT` whose PK already exists** (temporal INSERT-on-existing is legal and versions the document), even into a gap of its timeline. `OLD` is the version the write replaces at its start, not the version current at the clock: an `UPDATE ... VALID FROM` a scheduled instant sees the scheduled version. `DELETE` — even a future-dated `DELETE ... AT` — fires BEFORE and AFTER DELETE at write time, not at the expiry tick. `CANCEL VERSION` fires no triggers.

<a id="pitfall-collection-lifecycle-preserves-and-validates-trigger-definitions"></a>

### Collection lifecycle preserves and validates trigger definitions

`DROP COLLECTION` deletes the target's trigger catalog rows in the same migration. A trigger is bound to its target's identity, never its name, so a collection that later reuses the name starts without triggers. Collection and namespace renames preserve target and body identities in the same commit outcome. Persisted syntax and structural bindings survive renames and reopen. Current text renders the objects' current names; a static reference cannot be redirected by reusing an old name. A body write's WHERE may qualify a path with its target's short or namespace-qualified name (`WHERE [Docs].[Id] = NEW.[Id]`), also from a nested query correlated to the written row; those qualifiers follow a rename of the target or its namespace. A RETURNING list names paths relative to the written row and takes no qualifier.

Schema `ALTER COLLECTION` binds affected triggers against the resulting catalog. Compatible additions and unused-field edits succeed. Invalid OLD/NEW or WHEN paths, read-query inputs, assignment targets/expressions, predicates, and RETURNING fields name the trigger and abort the migration; a body write's RETURNING list is admitted as the write admits it when the body runs, so the check protects the fields the write reads. With `RETURNING [Docs].[Title]` on a target that declares an Object field `Docs`, dropping `[Docs]` is rejected and dropping the unused top-level `[Title]` succeeds; adding a `Body` root rejects a list that used `[Body]` shorthand, and a new subtype whose extension conflicts with another's rejects `RETURNING *`. A rejected explicit list names its first path the write rejects; a rejected `RETURNING *` reports the write's diagnostic. Body dependencies protect other collections too. Pending CREATEs, DROPs, replacements, and retargeting participate; commit rechecks current definitions and source generations. The check never runs a trigger body or requires runtime parameter values.

If an embedded host bypass or inconsistent restore nevertheless leaves a trigger row whose target no longer exists, the next MQL executor initialization fails closed with `TRIGGER_TARGET_MISSING`. The engine never keeps the trigger dormant: the orphaned row is evidence of a catalog changed outside its own DDL, so it is restored from backup or removed after an audit.

Persisted trigger enums are exact: `Timing = 0` (`BEFORE`) or `1` (`AFTER`) and a nonempty combination of event bits `1`, `2`, and `4` is executable. Zero and unknown event bits fail executor initialization with `TRIGGER_TIMING_INVALID` or `TRIGGER_EVENT_INVALID`; they never become silent no-op triggers.

<a id="pitfall-caller-cancellation-reaches-nested-trigger-statements"></a>

### Caller cancellation reaches nested trigger statements

Trigger bodies execute inside the source transaction and inherit the source statement's cancellation token. Cancelling the caller stops nested trigger DML and rolls back both source and trigger effects; the nested body does not continue as a detached statement after the source begins unwinding.

<a id="pitfall-multiple-same-event-triggers-fire-in-trigger-name-order-not-creation-order"></a>

### Multiple same-event triggers fire in trigger-NAME order, not creation order

When several triggers share the same collection + timing + event, they fire in **case-insensitive trigger-name order** (PostgreSQL parity), deterministic and identical before and after a restart. Do not rely on creation order — an order-dependent side effect (e.g. two audit-inserts) is keyed on the names. Name the triggers (`[a_first]`, `[b_second]`) to control the sequence.

<a id="pitfall-trigger-bodies-cannot-contain-transaction-control-migration-control-or-ddl"></a>

### Trigger bodies cannot contain transaction control, migration control, or DDL

`CREATE TRIGGER` rejects `BEGIN TRANSACTION` / `COMMIT` / `ROLLBACK` / `BEGIN MIGRATION` / `CREATE COLLECTION` / `DROP INDEX` / `BACKUP` / etc. inside a trigger body. Trigger bodies can only run DML (`INSERT` / `UPDATE` / `REPLACE` / `DELETE` / `SEAL` / `UNSEAL` / `CANCEL VERSION`) or read queries — anything else escapes the trigger sandbox or corrupts the source transaction.

SEAL/UNSEAL do not themselves fire INSERT/UPDATE/DELETE triggers, so a toggle in a trigger body cannot recursively trigger on the toggle; the incremental-view maintenance it causes shares the firing statement's nesting bound.

<a id="pitfall-before-sees-the-pending-image-but-cannot-replace-it"></a>

### `BEFORE` sees the pending image but cannot replace it

Every new candidate runs BEFORE validation, including a later rewrite from an AFTER body. BEFORE recursion is bounded by live nesting; AFTER retains its whole-cascade visited-row suppression.

`BEFORE INSERT|UPDATE|DELETE` executes before changing the source row and indexes. OLD/NEW remain read-only. A body failure vetoes the statement, including all earlier body effects. Validation may precede BEFORE; successful bodies do not bypass CHECK, FK, or uniqueness enforcement. A BEFORE body that recursively writes its pending source row fails (`TRIGGERED_DATA_CHANGE_VIOLATION`) rather than letting an outer merge overwrite a nested update. BEFORE and AFTER each run in name order and have separate recursion visitation.

<a id="pitfall-trigger-row-image-aliases-are-event-specific-and-read-only"></a>

### Trigger row-image aliases are event-specific and read-only

INSERT exposes `NEW.[Field]`, UPDATE exposes `OLD.[Field]` and `NEW.[Field]`, and DELETE exposes `OLD.[Field]`. Nested bracketed paths work. An unavailable alias or unknown or virtual path fails when the trigger is created. The images are immutable input values; they cannot be used as DML targets or assignment destinations.

<a id="pitfall-trigger-bodies-execute-ordered-statement-lists"></a>

### Trigger bodies execute ordered statement lists

`AS BEGIN <stmt>; <stmt>; END` runs 1–256 DML or read statements in order. A final semicolon is optional; empty statements reject. Semicolons in strings, bracketed identifiers, and comments are not separators. Every statement passes the body allowlist, including statements after the first. Any later failure rolls back the source mutation and all earlier body effects. Top-level requests still contain one statement; the separator belongs to trigger bodies.

<a id="pitfall-trigger-names-are-globally-unique-per-database"></a>

### Trigger names are globally unique per database

There is no per-collection scoping. `CREATE TRIGGER [t1] ON [A].[X]` followed by `CREATE TRIGGER [t1] ON [B].[Y]` is rejected — the second CREATE collides on name even though the targets differ.

<a id="pitfall-drop-trigger-requires-the-same-on-clause-as-create"></a>

### `DROP TRIGGER` requires the same `ON` clause as `CREATE`

```sql
CREATE TRIGGER [t1] ON [Sales].[Orders] AFTER INSERT AS BEGIN ... END;
-- correct
DROP TRIGGER [t1] ON [Sales].[Orders];
-- rejected ('registered ON [Sales].[Orders], not ON [Sales].[Audit]')
DROP TRIGGER [t1] ON [Sales].[Audit];
```

The mismatch error guards against typos that would otherwise silently drop a different trigger.

<a id="pitfall-triggers-on-mnemic--mnemic-are-rejected"></a>

### Triggers on `[Mnemic]` / `[Mnemic.*]` are rejected

The catalog namespace cannot be a trigger target — would create re-entrant write loops on engine open and migration replay.

<a id="pitfall-trigger-when-runs-only-for-true"></a>

### Trigger WHEN runs only for TRUE

`WHEN (<condition>)` follows the optional `FOR EACH ROW` and precedes `AS BEGIN`. It is a scalar BOOL condition over the event's read-only OLD/NEW images; only TRUE executes the body. FALSE and NULL skip it. Available images, stored paths, function arity, and the result type are validated before persistence and again on reopen. Bare field names, caller parameters, subqueries, and window functions are rejected; qualify row values as `NEW.[Value]` or `OLD.[Value]`. Scalar functions and NOW use the source statement's function snapshot and timestamp.

The condition is compiled once with the definition and evaluated immediately before that trigger's body at its BEFORE or AFTER boundary. It does not query the source collection again or replace an image. An evaluation error or cancellation rolls back the source mutation and any earlier trigger effects. A skipped condition does not increment the fired-body counter. Persisted condition syntax and identity bindings survive rename; public `WhenText` renders current enum names and OLD/NEW references.
