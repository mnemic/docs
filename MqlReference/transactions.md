# Transactions

> **In this section** — `BEGIN TRANSACTION` / `COMMIT` / `ROLLBACK`, failed-statement abort (the transaction becomes abort-only), first-writer-wins conflict detection, snapshot isolation, the opt-in `SERIALIZABLE` isolation tier, distributed (cross-connection) transactions.

## Single-connection transactions

```sql
-- Minimal commit
BEGIN TRANSACTION;
INSERT INTO [Store].[Products] ([Id]: '...', [Name]: 'Widget', [Price]: 10);
UPDATE [Store].[Inventory] SET ([Count]: 100) WHERE [ProductId] = '...';
COMMIT;

-- Explicit rollback
BEGIN TRANSACTION;
DELETE FROM [App].[Drafts] WHERE [UserId] = @userId;
ROLLBACK;
```

All operations between `BEGIN TRANSACTION` and `COMMIT` / `ROLLBACK` are atomic; `COMMIT` and `ROLLBACK` take no `TRANSACTION` keyword. First-writer-wins conflict detection is per page. Logical snapshot visibility does not hold a writer for the transaction's lifetime, but physical page and relocation safety can still cause bounded waits.

Queries inside a transaction include its own inserts, updates, and deletes. This applies to ordered scans and indexed predicates as well as primary-key lookups. The access path can change while index updates remain uncommitted; filtering, ordering, and paging still apply to the transaction's current rows.

Lazy query results may be consumed after a successful `COMMIT`. A `ROLLBACK` instead invalidates every still-open result created in that transaction; starting or resuming its row stream then fails before another row is decoded. Drain or dispose transaction-owned results before rolling back. Completion drains reads already executing, including their nested reference lookups, before publishing the outcome. Independently started row pulls wait for that outcome; canceling a pre-decision commit reopens read admission.

`ROLLBACK` reports success only after every load-bearing undo completes. If an eager index/count/catalog undo fails, the statement returns an error, clears the session transaction state, records recovery-required `ROLLBACK_CLEANUP_FAILED` in `[Mnemic].[Health]`, and refuses non-diagnostic MQL until close + reopen. Never retry work in the same process after this outcome. The same terminal-state rule applies to `COMMIT`: outcome publication and transaction disposal are both attempted, session state is cleared even when either fails, and a commit failure plus cleanup failure is preserved together under `ROLLBACK_CLEANUP_FAILED` rather than leaving a dead transaction attached to the session.

Plugin/executor recursion does not create a second terminal authority. If a nested helper attempts `COMMIT`, `ROLLBACK`, `COMMIT MIGRATION`, or `ABORT MIGRATION` while its parent statement still owns an operation on the same transaction, the child returns `TRANSACTION_COMPLETION_INSIDE_OPERATION`. This is a pre-terminal rejection: the transaction or migration remains active and no distributed outcome is published. If an ordinary nested DML error is ignored by a plugin, the parent still returns `TX_ABORTED`; for a migration, the rollback is deferred only until the parent operation exits and then the entire migration is rolled back.

### Session statement ownership

One `MqlExecutor` — one TCP connection or one embedded session; an embedded database opens more with `OpenSessionAsync` — is one logical session, and statements on different sessions run and commit concurrently. If multiple callers submit top-level statements to the same session at once, the executor runs them single-flight so transaction, migration, cancellation, snapshot, stage-observation, and `SET` state cannot overlap. Acquisition order between truly simultaneous callers is unspecified; await one statement before submitting the next whenever application order matters (especially around `BEGIN`, `COMMIT`, and `ROLLBACK`). A caller canceled while waiting receives the normal `Statement cancelled.` error without entering executor state.

Internal same-flow statements (trigger bodies, IVM maintenance, EXPLAIN, CTE, and set-operation helpers) are reentrant and remain inside the outer execution flow. The single-flight lease ends when the `MqlResult` is returned, before a `RowsAsync` stream is drained, so consuming a slow result stream does not hold the session gate.

### Isolation levels

```sql
BEGIN TRANSACTION;                          -- snapshot isolation (the default)
BEGIN TRANSACTION ISOLATION SNAPSHOT;       -- explicit snapshot
BEGIN TRANSACTION ISOLATION SERIALIZABLE;   -- opt-in serializable
```

