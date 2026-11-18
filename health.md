# Engine health

> **In this section** — the engine-derived health surface (`Healthy` / `Degraded` / `Quarantined`), the `[Mnemic].[Health]` virtual collection, and the admission gate that normally refuses ordinary MQL while the engine is `Quarantined`, plus the narrow read-only exception for a lone `WAL_FLUSH_FAILED`, and where the failure that raised a fault is kept.

The engine carries two orthogonal state machines that the MQL executor consults before dispatching a statement:

- **`EngineRoleState`** — operator intent set by plugins (`Writable` / `ReadOnly` / `Draining`). Documented under the plugin contract.
- **`EngineHealthState`** — engine-derived state (`Healthy` / `Degraded` / `Quarantined`). Flipped by detectors inside the engine (for example, a page-checksum mismatch is attributed to its catalog, page-store, or engine-wide domain according to the page's known ownership); cleared by an operator after they have inspected the cause.

Role admission has these statement-level semantics:

- `Writable` admits work subject to authorization, health, and the ordinary storage gates.
- `ReadOnly` refuses statements that open or continue mutation with `User / ENGINE_READ_ONLY`, the message naming the role's reason. Completion and rollback remain available, so work staged before the transition can settle and release its resources.
- `Draining` additionally lets a mutation continue only when it participates in an explicit transaction or migration that is already open. It refuses new scopes and independently committed work even when some unrelated scope is present, with the retryable `Transient / ENGINE_DRAINING`.

Reads remain role-admissible, including `BACKUP` and `SCRIPT MIGRATION`, because they do not publish engine state. `EXPLAIN ANALYZE` inherits the effect of the statement it executes. Database-management MQL and remote-connect auto-creation consult the master engine's role; ordinary statements consult the connected engine. The separate health gate may still refuse any of these operations when a fault prevents trustworthy execution.

An index whose generation recovery could not redo from the recovery log reports `INDEX_REDO_DIVERGED` as `Degraded` on its `Index{…}` domain when the database opens: the index is rebuilt from its collection's rows (a secondary answers by scans until its online rebuild publishes), so statements stay admitted, and the row remains until `ALTER ENGINE RESUME` clears it or the database reopens without the divergence.

An index posting with a slot outside the physical page format reports `INDEX_POINTER_INVALID` even when bitmap preparation encounters it before a heap read. The fault identifies the supplying collection and index and places the engine in `Quarantined` state in both buffered and streaming queries.

A commit whose typed index-page operations do not reproduce, over the live leaf, the image the commit logs files `TYPED_INDEX_RECORD_DIVERGED` on the `Wal` domain as `Degraded`, never `Quarantined`: that leaf's change is logged as its full image, typed index-page records stay off until the database reopens, and the engine runs a checkpoint so restart recovery no longer depends on earlier typed records. Statements stay admitted. `ALTER ENGINE RESUME 'Wal'` clears the row; it does not turn typed records back on.

Detectors reached by background or teardown work are equivalent to statement-time detectors. Auto-analyze, fuzzy checkpoint, incremental defrag, cold-page scrub, expiry-index sweeps, read-epoch tasks, deferred-WAL durability timers, temporal cascade/retry dispatch, change-tracker dependency scans, authorization-audit writer/pruner work, notification-dispatch and server shutdown, plugin service/shutdown callbacks, parallel-query owner joins, and statement/migration cancellation callbacks traverse aggregate/inner exception wrappers and publish every structural fault immediately. A loop may retry an ordinary operational failure, but corruption appears in `[Mnemic].[Health]` before that retry or task termination; it is not postponed until shutdown.

A background maintenance worker that ends on a failure while its engine serves files `BACKGROUND_WORKER_STOPPED` on the `EngineWide` domain as `Degraded`, never `Quarantined`, at the moment the worker ends, and the engine's log gets one error row under the same code that names the worker and carries the failure. A worker's loop retries an ordinary failure of one pass, so a worker ends this way on a failure it cannot absorb, such as memory exhaustion. Statements stay admitted, but the engine runs without that worker's maintenance until the database reopens. The standalone server stops on a worker's process-terminal failure and exits with code `3` ([server log](https://mnemic.io/docs/configuration/#standalone-server-log)), so there the row of such a failure is seen in the log, not in a later health query. `ALTER ENGINE RESUME 'EngineWide'` clears the row as an acknowledgement; it starts nothing. A structural fault in the worker's failure is reported under its own code and domain.

Expiry candidate processing and cleanup follow the shared failure arbitration: independent critical causes are retained, and a process-terminal cleanup failure displaces an ordinary selection or deletion error. Candidate reservations and spill readers are released after cancellation, memory exhaustion, and modeled process death alike; a terminal release failure is not retried.

Migration-owned catalog bootstrap likewise reclaims an unpublished initial page after a creation failure, including memory exhaustion and modeled process death. If reclamation also fails, both causes remain in the propagated failure graph. Successful reclamation retries preserve corruption and cancellation evidence. A later terminal cleanup failure retains earlier causes, including proofs from pages whose release already succeeded.

Within one buffer-pool write-behind pass, independently failed write runs retain their causes in attempt order for the durability barrier's failure graph. Corruption observed before memory exhaustion remains attributable to its original domain; that terminal signal stops subsequent writes without erasing the proof. An `ObjectDisposedException` wrapper in timer work does not make nested corruption or cancellation a benign shutdown race. Live evidence remains available at the durability barrier; memory exhaustion and modeled process death propagate. The first retained background write failure stops subsequent automatic cleaner work, including callbacks racing its publication. Clean close still retries dirty pages and reports the original barrier failure even if that retry succeeds. A terminal clean-close diagnostic failure retains that earlier durability graph, including every original structural proof, alongside the diagnostic cause.

Parallel query teardown joins every producer, worker, and merge pump before it releases spill/admission state. Early row-stream abandonment therefore cannot suppress corruption detected during teardown. Multiple independent detector failures produce their exact health rows, while cancellation caused only by an already-reported owner failure is not misclassified as a new fault. Throwing active-query registration/retirement, statement, migration-drain, or distributed-joiner cancellation callbacks likewise cannot skip terminal cleanup or hide a nested detector. The same rule applies to maintenance, notification, audit, and server shutdown: callback failure is retained, mandatory owners are joined, and clean shutdown is not certified over a nested corruption report.

Retryable resource cleanup retains every critical owner graph when another release exhausts memory or models process death. That failed owner is not retried, while independent owners and eligible retries still finish and proofs already observed, including from owners awaiting retry, are preserved.

Allocator-map publication likewise preserves corruption and cancellation even when a bounded write retry succeeds. Publication memory exhaustion or modeled process death leaves allocation unavailable and still returns its scratch; a terminal retry retains both publication failures.

A failed page-allocation compensation reports both the initiating allocation failure and the compensation failure. Earlier corruption and cancellation remain observable even when compensation exhausts memory or models process death, and every scratch rental is still returned.

Ordinary read failure or cancellation during free-page verification leaves the candidate available for revalidation by a later allocation. Corruption and process-terminal failures keep it excluded.

Read-scope cleanup preserves independent latch and provider-pin failures. A latch-release failure, including memory exhaustion or modeled process death, still attempts pin release. Earlier corruption and cancellation evidence remain in the reported failure graph. Rejecting a page also retains its verification proof when cleanup of the rejected read scope fails.

Bulk-index build cleanup preserves corruption, cancellation, and memory failures from unpublished-page reclamation even when a bounded free retry succeeds. Successful reclamation releases the page without erasing the earlier evidence. Modeled death during a free, retry, or cleanup diagnostic likewise retains the initiating build failure and earlier reclamation causes; every unpublished page still receives its free attempt and retry, and scratch is returned.

An async plugin hook may continue after its configured timeout because the hook contract cannot cancel it. The timeout is reported immediately; if the detached task later faults, `[Mnemic].[Health]` receives a separate `HOOK_<NAME>_LATE_FAILED` row. Structural proofs in that late failure remain quarantining and are never reduced to an observed-only task exception.

A statement is ordinarily admissible only while the role allows it and no fault domain is `Quarantined`. While any domain is `Quarantined`, every ordinary read and mutation is normally refused with `StructuralCorruption / ENGINE_QUARANTINED`, whose message names the stable code of each fault that refuses the statement ([refusal message shape](#refusal-message-shape)): a structural proof means the process cannot certify which user result paths remain independent of the fault. Only a direct `FROM [Mnemic].[Health]` query and the bounded recovery controls `ROLLBACK`, `ABORT MIGRATION`, `CANCEL QUERY`, and `ALTER ENGINE RESUME`, plus `RESTORE FROM '<backup>' VERIFY`, remain admissible. The server also accepts `UNSUBSCRIBE` under every quarantine, since it only removes one of the connection's own subscriptions.

A lone `WAL_FLUSH_FAILED` is narrower. The WAL writer is sealed and every mutation remains refused until reopen, but the live process retains one internally coherent read image. Queries (including set operations and `WITH`), `EXPLAIN`, `DESCRIBE`, [`COMPLETE` and `VALIDATE`](editor-assistance.md), `LIST DATABASE`, `LIST PLUGINS`, `SCRIPT MIGRATION`, `VERIFY DATABASE`, `SHOW PARAMETER`, and `LIST QUERIES` remain available. `BACKUP`, subscription registration (`SUBSCRIBE`, plugin channels included), and every state-publishing statement remain refused. If any other quarantining fault is present, the ordinary global refusal applies. Each row's `ReadsAdmitted` column reports this rule for its fault, and these reads run exactly while every row's is `TRUE`. In particular, an in-flight append/fsync failure pairs `WAL_FLUSH_FAILED` with `WAL_WRITE_FAILED`, and an incomplete post-marker page apply pairs it with `COMMIT_APPLY_FAILED_AFTER_MARKER`; both combinations close ordinary reads until restart recovery resolves the transaction outcome. Verified restore reads a separate backup, stages a validated replacement, and applies it only on reopen; it never serves or copies quarantined live user state. Recovery-required codes refuse in-place resume: `COMMIT_APPLY_FAILED_AFTER_MARKER`, `IMPLICIT_ROLLBACK_CLEANUP_FAILED`, `ROLLBACK_CLEANUP_FAILED`, `CATALOG_PUBLICATION_FAILED_AFTER_COMMIT`, `REBUILD_INCOMPLETE`, `REBUILD_REPEATED_FAILURE`, `WAL_WRITE_FAILED`, `WAL_FLUSH_FAILED`, and `PLUGIN_RECOVERY_REQUIRED`.

## A fault belongs to the engine that owns it

Every database has its own health state, and so does the master engine that holds the server's catalog and user directory. A fault is recorded against the engine whose files or statements raised it:

- A structural fault in one database quarantines that database alone. Every other database, `CREATE DATABASE`, `DROP DATABASE` and `REOPEN DATABASE` (including the faulted database), `CREATE USER` and `/health/ready` keep working.
- A database whose files cannot be opened has no engine to quarantine. Its `Connect` and `REOPEN DATABASE` return the classified `StructuralCorruption` pair on every attempt (see [connecting](connecting.md#a-database-that-cannot-be-opened)); nothing is recorded in the master engine.
- A failed statement always answers its client with the classified pair, including a failure that surfaces while the session cleans up after text that did not parse.
- The master engine's faults are visible on a server-level connection: `FROM [Mnemic].[Health]` and `ALTER ENGINE RESUME` run against the master engine, for `sa` only. While the master engine is quarantined `/health/ready` is `503` and the master-owned server statements (`CREATE DATABASE`, `CREATE USER`) are refused.

## `[Mnemic].[Health]`

Virtual system collection — one row per faulted domain. A healthy engine returns zero rows.

```mql
FROM [Mnemic].[Health] SELECT [Domain], [State], [Code], [Message];
FROM [Mnemic].[Health] WHERE [State] = 'Quarantined'
                       SELECT [Domain], [Code], [Message];
FROM [Mnemic].[Health] ORDER BY [LastObservedAtUtc] DESC
                       SELECT [Domain], [Code], [LastObservedAtUtc];
FROM [Mnemic].[Health] WHERE [Resumable] = FALSE
                       SELECT [Domain], [Code];
FROM [Mnemic].[Health] WHERE [ReadsAdmitted] = FALSE
                       SELECT [Domain], [Code];
```

Columns:

| Column | Type | Notes |
|---|---|---|
| `Domain` | `STRING` | Canonical fault-domain string. Shapes: `EngineWide`, `Catalog`, `Wal`, `PageStore{<CollectionName>}` with the bracketed collection name (`PageStore{[Shop].[Orders]}`), `Index{<CollectionName>/<IndexName>}` with the index's `[Mnemic].[Indexes]` name (`Index{[Shop].[Orders]/secondary_Status}`; a compound index has a `compound1_…` name), `Plugin{<plugin UUID>}`, `Transaction{<Guid>}`. |
| `State` | `STRING` | One of `Healthy` (never observed in this view), `Degraded`, `Quarantined`. |
| `Code` | `STRING` | Stable token from the detector — e.g. `PAGE_CHECKSUM`. Same string that appears in logs and in `IClassifiedException.Code`. |
| `Resumable` | `BOOL` | Whether `ALTER ENGINE RESUME` can clear the fault. `FALSE` for a [recovery-required code](#clearing-quarantine--alter-engine-resume), which only the recovery of reopening the engine clears: `REOPEN DATABASE` for a user database on a server, a server restart for the server's own engine. The engine reports it by the same rule it refuses `ALTER ENGINE RESUME` with, so a client reads it here instead of keeping its own list of codes. |
| `ReadsAdmitted` | `BOOL` | Whether ordinary reads stay admitted while the fault stands. `TRUE` for a `Degraded` fault and for a quarantined `WAL_FLUSH_FAILED`, which keeps the bounded read shapes listed above on its one coherent image; `FALSE` for any other quarantined fault. The engine admits an ordinary read exactly while every row's is `TRUE`, and reports it by the same rule its admission follows, so a client reads it here instead of keying on a code. |
| `Message` | `STRING` | Human-readable diagnostic from the detector (page id, byte offset, etc.). Not a parsing target. |
| `FirstObservedAtUtc` | `DATETIME` | UTC of the first fault filed against this domain since open. |
| `LastObservedAtUtc` | `DATETIME` | UTC of the most recent fault. |

Health state lives in memory. Reopen creates a fresh health overlay after the engine's durable recovery and reconstruction steps have run. For an ordinary resumable fault, reopening alone does not repair a persistent cause: detectors report it again on open validation or the next touch. Recovery-required faults are different because that reopen path is what establishes a new trustworthy process image.

## Where a fault began

A fault keeps the failure its detector reported, with its type, message, stack trace and inner exceptions, beside its code and message. That failure stays inside the engine's process: `[Mnemic].[Health]` and every wire answer carry the code and message only, so no engine stack reaches a remote client. Inside the process it is never dropped:

- The standalone server writes it on the log row of the health transition ([server log](https://mnemic.io/docs/configuration/#standalone-server-log)).
- An embedded host reads it from the fault, as the `Cause` of each entry of `IEngineHealthState.Faults`, and from the statement that met it: an in-process error result carries it as its `Cause`, and the typed client's `MqlException` raises it as its inner exception. A document the validator could not read reports `DOCUMENT_PAYLOAD_CORRUPT`, and the inner exception of its cause is the failure the validator met.
- Every fault report, a refresh of a standing fault included, is an event of the engine's `Mnemic-Engine` event source carrying the fault's `domain`, `code` and `message` and the failure's `failureType` and full text as `failure`: `StructuralCorruptionReported` at the `Critical` level, `DegradationReported` at `Warning`. The source needs no configuration, so a host that attached no logger still captures it, with `dotnet-trace collect --providers Mnemic-Engine -p <pid>`, PerfView, or an in-process `EventListener` enabled for the source.

## Refusal message shape

When the engine is `Quarantined`, refused MQL returns an error shaped like:

```text
[Insert] Engine is quarantined; mutating statements refused — PAGE_CHECKSUM: Page 42 failed checksum verification: stored=0x..., computed=0x.... The page is corrupted on disk; the database file is not safe to open.
[Query] Engine is quarantined; only direct [Mnemic].[Health] queries and recovery controls are allowed — PAGE_CHECKSUM: Page 7 failed checksum verification.
[Backup] Engine is quarantined and requires recovery on reopen; only ordinary reads, direct [Mnemic].[Health] queries and recovery controls are allowed — WAL_FLUSH_FAILED: The data-file flush after a commit was incomplete; writes are quarantined until reopen recovery succeeds.
```

The leading `[<StatementType>]` matches the parser's AST node name without the `Node` suffix. The message then says what the engine still admits: mutations are refused; while a fault closes reads only direct `[Mnemic].[Health]` queries and the recovery controls run; under the lone `WAL_FLUSH_FAILED` exception above the bounded ordinary reads run too. It names, after the dash, every fault that refuses this statement and no other, each as its code and its own message, oldest first: a read names the rows whose `ReadsAdmitted` is `FALSE`, so a read refused beside a `WAL_FLUSH_FAILED` names the fault that closes reads rather than the flush, and any other statement names every quarantined row; a `Degraded` row is never named. `requires recovery on reopen` appears when a named fault is a [recovery-required code](#clearing-quarantine--alter-engine-resume), so only reopening clears the refusal; otherwise resuming the named domains does. `BACKUP` is not admitted from any quarantined process image because copying live state would create an artifact whose durable cut cannot be certified.

## Clearing quarantine — `ALTER ENGINE RESUME`

```mql
ALTER ENGINE RESUME '<domain-spec>';
```

Clears the admission gate for one fault domain. The argument is the canonical `Domain` string from `[Mnemic].[Health]` — same shape that appears in logs and in `IClassifiedException.Code`-correlated messages. Examples:

```mql
ALTER ENGINE RESUME 'EngineWide';
ALTER ENGINE RESUME 'Catalog';
ALTER ENGINE RESUME 'Wal';
ALTER ENGINE RESUME 'PageStore{[Shop].[Orders]}';
ALTER ENGINE RESUME 'Index{[Shop].[Orders]/secondary_Status}';
ALTER ENGINE RESUME 'Plugin{3f2504e0-4f89-11d3-9a0c-0305e82c3301}';
ALTER ENGINE RESUME 'Transaction{1f9c8c18-7e44-4b5d-9f2c-9c8a3a3e2a01}';
```

Errors:

- `User / INVALID_VALUE` `invalid fault domain '...'` — the spec doesn't parse against the canonical grammar; copy the string verbatim from `[Mnemic].[Health]`.
- `User / OBJECT_NOT_IN_PREREQUISITE_STATE` `domain '...' is not faulted; nothing to clear` — no fault is currently filed for that domain.
- `User / OBJECT_NOT_IN_PREREQUISITE_STATE` `domain '...' requires recovery on reopen` — the domain carries one of the recovery-required codes below.
- `Transient / FAULT_STATE_CHANGED` `fault state for domain '...' changed concurrently` — a concurrent detector replaced the fault, or another resume cleared it, after the statement inspected it; nothing was cleared. Inspect `[Mnemic].[Health]` and retry.

Authorization: `ServerAdmin` (same tier as `CREATE DATABASE` / `DROP DATABASE`). The statement is admissible while the engine is `Quarantined` and is the in-process way out for a resumable fault.

Recovery-required codes are exceptions:

- `COMMIT_APPLY_FAILED_AFTER_MARKER` — an explicit or implicit commit reached its WAL marker but did not finish page apply in the current process.
- `IMPLICIT_ROLLBACK_CLEANUP_FAILED` — an auto-commit write failed and one or more eager index/count/catalog cleanup steps could not be completed, so the live process image may disagree with durable storage.
- `ROLLBACK_CLEANUP_FAILED` — an explicit rollback or dispose-time implicit rollback could not complete every eager-state undo callback. The session is cleared and the failure is returned, but the live process image is not trusted until reopen.
- `CATALOG_PUBLICATION_FAILED_AFTER_COMMIT` — durable catalog state committed, but the process could not publish a mutually consistent set of live metadata snapshots.
- `REBUILD_INCOMPLETE` — destructive index replacement began but did not produce a complete usable generation.
- `REBUILD_REPEATED_FAILURE` — repeated rebuild attempts failed to certify a usable index generation in this process.
- `WAL_WRITE_FAILED` — a write or durability operation left the WAL tail indeterminate, so the writer is sealed against further use.
- `WAL_FLUSH_FAILED` — a committed WAL group or its data-file apply could not prove completion, so retained WAL must be replayed on reopen.
- `PLUGIN_RECOVERY_REQUIRED` — a plugin's derived state, such as a vector index, can no longer prove that it covers every committed source mutation; the clean-shutdown marker is withheld so the next open reconstructs that state from its authoritative source.

`ALTER ENGINE RESUME` is refused for a domain carrying any of these codes, and the health row remains, its `Resumable` column `FALSE`. The direct health diagnostic plus rollback, migration abort, and query cancellation controls remain available. A lone `WAL_FLUSH_FAILED` also retains the bounded read-only statements listed above; it does not make the code resumable. Reopen the database so recovery and reconstruction can re-enable mutations: on a server, `REOPEN DATABASE [name]` from a server-level connection reopens a user database, terminating the sessions connected to it ([databases](databases.md#reopening-a-database)). The master engine is no database `REOPEN DATABASE` can name (`_master` is refused with `RESERVED_NAME`), so a recovery-required fault of the master engine, which a server-level connection shows, needs a server restart.

For resumable faults, `ALTER ENGINE RESUME` clears only the admission gate. The underlying corruption is unaffected: restore from a known-good backup before expecting writes to succeed against the affected domain. Detectors re-fire on the next touch if the corruption persists.

Plugins read the same faults through `IPluginContext.HealthState` (`Current` and `Faults`) for diagnostics; clearing a fault is engine-owned, through `ALTER ENGINE RESUME` and the engine's own code-scoped recoveries.

## Tuning admission watermarks — `ALTER ENGINE SET ADMISSION`

```mql
ALTER ENGINE SET ADMISSION (Key = Value [, Key = Value ...])
```

Re-tunes the engine-wide admission policy at runtime without restart. Only available when the engine was opened with a `BoundedAdmissionController` — the default no-op controller refuses with a typed error so silent no-ops can't hide.

The statement persists and publishes its policy outside any transaction, so a ROLLBACK or ABORT could not undo it: inside an explicit transaction or a migration it is refused with `INVALID_TRANSACTION_STATE` (`Cannot ALTER ENGINE SET ADMISSION while a transaction is active. COMMIT or ROLLBACK first.`) before it changes anything, as `ALTER ENGINE SET SLOWQUERY` is, and like any failed statement the refusal ends the migration or makes the transaction abort-only. Run it on its own.

Accepted keys (case-insensitive, at least one required, each at most once):

| Key | Default | Effect |
|---|---|---|
| `WarnHighWaterPct` | 0.70 | Crossing this threshold publishes a `Warn` pressure event. |
| `SpillHighWaterPct` | 0.85 | Crossing this threshold publishes a `Spill` pressure event. Operators still decide when to spill from their own local memory cap. |
| `RejectHighWaterPct` | 0.95 | A fresh reservation above this threshold fails with `RESOURCE_PRESSURE`. |
| `MaxBytes` | controller-construction value | The policy budget used to derive the three byte thresholds. It is not a hard process-RSS cap: only explicit admission tickets are counted, and in-flight spillable growth remains admitted and accounted. |

Percentage values accept both fractional (`0.70`) and percentage (`70`) form; values above 1 are scaled by 100 so an operator-typed `WarnHighWaterPct = 65` resolves to 0.65. `MaxBytes` is a positive int64 (no auto-scaling); zero or a negative value is `INVALID_VALUE`.

Spilled GROUP BY processing also accounts its per-partition rebuild state and terminal sort buffers. Recursive warm groups remain charged while child partitions run; a completed sort spill releases its resident key and row reservation. These tickets retain the statement and collection fault domain.

The post-update percentages must satisfy `warn < spill < reject` and every value must lie strictly inside `(0, 1)`. `MaxBytes` must be positive, and conversion to byte thresholds must preserve `0 < warn < spill < reject < max`; a budget that collapses two thresholds onto the same byte is invalid. Invalid combinations return a typed error and leave the controller unchanged; updates are published atomically so concurrent acquire / adjust callers never see a torn (mid-publish) tuple. Concurrent partial ALTER statements rebase omitted fields on the newest generation, so neither accepted transition can silently erase the other. `ReadOnly` and new mutations under `Draining` reject this statement because it changes the persisted engine configuration.

**Persistence across restart.** The engine validates the complete candidate, persists it to the `[Mnemic.Configuration].[Admission]` system collection (PK=1, singleton row), and only then publishes it to the live controller. A failed or cancelled persistence attempt therefore leaves the live policy unchanged. Waiting behind another admission-policy update observes cancellation and the host's lock-acquisition deadline; expiry returns `Transient / LOCK_TIMEOUT` before the waiting statement changes configuration. Subsequent openings with a `BoundedAdmissionController` automatically reapply the persisted snapshot as one validated generation before request dispatch resumes — the operator's choice survives without an external bootstrap script. A fresh database with no prior ALTER keeps the controller's construction-time defaults. If the singleton row is present but its fixed key/layout, required fields, or complete policy tuple is invalid, engine open fails with `DOCUMENT_PAYLOAD_CORRUPT`; corrupt persisted policy never falls back to those defaults or partially updates the controller.

Examples:

```mql
ALTER ENGINE SET ADMISSION (SpillHighWaterPct = 80);
ALTER ENGINE SET ADMISSION (WarnHighWaterPct = 0.6, SpillHighWaterPct = 0.8, RejectHighWaterPct = 0.92);
ALTER ENGINE SET ADMISSION (MaxBytes = 134217728);                                  -- 128 MiB
ALTER ENGINE SET ADMISSION (MaxBytes = 268435456, WarnHighWaterPct = 60);           -- shrink + re-tune
```

Errors:

- `not running with a BoundedAdmissionController` — an embedded host supplied the no-op or a custom non-bounded controller; no watermarks to tune.
- `Watermarks must be strictly ordered: warn < spill < reject` — the post-update triple violates the invariant.
- `Watermarks must lie strictly inside (0, 1)` — a value pins to 0 or 1.
- `MaxBytes must be positive` — `MaxBytes` is zero or negative.
- `MaxBytes and watermarks must produce distinct positive byte thresholds` — percentage-to-byte conversion would collapse or eliminate a threshold.
- `persisting to [Mnemic.Configuration].[Admission] failed before the live configuration was published: …` — the catalog upsert failed and the prior live policy remains active.

EXPLAIN ANALYZE's `AdmissionDecisions` column surfaces both the consumption shape and one coherent policy snapshot: `reserved=X/max=Y warn=W%/spill=S%/reject=R% crossings=Z`. `crossings` is the engine-wide change during the statement's wall-clock window; concurrent sibling statements can contribute to it.

Authorization: `ServerAdmin` (same tier as `CREATE DATABASE` / `DROP DATABASE`) — engine-wide configuration is server-scoped.

## Metrics

`[Mnemic].[Health]` is a point-in-time view; for continuous alerting, the engine also publishes health transitions on the `Mnemic.Engine` OpenTelemetry meter:

| Instrument | Type | Tags | Notes |
|---|---|---|---|
| `mnemic.health.transitions` | `Counter<long>` | `domain`, `state` | One increment per real transition: a domain going `Healthy → Degraded`/`Quarantined` (detector report) or back to `Healthy` (`ALTER ENGINE RESUME`). |

`domain` is the bounded fault-domain *kind* — `engine_wide`, `page_store`, `catalog`, `wal`, `index`, `plugin`, `transaction` (never the full domain string, so chart cardinality stays bounded). `state` is `degraded`, `quarantined`, or `healthy`. A second fault at the same severity does not re-emit. Alert on `state="quarantined"` from Prometheus/Grafana to page on corruption without polling the view on a timer — the same meter-vs-collection pairing `mnemic.lock.timeouts` gives `[Mnemic].[LockTimeoutsBySite]`.

## Detailed syntax and execution


<a id="syntax-alter-engine-resume"></a>

### ALTER ENGINE RESUME

The executor also applies an engine-local role before dispatch. `Writable` admits work subject to the ordinary gates. `ReadOnly` refuses statements that open or continue mutation but leaves completion and rollback available. `Draining` additionally lets scoped mutation continue only inside an already open transaction or migration; it refuses new scopes and independently committed work. Reads—including `BACKUP` and `SCRIPT MIGRATION`—remain role-admissible, while `EXPLAIN ANALYZE` inherits the effect of its inner statement. Database-management MQL and remote-connect auto-creation consult the master engine's role; ordinary statements consult the connected engine. Health quarantine is a separate gate and may still refuse role-admissible work. A lone `WAL_FLUSH_FAILED` retains bounded queries, metadata/integrity reads, and `VERIFY DATABASE`, but refuses every mutation, backup, and subscription registration until reopen; any other quarantining fault closes those reads. Indeterminate in-flight append/fsync and incomplete post-marker page apply also publish `WAL_WRITE_FAILED` or `COMMIT_APPLY_FAILED_AFTER_MARKER`, respectively, so those transaction states do not receive the read exception.

```sql
ALTER ENGINE RESUME '<domain-spec>';
```

Clears a single fault domain's quarantine in `[Mnemic].[Health]`. The argument is the canonical `Domain` string copied verbatim from the health view — `'EngineWide'`, `'Catalog'`, `'Wal'`, `'PageStore{[Shop].[Orders]}'`, `'Index{[Shop].[Orders]/secondary_Status}'`, `'Plugin{<plugin UUID>}'`, `'Transaction{<guid>}'`; a `Plugin{}` domain names the plugin by the UUID `LIST PLUGINS` lists, and any other spelling is `INVALID_VALUE`. Admissible while the engine is `Quarantined` — it is the only way out short of restart.

`COMMIT_APPLY_FAILED_AFTER_MARKER`, `IMPLICIT_ROLLBACK_CLEANUP_FAILED`, `ROLLBACK_CLEANUP_FAILED`, `CATALOG_PUBLICATION_FAILED_AFTER_COMMIT`, `REBUILD_INCOMPLETE`, `REBUILD_REPEATED_FAILURE`, `WAL_WRITE_FAILED`, `WAL_FLUSH_FAILED`, and `PLUGIN_RECOVERY_REQUIRED` are non-resumable. They prove that page apply, eager-state rollback, live metadata publication, index-generation reconstruction, WAL durability, or a plugin's derived-state coverage did not leave a resumable mutation path. The direct health diagnostic plus rollback, migration-abort, and query-cancel controls remain available. A lone `WAL_FLUSH_FAILED` additionally retains query/set/`WITH`, `EXPLAIN`, `DESCRIBE`, `COMPLETE`, `VALIDATE`, `LIST DATABASE`, `LIST PLUGINS`, `SCRIPT MIGRATION`, `VERIFY DATABASE`, `SHOW PARAMETER`, and `LIST QUERIES` reads; it still requires reopen before mutation. `RESTORE FROM '<backup>' VERIFY` is also admitted: it validates and stages a separate replacement without reading quarantined live user data, then requires close and reopen before the replacement is applied.

For resumable faults, `ALTER ENGINE RESUME` clears only the admission gate; the underlying corruption is unaffected. Restore from a known-good backup before expecting writes to succeed against the affected domain. See [health.md](health.md).

<a id="syntax-alter-engine-set-slowquery"></a>

### ALTER ENGINE SET SLOWQUERY

Configure the current database engine's default slow-query threshold and shared history capacity without restarting. Requires `ServerAdmin` on the server and no active transaction or migration in the calling session.

```sql
ALTER ENGINE SET SLOWQUERY (Threshold = '100ms', RingCapacity = 2048);
ALTER ENGINE SET SLOWQUERY (Threshold = 2);        -- integer seconds
ALTER ENGINE SET SLOWQUERY (Threshold = NONE);     -- disable default selection
ALTER ENGINE SET SLOWQUERY (RingCapacity = 512);   -- retain newest events
```

Keys are case-insensitive; supply at least one, without duplicates. `Threshold` accepts a non-negative duration (`'0ms'`, `'500ms'`, `'30s'`, `'2m'`), integer seconds, or `NONE`. Zero selects every statement. `RingCapacity` is a positive `INT32`. Omitted keys retain their current engine value, including when another administrator updates them concurrently.

The complete policy is persisted before publication and overrides host `EngineOptions` on subsequent opens. A failed or cancelled update before durable persistence leaves the live policy unchanged. Success returns `Ok` with the applied threshold and capacity. Ordinary DML cannot edit the internal settings. Waiting behind another slow-query-policy update observes cancellation and the host's lock-acquisition deadline; expiry returns `Transient / LOCK_TIMEOUT` before the waiting statement changes configuration.

Existing and new sessions without an override use the live engine default at statement start. Explicit `SET slow_query_threshold` overrides, including `NONE`, continue to win. `SET slow_query_threshold = DEFAULT` resumes inheritance. An in-flight statement keeps the selection threshold captured when it started.

Every successful global configuration enables shared history, including `Threshold = NONE` or a capacity-only change. Disabling default selection keeps retained events and allows session overrides to contribute new events. Resizing keeps the newest events, including observations arriving during the update. History events themselves remain process-lifetime data and are empty on reopen.

`SHOW slow_query_threshold` reports the current session's effective value and source. `[Mnemic].[Capabilities]` reports the engine default selection toggle as `SlowQueryLog` and allocated history capacity as `SlowQueryRingCapacity` (zero before history is enabled). See [system collections](system-collections.md).

<a id="syntax-alter-engine-set-admission"></a>

### ALTER ENGINE SET ADMISSION

```sql
ALTER ENGINE SET ADMISSION (Key = Value [, Key = Value ...])
```

Re-tunes the engine-wide admission gate's MaxBytes + high-water percentages at runtime without restart. Accepted keys (case-insensitive, at least one required): `WarnHighWaterPct`, `SpillHighWaterPct`, `RejectHighWaterPct` (percent / fractional, values above 1 are scaled by 100), and `MaxBytes` (positive int64, the engine-wide byte budget). The post-update percentages must satisfy `warn < spill < reject` strictly inside `(0, 1)`. `MaxBytes` must be positive, and their conversion to byte thresholds must preserve `0 < warn < spill < reject < max`; configurations that collapse thresholds onto the same byte are rejected. Invalid updates leave the prior policy untouched. Lowering `MaxBytes` does not revoke in-flight tickets; fresh reservations observe the lower reject threshold, while spillable in-flight growth remains admitted and exactly accounted. The complete tuple publishes atomically; concurrent partial updates rebase omitted fields on the newest generation instead of losing an accepted transition. Because this statement persists engine configuration, `ReadOnly` and new mutations while `Draining` reject it, and it runs only outside an explicit transaction or migration: inside one it is refused with `INVALID_TRANSACTION_STATE` before it changes anything, since ROLLBACK and ABORT could not undo it.

```sql
ALTER ENGINE SET ADMISSION (SpillHighWaterPct = 80);
ALTER ENGINE SET ADMISSION (WarnHighWaterPct = 0.6, SpillHighWaterPct = 0.8, RejectHighWaterPct = 0.92);
ALTER ENGINE SET ADMISSION (MaxBytes = 134217728);                            -- 128 MiB
ALTER ENGINE SET ADMISSION (MaxBytes = 268435456, WarnHighWaterPct = 60);     -- shrink + re-tune
```

The engine validates the complete candidate, upserts it into the `[Mnemic.Configuration].[Admission]` system collection, and only then publishes it live. Failure or cancellation before persistence completes leaves the live policy unchanged. Subsequent engine opens re-apply the persisted snapshot, so the operator's choice survives restart without an external bootstrap script. A present but structurally or semantically invalid singleton fails engine open with `DOCUMENT_PAYLOAD_CORRUPT`; it never falls back to construction defaults or partially publishes a policy tuple. EXPLAIN ANALYZE's `AdmissionDecisions` column carries both consumption and one coherent policy snapshot: `reserved=X/max=Y warn=W%/spill=S%/reject=R% crossings=Z`. The crossing delta is engine-wide for the statement's wall-clock window, so concurrent sibling statements can contribute.

Only available when the engine runs with a `BoundedAdmissionController`; the default no-op controller refuses with a typed error. Authorization: `ServerAdmin` (same tier as `CREATE / DROP DATABASE`). See [health.md](health.md).

## Usage pitfalls


<a id="pitfall-executable-metadata-initialization"></a>

### Executable metadata initialization

Before the first statement, MQL asynchronously validates persisted CHECK and computed expressions, views, materialized views, and triggers as one metadata generation. Cancellation and statement deadlines cover this work; a cancelled attempt has no statement effects and leaves initialization retryable. Corrupt metadata fails closed before execution. Embedded hosts can explicitly await `MqlExecutor.InitializeAsync` before accepting requests; constructing an executor alone does not validate its catalogs.
