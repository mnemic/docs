# WAL Archiving and PITR Operator Guide

This guide covers the recovery contract of a single-node database: archiving its WAL, proving what a restore can reach, rehearsing restores, restoring during an incident, advancing retention and rotating archive keys.

## Configure the archive

Open a file-backed database with an absolute archive URI:

```csharp
var options = new EngineOptions {
    FilePath = databasePath,
    WalArchiveUri = new Uri(Path.GetFullPath(archivePath)),
    WalArchiveInterval = TimeSpan.FromMinutes(1),
    WalArchiveFailureTimeout = TimeSpan.FromMinutes(5),
    WalArchiveOperationTimeout = TimeSpan.FromSeconds(30),
    WalArchiveCompression = WalArchiveCompression.None,
    WalCapacityBytes = 512L * 1024 * 1024
};
```

`WalArchiveInterval` is a requested cut cadence, not an RPO guarantee. The achieved RPO is the age of durable `CoveredThroughUtc`. Natural segment rollover may archive sooner. `WalCapacityBytes` must cover the largest expected failure/retry window plus safety margin; reaching the high watermark blocks new mutations before active WAL can be overwritten.

The built-in `file:` provider is crash-safe and fenced: its per-database writer lease admits exactly one writer across all processes on every platform, passes to the next writer when its holder dies, and is refused outright on a file system whose locks it cannot prove (for example a network mount without lock coordination). Objects are staged in each database's `.staging` directory and renamed into place; staging is never recovery input, and the next writer for that database deletes whatever a killed writer left there. A directory on the same host or device is still not an independent disaster-recovery copy. Put it on a separately monitored volume and replicate or snapshot that volume outside the host. The archive path must not equal or alias the database, active WAL, pending-restore sidecar, or backup output.

Authenticated encryption is optional. Register an `IWalArchiveProtector` through `EngineOptions.PluginExtensionRegistries`; the default package provides `Aes256GcmWalArchiveProtector`, which requires an `IWalArchiveKeyProvider` and a durable, never-reusing `IWalArchiveNonceProvider`. Do not place raw key material in engine options, URIs, logs, or manifests. With a protector configured, archive and PITR scratch that would hold segment plaintext is encrypted under a key that never leaves the process, so plaintext never reaches the temporary volume.

## Establish and monitor recoverability

Create regular protected base backups and force coverage before declaring them usable:

```sql
BACKUP TO '/backups/mydb-2026-08-22.mnemic' REQUIRE PITR
```

Monitor the one-row operational view:

```sql
FROM [Mnemic].[WalArchive]
```

Alert before admission blocks, not after:

- `Enabled` must be true and `State` should be `Healthy`.
- `AchievedRecoveryLagSeconds` should stay below the intended RPO and well below `FailureTimeoutSeconds`.
- `PendingBytes` and `RetainedWalBytes` should converge rather than grow without bound.
- `AdmissionHeadroomBytes` must remain comfortably positive.
- `ConsecutiveFailures`, `FailureAgeSeconds`, `WriteBlocked`, and `BlockReason` are paging signals.
- Record `ProgressGeneration`, `RecoveryFloorGeneration`, protection mode, and capability flags in recovery evidence.

The `Mnemic.Engine` meter publishes archive cut/request latency, archive bytes, failures, pending/retained bytes, lag, admission headroom, and PITR verify/replay/rehearsal measurements. Timeline and key IDs deliberately are not metric attributes; read them from the system row or structured logs.

## Run restore drills

Run metadata preflight frequently enough to catch missing segments, keys, unsupported critical extensions, or broken timeline lineage:

```sql
VERIFY RESTORE FROM '/backups/mydb-2026-08-22.mnemic'
USING ARCHIVE 'file:///archives/mydb'
TO LATEST
```

Schedule full rehearsal as the actual RTO proof:

```sql
VERIFY RESTORE FROM '/backups/mydb-2026-08-22.mnemic'
USING ARCHIVE 'file:///archives/mydb'
TO LATEST REPLAY
```