`BEGIN TRANSACTION` defaults to the **snapshot conflict mode**: writers conflict first-writer-wins per page, preventing lost updates, but write skew is still possible when transactions read a shared set and write disjoint parts. Repeatable-read visibility is controlled separately by the host's opt-in `EngineOptions.SnapshotIsolation`. When enabled, every collection read in the explicit transaction observes its transaction-start snapshot (plus its own writes), for both in-process and TCP clients. This includes the target-selection read for `UPDATE`, `REPLACE`, and `DELETE`: a row committed after `BEGIN` is not eligible for mutation by that transaction. Each matching primary key appears at most once in a snapshot result, including when a concurrent update physically relocates the row. Temporal segment identity additionally includes `ValidFrom`: body and validity metadata resolve at the transaction CSN before `AS OF`, `IN BETWEEN`, or current-time filtering is applied. The same membership guarantees apply to auto-commit read statements when snapshot isolation is enabled, including temporal primary-key lookups and history ranges with lock-free reads: concurrent commits cannot add post-snapshot rows or duplicate an already observed segment. When disabled (the no-version-capture default), each read observes its own committed read window and a later statement may see a concurrent commit; the isolation clause still controls write-conflict and serializable validation behavior. A read statement still returns every row that stays committed while it runs exactly once, as PostgreSQL's `READ COMMITTED` does, outside an explicit transaction and inside one alike, and with `EngineOptions.LockFreeReads` on or off: a concurrent `UPDATE` or `REPLACE` that commits while the statement runs, and moves the row to another page or changes the value its index finds it by, neither hides the row nor returns it twice. The statement returns the row in its before image or its after image, each evaluated against its `WHERE`, so a row whose new value no longer matches can be absent. A row a concurrent statement inserts or deletes may or may not appear.

While a snapshot is open, the engine keeps in memory every version it can still read, up to `EngineOptions.MaxSnapshotHistoryBytes`. A commit whose superseded versions would cross that budget fails with `SNAPSHOT_HISTORY_LIMIT` (`ResourcePressure`) and changes nothing; end long-running transactions and retry. The history is never written to the database file, so no snapshot outlives the engine process.

`BEGIN TRANSACTION ISOLATION SERIALIZABLE` adds **commit-time read-set validation**: the transaction records the latest committed mutation identity of every collection it reads and re-checks it at commit. If any collection it read was written by another committed transaction, the `COMMIT` aborts with a transient `TX_CONFLICT_SERIALIZABLE` error — retry the whole transaction. With `SnapshotIsolation` enabled, a collection mutation newer than the transaction-start snapshot also conflicts even when the transaction delays its first read until after that mutation commits. This eliminates write skew among serializable transactions. Snapshot transactions pay no serializable read-set or decision-gate cost; only serializable transactions pay the validation.

Validation is **collection-grain**: a serializable transaction conflicts if any write touched a collection it read, even a write to a *different* row — so a serializable read of a hot collection can be aborted by unrelated concurrent writes to it. Keep serializable transactions short and retry on `TX_CONFLICT_SERIALIZABLE` (its error class is `Conflict`, the same retry contract as `TX_CONFLICT_FWW`).

### Money-transfer pattern

Multi-document mutation with atomic rollback on business-rule failure:

```sql
BEGIN TRANSACTION;

UPDATE [Bank].[Accounts] SET ([Balance]: 900)  WHERE [Id] = @from EXPECTED REVISION @fromRev;
UPDATE [Bank].[Accounts] SET ([Balance]: 1100) WHERE [Id] = @to   EXPECTED REVISION @toRev;
INSERT INTO [Bank].[Ledger] ([Id]: @txId, [From]: @from, [To]: @to, [Amount]: 100, [At]: NOW);

COMMIT;
-- Any failing EXPECTED REVISION aborts the whole sequence — no partial state on disk.
```

