# Maintenance

> **In this section** — `BACKUP`, WAL archiving and point-in-time recovery, verified `RESTORE`, online `VERIFY DATABASE`, `DEFRAGMENT`, `SHRINK`, `RECLAIM ORPHAN PAGES`, and `SET CREATEDBY`, engine slow-query policy, and session parameters.

## BACKUP

```sql
BACKUP TO '/backups/mydb.bak';
BACKUP TO 'C:\Backups\store-2024-06-01.mnemic';
```

A string literal escapes only `''`, so a Windows path is written with single backslashes.

Creates a point-in-time backup using before-image copy-on-write snapshots. Writes continue during backup. Result is a valid `.mnemic` file that can be opened directly. The destination is a single-quoted string literal — the parser does not accept `@parameter` paths or function calls here; format the path on the client before sending the statement.

For an encrypted database, the backup remains a self-contained encrypted database. Opening or restoring it requires the keys used by that backup. Restore authenticates the staged candidate before replacing live storage. See [data at rest](https://mnemic.io/docs/security/#data-at-rest).

`BACKUP` returns exactly one row:

| Column | Type | Meaning |
|--------|------|---------|
| `Destination` | `STRING` | Published backup path. |
| `SnapshotCommitSequenceNumber` | `INT64` | Commit-sequence number of the exact copied snapshot. |
| `IsPointInTimeRecoveryCapable` | `BOOL` | Whether durable archive coverage makes this backup eligible as a PITR base. |

Before the copy starts, the engine briefly freezes commit-sequence allocation publication, drains page applies, and waits for the physical pages to form one exact applied prefix. The copied header and snapshot metadata therefore name a single LSN `K`: every committed LSN through `K` is present and no later commit is present. The wait is bounded and observes statement cancellation. A published commit whose apply failed must be finished by reopen recovery first; `BACKUP` then fails with `COMMIT_APPLY_FAILED_AFTER_MARKER` rather than emitting bytes that no scalar replay LSN can describe. Once the prefix is pinned, writes continue normally and before-images exclude those later commits from the backup.

**Durability.** The backup file is fsynced (`FlushFileBuffers` / `fsync`) before `BACKUP` returns, so a process or OS crash immediately after the statement completes leaves the on-disk file durable through its last byte. The manifest is separately fsynced and renamed into place only after both files are complete. Matches PostgreSQL `pg_basebackup --sync` default.

**Local-file confinement.** The default file destination retains the resolved destination-directory handle for the session and creates, inspects, removes, and atomically publishes its staging entry relative to that handle. It obtains stable database and WAL identities from the engine's already-open handles and checks the actual destination entry again immediately before publication. Exchanging a symlink, junction, or writable ancestor after `BACKUP` starts therefore cannot redirect publication or cleanup onto another path. Identity or permission/I/O uncertainty fails the backup closed; there is no textual-path fallback. This contract is implemented on Windows, Linux, and macOS. A failed or cancelled backup removes its private staging entry through the held directory and leaves any prior completed snapshot untouched.

**Restore.** Opening the backup file *is* the restore — there is no separate statement. The file identifies itself as a restored snapshot (a backup-origin marker in its header), and its first open rebuilds every secondary index from the heap because derived structures inside a time-window copy are not trusted. The captured allocator state is retained without a scan; document counts are recounted from the restored data chains as the file opens, as on every open. Restore to a **fresh path**: the open refuses to run with a pre-existing `.wal` file next to the restored snapshot (`RESTORED_BACKUP_STALE_WAL`) — the snapshot carries the origin database's identity, and replaying the origin's stale log over it would silently roll the file into a state that never existed.

**Observability.** Three `Mnemic.Engine` meter instruments — `mnemic.backup.duration_us` (wall-clock), `mnemic.backup.pages_copied` (page count), and `mnemic.backup.before_images_saved` (per-page memory pressure signal during concurrent writes) — surface per-backup cost.

### PITR-capable backup

```sql
BACKUP TO '/backups/mydb.mnemic' REQUIRE PITR;
```

`REQUIRE PITR` captures an exact WAL cut and publishes neither the backup nor its manifest until durable, read-verified archive progress covers that cut. The replay anchor is pinned when the snapshot is established, so commits made during the copy remain available for subsequent replay even if archiving advances. It fails instead of silently producing an ordinary backup when archive coverage is unavailable; on a database opened without a WAL archive it is refused with `OBJECT_NOT_IN_PREREQUISITE_STATE`. Plain `BACKUP` retains its ordinary snapshot behavior and reports that capability through `IsPointInTimeRecoveryCapable`.

The backup output must not be inside, equal to, or alias the configured archive root (`INVALID_VALUE`). This includes symlinks and Windows junctions. Aliases in the ancestors of a link's target are resolved too; target ancestry deeper than 40 resolutions is rejected when isolation cannot be established.

## RESTORE … VERIFY

```sql
RESTORE FROM '/backups/mydb.bak' VERIFY;
RESTORE FROM 'C:\Backups\store-2024-06-01.mnemic' VERIFY;
```

Copies the selected backup into private staging beside the live database (`.mnemic-restore-staging-*`, so size that volume for one more copy), fsyncs it, and verifies that staged copy against `<backup>.manifest.json`. Verification checks the exact byte length and re-hashes every page using SHA-256. Only a complete match is atomically published as the database's pending-restore sidecar; a missing manifest, malformed manifest, truncation, extra bytes, page-size mismatch, or changed page rejects the statement with `User / BACKUP_INVALID` without replacing that sidecar, and a backup path that does not exist reports `User / BACKUP_NOT_FOUND`. The staging is removed when the statement ends; a restore interrupted by a crash leaves it until the next restore or engine open in that directory reclaims it. On a volume whose file locks cannot prove the owner dead, staging uses an `-unleased-` directory that nothing reclaims, and one left by a crash is removed by hand once its process has exited.

Restore is offline: a successful statement does not mutate the currently-open engine. Reopen the database to promote the verified sidecar: on a server, `REOPEN DATABASE [name]` from a server-level connection closes it, terminating every session connected to it, opens it again and reports `RestorePromoted` ([databases](databases.md#reopening-a-database)); a server restart or any later open promotes it too. That open first validates a disposable copy of the sidecar in the same kind of staging, then removes recovery-log artifacts belonging to the old database before replacing the data file, preventing stale WAL from replaying over the restored snapshot.

`VERIFY` is mandatory—there is no unverified MQL restore form. The statement accepts server-local paths and `file://` URIs, requires no active transaction or migration, and requires `ServerAdmin` permission on the server because it can replace the whole database on reopen. The typed client wrapper is `IDatabase.RestoreBackupAsync(path)`.

## VERIFY DATABASE

```sql
VERIFY DATABASE;
```

Runs a cancellable online integrity pass over every allocated page. The pass rechecks page checksums and common page-header invariants, including embedded page identity, allocation incarnation, page/owner types, item count, free-space offset, and continuation references. Catalog records have already been decoded and validated during engine open; malformed catalog lengths and references therefore fail closed before this command can run.

The pass captures its upper page bound at start and visits each surviving page ID at most once. Pages added beyond that bound wait for the next verification. Concurrent shrink can reduce the pages checked; it does not restart the pass at the beginning or count surviving corruption again.

An admitted verification keeps storage alive until its result is built or a failure unwinds. Engine shutdown waits for it, and a verification callback cannot close the same engine while the pass awaits that callback. Every failure releases that admission, memory exhaustion and modeled process death included, so later graceful shutdown still proceeds.

The command returns exactly one row:

| Column | Type | Meaning |
|---|---|---|
| `Status` | `STRING` | `Healthy` when no corruption was found; otherwise `Corrupt`. |
| `PagesChecked` | `INT64` | Number of distinct pages checked, including pages found corrupt. |
| `CorruptionCount` | `INT64` | Number of corrupt pages found during this pass. |
| `StartedAtUtc` | `DATETIMEOFFSET` | UTC start time. |
| `CompletedAtUtc` | `DATETIMEOFFSET` | UTC completion time. |
| `ElapsedMicroseconds` | `INT64` | Monotonic elapsed duration in microseconds. |

Corruption does not stop the remaining physical scan. Each failure is also published through `[Mnemic].[Health]`, which quarantines the affected fault domain. Cancellation stops promptly between page reads. The command is read-only, but it requires no active migration or transaction and uses the DDL permission tier on the server.

## WAL archiving and point-in-time recovery

WAL archiving is enabled when the engine opens with `EngineOptions.WalArchiveUri`. That option is rejected with `EncryptionKey` or `EncryptionKeyProvider` before database or archive creation: the active segmented log used for archiving does not use the keyed framing required by authenticated page storage. Protecting sealed archive objects with `IWalArchiveProtector` does not change that active-log boundary. Ordinary encrypted `BACKUP` and verified `RESTORE` remain available.

The built-in provider accepts an absolute `file:` URI. An in-memory database/WAL cannot provide a durable archive. The default one-minute cut interval bounds healthy archive lag; natural segment rollover may publish sooner.

```sql
ARCHIVE WAL;
CREATE RESTORE POINT 'before-schema-v2';
```

`ARCHIVE WAL`, `CREATE RESTORE POINT` and a publishing `RESTORE ... USING ARCHIVE` need a database opened with a WAL archive (`EngineOptions.WalArchiveUri`); without one they are refused with `OBJECT_NOT_IN_PREREQUISITE_STATE`, as is a publishing restore that names another archive than the database's own. `ARCHIVE WAL` seals the current tail when needed and waits for durable progress. With no new WAL it advances the proven cut time without creating an empty segment. `CREATE RESTORE POINT` appends a no-page-change physical marker, returns its generated GUID, and waits for archive coverage. Labels are diagnostic and may repeat; recovery targets the returned GUID.

Point-in-time restore supports four exact target forms:

```sql
RESTORE FROM '/backups/mydb.mnemic' VERIFY
USING ARCHIVE 'file:///archives/mydb'
TO LATEST;

RESTORE FROM '/backups/mydb.mnemic' VERIFY
USING ARCHIVE 'file:///archives/mydb'
TO CSN 184467;

RESTORE FROM '/backups/mydb.mnemic' VERIFY
USING ARCHIVE 'file:///archives/mydb'
TO TIMESTAMP '2026-08-22T14:30:00+00:00';

RESTORE FROM '/backups/mydb.mnemic' VERIFY
USING ARCHIVE 'file:///archives/mydb'
TO RESTORE POINT '6cd61e72-8377-4b63-b6c0-c10de374db06';
```

`TO LATEST` means the latest boundary proven by archive progress, never an unarchived local tail. A timestamp must contain an explicit UTC offset and selects the latest physical commit at or before it. A CSN selects the exact commit record. A target before the base backup or beyond proven coverage fails without publishing pending state.

A backup names its parent timeline. Select an exact descendant branch when needed; the engine never guesses among siblings:

```sql
RESTORE FROM '/backups/mydb.mnemic' VERIFY
USING ARCHIVE 'file:///archives/mydb'
ON TIMELINE '9f1440ce-354e-4cc7-b593-ecf6b96d27c0'
TO LATEST;
```

Use metadata preflight before an incident and full rehearsal for a scheduled restore drill:

```sql
VERIFY RESTORE FROM '/backups/mydb.mnemic'
USING ARCHIVE 'file:///archives/mydb'
TO LATEST;

VERIFY RESTORE FROM '/backups/mydb.mnemic'
USING ARCHIVE 'file:///archives/mydb'
TO LATEST REPLAY;
```

Preflight resolves lineage, the exact target, required key IDs, input bytes, and workspace without staging or mutation. `REPLAY` additionally copies the base into private staging, runs bounded redo and final validation, reports measured timings, and removes the staging directory without publishing a child timeline or pending restore.

A preflight, a rehearsal and a point-in-time restore each return exactly one row:

| Column | Type | Meaning |
|---|---|---|
| `Mode` | `STRING` | `Preflight`, `Rehearsal` or `Restore`: the form that ran. |
| `DatabaseId` | `GUID` | The database the base backup and the archive belong to. |
| `Lineage` | `ARRAY STRING` | The timelines replay follows, from the base backup's timeline to the target's. |
| `BaseWalEndLsn` | `STRING` | The base backup's WAL end, as 16 hexadecimal digits. |
| `TargetWalEndLsn` | `STRING` | The WAL end of the selected target, as 16 hexadecimal digits. |
| `TargetCommitSequenceNumber` | `INT64` | The CSN of the last commit replay includes. |
| `TargetCommitUtc` | `DATETIMEOFFSET` | The ordered decision time of the selected target. |
| `ProvenCoveredEndLsn` | `STRING` | The end of proven archive coverage on the target timeline, as 16 hexadecimal digits. |
| `BytesToRead` | `INT64` | Archive bytes read up to the target. |
| `EstimatedWorkspaceBytes` | `INT64` | The largest scratch workspace replay needs at once. |
| `RequiredKeyIds` | `ARRAY STRING` | The key IDs the archive's protection needs. |
| `ProtectionMode` | `STRING` | `IntegrityOnly` or `AuthenticatedEncryption`. |
| `ProtectionAlgorithm` | `STRING` | The archive's protection algorithm. |
| `SpilledTransactionCount` | `INT32` | Transactions larger than the memory budget, which replay spills to a private file. |
| `PeakTransactionMemoryBytes` | `INT32` | The largest transaction replay holds in memory. |
| `ChildTimelineId` | `GUID` | The child timeline a restore published; NULL for the other forms. |
| `OperationId` | `GUID` | The rehearsal's operation identity; NULL for the other forms. |
| `ReplayedTransactions` | `INT64` | Committed transactions replayed. |
| `ReplayedRecords` | `INT64` | Redo records replayed, each counted once in one of the next four columns, which sum to it. |
| `AppliedRecords` | `INT64` | Replayed records redo wrote to the pages they touch. |
| `SkippedByPageLsn` | `INT64` | Replayed records the PageLSN of every page they touch already covered. |
| `SkippedByIncarnation` | `INT64` | Replayed records another lifetime of a page they touch made stale. |
| `LeftOutRecords` | `INT64` | Replayed records redo wrote nothing for: a typed index-page run that a later full image of its page supersedes, or the records of an index page that cannot be redone up to its next full image, whose index the promoted database rebuilds. |
| `ReplayedBytes` | `INT64` | Bytes of the replayed redo records. |
| `StagingDurationMicroseconds`, `ManifestVerificationDurationMicroseconds`, `ReplayDurationMicroseconds`, `ValidationDurationMicroseconds`, `TotalDurationMicroseconds` | `INT64` | The rehearsal's measured phase durations; NULL for the other forms. |

A preflight replays nothing and reports zero in every replay count. A rehearsal and a restore of the same base, archive and target replay the same records and complete each the same way.

The base must be a backup of this database taken with `REQUIRE PITR`: a plain backup, or one of another database, is refused with `OBJECT_NOT_IN_PREREQUISITE_STATE` before any archive is read, and a PITR-capable manifest missing its archive anchor, identity or protection record is `BACKUP_INVALID`. A real PITR verifies and replays into private staging, validates the result, publishes a child timeline, then atomically publishes the pending-restore sidecar. The live engine is unchanged until the database reopens: `REOPEN DATABASE` promotes the sidecar on a server ([databases](databases.md#reopening-a-database)), and so does any later open of it. That open needs the same `WalArchiveUri` the restore published the child timeline with. See the [operator guide](https://mnemic.io/docs/wal-archiving-pitr/) for monitoring, drills, offline tail capture, retention floors, and key rotation.

## DEFRAGMENT

```sql
DEFRAGMENT;
```

Compacts every collection by reclaiming the space that `DELETE`/`UPDATE` strands. It drives the incremental compactor to convergence over the space that is reclaimable when the statement starts — relocating live slots off pages carrying dead space (and reclaiming fully-dead head/ tail pages) in bounded, brief-locked, crash-safe units, and following its own relocations until none of that space remains. Space that concurrent writes strand while it runs is left to the next `DEFRAGMENT` or `DEFRAGMENT INCREMENTAL`, so sustained write traffic cannot prolong the statement; on a database without concurrent writes it leaves no reclaimable space behind. It holds **no engine-global lock** and runs concurrently with queries and writers; indexes stay live throughout (a crash mid-`DEFRAGMENT` leaves a consistent database that the next `DEFRAGMENT` resumes — no repair). Typically followed by `SHRINK` to release the freed pages back to the filesystem:

```sql
DEFRAGMENT;
SHRINK;
```

### DEFRAGMENT INCREMENTAL

```sql
DEFRAGMENT INCREMENTAL;
DEFRAGMENT INCREMENTAL [Namespace].[Collection];
DEFRAGMENT INCREMENTAL BUDGET 100;
DEFRAGMENT INCREMENTAL [Namespace].[Collection] BUDGET 5;
```

A single bounded pass of the same machinery `DEFRAGMENT` drains to convergence — useful as a non-interfering background tick or with `BUDGET` to cap the work. Two jobs:

- **Space reclamation** (all collections): relocates the live slots off mostly-dead interior data pages and frees those pages — reclaiming the space that `DELETE` strands (a deleted row is logically gone but its bytes stay on the page until something rewrites it).
- **Temporal co-location** (temporal collections): relocates a primary key's scattered versions into contiguous pages so a history scan (`GetAllVersions` / `AS OF` / `IN BETWEEN`) shares disk reads. Every version and its AS OF resolution is preserved. Histories whose complete records already occupy as few home pages as append packing allows are skipped, including large keys, large inline bodies, and spilled bodies. An admitted run starts on a fresh page so partially filled append pages cannot keep the same history eligible for repeated relocation.

Reports the number of units compacted.

- An optional `[Namespace].[Collection]` scopes the drain to one collection (default: all collections); a name the catalog does not hold is `User / COLLECTION_NOT_FOUND`, never an empty success.
- `BUDGET n` caps the number of pages compacted in this call (`INCREMENTAL` only; `BUDGET` without `INCREMENTAL` is a parse error).
- Reclaimed pages return to the free pool for reuse; run `SHRINK` afterward to return any that end up trailing to the filesystem.
- Temporal collections keep all versions through relocation (`AS OF` and history are unaffected). A single bounded pass reclaims interior + fully-dead head/tail pages; a full `DEFRAGMENT` (drain to convergence) additionally relocates a tail's or a lone page's live slots to reclaim that dead space too.

For scoped per-collection reclamation use `DEFRAGMENT INCREMENTAL [Namespace].[Collection]`; plain `DEFRAGMENT` always drains every collection. The scoped collection name is case-insensitive, as in other collection commands.

## SHRINK

SHRINK reclaims pages that are safe to reuse. Work still protected by an active read, or queued after the current reclamation pass began, stays queued for a later pass. A reader finishing during maintenance does not turn that ordinary postponement into a stalled-reclamation error.

```sql
SHRINK;
```

Reduces the database file size. First relocates the live collection data pages sitting above a freed hole down into the holes (drawing destinations from the free list, lowest first; an overflow chain moves with the row that owns it), then truncates the run of free pages at the end of the file. Run after `DEFRAGMENT` to return the reclaimed space to the filesystem.

Only the free pages above the highest page SHRINK cannot move are returned. Collection data pages move, including those of hidden plugin collections such as vector indexes; these stay in place: index pages (primary-key, secondary, array-element, temporal and attribution trees), catalog and registry pages, the free-space map pages at their fixed positions, a collection's lone remaining data page, and an overflow page whose owning row sits below every hole. A free hole beneath one of them stays inside the file and is reused by later writes rather than returned. Rebuilding a secondary index after `DEFRAGMENT` (`DROP INDEX` and `CREATE INDEX` in a migration) writes the new tree into the lowest free pages and frees the old one, after which `SHRINK` can return more.

## RECLAIM ORPHAN PAGES

```sql
RECLAIM ORPHAN PAGES;
```

Requests repair of leaked pages: pages that stay allocated although no storage root owns them, as an unclean shutdown can leave when recovery cannot prove a page free. The statement records a durable request beside the database file (`<file>.reclaim-orphans`) and returns; the repair runs on the next open of the database, such as the one `REOPEN DATABASE` runs ([databases](databases.md#reopening-a-database)), after recovery and before any statement, plugin or background work is admitted, because only then can nothing else be allocating. It walks every storage root and returns each allocated page none of them reaches to the free pool, and the server log reports how many pages it reclaimed. A defect anywhere in the walked structures aborts the pass without changing a page. If the database opens quarantined, or its deferred page frees or recovery log cannot be settled first, the request waits for a later open. Run `SHRINK` afterwards to return the reclaimed space to the filesystem.

Requires `ServerAdmin` on the server, a file-backed database (an in-memory database has no next open and returns `FEATURE_NOT_SUPPORTED`), and no active transaction or migration. `DROP DATABASE` removes a pending request.

## Session parameters — `SET` / `SHOW`

Override a registered parameter for the **current connection only** (PostgreSQL `SET` / `SHOW` model). The override lives for the connection's lifetime; a new connection starts at the engine defaults.

```sql
SET slow_query_threshold = '250ms';  -- duration: '30s' / '500ms' / '2m'
SET slow_query_threshold TO 30;      -- bare integer = seconds
SET slow_query_threshold = 0;        -- observe every statement
SET slow_query_threshold = NONE;     -- disable for this session
SET slow_query_threshold = DEFAULT;  -- clear the override (engine default applies)

SHOW slow_query_threshold;           -- effective value for this session
SHOW ALL;                            -- every settable parameter
```

String parameter values use ordinary MQL string escaping: represent an embedded single quote by doubling it (`'it''s'`). Canonical statement text preserves that escaping when a `SET` statement is printed or replayed.

`SHOW` returns columns `Name`, `Value`, `Source` (`session` if overridden, else `engine-default`), and `Description`.

Settable parameters:

| Parameter | Meaning |
|---|---|
| `slow_query_threshold` | Per-session slow-query threshold; overrides the current engine default. A non-negative duration, a non-negative integer (seconds), or `NONE`. Zero observes every statement; `NONE` disables observation. A session may tighten, loosen, disable, or enable observation for its own connection. |

If a session enables observation while the engine default is off, events still reach enabled structured `Mnemic.Mql.SlowQuery` logging and `mnemic.slow_query.elapsed_us` meter sinks. Session settings do not allocate shared history. `[Mnemic].[SlowQueries]` retains events only when a host threshold or `ALTER ENGINE SET SLOWQUERY` has enabled the engine-owned ring.

Unknown parameter names and invalid values are rejected. The dedicated `SET STATEMENT TIMEOUT` verb — see [Timeouts and cancellation](cancellation.md) — keeps its own syntax and is not part of this registry.

## ALTER ENGINE SET SLOWQUERY

Configure the current database engine's default slow-query threshold and shared history capacity without restarting. Requires `ServerAdmin` on the server and no active transaction or migration in the calling session.

```sql
ALTER ENGINE SET SLOWQUERY (Threshold = '100ms', RingCapacity = 2048);
ALTER ENGINE SET SLOWQUERY (Threshold = 2);        -- integer seconds
ALTER ENGINE SET SLOWQUERY (Threshold = NONE);     -- disable default selection
ALTER ENGINE SET SLOWQUERY (RingCapacity = 512);   -- retain newest events
```

Keys are case-insensitive; supply at least one, without duplicates. `Threshold` accepts a non-negative duration (`'0ms'`, `'500ms'`, `'30s'`, `'2m'`), integer seconds, or `NONE`. Zero selects every statement. `RingCapacity` is a positive `INT32`. Omitted keys retain their current engine value, including when another administrator updates them concurrently.

The complete policy is persisted before publication and overrides host `EngineOptions` on subsequent opens. A failed or cancelled update before durable persistence leaves the live policy unchanged. Success returns `Ok` with the applied threshold and capacity. Ordinary DML cannot edit the internal settings.

Existing and new sessions without an override use the live engine default at statement start. Explicit `SET slow_query_threshold` overrides, including `NONE`, continue to win. `SET slow_query_threshold = DEFAULT` resumes inheritance. An in-flight statement keeps the selection threshold captured when it started.

Every successful global configuration enables shared history, including `Threshold = NONE` or a capacity-only change. Disabling default selection keeps retained events and allows session overrides to contribute new events. Resizing keeps the newest events, including observations arriving during the update. History events themselves remain process-lifetime data and are empty on reopen.

`SHOW slow_query_threshold` reports the current session's effective value and source. `[Mnemic].[Capabilities]` reports the engine default selection toggle as `SlowQueryLog` and allocated history capacity as `SlowQueryRingCapacity` (zero before history is enabled). See [system collections](system-collections.md).

## SET CREATEDBY

Committing a policy that changes which actors it admits (another mode, byte limit or target) waits for existing source writers to settle; under `COLLECTION` it scans existing source metadata to build or refresh the target's reverse index, and disabling or changing the target retires the previous target's reverse index. A migration whose policies all keep the committed policy's actors, restating it to change no more than how they read, does neither, so it commits while source transactions stay open. A policy change does not change stored attribution values. On reopen, a missing or invalid active reverse index is rebuilt before queries are admitted. `CREATEDBY_SOURCE_MISSING` is an engine contract violation: the engine refuses an incomplete backfill. Its description is in `[Mnemic].[ErrorCatalog]`.

Configure created-by tracking mode (requires migration context):

```sql
SET CREATEDBY NONE;                         -- Attribution unavailable; all collections must be disabled
SET CREATEDBY EXTERNAL 64;                  -- Accept opaque attribution bytes (max 64 bytes)
SET CREATEDBY EXTERNAL 64 DISPLAY AS INT32; -- … which hold a little-endian INT32
SET CREATEDBY COLLECTION [Audit].[Users];   -- Track as reference to a collection
SET CREATEDBY COLLECTION [Audit].[Users] DISPLAY CONCAT([First], ' ', [Last]); -- … whose actors read as a name
```

Each form replies with the policy in its own words: `CreatedBy set to NONE.`, `CreatedBy set to EXTERNAL 64.` or `CreatedBy set to COLLECTION [Audit].[Users].`, naming the target as the catalog spells it, with its display, if any, as `[Mnemic].[CreatedBy]` reports it.

Under `COLLECTION`, the actor named in a write's `CREATEDBY` is a primary-key value of the target collection, and `Meta.CreatedBy` stores that key in its binary form: an `INT64` key is its 8 little-endian bytes, so `CREATEDBY 7` reads back as the BYTES value `'BwAAAAAAAAA='` in Base64, which `WHERE [Meta].[CreatedBy] = 'BwAAAAAAAAA='` matches.

These statements configure the database-wide actor policy. Collections default to attribution disabled and independently opt in using `(CreatedBy: TRUE)` on CREATE or `ALTER COLLECTION [Namespace].[Collection] SET CREATEDBY TRUE`. Configure EXTERNAL or COLLECTION before enabling a collection; disable every participating collection before switching to NONE. Both changes require a migration. Read the effective policy through `FROM [Mnemic].[CreatedBy]`.

Enabled collections require an explicit actor or COPY intent on every authored write. Disabled collections reject attribution clauses. Supply an actor with the [DML CREATEDBY clause](dml.md#createdby-attribution), for example `INSERT INTO [App].[Items] ([Id]: 1) CREATEDBY @actor`. `SET CREATEDBY '<id>'` is not valid MQL.

The singleton and active policy remain private to the owning migration until commit. Abort discards both. Internal policy binds the target's stable catalog identity, so collection and namespace renames preserve it. The target may be a collection or materialized view that the same migration creates or renames, before or after `SET CREATEDBY`. A materialized view target follows `REFRESH MATERIALIZED VIEW`, `ALTER MATERIALIZED VIEW` and a source rebuild to the generation they publish; a key that leaves the view while a retained version still names it as actor refuses the statement with `CREATEDBY_ACTOR_REFERENCED` and rolls the migration back. Changing the target's key is refused with `DEPENDENT_OBJECTS_EXIST` naming the policy, whether or not an actor is recorded: `SET PRIMARY KEY` on a collection target, and a view replacement keyed by another type ([references to a materialized view](materialized-views.md#references-to-a-materialized-view)). DROP of that target is rejected unless the migration first reconfigures CreatedBy. Conflicting policy and target-drop migrations cannot both commit. Policy changes do not backfill stored attribution.

<a id="createdby-display"></a>

### How actors read: `DISPLAY`

`DISPLAY` names how the policy's actors read, so every client shows them alike instead of as Base64 bytes. It is part of the policy: restating the policy without it clears it, and it is set only in a migration, published with the policy, and discarded with an aborted migration. The engine stores it and never evaluates it.

- **`COLLECTION … DISPLAY <expression>`** is a scalar expression over one document of the actor collection. It binds as a CHECK of that collection does: its fields and its subtypes' fields, and only subtypes the collection declares in `SUBTYPE IS` or a `[_type]` comparison; a field it does not have, a subtype it does not declare, a parameter, a subquery, a window or a virtual computed field is `INVALID_DEFINITION`, and an aggregate is a `SYNTAX_ERROR`. It must evaluate to `STRING` under every subtype (`TYPE_MISMATCH` otherwise; `CAST` a number `AS STRING`); arms of different types read as `STRING`, as a `CASE` does. A static `ENUM_VALUE` binds its enum by identity, so the display follows a rename of it, and the enum cannot be dropped while the display names it (`DEPENDENT_OBJECTS_EXIST`).
- **`EXTERNAL <size> DISPLAY AS <type>`** names what the opaque bytes hold: `STRING` (strict UTF-8), `BYTES` (shown in Base64), `INT8`, `INT16`, `INT32` or `INT64` (little-endian) or `GUID` (the byte order the engine stores a `GUID` key in). Any other type is `INVALID_VALUE`; it never changes which actors are admitted.
- `NONE` takes neither, an expression after `EXTERNAL` or a type after `COLLECTION` is a `SYNTAX_ERROR`.

A polymorphic actor collection can read each subtype its own way:

```sql
BEGIN MIGRATION 12;
SET CREATEDBY COLLECTION [Core.LegalEntity].[LegalEntity]
    DISPLAY CASE WHEN SUBTYPE IS [User]
        THEN CONCAT([FirstName], ' ', [LastName], ' (', [Email], ')')
        ELSE [Name] END;
COMMIT MIGRATION;
```

`[Mnemic].[CreatedBy]` reports the expression in canonical text with current names (`SUBTYPE IS [User]` reads as `[_type] = 'User'`) in `Display`, and the type in `DisplayType`. A client labels the actors of a page, the values of every column its result schema marks as [CreatedBy attribution](queries.md#createdby-attribution-columns), whatever the column is called, by reading the actor collection itself, in plain MQL, with the keys it holds as parameters in the actor collection's key type:

```sql
FROM [Mnemic].[CreatedBy];  -- ReferencedCollection, PrimaryKeyField, Display
FROM [Core.LegalEntity].[LegalEntity] WHERE [Id] IN (@k0, @k1)
    SELECT [Id], (CASE WHEN [_type] = 'User' THEN CONCAT([FirstName], ' ', [LastName], ' (', [Email], ')') ELSE [Name] END) AS [Label];
```

The read is current: an actor whose versions have all ended, or a label that evaluates to `NULL`, has no label, and the client falls back to the key. Over a temporal actor collection (`[Mnemic].[Collections]` lists `IsTemporal`), the actors the current read does not find can be read again over their whole retained history, `IN BETWEEN '0001-01-01T00:00:00Z' AND '9999-12-31T23:59:59.9999999Z'` with `[Meta].[ValidFrom]` selected beside the label, plus `AS OF '9999-12-31T23:59:59.9999999Z'` for a version that starts at that last instant, which the half-open range leaves out ([`IN BETWEEN`](temporal.md#in-between-temporal-range)), and labelled by the version that starts latest, as Studio labels an actor that has left.

The display depends on the actor collection as a view does on its source. `RENAME FIELD` or `DROP FIELD` of a field it reads, a `SET TYPE` that converts that field's values, and `RENAME SUBTYPE` of a subtype it names are refused at the statement with `DEPENDENT_OBJECTS_EXIST`; any other change of the collection that leaves the display unable to bind or no longer `STRING`, such as `DROP SUBTYPE`, and a materialized view target's `REFRESH` or `ALTER` that drops a column it reads, are refused the same way and roll the migration back. Restate the policy without `DISPLAY` earlier in the same migration to make such a change, and with the new display after it; that restatement changes no actors, so it commits without draining writers or rebuilding the reverse index. Collection and namespace renames keep the display, since the policy binds its target by identity.

DELETE requires no new attribution: physical removal discards it, while temporal end preserves the existing attribution and its reference restriction.

Physical deletion of an Internal-mode target probes the persisted reverse index and validates matching source slots, including inserts and removals made earlier in the deleting transaction. The first confirmed reference rejects the delete. A migration that changes which actors the policy admits uses a cross-collection scan until commit builds or refreshes the index; a miss in that fallback can require a database-wide scan. `EXPLAIN DELETE` describes the indexed check and private-policy fallback. The `mnemic.createdby.restrict_scans` counter counts fallback scans only. An older query snapshot cannot authorize new attribution to a physically deleted actor; admission checks current state plus the transaction's own writes. Concurrent reference writes may share an actor. A competing physical actor delete conflicts with those transactions until they complete; it cannot commit past a pending reference. These conflicts use the ordinary transient transaction conflict response.

## See also

- [Transactions](transactions.md) — `BACKUP`, `RESTORE … VERIFY`, `VERIFY DATABASE`, `DEFRAGMENT`, `SHRINK` require no in-flight transaction
- [WAL archiving and PITR operator guide](https://mnemic.io/docs/wal-archiving-pitr/)
- [Migrations](migrations.md) — `SET CREATEDBY` requires migration context
- [Timeouts and cancellation](cancellation.md) — `SET STATEMENT TIMEOUT`
- [Practical examples](examples.md#full-db-maintenance-cycle)

## Detailed syntax and execution


<a id="syntax-9-maintenance"></a>

### Maintenance

```sql
BACKUP TO 'path/to/backup.mnemic';
BACKUP TO 'path/to/backup.mnemic' REQUIRE PITR;
DEFRAGMENT;                                       -- drain every collection's starting dead space to convergence (online; no global lock; indexes stay live)
DEFRAGMENT INCREMENTAL [[Ns].[Coll]] [BUDGET n]; -- one bounded pass of the same machinery: reclaim dead-slot space (all) + co-locate temporal version chains
SHRINK;
RECLAIM ORPHAN PAGES;               -- request repair of leaked pages; it runs on the next open
ANALYZE [Ns].[Coll];               -- per-secondary-index stats; the collection is required
SET CREATEDBY NONE;
SET CREATEDBY EXTERNAL 64 [DISPLAY AS <type>];
SET CREATEDBY COLLECTION [Audit].[Users] [DISPLAY <expression>];
```

Authenticated storage encryption is configured at engine open through `EncryptionKey` or `EncryptionKeyProvider`, without a separate MQL encryption clause. `BACKUP` preserves encrypted storage; opening or restoring a backup requires its retained keys. Restore authenticates the candidate before live replacement. `WalArchiveUri` is rejected with either encryption option because the active segmented archive log does not use the required keyed WAL framing; protecting sealed archive objects alone does not make that combination valid. See [maintenance](maintenance.md).

SHRINK reclaims currently safe pages; retirements pinned during a sweep or queued after its cutoff remain pending for a later pass. A reader leaving between the sweep and diagnostics is ordinary concurrency, not a stalled-drain error. Clean shutdown still requires complete reclamation.

`SET CREATEDBY` is migration-only policy configuration, not a statement or session attribution setter. The three forms above are the complete grammar; `SET CREATEDBY '<id>'` is unsupported. DML supplies attribution through `CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>` before `RETURNING`. `COLLECTION` mode encodes that actor using the target's native primary-key type and validates its existence. Policy changes remain private until migration commit. Physical deletion of the target is restricted while another live source slot references it. DELETE requires no new attribution; temporal end preserves the existing attribution and physical removal discards it. The check observes inserts, reassignments, and removals made earlier in the deleting transaction. A published Internal policy probes its persisted reverse index and validates candidate source slots against current metadata. Changing the policy inside a migration uses a cross-collection scan until commit builds or refreshes the index. That fallback stops at the first match; a miss can require a complete scan. `EXPLAIN DELETE` describes the indexed check and private-policy fallback. The `mnemic.createdby.restrict_scans` counter counts fallback scans only. Attribution admission checks current state plus own writes; a historical snapshot cannot authorize a new reference to a physically deleted actor. Concurrent source writers may share an actor, but physical actor deletion and reference creation hold conflicting transaction-lived claims. A competing operation gets a transient transaction conflict; retry the whole transaction.

`ANALYZE` writes per-secondary-index rows to `[Mnemic].[Statistics]`: single-column, scalar compound (collision-free length-framed ordered path), array-component, and partial (tagged with the index's exact canonical filter text). Identifier case is insensitive in the deterministic identity, while literal case is preserved because Base64 `BYTES` literals can differ semantically by case. Each row carries NDV that a full scan computes exactly until the bounded distinct-key dictionary saturates; incremental rows use the merged HLL estimate. The persisted HLL sketch enables eligible incremental refreshes, and the configurable row threshold controls sketch persistence independently of exact-dictionary saturation. Each row also carries an equi-depth histogram (100 buckets), an MCV list (top 20 heavy-hitters), full-collection scalar rows for each compound component even without a standalone index, a compound-correlation ratio for scalar multi-column indexes, and the cost-model page-packing inputs `AvgRowWidthBytes` (collection-level average document width) + `AvgKeyWidthBytes` (per-column average encoded-key width). Compound NDV and width use the physical index's escaped, terminated tuple encoding; variable-width binary components and embedded zero bytes cannot collapse tuple boundaries. Compound correlation measures both component NDVs and joint NDV over the physical index population, where every key component is present. A partial compound index additionally restricts both to filter-satisfying rows, so missing or excluded rows cannot inflate its correlation ratio. Correlation ratios remain in the finite `[1, FLOAT64_MAX]` domain: approximate NDV crossings clamp to 1 and larger mathematical results saturate instead of failing `ANALYZE`. Scalar statistics rows carry exactly `1.0`; any durable row outside its scalar/compound ratio domain fails closed as `STATISTICS_CATALOG_CORRUPT`. The code-owned statistics schema is exactly version `1`; `IsCompound`, `IsArrayComponent`, `CompoundCorrelationRatio`, and `IsIncremental` are mandatory current state, while estimator payloads, widths without observations, and the first usable write watermark remain nullable. A row of any other shape is refused ([compatibility](https://mnemic.io/docs/compatibility/)). `LastAnalyzedWriteCsn` records the gap-free commit cut covered by the row; `IsIncremental` marks additive single-column HLL/count state produced by background auto-analyze. Explicit `ANALYZE` always performs a full scan. Auto-analyze refreshes existing statistics when committed-write coverage trails or row-count drift requires it; first analysis remains explicit. Besides the periodic `EngineOptions.AutoAnalyzeInterval` (15 minutes), committed row bursts wake analysis at `AutoAnalyzeWriteThreshold + AutoAnalyzeWriteRatio * size` (defaults 50 and 0.1, rounded up to at least one). Rollback does not count. `AutoAnalyzeWriteThreshold = null` preserves only periodic refresh; `AutoAnalyzeInterval = null` disables both. These host settings affect scheduling, not query results or the separate estimator reconciliation threshold. Counters reset on reopen; persisted watermarks and periodic freshness checks remain. Idle waits round up to one millisecond so very short cadences still yield. Explicit and background requests for the same collection serialize as one refresh, preventing stale-watermark overwrite and first-use statistics-catalog publication deadlock; different collections remain concurrent. A request whose collection identity changed while waiting fails rather than publishing under a reused name. Background passes use the incremental path only for non-temporal, non-compound, non-array-component, non-partial rows that all share one valid watermark and HLL; otherwise they perform a full scan. An incremental pass observes live rows whose latest logical-write CSN falls in the next gap-free interval. Deleted rows do not enter that delta, so their prior observations remain in the additive state until drift forces a reconciling full scan. The cost-based planner reads MCV first, histogram next, uniform `TotalCount / NDV` last when sizing an equality probe, and costs index-vs-scan in *pages*: a wide-matching index seek whose random heap fetches saturate the heap's pages loses to a sequential scan, while a selective seek stays cheap (page-packing). That `FullScan` verdict is authoritative when every modeled candidate has a fresh estimate and the query has no unpriced `ORDER BY` benefit; unknown-estimate candidates and primary-key ranges remain eligible. Full compound-key equality uses the frequency of the requested tuple, with schema normalization and escaped, terminated components matching ANALYZE. Partial index statistics apply only when the query implies that index's filter. For a non-temporal full scan, scalar equality conjunctions use complete declared or full-index joint populations, then multiply value-aware scalar selectivities for remaining fields. Repeated equivalent equalities count once; conflicting equalities on one field estimate zero. Missing or stale evidence preserves the scan fallback. A sole `!=` predicate subtracts the requested value's estimated frequency from the non-null population. These statistics guide cost and explain estimated output; predicate evaluation still determines the actual result.

Cross-field equality OR is costed too: a fully estimated union that would touch most heap pages runs as `FullScan`, while a selective or partly unknown union retains `BitmapOr`. DROP COLLECTION purges these rows transactionally. Statistics are keyed by the stable collection identity, so `ALTER COLLECTION … RENAME TO` and `ALTER NAMESPACE … RENAME TO` leave their rows, deterministic IDs and cache keys unchanged and only the rendered `CollectionName` follows the new name ([lifecycle](analyze.md#lifecycle)). The standalone engine rename APIs enroll structure and statistics in one WAL transaction too.

`BACKUP` writes the file via 1 MB batched `WriteAsync` calls (256 pages at the 4 KB default page size) into an `ArrayPool`-rented buffer, then fsyncs the output file (`FlushFileBuffers` / `fsync`) before returning. It returns exactly one row with `Destination STRING`, `SnapshotCommitSequenceNumber INT64`, and `IsPointInTimeRecoveryCapable BOOL`. `REQUIRE PITR` fails instead of publishing unless durable, read-verified archive coverage protects the exact backup anchor; plain `BACKUP` reports the resulting capability through that boolean column. The destination string-literal is required — the parser rejects `@parameter` paths and function calls in this position. Before copying, backup briefly freezes CSN allocation publication, drains page apply, and waits for the physical database to form an exact applied prefix. Its header/metadata LSN therefore includes every committed LSN through that value and no later commit. The wait is bounded and cancellable; after a published commit apply failure, backup returns `COMMIT_APPLY_FAILED_AFTER_MARKER` until reopen recovery establishes a trustworthy prefix. Later writes continue during the copy and are excluded through saved page before-images. For the default local-file destination, staging, inspection, cleanup, and atomic publication are relative to one retained destination-directory handle. Stable database and WAL identities come from the engine's already-open handles and the actual destination entry is checked again immediately before publication. An exchanged symlink, junction, or writable ancestor cannot redirect the operation; an identity or permission/I/O uncertainty fails closed. Windows, Linux, and macOS are supported. Cancellation or failure removes the private staging entry through the retained handle and preserves any prior completed snapshot. Restore = open the backup file (no statement); its header carries a backup-origin marker, so the first open force-rebuilds every secondary index from the heap and refuses to run with a pre-existing `.wal` next to the restored file (`RESTORED_BACKUP_STALE_WAL`) — restore to a fresh path.

See [`maintenance.md`](maintenance.md).

## Usage pitfalls


<a id="pitfall-attribution"></a>

### Attribution

- `SET CREATEDBY` configures policy; it does not set the current actor. Committing an Internal policy builds or refreshes a reverse index from existing source metadata without changing stored attribution values. Disabling or retargeting Internal waits for existing source writers to settle and retires the previous target's reverse index; rollback preserves it. Reopening repairs a missing or invalid active reverse index before serving queries. `CREATEDBY_SOURCE_MISSING` reports an engine contract violation, not an invalid actor value; consult `[Mnemic].[ErrorCatalog]`. Physical actor deletion uses that index; a policy changed in the current migration uses a source scan until commit backfill.
- Use trailing `CREATEDBY @actor` before RETURNING on INSERT/UPDATE/REPLACE. Enabled collections require a non-empty actor or explicit `COPY`; `COPY OR @actor` supplies a fallback only when the selected source is absent or unattributed. Copied actors are revalidated; Internal mode requires a currently valid native target key. NULL and empty attribution are rejected, as is any attribution clause on a disabled collection. There is no session actor.
- Rolling back an INSERT removes its attribution reference with the row; do not issue a separate attribution cleanup statement. Physical source deletion and its attribution-reference removal also roll back together. UPDATE/REPLACE attribution changes roll back to the previous reference with the row. Temporal version insertion keeps historical references; soft DELETE keeps them too. DROP COLLECTION removes references from all retained source versions, with migration rollback restoring them; other source collections are unaffected. Canceling a future version removes that version's reference; rewriting its predecessor body requires its own explicit attribution intent. Validity-only restoration and timeline splits preserve actors on retained fragments and remove references for fully removed versions. Compaction and schema rewrites preserve attribution references when moving rows. Aborting a rewrite restores the original references, including for rows inserted earlier in the migration. Unpublished attribution indexes belong to their creating transaction and are unavailable to other transactions.
- `Meta.CreatedBy` is BYTES. Compare byte parameters or base64 literals, with exact byte equality. ORDER BY compares unsigned bytes; text operators are unsupported. NULL and empty CREATEDBY are rejected; absent retained metadata reads as NULL.
- Reconfigure CreatedBy before dropping its Internal target. Renames preserve the reference. An uncommitted policy affects only its migration owner.

<a id="pitfall-set-createdby-configures-policy-not-the-current-actor"></a>

### `SET CREATEDBY` configures policy, not the current actor

Only `SET CREATEDBY NONE`, `SET CREATEDBY EXTERNAL <positive-int> [DISPLAY AS <type>]`, and `SET CREATEDBY COLLECTION [Namespace].[Collection] [DISPLAY <expression>]` are accepted, and they must run in a migration. `SET CREATEDBY '<id>'` is not a session form. Use `CREATEDBY <literal|@parameter|COPY [OR <literal|@parameter>]>` before `RETURNING` on INSERT, UPDATE, or REPLACE to supply statement attribution. Internal/`COLLECTION` mode encodes the actor in the target collection's native primary-key type and requires a currently valid actor for enabled collections, including an explicitly enabled actor collection. Initial identities can be seeded while the actor collection is disabled.

<a id="pitfall-backup-requires-an-exact-applied-prefix"></a>

### `BACKUP` requires an exact applied prefix

`BACKUP` does not merely lower its header LSN when an earlier commit is pending below a higher applied commit. The physical pages already contain that higher commit, so catch-up from a lowered scalar LSN would replay it twice. Capture briefly freezes CSN allocation publication and drains/retries until the pages form an exact prefix. The wait is bounded and observes statement cancellation. If a published commit failed during page apply, reopen the engine so WAL recovery finishes it; `COMMIT_APPLY_FAILED_AFTER_MARKER` is a fail-closed backup refusal, not a retryable snapshot error.

<a id="pitfall-a-single-defragment-incremental-pass-may-leave-headtail-dead-space"></a>

### A single `DEFRAGMENT INCREMENTAL` pass may leave head/tail dead space

`DEFRAGMENT INCREMENTAL` runs **one bounded pass**: it relocates live slots off mostly-dead interior pages and reclaims fully-dead head/tail pages, but it does not relocate a tail's or a lone page's *live* slots. So a tiny collection whose only dead space is in its single page may report `0 page(s) compacted` from one incremental pass — that is correct. Plain `DEFRAGMENT` (drain to convergence) *does* reclaim it (it relocates the lone/tail page's live slots), and `SHRINK` then returns the freed pages to the filesystem. `DEFRAGMENT` holds no global lock and runs concurrently with queries/writers; indexes stay live throughout.