Rehearsal copies the base into operation-scoped private staging beside the database (`.mnemic-pitr-rehearsal-*`, so size that volume for one more copy), performs bounded replay and complete validation, reports phase timings and how redo completed each replayed record ([result columns](MqlReference/maintenance.md#wal-archiving-and-point-in-time-recovery)), and removes its state without publishing a child timeline or pending restore. A real restore stages its copy and pending metadata the same way (`.mnemic-restore-staging-*`), and so does the open that validates the copy before promotion. Staging interrupted by a crash stays until the next rehearsal, restore, or engine open in that directory reclaims it. On a volume whose file locks cannot prove the owner dead, staging falls back to an `-unleased-` directory that nothing reclaims; remove one left by a crash by hand once its process has exited. Track measured RTO and workspace for small, normal, and larger-than-memory recovery sets. Metadata preflight alone is not a substitute for this drill.

## Restore during an incident

1. Stop application writes and preserve the source database, WAL, archive, and selected base backup. Do not delete the source's future history.
2. If the source host survived with a newer active WAL tail, close the engine and run `DefaultWalOfflineRecovery.CaptureTailAsync` first with `DryRun = true`, then with `DryRun = false`. It requires exclusive ownership and publishes only the last complete validated physical group through the normal fenced archive protocol. Restore never consumes an arbitrary local tail.
3. Run `VERIFY RESTORE` for the exact target and save the resolved timeline, LSN, CSN/time, required key IDs, input bytes, and workspace estimate.
4. Prefer `VERIFY RESTORE ... REPLAY` when the incident window permits. It proves the full data path without changing the live database.
5. Issue the real `RESTORE ... VERIFY USING ARCHIVE ... TO ...`. Use `ON TIMELINE '<guid>'` when siblings exist; never infer a branch from a human label.
6. Promote the validated pending sidecar with `REOPEN DATABASE [name]` from a server-level connection, which terminates the sessions connected to the database and reports `RestorePromoted` ([databases](MqlReference/databases.md#reopening-a-database)); an embedded engine is disposed and opened again. Verify application invariants and archive the new child timeline before resuming writes.

Every real PITR creates a child timeline. The parent and its abandoned future remain immutable and can support another recovery.

## Advance retention safely

Archive deletion is deliberately external and two-phase. Keep the database offline while applying a local-provider floor.

1. Construct `WalArchiveRetentionPlanOptions` with the exact database ID, archive URI, every retained protected backup manifest, every explicitly retained timeline, protection provider, and supported critical extension namespaces.
2. Call `DefaultWalArchiveRetention.PlanAsync`. Review `Blockers`, retained roots, per-timeline floors, and proposed safe prefix ranges. Persist the canonical bytes from `SerializePlan` and its `PlanSha256` in change-control evidence.
3. Call `ApplyAsync` with the same plan options, `WalControlPath = databasePath + ".wal"`, and `ExpectedPlanSha256`. Application revalidates every root and metadata generation, obtains archive/source ownership, publishes monotonic protected floors, and read-verifies the complete plan.
4. Delete nothing unless `DeletionAuthorized` is true. An external lifecycle job may then remove only the returned `SafePrefixRanges`. Keep timeline manifests and progress/floor metadata unless a reviewed plan explicitly proves an entire timeline obsolete.

Omitting a backup or timeline from the inventory declares it expired. A floor never retreats, and stale bytes below an advanced floor do not become valid recovery input merely because they still exist. A PITR-participant plugin may attach protected legal-hold or epoch state or veto a floor, but cannot move it beyond the closure the engine proved.

## Rotate archive keys

1. Provision and test the new key and durable nonce allocator before changing the active key ID.
2. Reopen the engine with a protector whose active descriptor names the new key and whose key provider can still open every historical key reported by preflight. New segments use the new key; existing timeline history remains readable with its recorded descriptors.
3. Create a new `REQUIRE PITR` base backup, force archive coverage, and run both preflight and full rehearsal.
4. Retain old keys until every retained base, segment, child timeline, metadata document, and recovery floor that references them is past an authorized retention floor. A successful write with the new key is not proof that an old key is safe to destroy.
5. Test missing-key alerts and recovery in a non-production drill. Key IDs may appear in the system row and plans; raw keys must never appear there.

## What the engine owns, and what an extension may add

The engine owns physical WAL ordering, sealing, checksums, immutable publication, provider capability proof, local writer ownership, exact targets, bounded replay, child timelines, staging atomicity, and monotonic retention floors.

A plugin that implements the cut-fence and PITR-participant contracts may narrow an eligible cut, attach protected critical extensions, provide its own archive and key transports, and veto a fork, restore, or retention publication. Without such a plugin none of this runs, and the archive format is the same either way.

See [maintenance](MqlReference/maintenance.md) for the statements.
