# Schema migrations

> **In this section** — `BEGIN MIGRATION` / `COMMIT MIGRATION`, schema-level tracking, multi-step migrations, `MERGE MIGRATION` / `SCRIPT MIGRATION`.

Migrations wrap DDL changes atomically and track schema level:

```sql
-- Start migration from expected schema level (0 = fresh database)
BEGIN MIGRATION 0;

CREATE COLLECTION [Store].[Products]
WITH SCHEMA ([Id] GUID PRIMARY KEY, [Name] STRING, [Price] DECIMAL);

COMMIT MIGRATION;
-- Schema level incremented to 1

-- Subsequent migration
BEGIN MIGRATION 1;

ALTER COLLECTION [Store].[Products] ADD FIELD [Currency] STRING DEFAULT 'USD';
CREATE INDEX ON [Store].[Products] ([Name]);

COMMIT MIGRATION;
-- Schema level incremented to 2
```

`BEGIN MIGRATION <fromLevel>` first acquires exclusive migration admission, then validates that the database is exactly at the expected level: a database below or above it is rejected with `User / SCHEMA_LEVEL_MISMATCH` ("Schema level mismatch: expected 5 but database is at 4."), and no migration starts. A successful `COMMIT MIGRATION` applies the migration atomically and advances that level by one. Admission is exclusive, so nothing else can move the level between the check and that commit; the history persists to `[Mnemic].[Migrations]`.