Cannot nest transactions. Cannot begin a transaction while a migration is active, or vice versa; inside a migration the refused `BEGIN TRANSACTION` ends the migration, as any failed statement does ([migrations](migrations.md#a-failed-statement-ends-the-migration)). `ANALYZE`, `BACKUP`, `RESTORE … VERIFY`, `DEFRAGMENT`, `SHRINK`, `RECLAIM ORPHAN PAGES`, and the [DIAGRAM statements](diagrams.md#diagram-statements-stand-alone) also require no transaction or migration in flight.

### A failed statement aborts the transaction

A DML statement (`INSERT` / `UPDATE` / `REPLACE` / `DELETE` / `CANCEL VERSION`) that **fails inside an explicit transaction marks the transaction abort-only** — the PostgreSQL contract (SQLSTATE 25P02 `in_failed_sql_transaction`):

```sql
BEGIN TRANSACTION;
INSERT INTO [App].[Items] ([Id]: 2, [Name]: 'ok');     -- staged
INSERT INTO [App].[Items] ([Id]: 1, [Name]: 'dup');    -- Error: duplicate PK
FROM [App].[Items];
-- Error: Current transaction is aborted — commands ignored until ROLLBACK …
COMMIT;
-- Error: COMMIT refused — the transaction was aborted by an earlier failed
-- statement and has been rolled back. …
```

- Every statement after the failure is refused with error code `TX_ABORTED` until the transaction ends. `ROLLBACK` is the clean exit; `COMMIT` is also accepted but **acts as a rollback** and reports the refusal — it never persists partial work.
- The rule is uniform across all DML failures (constraint violation, FK violation, type mismatch, cancellation mid-statement): there is no per-statement undo for the transaction's buffered writes, so after any failed DML the only safe exit is rolling back the whole transaction.
- **Statement text that does not parse aborts too.** Text that fails to parse (`SYNTAX_ERROR`, or `LIMIT_EXCEEDED` / `INVALID_VALUE` from the parser) could have been any statement, a write included, so it marks the transaction abort-only exactly like a failed DML statement, over TCP and in-process alike; the writes before the typo never commit.
- **A request the server did not run aborts too.** A client can send statements behind ones it has not seen answered, so over TCP every request the server does not run — refused because the connection's pending budget was full (`REQUEST_QUEUE_FULL`), refused by authorization, malformed, or cancelled before it started — marks the open transaction abort-only (an open migration rolls back) exactly like text that does not parse, once every request sent before it ran. The refusal's answer may arrive before the answers of earlier requests; its effect never lands before them. A `COMMIT` sent behind such a request therefore commits nothing, whatever the request was ([connecting](connecting.md#session-execution-ownership)).
- **A transaction the session lost refuses every statement until `ROLLBACK`.** When the server rolls an idle transaction back, or the creator of a transaction the session joined ends it, the session's statements are refused with `TX_ABORTED`, naming why the transaction was lost, until its `ROLLBACK` ends the lost transaction (`COMMIT` ends it too, reporting that nothing was committed); a statement sent behind one the client has not seen answered never runs in autocommit outside the transaction it was sent into. This is a rule of TCP sessions, where idle rollback exists and statements can be pipelined.
- **Failed reads do not abort** — a query that parses and runs but fails (unknown collection, unknown field, a type error) stages no writes, and the transaction continues. This is the one deliberate divergence from PostgreSQL, which aborts on read errors too.
- Auto-commit DML (no explicit transaction) is unaffected: a failed statement rolls back its own implicit mini-transaction.
- If that implicit rollback itself cannot complete an eager index/count/catalog cleanup step, the statement returns `IMPLICIT_ROLLBACK_CLEANUP_FAILED` instead of retrying. The engine becomes recovery-required: the direct `[Mnemic].[Health]` diagnostic plus rollback, migration-abort, and query-cancellation controls remain available until close + reopen.
- If a multi-match statement failure and its rollback/disposal both fail, the returned error preserves the statement failure together with every cleanup failure. After a durable WAL commit marker, the engine never reports or attempts rollback; reopen recovery completes the committed work.
- Structural DDL uses the same rule: an incomplete rollback/disposal is never hidden behind the original DDL error, and work past a durable commit marker is recovery-owned rather than reported as rolled back. `DROP FIELD` restores removed indexes only for an abort before that decision; recovery completes durable field and index removals together even if apply or publication fails.
- The abort applies to the shared transaction in distributed (`JOIN TRANSACTION`) sessions too: a joiner's failed DML poisons the transaction for every bound connection, and the creator's `COMMIT` is refused.

## Distributed (cross-connection) transactions

`BEGIN TRANSACTION` returns the transaction's identity (`TransactionId`, a Guid) to the issuing connection. It is the one identity a client sees for the transaction: `TX_CONFLICT_SERIALIZABLE` and `DEADLOCK` messages, `[Mnemic].[Transactions]`, `[Mnemic].[Locks]` and `[Mnemic].[SlowQueries]` all name the transaction by it. A second connection of the same user can bind to the same uncommitted transaction with `JOIN TRANSACTION`:

```sql
-- Connection A (creator):
BEGIN TRANSACTION;
-- returns { TransactionId: 'b1c5…9f' }
INSERT INTO [Store].[Orders] ([Id]: '...', [Total]: 100);

-- Connection B (joiner):
JOIN TRANSACTION 'b1c5…9f';
-- returns { TransactionId: 'b1c5…9f' } like BEGIN; the answer reports
-- the session holding a joined transaction, which connection pools
-- read to reset the connection before another tenant leases it
INSERT INTO [Store].[LineItems] ([OrderId]: '...', [Qty]: 1);

-- Connection A:
COMMIT;
```

### Rules

- The handle is the transaction's identity, not a secret, so joining is bound to the user that began the transaction (compared case-insensitively, as user names are): `JOIN TRANSACTION` from another user fails with `INSUFFICIENT_PRIVILEGE` and binds nothing. Sessions without a user, such as an embedded engine's executors, join each other's transactions.
- The identity is non-empty and registered with checked uniqueness: an engine identity that is empty or already registered fails closed with `DISTRIBUTED_TRANSACTION_IDENTITY_CONFLICT` and never replaces the registered transaction.
- Only the **creator** can issue `COMMIT` or `ROLLBACK`. A joiner's attempt is refused with `INVALID_TRANSACTION_STATE`: `"COMMIT can only be issued by the transaction creator."` or `"ROLLBACK can only be issued by the transaction creator."`
- Any bound connection can read and write — the engine serializes concurrent ops across connections via a per-transaction lock. If a joiner statement is already queued behind the creator's `COMMIT`, `ROLLBACK`, or disconnect cleanup, it re-checks the outcome after acquiring that lock and reports the terminal message below; it never dispatches against the completed transaction, and a plugin execution interceptor is consulted only after that check, under the same lock. Waiting for this lock is part of the statement lifetime: caller cancellation, `SET STATEMENT TIMEOUT`, `CANCEL QUERY`, creator-side cancellation, and migration-drain cancellation can retire the waiter before the current owner releases the lock, and the cancelled statement never dispatches.
- Legitimate trigger/plugin/query recursion on the same logical call chain reuses the already-owned transaction lock. That ownership is exact and flow-local; once its parent phase ends, delayed work with a captured context must queue normally and re-check the transaction outcome rather than bypass the lock.
- A joiner disconnect detaches silently and leaves the transaction intact.
- After the creator finishes (commit, rollback, or disconnect), any subsequent `JOIN TRANSACTION` on that handle returns `"Transaction '<id>' not found."`. An in-flight joiner's next statement reports the actual outcome with `INVALID_TRANSACTION_STATE`, and over TCP every later statement of that joiner is refused with `TX_ABORTED` naming the same outcome until the joiner's `ROLLBACK` ([a lost transaction](#a-failed-statement-aborts-the-transaction)):
  - `"The distributed transaction was committed by the creator; no further statements may be submitted."`
  - `"The distributed transaction was rolled back by the creator."`
  - `"Transaction has been aborted by the creator."` (on creator disconnect)

  Joiner retry logic can use this distinction to avoid double-applying committed work.
- The durable WAL commit marker is the outcome boundary. If the creator's `COMMIT` returns an error after that marker because page apply or flush failed, the engine enters recovery-required quarantine and reopen recovery completes the commit. In-flight joiners still receive the **committed** terminal message; never retry that transaction as rolled back. The failed commit also remains a hole in the process's snapshot watermark, so a higher disjoint commit cannot make the incomplete page state visible as a valid snapshot before reopen.
- The typed .NET client exposes this as `MnemicDatabase.BeginDistributedTransactionAsync()` / `JoinTransactionAsync(handle)`; the returned `TransactionHandle` is a simple record struct around the Guid.

## See also

- [DML](dml.md) — `EXPECTED REVISION`, `SKIP IF UNCHANGED`, `FORCE WRITE`
- [Migrations](migrations.md) — DDL transaction context

## Detailed syntax and execution


<a id="syntax-7-transactions"></a>

### Transactions

```sql
BEGIN TRANSACTION;                          -- snapshot isolation (default)
  -- multi-statement work
COMMIT;
ROLLBACK;

BEGIN TRANSACTION ISOLATION SNAPSHOT;       -- explicit snapshot (same as bare)
BEGIN TRANSACTION ISOLATION SERIALIZABLE;   -- opt-in serializable

-- distributed: BEGIN returns Guid, JOIN binds; only creator commits.
-- Both BEGIN and JOIN return { TransactionId }; every answer reports the
-- session state the issuing connection holds (connecting.md).
BEGIN TRANSACTION;
JOIN TRANSACTION '<guid>';
```

The returned handle is the transaction's own non-empty identity, registered with checked uniqueness; an empty or already-registered identity returns `DISTRIBUTED_TRANSACTION_IDENTITY_CONFLICT` and never replaces a live transaction. Only the user that began the transaction can join it (`INSUFFICIENT_PRIVILEGE` otherwise).

Distributed statements are serialized across bound connections. A joiner statement queued behind the creator's `COMMIT`, `ROLLBACK`, or disconnect cleanup observes that terminal outcome after it reaches the gate and fails with the corresponding committed/rolled-back/aborted message; it does not run against the completed transaction. The gate wait is itself cancellable by the caller token, statement timeout, `CANCEL QUERY`, creator-terminal signal, or migration drain; a cancelled waiter never dispatches and never releases a gate it did not acquire. The durable WAL commit marker is the outcome boundary: a later page-apply/flush failure may make `COMMIT` return an error and quarantine the engine, but joiners still observe **committed** because reopen recovery will finish that work. Its LSN remains a hole in the process's gap-free snapshot watermark, so a higher disjoint commit cannot make the incomplete apply a valid read cut. Never retry it as rolled back.

`BEGIN TRANSACTION` uses the **snapshot conflict mode** (first-writer-wins per page, prevents lost updates but NOT write skew; foreign-key and created-by references are the exception, see [concurrent removal](ddl.md#pitfall-a-reference-and-a-concurrent-removal-cannot-both-commit)). Repeatable-read visibility is the separate host option `EngineOptions.SnapshotIsolation`: when enabled, every collection read reuses the transaction-start snapshot (plus own writes), identically over in-process and TCP execution. DML target selection is part of that read contract: `UPDATE`, `REPLACE`, and `DELETE` cannot target a row committed after `BEGIN`. Each matching primary key appears at most once in a snapshot result, including when a concurrent update physically relocates the row. Temporal segment identity additionally includes `ValidFrom`; its body and validity metadata resolve at the CSN before valid-time filtering. When disabled, each statement observes its own committed read window and may see a concurrent commit. `BEGIN TRANSACTION ISOLATION SERIALIZABLE` adds commit-time read-set validation: the transaction records the latest committed mutation identity of every collection it reads and aborts the `COMMIT` with a transient `TX_CONFLICT_SERIALIZABLE` if any read collection was written under it — retry the whole transaction. With `SnapshotIsolation` enabled, a mutation newer than the transaction-start snapshot conflicts even when the first read of that collection happens after the mutation commits. Validation is collection-grain (a write to any row in a read collection conflicts), so keep serializable transactions short. The clause defaults to the engine's `DefaultIsolationLevel` (Snapshot) when omitted.

Logical snapshot visibility does not hold a writer for the transaction's lifetime, but physical page and relocation safety can still cause bounded waits, and a write whose unique key another transaction is removing waits for that transaction's outcome ([unique indexes](indexes.md)).

A bare DML statement runs in an implicit mini-transaction (always snapshot). A multi-row INSERT without an explicit transaction runs all its rows in one implicit mini-transaction of the session, which commits once after its last row and is retried as a whole on a transient conflict before it commits ([multi-row INSERT](dml.md#multi-row-insert)). The engine retries an implicit transaction that fails transiently while it writes or commits, before its commit marker — a write-write conflict, a deadlock victim, an expired lock wait or another transient failure — up to eight times with a jittered pause, and only then fails the statement with `WRITE_CONTENTION`. An attempt refused by another transaction it can name — one writing the data page the statement needs, one owning the index state it changes, or one whose pending removal claims a unique key it inserts — instead waits for that transaction to commit or roll back and then runs again, without spending a retry; the wait is bounded by its lock-wait deadline (`LOCK_TIMEOUT`) and shows in `[Mnemic].[Locks]`. An explicit transaction's statement that meets another transaction's page write fails with `TX_CONFLICT_FWW`. Its client sees none of these retries, which count by cause in `[Mnemic].[Metrics]` and appear in the server log ([virtual collections](system-collections.md#virtual-collections)). Multi-match UPDATE without an explicit transaction is wrapped in an implicit engine transaction — mid-iteration failures roll back all earlier mutations. If eager cleanup itself fails, retries stop with `IMPLICIT_ROLLBACK_CLEANUP_FAILED`; the direct health diagnostic plus rollback, migration-abort, and query-cancel controls remain available until close + reopen. The returned error preserves both the statement failure and every failed rollback/disposal step. A transaction whose WAL commit marker is already durable is never rolled back; reopen recovery completes its committed work. Standalone structural DDL follows the same phase-aware cleanup and error aggregation rules.

**A failed DML statement aborts the transaction** (PostgreSQL 25P02 posture): after any `INSERT` / `UPDATE` / `REPLACE` / `DELETE` / `CANCEL VERSION` error inside `BEGIN TRANSACTION`, the transaction is abort-only — every further statement errors with code `TX_ABORTED` until `ROLLBACK` (the clean exit) or `COMMIT` (accepted, but it rolls back and reports the refusal; partial work never persists). Statement text that does not parse aborts the same way, and so does a TCP request the server did not run (refused over the pending budget or by authorization, malformed, or cancelled before it started), in its place in the request order; a TCP session that lost its transaction (an idle rollback, a joined transaction its creator ended) refuses every statement with `TX_ABORTED` until `ROLLBACK`. Failed *reads* that parsed do not abort — the transaction continues (the one deliberate divergence from PostgreSQL). Retry pattern: `ROLLBACK`, then re-run the whole transaction. Lazy query rows remain consumable after a successful commit. Rollback invalidates still-open rows from that transaction; starting or resuming their stream fails before another row is decoded, so drain or dispose them first. `ROLLBACK` returns success only after every load-bearing undo completes. An undo failure returns an error, clears the session transaction, records recovery-required `ROLLBACK_CLEANUP_FAILED`, and retains only the direct health diagnostic plus rollback, migration-abort, and query-cancel controls until close + reopen.

A recursively invoked terminal command is not a second transaction owner. `COMMIT`, `ROLLBACK`, `COMMIT MIGRATION`, and `ABORT MIGRATION` return `TRANSACTION_COMPLETION_INSIDE_OPERATION` when a parent statement still owns an operation on that transaction; this leaves the transaction/migration active and publishes no distributed outcome. A plugin cannot hide a nested DML failure: the parent returns `TX_ABORTED`. In a migration, cleanup is deferred only until the parent operation exits and then rolls the entire block back.

## Usage pitfalls


<a id="pitfall-cancel-a-migration-with-abort-migration-a-transaction-with-rollback"></a>

### Cancel a migration with `ABORT MIGRATION`, a transaction with `ROLLBACK`

`ABORT MIGRATION` cancels the in-flight migration block (and atomically reverts every catalog mutation in it — created collections vanish, dropped collections + data return, renames/indexes/enums revert, `SET CREATEDBY` reverts to the prior mode; schema level unchanged). `ROLLBACK` cancels a `BEGIN TRANSACTION` data scope, and inside a migration it aborts the migration exactly as `ABORT MIGRATION` does. `ABORT MIGRATION` outside a migration errors with `"No migration in progress."`. You don't need `ABORT MIGRATION` on a statement error inside a block — any failed or refused statement rolls the whole migration back already; `ABORT MIGRATION` is for cancelling a still-valid migration you changed your mind about.

Lazy rows created inside a transaction or migration may be drained after a successful commit. Rollback is not equivalent: it invalidates every still-open row stream from that transaction, and the next row request fails before decode. Drain or dispose those results before `ROLLBACK` / `ABORT MIGRATION`. For a schema-changing ALTER, successful `COMMIT MIGRATION` atomically retains an open owner stream's schema-generation pin, so a later rewrite waits until the stream is disposed. The private catalog overlay is rebound only for each iterator call and restored before returning control; it does not leak across consumer back-pressure into a later statement.

<a id="pitfall-a-failed-dml-statement-aborts-the-explicit-transaction--rollback-and-retry-the-whole-thing"></a>

### A failed DML statement aborts the explicit transaction — ROLLBACK and retry the whole thing

Any `INSERT` / `UPDATE` / `REPLACE` / `DELETE` / `SEAL` / `UNSEAL` / `CANCEL VERSION` that fails inside `BEGIN TRANSACTION` (constraint violation, FK violation, type mismatch, cancellation) marks the transaction **abort-only** (PostgreSQL 25P02 posture). Every later statement — including reads — errors with code `TX_ABORTED` ("Current transaction is aborted — commands ignored until ROLLBACK …") until the transaction ends. `ROLLBACK` is the clean exit; `COMMIT` is accepted but **rolls back** and reports the refusal — partial work never persists. There is no "commit the statements that worked" — the engine has no per-statement undo, so retry means `ROLLBACK` + re-run the whole transaction. Statement text that does not parse aborts the same way, since it could have been a write, and over TCP so does every request the server did not run — refused over the pending budget or by authorization, malformed, cancelled while queued — even when its answer arrives before the answers of the statements sent ahead of it. Failed *reads* that parsed do NOT abort (the one divergence from PostgreSQL); auto-commit DML outside a transaction is unaffected (its implicit mini-transaction rolls back by itself).

Structural quarantine normally refuses ordinary reads as well as mutations; only a direct `[Mnemic].[Health]` query and rollback/abort/cancel/resume controls remain. A lone `WAL_FLUSH_FAILED` retains bounded queries, metadata reads, and `VERIFY DATABASE` while refusing every mutation, backup, and subscription registration (`SUBSCRIBE`, plugin channels included) until reopen. Any second quarantining fault closes those reads. In-flight append/fsync uncertainty adds `WAL_WRITE_FAILED`; incomplete post-marker page apply adds `COMMIT_APPLY_FAILED_AFTER_MARKER`, so both fail ordinary reads closed until reopen recovery. The reported domain is diagnostic scope, not permission to query unrelated collections. `PAGE_CHECKSUM` therefore closes ordinary MQL globally when reported by a live engine. `PAGE0_CHECKSUM` instead means open found no usable current-format header slot, so no live engine or health row exists to query. Repair the reported domain before resume. If an auto-commit attempt cannot finish an eager index/count/catalog undo, the engine does not retry or continue serving user reads. It returns `IMPLICIT_ROLLBACK_CLEANUP_FAILED`, retains only the direct `FROM [Mnemic].[Health]` diagnostic plus rollback, migration-abort, and query-cancel controls, and requires close + reopen; `ALTER ENGINE RESUME` cannot clear this recovery-required fault.

The same fail-closed rule applies to an explicit `ROLLBACK` and to connection or transaction disposal. If any eager-state undo fails, `ROLLBACK` returns an error instead of false success, clears the session transaction, records `ROLLBACK_CLEANUP_FAILED`, and retains only the direct health diagnostic plus rollback, migration-abort, and query-cancel controls until close + reopen. Do not retry the transaction in that process image. `COMMIT` also clears the session after terminal cleanup even when disposal or distributed-outcome publication fails. A primary commit refusal plus cleanup failure is returned as `ROLLBACK_CLEANUP_FAILED` with both causes preserved; the session never remains attached to the completed transaction.

Plugin/executor recursion cannot own a second terminal transition. A nested `COMMIT`, `ROLLBACK`, `COMMIT MIGRATION`, or `ABORT MIGRATION` attempted while the parent still owns an operation returns `TRANSACTION_COMPLETION_INSIDE_OPERATION`; it does **not** clear the session, publish a distributed outcome, or end the migration. If a plugin ignores an ordinary nested DML error, the parent still returns `TX_ABORTED`. Inside a migration, rollback waits only for the parent operation to exit and then destroys the whole migration; the ignored child error cannot become success.

For a distributed transaction, "the statement was submitted before the creator's `COMMIT` returned" does not guarantee it runs first. Statements from all bound connections serialize on one transaction gate. A joiner queued behind the creator's terminal operation receives the accurate committed / rolled-back / creator-disconnected outcome without executing. Treat that result as terminal; do not resubmit the statement against the old handle. The durable WAL marker is the commit boundary: even when a later page-apply/flush failure makes the creator's `COMMIT` return an error and quarantines the engine, joiners receive the committed outcome because reopen recovery will finish the work. Retrying it as rolled back can double-apply the transaction. The recovery-owned commit also remains a hole in the process's snapshot watermark; a later disjoint commit cannot make that incomplete page state a valid read cut before reopen. Same-flow trigger/plugin/query recursion may reuse the already-owned distributed transaction gate, but delayed work cannot: ownership is exact and flow-local, retired before release, and stale captured work queues normally and re-checks the terminal outcome.

Distributed handles are checked non-empty identities. If `DISTRIBUTED_TRANSACTION_IDENTITY_CONFLICT` is returned, no registry entry was published or replaced; treat it as an engine identity fault, not as a handle that can be retried with `JOIN TRANSACTION`.

<a id="pitfall-isolation-serializable-can-abort-commit--always-retry-the-whole-transaction"></a>

### `ISOLATION SERIALIZABLE` can abort `COMMIT` — always retry the whole transaction

`BEGIN TRANSACTION` defaults to snapshot isolation, which does NOT prevent write skew (two transactions each read a shared invariant and write disjoint parts — both commit, the invariant breaks). Referential integrity is not exposed to it: a reference and a concurrent removal of its target cannot both commit. `BEGIN TRANSACTION ISOLATION SERIALIZABLE` fixes that by validating its read-set at commit, but the price is that `COMMIT` can fail with a transient `TX_CONFLICT_SERIALIZABLE` (error class `Conflict`). Callers MUST be idempotent across retries and re-run the whole transaction (re-read, re-decide, re-write) — exactly the same retry contract as `TX_CONFLICT_FWW`. Two extra traps: (1) validation is **collection-grain**, so a serializable transaction aborts if *any* row in a collection it read was written — even a row it didn't touch; a serializable read of a hot collection can be aborted by unrelated concurrent writes, so keep serializable transactions short. (2) Serializable is opt-in only — a bare `BEGIN TRANSACTION` (or any implicit single-statement write) is snapshot and gives no write-skew protection. (3) When the host enables `SnapshotIsolation`, delaying the first read does not adopt a concurrent writer's newer state: the read still uses the transaction-start snapshot and `COMMIT` conflicts if that collection changed after the snapshot. Do not treat a late read as a way to refresh a serializable transaction.

<a id="pitfall-wal-transaction-sequence-corruption-is-not-retryable"></a>

### WAL transaction-sequence corruption is not retryable

Recovery that finds an impossible transaction structure in a checksum-valid WAL fails the open with a message naming the defect ([WAL transaction structure](errors.md#wal-transaction-structure)). Do not retry statements or resume the engine in place. Preserve the files for diagnosis.

`TRANSACTION_LIFECYCLE_OBSERVER_FAILED` is a health-only structural signal raised when a rollback-completed or settled observer fails after the transaction outcome is authoritative; the engine quarantines because lifecycle ownership may remain retained.

The `CHECK_REGISTRY_*_OWNER_*` and `SCHEMA_BARRIER_PUBLICATION_ACQUISITION_MISSING` catalog entries are `ContractViolation` signals for missing durable collection or schema-generation ownership during DDL publication. They are engine defects, not invalid CHECK expressions or retryable catalog conflicts.
