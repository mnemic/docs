# Operations

Running a Mnemic server day to day: starting and stopping it, watching its health, checking its integrity, backing it up, recovering it, and what to do in an incident.

## Deployment baseline

Run the server with a dedicated identity and a dedicated writable data volume. Keep the executable/container filesystem read-only. Limit access to the data, backup, archive, certificate, and secret paths to that identity. Put the data and WAL on storage whose durability guarantees match the workload.

The database listener binds loopback by default. Expose another address only behind an explicit network policy. Keep TLS enabled. The diagnostics listener always binds loopback and exposes no database names or user-controlled labels.

## Start and stop

Supply exactly one root-password source, then start the host. Wait for readiness before routing traffic. Send Ctrl+C or `SIGTERM` for shutdown; the host stops accepting traffic, drains work, checkpoints, flushes, and then exits. Configure an orchestrator grace period long enough for that path; the included container uses 30 seconds.

Avoid hard kill during routine operations. Crash recovery makes an unclean stop safe, but it is an incident path rather than the normal lifecycle.

## Health and metrics

The loopback diagnostics listener serves:

| Endpoint | Meaning |
|---|---|
| `/health/live` | The diagnostics process is alive. |
| `/health/ready` | The main listener accepts connections and the master engine is outside quarantine. A user database's quarantine is not a readiness fault; read it from that database's `[Mnemic].[Health]`. |
| `/metrics` | Prometheus text metrics without database, user, statement, or path labels. |

Readiness returns HTTP `503` when false. The container health check runs `mnemic-server --check-ready --diagnostics-port 5151`. The MQL `[Mnemic].[Health]`, `[Mnemic].[Capabilities]`, active-query, session, archive, and statement-statistics system collections provide authenticated detail.

Alert on readiness loss, quarantined health, WAL/archive failures, admission rejections, sustained spill pressure, authentication throttling, backup failure, and growing archive lag. Establish workload-specific latency and space alerts from a healthy baseline.

## Integrity checks

Run this online and after unusual storage or recovery events:

```sql
VERIFY DATABASE
```

It checks every allocated page and returns `Status`, `PagesChecked`, `CorruptionCount`, UTC start/completion timestamps, and elapsed microseconds. Corruptions are also published through `[Mnemic].[Health]`. The statement is cancellable and does not stop after the first corrupt page.

The server also scrubs 512 cold pages per minute by default. Scrubbing is a detection mechanism, not a replacement for verified backups and restore drills.

## Backups

Create an online, fsynced backup and integrity manifest:

```sql
BACKUP TO '/backups/app.mnemic'
```

For a backup that must be a valid PITR base:

```sql
BACKUP TO '/backups/app.mnemic' REQUIRE PITR
```

Keep backup output outside the database and archive directories. Copy completed backup plus manifest to independent storage. Periodically restore into a fresh path, open it, run representative reads, and run `VERIFY DATABASE`.

Ordinary restore stages and verifies the backup, then activates it on reopen:

```sql
RESTORE FROM '/backups/app.mnemic' VERIFY
```

Promote the verified sidecar by reopening the database after a successful restore statement: on a server, `REOPEN DATABASE [app]` from a server-level connection closes it cleanly, terminating the sessions connected to it, and opens it again ([database management](MqlReference/databases.md#reopening-a-database)); an embedded engine is disposed and opened again. Never place an origin `.wal` beside a restored snapshot. See [Maintenance](MqlReference/maintenance.md) for the exact contract.

## Point-in-time recovery

Configure an absolute `file:` archive URI through `EngineOptions.WalArchiveUri`, monitor durable archive progress, take a `REQUIRE PITR` base, and rehearse:

```sql
VERIFY RESTORE FROM '/backups/app.mnemic'
USING ARCHIVE 'file:///archives/app'
TO LATEST REPLAY
```

The complete sequence, retention floors, timelines, restore points, offline tail capture, and key rotation are documented in the [PITR operator guide](wal-archiving-pitr.md).

## Space maintenance

`DEFRAGMENT` compacts logical dead space while queries and writes continue. Follow it with `SHRINK` to release free tail pages to the filesystem. Use `DEFRAGMENT INCREMENTAL ... BUDGET n` for bounded maintenance ticks. When an unclean shutdown has left pages allocated that nothing owns, `RECLAIM ORPHAN PAGES` requests a repair that runs on the next open, such as the one `REOPEN DATABASE` runs. All three are covered in [Maintenance](MqlReference/maintenance.md).

## Incident sequence

1. Stop routing new work when readiness withdraws.
2. Preserve logs, the database file, WAL, archive state, and health/metrics output before changing files.
3. Do not hand-edit, truncate, copy over, or delete recovery files.
4. Attempt a normal reopen so verified recovery can complete; on a server, `REOPEN DATABASE [name]` reopens one user database while the others keep serving, and the master engine, which `REOPEN DATABASE` cannot name, reopens only when the server restarts.
5. Run `VERIFY DATABASE` after recovery if the engine opens.
6. Restore from a verified backup or perform PITR when integrity remains in doubt; retain the original evidence for diagnosis.
7. Record the exact Mnemic version, OS, filesystem, storage fault, and last successful backup/restore drill when requesting support.