> **Exception — online index builds run OUTSIDE a migration.** `CREATE INDEX … WITH (ONLINE = ON)` and `ALTER INDEX … INCLUDE … WITH (ONLINE = ON)` commit independently (write lock released during the backfill) and are **rejected inside** `BEGIN MIGRATION` / `COMMIT MIGRATION` — the same standalone rule as PostgreSQL `CREATE INDEX CONCURRENTLY` — and the refusal ends the open migration like any failed statement. Run them as bare statements. See [Indexes → Online index build](indexes.md#online-index-build--with-online--on).

## Multi-step migration

Each DDL sees the prior's effect because they share a migration transaction:

```sql
BEGIN MIGRATION 2;

CREATE ENUM [Catalog].[Status] (INT32)
VALUES (1 = 'Draft', 2 = 'Published', 3 = 'Archived');

ALTER COLLECTION [Catalog].[Articles]
  ADD FIELD [Status] [Catalog].[Status] DEFAULT 1;

CREATE INDEX ON [Catalog].[Articles] ([Status]);
COMMIT MIGRATION;
```

Row-rewriting ALTER and offline index DDL can follow DML or earlier rewrites of the same collection in one migration. Each scan includes the owner's dirty pages and unspliced appended rows, translated through the current private schema. Later statements see that complete result; commit or abort settles all rows, indexes, and schema changes together. For example, ADD a field with a default, backfill it with UPDATE, then CREATE INDEX without an intervening commit. Rewrites may allocate additional pages as rows grow; new rows and index locations remain transaction-owned through abort, commit, and recovery.

Standalone raw-engine rewrites still refuse a collection owned by an open data transaction. A competing migration is rejected at BEGIN until the owner settles. See the [combined migration example](examples.md#migration-with-multiple-ddl-steps).

> Once a migration stages any query-visible field or subtype schema change, committed readers and competing writers for that collection are fenced until `COMMIT MIGRATION` or rollback publishes/restores one complete schema+row+index generation. DML in the owning migration is still allowed after the ALTER and uses the new metadata/layout; its queries and FK probes also read that private generation without waiting on its own fence. If an owner query stream remains open at `COMMIT MIGRATION`, its generation pin transfers atomically to a read lease and a later rewrite waits for that stream to drain. Rollback cannot preserve the abandoned private generation: `ABORT MIGRATION` or automatic rollback invalidates such a stream, and its next row request fails before decode. The private catalog overlay is rebound only during each lazy iterator operation and restored before control returns, so consumer back-pressure never leaks it into another statement. Subtype-only changes advance both collection and extension-schema versions, invalidating cached plans that mention old extension fields. Competing DML, index DDL, or schema DDL may receive a transient structural/lock error; retry the statement after the migration settles. A query submitted after the generation fence is already held waits before lowering its plan, then uses the newly published generation. A lazy result created before the ALTER but first enumerated after publication instead fails closed because its already-lowered descriptor belongs to the old generation. If catalog publication cancels a concurrently admitted statement at this boundary, that statement returns transient code `CATALOG_PUBLICATION_RETRY` rather than continuing against a mixed generation. Retry the complete statement; its next execution binds the newly committed catalog snapshot. Within one statement every reference to a collection (a self-join, a CTE or view read twice, set-operation branches, nested queries and the rows they evaluate, the statement `EXPLAIN ANALYZE` measures) reads the schema generation the statement captured first. A migration that commits meanwhile drains the statement as described above; it never changes what the statement's later references read. Inside a migration, the owner's own statements are not drained: if a later statement of the migration changes a collection while an earlier statement's rows are still being read, the nested queries those remaining rows evaluate (scalar subqueries, `APPLY` bodies, correlated `HAVING` and `WHERE` subqueries) read the changed collection.

## MERGE / SCRIPT MIGRATION

```sql
-- Consolidate migration history (squash levels 0-5 into one)
MERGE MIGRATION 0 TO 5;

-- Generate DDL script to replay migrations from level 3 to 7
SCRIPT MIGRATION 3 TO 7;
```

`MERGE MIGRATION` rewrites the `[Mnemic].[Migrations]` history so the range `(from..to]` becomes a single entry at level `to` (the same per-level window `SCRIPT MIGRATION` reads — the record at level N holds the statements that *produced* level N, so the record at `from` itself is outside the range and survives). The rewrite is atomic — the per-level deletes and the merged record commit under one transaction. The merged entry retains each original migration's statement group; it does not flatten distinct transactions into one replay transaction. A later MERGE range must start at an intact group boundary and cover every level through `to`; a range that cuts through an earlier merged entry or a history gap is rejected instead of silently moving or omitting DDL. Both levels must be non-negative, `from` must be less than `to`, and `to` may not exceed the current committed schema level. Frontier validation, history reads, and the rewrite serialize with concurrent `COMMIT MIGRATION` / `MERGE MIGRATION` operations, so a commit cannot overwrite a future-target merged row. The **live schema level is unchanged**: MERGE compacts history, it does not move the database to a different level, so the next `BEGIN MIGRATION` continues from the current level and never overwrites the merged record. Run MERGE only outside an active transaction or migration.

Once the rewrite transaction has durably applied, an ordinary post-commit observer failure is logged and MERGE reports success so the committed rewrite cannot be retried. A wrapped memory-exhaustion or simulated-crash failure waits for rewrite-transaction and spill-owner cleanup, then escapes with its exact outer exception identity; the history rewrite remains committed and non-retryable.

`SCRIPT MIGRATION` returns one row with a `Script` string containing the MQL statements that reproduce the range. Statement boundaries are NUL (`U+0000`), not line breaks: split the value on `\0` and pass each non-empty frame to `ExecuteAsync`. Literal newlines remain inside their original statement. NUL is unambiguous because valid MQL rejects it. Like MERGE, both levels must be non-negative, `from` must be less than `to`, and `to` may not exceed the current committed schema level. Run SCRIPT only outside an active transaction or migration. Scripting a merged range emits empty `BEGIN/COMMIT` blocks only for migrations that were originally empty and otherwise restores the original transaction boundaries. This matters for scan-based DDL such as two row-rewriting ALTERs that must commit separately. For both MERGE and SCRIPT, structural-latch release is retried once. If the operation and both release attempts fail, the executor preserves both failures in one aggregate instead of replacing the operation failure with cleanup. The requested range must cover complete retained groups; a range that begins inside an earlier merged entry or crosses an uncovered history level errors. The entire history range is read once under the same frontier latch used by MERGE and COMMIT, so a concurrently requested history rewrite waits and the returned script never mixes rows from before and after that rewrite.

A present `[Mnemic].[Migrations]` row must contain a non-blank, exactly framed UTF-8 `Statements` value. If that catalog field is null, blank, truncated, or otherwise malformed, both SCRIPT and MERGE fail with `DDL_CATALOG_STRING_INVALID`; the row is never treated as an empty migration.

MERGE and SCRIPT assemble output incrementally and refuse a result above `EngineOptions.MaxDocumentBytes` before allocating an oversized history/script string. MERGE's potentially large set of history-row ids is bounded by `MaxInMemoryStreamingState`, admission-accounted, and spills to authenticated temporary storage. The size failure publishes neither a partial history group nor any catalog rewrite.

Migration-history rows use exactly `MQLH1` with NUL-delimited framing. The header is `MQLH1\0`, followed by the transaction-group count and, for each group, its statement count and statement frames. Unframed, newline-delimited, count-free, truncated, or alternate grouped encodings are rejected. A history row of any other shape is refused ([compatibility](https://mnemic.io/docs/compatibility/)).

## Concurrent migrations

Only one migration may be active **per database**, including its setup and terminal cleanup. Another connection's `BEGIN MIGRATION` fails with transient `MIGRATION_IN_PROGRESS`; it does not wait, fork an overlay, or start a transaction. This applies even when the migrations would touch disjoint collections. Ordinary statement admission, cancellation, and brief publication fences still apply.

The owner retains admission until commit, abort, automatic rollback, or connection cleanup finishes. A failed begin releases admission. Ordinary statements can continue against committed metadata under the existing collection-generation and publication fences. Migrations on different databases remain independent.

Retry after the owner finishes using the migration's original expected level. If the owner committed, a stale level fails with `User / SCHEMA_LEVEL_MISMATCH`. Do not replace it with the current level blindly: the intended upgrade may already have been applied. Exclusion covers one migration block, not an entire multi-block application upgrade or a named migration registry.

Private catalog, enum, and type snapshots remain invisible to other sessions until commit. Structural identity checks also protect publication against maintenance and standalone online index DDL.

**Migrations are atomic for schema metadata.** Every DDL the block runs — `CREATE` / `DROP` / `RENAME COLLECTION`, `ALTER NAMESPACE RENAME`, index DDL, enum DDL, named-type DDL, and `ALTER` schema changes — participates in the migration transaction. If the migration aborts for any reason (a statement errors, a plugin commit hook throws, FK validation fails, or you issue `ABORT MIGRATION`), committed metadata is left exactly as it was before `BEGIN MIGRATION`: created collections/enums/types vanish, dropped objects and collection data come back, and renames/index changes revert. One migration's enum or type can never ride another migration's WAL commit.

<a id="a-failed-statement-ends-the-migration"></a>

**Any failed statement ends the migration — one that fails to parse or is refused included.** A statement sent into an open migration that returns an error rolls the whole migration back before the error is returned, over TCP and in-process alike: one that fails while it executes, one whose text does not parse (`SYNTAX_ERROR`, or `LIMIT_EXCEEDED` / `INVALID_VALUE` from the parser), and one the engine refuses before running it — online index DDL, DML on a view or a materialized view, a read a plugin's read gate vetoes, an engine-role or health refusal, and the session-control commands that cannot run inside a migration (`BEGIN TRANSACTION`, `JOIN TRANSACTION`, a second `BEGIN MIGRATION`, `COMMIT`, `MERGE MIGRATION`, `SCRIPT MIGRATION`). Nothing before the failure publishes: the `COMMIT MIGRATION` that follows fails with `INVALID_TRANSACTION_STATE` ("No migration in progress."), and the schema level is unchanged, so correct the script and run the migration again from its `BEGIN MIGRATION`. Over TCP a request the server does not run ends the open migration the same way, in that request's place in the request order: one refused over the pending budget (`REQUEST_QUEUE_FULL`) once the requests queued before it ran, one refused by authorization, a malformed one, and one cancelled before it ran ([connecting](connecting.md#session-execution-ownership)). Nothing resumes a migration that ended: statements a client pipelined behind the failure run outside it, against the committed schema — DDL there is refused (online index DDL aside, which runs on its own), while DML such as a backfill `UPDATE` runs in autocommit and persists — and only the trailing `COMMIT MIGRATION` reports that no migration is in progress. Send a migration's statements one at a time, each behind the answer of the one before it.

`COMMIT MIGRATION` failures are interpreted by durability phase. An unexpected failure before the WAL commit marker rolls the migration back and clears the session state, so it may be corrected and retried. Once all WAL-governed pages have applied, a later ordinary observer failure does not turn the durable migration into an error: the catalog overlay is published and `COMMIT MIGRATION` succeeds. A wrapped memory-exhaustion or simulated-crash signal still waits for that overlay publication and terminal owner cleanup, then escapes with its outer exception identity; the committed migration must not be retried. If the marker is durable but page apply is incomplete, the command returns an error and the engine enters a non-resumable `EngineWide` quarantine with code `COMMIT_APPLY_FAILED_AFTER_MARKER`. The direct `FROM [Mnemic].[Health]` diagnostic plus rollback, migration-abort, and query-cancellation controls remain accepted; close and reopen the database so WAL recovery finishes the commit before issuing ordinary MQL. Do not retry the migration based on that error.

`ABORT MIGRATION` explicitly cancels the in-flight migration — the user-initiated counterpart of the automatic rollback that fires when a statement errors mid-block. It rolls the migration transaction back, leaves the schema level unchanged, and releases the migration so a corrected one can start:

```sql
BEGIN MIGRATION 3;
CREATE COLLECTION [Store].[Draft] WITH SCHEMA ([Id] GUID PRIMARY KEY);
-- changed your mind:
ABORT MIGRATION;
-- schema level still 3, [Store].[Draft] never created
```

`ABORT MIGRATION` errors with `"No migration in progress."` outside a migration block. (A plain transaction is cancelled with `ROLLBACK`, not `ABORT`.) Inside a migration, `ROLLBACK` aborts it exactly as `ABORT MIGRATION` does and returns the same result; closing the session aborts an open migration the same way.

**Foreign-key targets are validated at `COMMIT MIGRATION`.** A `CREATE COLLECTION` may declare a foreign key to a collection that doesn't exist *yet* — forward references are tolerated so self-referential (`A.Parent → A`) and cyclic (`A ↔ B`) FKs can be declared inline in one migration. At `COMMIT MIGRATION`, after every statement has applied, a closing pass re-checks the FKs registered during the migration against the now-complete catalog: every referenced collection must resolve, the FK field type must match the target's PK type, and every `PINNED REFERENCES` target must be temporal. Any failure aborts and rolls back the **whole** migration (the collections it created are dropped, so a corrected migration can be retried) with a collection+field-qualified error. This is deferred constraint checking at the commit boundary — the analogue of SQL `SET CONSTRAINTS … DEFERRED`. (FKs added via `ALTER ADD FIELD` are validated eagerly against the live catalog instead — see [DDL](ddl.md#alter-collection).)

An existing target binds by collection ID when the declaration is accepted. A forward target binds to the first relation that takes its name in the migration, whichever statement gives the name one: `CREATE COLLECTION`, `CREATE MATERIALIZED VIEW`, `ALTER COLLECTION … RENAME TO`, or `ALTER NAMESPACE … RENAME TO`, as a rename or a merge, moving a collection under that name. From then on the key follows that relation, never the name: renaming the relation away keeps the key on it, under its new name, and a relation that takes the name afterwards is another one. A key no statement of the migration gave a target is refused at commit as referencing a collection that does not exist. Commit validates the merged catalog, including all committed and migration-owned dependencies. Dropping and recreating a target under the same name does not satisfy a foreign key bound to the retired collection: commit refuses it with `DEPENDENT_OBJECTS_EXIST`, naming the key the drop left and the target by the name the migration dropped it under.

`BEGIN MIGRATION` and an explicit `BEGIN TRANSACTION` are mutually exclusive. `BEGIN MIGRATION` inside a transaction is rejected with `"Cannot begin migration while a transaction is active."` and leaves the transaction open; `COMMIT` or `ROLLBACK` it first. `BEGIN TRANSACTION` inside a migration is rejected too, and the refusal ends the migration like any failed statement: a migration's statements already run in its own transaction. `MERGE MIGRATION` and `SCRIPT MIGRATION` likewise refuse to run inside an in-flight migration or transaction.

For engine-role admission, `SCRIPT MIGRATION` is a read and remains available in `ReadOnly` and `Draining`. `MERGE MIGRATION` rewrites committed history as an independent mutation and therefore requires `Writable`. An already-open migration may continue issuing its scoped DDL while draining; completion or abort remains available after either non-writable transition.

## See also

- [Schema definition (DDL)](ddl.md) — every CREATE / ALTER / DROP form
- [Indexes](indexes.md) — backfill rules and ALTER INDEX ordering
- [System virtual collections](system-collections.md) — `[Mnemic].[Migrations]` is the persistent history collection
- [Practical examples](examples.md#schema-level-sync-with-begin-migration-fromlevel)

## Detailed syntax and execution


<a id="syntax-8-migrations"></a>

### Migrations

```sql
BEGIN MIGRATION 1;
  CREATE COLLECTION [...];
  CREATE INDEX [...];
COMMIT MIGRATION;

-- Explicitly cancel the in-flight migration. Rolls every catalog mutation in
-- the block back atomically, leaves the schema level unchanged, and releases
-- the migration. Errors "No migration in progress." outside a block. (A plain
-- transaction is cancelled with ROLLBACK, not ABORT.)
BEGIN MIGRATION 1;
  CREATE COLLECTION [...];
ABORT MIGRATION;

-- History squash: collapse the transitions (2..5] — the records at
-- levels 3, 4, 5 — into ONE [Mnemic].[Migrations] record at level 5.
-- Atomic (one transaction); the LIVE schema level is unchanged (MERGE
-- rewrites history, it does not move the database to another level).
-- The compacted row retains each original transaction group so SCRIPT
-- recreates those BEGIN/COMMIT boundaries. A range that cuts through an
-- earlier merged group or an uncovered history level is rejected.
-- History uses exact MQLH1 NUL framing with explicit group and statement
-- counts. Unframed, count-free, truncated, or alternate shapes are rejected.
-- TO must not exceed the current committed schema level. Validation,
-- reads, and rewrite serialize with concurrent COMMIT/MERGE operations.
-- Levels must be non-negative and FROM < TO. Run outside an active
-- transaction or migration.
-- No other clauses — there is no SCHEMA / statement-block form.
MERGE  MIGRATION 2 TO 5;

-- Replay-script generation: returns one row with a [Script] column —
-- the BEGIN/COMMIT-wrapped MQL that reproduces transitions (3..7].
-- The string is statement-framed with NUL (U+0000), not newline: split
-- on '\0' and execute each non-empty frame. Literal newlines stay intact.
-- Merged history restores the original migration transaction boundaries;
-- originally-empty migrations remain empty BEGIN/COMMIT pairs. The requested
-- range must cover whole retained groups, not start inside a prior merge.
-- A malformed history row cannot stand in for an absent lower-boundary row.
-- The range read serializes with COMMIT/MERGE, so one script cannot mix
-- history rows from before and after a concurrent rewrite.
-- TO must not exceed the current committed schema level.
-- Levels must be non-negative and FROM < TO. Run outside an active
-- transaction or migration.
-- Read-only; takes no statement block.
SCRIPT MIGRATION 3 TO 7;
```

For engine-role admission, `SCRIPT MIGRATION` is a read and remains available in `ReadOnly` and `Draining`. `MERGE MIGRATION` rewrites committed history as an independent mutation and requires `Writable`. DDL inside a migration that was already open may continue while draining; completion or abort remains available after either non-writable transition.

A present `[Mnemic].[Migrations]` row must carry a non-blank, exactly framed UTF-8 `Statements` value. `SCRIPT MIGRATION` and `MERGE MIGRATION` return `DDL_CATALOG_STRING_INVALID` for a null, blank, truncated, or malformed value; they never reinterpret catalog corruption as an empty migration. MERGE and SCRIPT build output incrementally under `MaxDocumentBytes`; an oversized range fails before publishing a partial group. MERGE's history-row id set is admission-accounted and spills above `MaxInMemoryStreamingState`. After MERGE durably applies its rewrite, ordinary post-commit observer failures are logged and the statement reports success. A wrapped memory-exhaustion or simulated-crash failure escapes with its exact outer identity only after the rewrite transaction and spill owner are cleaned up; the rewrite remains committed and must not be retried. MERGE and SCRIPT retry structural-latch release once. If the operation and both release attempts fail, both failures are preserved in one aggregate rather than letting cleanup replace the primary failure.

Database-schema and DDL-registry mutations — `CREATE/ALTER/DROP/RENAME COLLECTION`, `RENAME NAMESPACE`, `CREATE/ALTER/DROP INDEX`, `CREATE/DROP STATISTICS`, `CREATE/ALTER/DROP ENUM`, `CREATE/ALTER/DROP TYPE`, `SET CREATEDBY`, `CREATE/DROP TRIGGER`, `CREATE/ALTER/DROP VIEW`, and `CREATE/ALTER/REFRESH/DROP MATERIALIZED VIEW` — run inside a migration by default; bare `CREATE COLLECTION` (or any of the migration-scoped forms) outside `BEGIN MIGRATION` is rejected. One migration at a time owns admission for the database. A competing BEGIN fails with `MIGRATION_IN_PROGRESS`, and its expected level is validated only after it acquires admission. Online index DDL is the schema-changing exception: it commits independently, must run standalone, and is rejected inside a migration, which the refusal ends. Read-only DDL (`DESCRIBE COLLECTION`, `EXPLAIN`) also runs standalone. History persists in `[Mnemic].[Migrations]`.

Atomic catalog publication cancels a statement admitted against the retiring generation rather than allowing its old descriptor to resume against new metadata. The statement returns transient `CATALOG_PUBLICATION_RETRY`; retry the complete statement so it binds the newly committed generation.

Migrations are **atomic for catalog mutations**: every DDL in the block participates in the migration transaction, so an abort (statement error, throwing plugin commit hook, FK-validation failure, or explicit `ABORT MIGRATION`) leaves the catalog exactly as it was before `BEGIN MIGRATION` — created collections vanish, dropped collections and their data return, renames revert, built/dropped indexes and enums are restored.

COMMIT failures are phase-aware: a pre-marker failure rolls back and clears the migration state; an ordinary post-apply observer failure still publishes the durable overlay and succeeds. Wrapped memory exhaustion and simulated crash wait for overlay publication and terminal owner cleanup, then escape without making the committed migration retryable. A marker-durable but incomplete page apply returns an error, files non-resumable `COMMIT_APPLY_FAILED_AFTER_MARKER` against `EngineWide`, and retains only the direct health diagnostic plus rollback, migration-abort, and query-cancel controls until close + reopen recovery. Do not retry that migration based on its error result.

See [`migrations.md`](migrations.md).

## Usage pitfalls


<a id="pitfall-concurrent-migrations-are-rejected"></a>

### Concurrent migrations are rejected

A second session receives `MIGRATION_IN_PROGRESS` while a migration owns the database. After the owner commits, retrying the original `BEGIN MIGRATION n` fails if its expected level is stale. After abort, the unchanged level may be retried. Different databases can migrate concurrently.

Atomic catalog publication may cancel a statement admitted against the retiring generation. It returns transient `CATALOG_PUBLICATION_RETRY`; retry the complete statement so it binds one fresh committed generation. Do not continue consuming or reuse a descriptor from the cancelled execution.

<a id="pitfall-merge-preserves-migration-boundaries-ranges-cannot-cut-through-a-merge"></a>

### MERGE preserves migration boundaries; ranges cannot cut through a merge

`MERGE MIGRATION` stores one physical history row at `to`, but retains every original transaction group. `SCRIPT MIGRATION` therefore replays separate migrations separately, preserving each original commit boundary even when the same collection is rewritten more than once. Start later MERGE/SCRIPT ranges on an intact history-group boundary. A range beginning inside a previously merged entry, or crossing a history gap, errors instead of replaying the wrong statements. History uses exact `MQLH1` NUL framing with explicit transaction-group and statement counts. Unframed, newline-delimited, count-free, truncated, and alternate grouped shapes are rejected. Likewise, a present history row with null, blank, truncated, or malformed `Statements` fails as `DDL_CATALOG_STRING_INVALID`; do not treat that error as an empty migration or retry by deleting the row without an audit.

The persisted trigger, view, and materialized-view catalogs use the same error when a required identity, body, target, or primary-key string is blank or malformed. The executor will not normalize such values to empty text and publish anonymous executable metadata. Duplicate persisted executable names fail as `DDL_CATALOG_DUPLICATE_NAME`; Mnemic never guesses a winner or deletes catalog rows during hydration. Their synthesized `Id` values must decode as non-empty, unique GUIDs. Empty or repeated identities fail as `DDL_CATALOG_ID_INVALID` before a UNIQUE-on-`Name` index or executable registry generation is published.

<a id="pitfall-begin-migration-takes-the-exact-current-level"></a>

### BEGIN MIGRATION takes the exact current level

`BEGIN MIGRATION n` names the level the database is at when the migration starts, not a minimum: a database at any other level, lower or higher, rejects it with `SCHEMA_LEVEL_MISMATCH` and starts nothing. The highest `Level` in `[Mnemic].[Migrations]` is the current level (an empty history is level 0), which a refused attempt's message also names. A committed migration advances the level by one, so the next block begins at `n + 1`; an aborted or rolled-back one leaves it at `n`.

<a id="pitfall-ddl-outside-a-migration-block"></a>

### DDL outside a migration block

Database-schema and DDL-registry mutations — `CREATE/ALTER/DROP/RENAME COLLECTION`, `RENAME NAMESPACE`, `CREATE/ALTER/DROP INDEX`, `CREATE/DROP STATISTICS`, `CREATE/ALTER/DROP ENUM`, `CREATE/ALTER/DROP TYPE`, `SET CREATEDBY`, `CREATE/DROP TRIGGER`, `CREATE/ALTER/DROP VIEW`, and `CREATE/ALTER/REFRESH/DROP MATERIALIZED VIEW` — are rejected outside `BEGIN MIGRATION ... COMMIT MIGRATION` by default. Online index DDL is the schema-changing exception: it must run standalone, and inside a migration it is refused and ends the migration. A second migration on the same database is rejected at BEGIN (see "Concurrent migrations" above). Read-only DDL (`DESCRIBE COLLECTION`, `EXPLAIN`) is also exempt — run those as standalone statements.

<a id="pitfall-dropcreate-name-reuse-follows-migration-order"></a>

### DROP/CREATE name reuse follows migration order

A materialized view can be dropped and recreated in either mode within one migration, including an INCREMENTAL view whose initial seed was created in that block. DROP closes that seed and retains every old incremental source fence. Owner writes between DROP and CREATE are valid; later writes maintain a new INCREMENTAL definition. MANUAL replacements retain their seed snapshot. Older DROP callbacks cannot unregister the replacement. Abort restores the pre-migration identity and maintenance, or leaves a newly introduced name absent. Definition changes retain exact ownership instead of restoring obsolete subscriptions. DROP still checks dependent views and triggers.
