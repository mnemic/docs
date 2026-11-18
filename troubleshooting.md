# Troubleshooting

What a failure means and what to do about it, from a server that will not start to a database that reports corruption, and how to ask for support.

## Server exits with configuration code 2

Run `mnemic-server --help`. The common causes are a missing/empty root password, both password sources being configured, an unreadable secret file, an invalid TLS mode or certificate, duplicate arguments, equal data/diagnostics ports, or an invalid admission byte count. Configuration errors are written to stderr before any listener starts.

## Server exits with failure code 3

The server ended on a failure at startup or while it ran or stopped. The failure is the last row of the [server log](configuration.md#standalone-server-log), at `crit`, with its stack. Where the server stopped itself on the failure of a connection, of one of its loops or of a background worker, a `fail` row under `CONNECTION_TERMINAL_FAILURE`, `SERVER_LOOP_STOPPED` or `BACKGROUND_WORKER_STOPPED` shows where and when the failure happened, and a `warn` row after it that the server is stopping on it. A server that keeps exiting this way on memory exhaustion needs more memory or less concurrent work, not only its restart. Any other non-zero status is the runtime's or the operating system's, such as `137` for a container killed at its memory limit.

## TLS connection fails

- `tcps://` uses OS trust by default. Confirm the certificate chain, hostname, validity window, and TLS 1.2/1.3 support.
- For automatic self-signed TLS, pin the exact SHA-256 hash logged by the server.
- A changed automatic-certificate hash indicates a different master database or certificate rotation; verify it out of band before replacing the pin.
- `--insecure` and `AcceptAnyServerCertificate` are diagnosis-only controls.
- A `tcp://` client cannot talk to a TLS listener, and `tcps://` cannot talk to plaintext.

## Authentication fails

Confirm URL-encoding of userinfo, database scope, username, password source, role grants, and whether throttling is active after repeated failures. Connect to the server-level URL ending in `/` for `LIST DATABASES` or user/role admin; database-scoped statements require a database segment.

## Readiness is false

Liveness can remain healthy while readiness returns `503`. Readiness is false during listener shutdown or while the master database is quarantined; a quarantined user database affects that database alone ([health](MqlReference/health.md#a-fault-belongs-to-the-engine-that-owns-it)). Inspect server logs, `/metrics`, and authenticated `[Mnemic].[Health]`. Do not restart in a loop without preserving the first failure evidence.

## Writes return `WAL_FLUSH_FAILED`

An asynchronous WAL flush failed after a write was accepted for durability. The engine quarantines mutations and keeps safe reads available. Stop writes, preserve the storage error and files, correct the underlying disk/permission fault, then reopen the database so recovery can verify the durable prefix: on a server, `REOPEN DATABASE [name]` reopens a user database, and a fault of the master engine, which withdraws readiness, needs a server restart, since `REOPEN DATABASE` cannot name it ([health](MqlReference/health.md#a-fault-belongs-to-the-engine-that-owns-it)). Do not resume mutations merely because the storage device appears writable again.

## `VERIFY DATABASE` reports corruption

Take the database out of service, preserve the database/WAL/logs, and restore a verified backup or perform point-in-time recovery. The verifier continues its scan and publishes failures through `[Mnemic].[Health]`; its count is evidence, not an instruction to edit the reported pages.

## Query spills or is rejected

Inspect admission metrics and `[Mnemic].[Capabilities]`. The server shares one bounded budget across all databases. Reduce concurrency or query state, add an appropriate index, paginate results, or adjust the admission budget with the ServerAdmin command. Increasing the budget cannot exceed actual process and container memory safely.

## Restore is rejected

Keep the backup and `<backup>.manifest.json` together. A mismatch, truncation, extra bytes, wrong page size, stale origin WAL, invalid archive lineage, target outside proven coverage, or insufficient workspace fails closed. Restore to a fresh path or stage through the verified restore statement; never reuse the origin `.wal` beside a copied backup.

## Permission or read-only filesystem failure

The data directory, backup destination, archive root, and spill directory (`--spill-dir`, by default `spill` inside the data directory) must be writable by the server identity; a data directory the server cannot create, or a spill directory it cannot create or write, fails startup with exit code `2`; an existing data directory it cannot write fails when the master database opens. A startup warning that the spill directory's filesystem showed no working file lock means temporary files a crash leaves there are never reclaimed; move `--spill-dir` to a filesystem with working locks ([configuration](configuration.md#standalone-server-arguments)). The container root filesystem may be read-only when `/data` and `/tmp` are writable. Check UID/GID, mount flags, free space, inode availability, quotas, and parent-directory permissions.

## Requesting support

Ask for support at [support@mnemic.io](mailto:support@mnemic.io). Include:

- the exact Mnemic version and the package, container or artifact digest;
- the .NET runtime, operating system, architecture, filesystem and deployment mode;
- the statement or API call, what you expected, what happened, and the stable error code;
- redacted logs, with the output of `/health/ready`, `/metrics` and `[Mnemic].[Health]`;
- whether the problem reproduces on a fresh database, and whether `VERIFY DATABASE` and backup verification succeed.

Never send credentials, connection strings that hold secrets, certificates with private keys, production database files, backups, WAL archives or unredacted customer data. Report a vulnerability to the security address instead ([reporting a vulnerability](security.md#reporting-a-vulnerability)). During an incident, first follow the preservation and recovery sequence in [operations](operations.md#incident-sequence).
