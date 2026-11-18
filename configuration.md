# Configuration

How the standalone server is configured, through its arguments and its environment, and how an application configures a server or an embedded engine it hosts.

## Standalone server arguments

Run `mnemic-server --help` for the executable's authoritative help.

| Argument | Default | Meaning |
|---|---:|---|
| `--data-dir <path>` | `./data` | Directory containing master and user databases. |
| `--spill-dir <path>` | `<data-dir>/spill` | Directory holding the process's temporary files: sort, hash-join and index-build spill, and backup and archive scratch. |
| `--log-dir <path>` | `<data-dir>/logs` | Directory holding the server's [log file](#standalone-server-log). |
| `--port <0-65535>` | `5150` | Database TCP port; `0` selects an ephemeral port. |
| `--diagnostics-port <1-65535>` | `5151` | Loopback HTTP diagnostics port. |
| `--listen-address <ip>` | `127.0.0.1` | Numeric IPv4 or IPv6 database-listener address. |
| `--check-ready` | — | Probe local `/health/ready`; exit `0` when ready and `1` otherwise. |
| `--help` | — | Print help and exit. |
| `--version` | — | Print semantic version and exit. |

Duplicate, positional, malformed, and unknown host arguments fail with exit code `2`. The database port and diagnostics port must differ.

The server process exits with `0` when it stopped on request, with `2` when it refused its arguments or configuration, a loaded plugin's settings included, and with `3` when it ended on a failure, at startup or while it ran or stopped; that failure is the last row of its [log](#standalone-server-log), at `crit`. `--check-ready` exits with `0` for a ready server and with `1` otherwise. Any other status is the runtime's or the operating system's, such as `137` for a process that was killed.

At start the server creates a missing data directory with the process's default permissions and then the spill directory, owner-only on Linux and macOS, and keeps its temporary files in one leased directory inside the spill directory, which a graceful shutdown deletes and the next start reclaims after a crash or kill. The default keeps spill on the data volume, because a container's `/tmp` is typically a small tmpfs: the shipped compose file mounts 64 MiB there, which a large `CREATE INDEX` or a spilling sort or join exceeds. Spill is encrypted under a key that never leaves the process, but it takes data-volume space while a statement runs. A data directory the server cannot create, or a spill directory it cannot create or write, fails startup with exit code `2`. Leftovers are reclaimed only from the directory the current run uses, so remove the old one by hand after moving `--spill-dir`. The spill directory needs a filesystem whose file locks every process sharing it can see, which rules out a network or bind mount without lock coordination: where the server cannot prove the lock on its leased directory, it logs a warning at start, and what a crash or kill leaves in the spill directory is never reclaimed.

## Standalone server environment

Exactly one root-password source is required.

| Variable | Contract |
|---|---|
| `MNEMIC_ROOT_PASSWORD` | Root password value. Mutually exclusive with the file source. |
| `MNEMIC_ROOT_PASSWORD_FILE` | UTF-8 secret file. Terminal CR/LF characters are removed. |
| `MNEMIC_TLS` | `auto` (default), `file`, or `off`. |
| `MNEMIC_TLS_CERT_FILE` | PFX path required by `MNEMIC_TLS=file`. |
| `MNEMIC_TLS_CERT_PASSWORD` | Optional PFX password. |
| `MNEMIC_ACCEPT_PLAINTEXT_CREDENTIALS` | Must be `true` for any `MNEMIC_TLS=off` listener. |
| `MNEMIC_UNSAFE_ALLOW_REMOTE_PLAINTEXT` | Must also be `true` for non-loopback plaintext. |
| `MNEMIC_ADMISSION_MAX_BYTES` | Positive Int64 process-wide query-admission budget in bytes. |
| `MNEMIC_BUFFER_POOL_BYTES` | Non-negative Int64 buffer-pool capacity in bytes of every database the server opens, master included. Unset, empty or `0` leave each pool automatic, so the databases share the process's automatic storage-cache budget ([embedded engine](#embedded-engine)); a positive value gives each database a pool of its own of that capacity, and a node budget of its own as large, never below 64 MiB. Another value fails startup with exit code `2`. |
| `MNEMIC_PLUGIN_DIR` | Plugin assembly directory; default is `plugins` beside the server binary. |
| `MNEMIC_VAMANA_PAGE_CACHE_BYTES` | Non-negative Int64 byte count of decoded adjacency pages the Vamana vector-index plugin caches for each database, shared by every vector index of it; unset, empty or `0` take an eighth of a buffer-pool capacity of the database's own, never below 4 MiB, and 32 MiB beside an automatic pool. Read by the plugin when the server loads it; another value fails startup with exit code `2`. |
| `LOG_LEVEL` | Lowest .NET logging level written to the [log](#standalone-server-log); unrecognized values use `Information`. |
| `MNEMIC_LOG_FILE_MAX_BYTES` | Size in bytes a log file is rolled at; default `16777216` (16 MiB). `0` keeps no log file. |
| `MNEMIC_LOG_FILE_COUNT` | Log files kept, the current one included, from `1` through `1024`; default `8`. |

Automatic TLS persists a self-signed RSA certificate in the master database. The startup log reports its SHA-256 hash for client pinning. File TLS loads an operator-owned PFX and does not copy or retain its private-key lifecycle.

When no admission byte limit is supplied, the server uses 25% of .NET's available-memory estimate, clamped from 256 MiB through 1 GiB. Master and every user database share this one process budget. A `ServerAdmin` may inspect and change the runtime budget through the admission MQL commands; changes are stored in master and restored at startup.

The server enables cold-page scrubbing every minute with a maximum of 512 pages per tick. Programmatic hosts can change `ColdPageScrubInterval` and `ColdPageScrubBatchPages`; embedded engines leave scrubbing opt-in.

The standalone server resolves no other settings; a plugin it loads reads its own, such as `MNEMIC_VAMANA_PAGE_CACHE_BYTES`, and a plugin setting it refuses fails startup with exit code `2` before any database opens. Master and every user database open with the default `EngineOptions` apart from that scrub cadence and the buffer-pool capacity `MNEMIC_BUFFER_POOL_BYTES` sets, plus the plugins the server loads, so engine options such as `EncryptionKey`, `WalArchiveUri`, `SnapshotIsolation`, and `MaxInMemoryStreamingState` keep their defaults, and unless that variable sets a capacity the open databases together cache page frames and decoded index nodes within the process's automatic storage-cache budget ([embedded engine](#embedded-engine)). Its `MnemicServerOptions` and `AuthOptions` carry only the listener, TLS, plaintext acknowledgements, and root password, so authorization audit, client certificates, statement and idle-session timeouts, and the connection limit also keep their defaults. Changing any of them takes a [programmatic server host](#programmatic-server-host) or an [embedded engine](#embedded-engine).

## Standalone server log

The server writes its log to standard output and appends the same rows to `mnemic-server.log` in the log directory, `logs` inside the data directory unless `--log-dir` names another. A container runtime deletes a container's log with the container, so the file on the data volume is the copy that survives recreating it. The file is appended to across server runs. A row that would grow it past `MNEMIC_LOG_FILE_MAX_BYTES` rolls it first: the file becomes `mnemic-server.1.log`, each older file moves one number up, and the oldest of the `MNEMIC_LOG_FILE_COUNT` files kept is deleted, so the defaults bound the log at 128 MiB of the data volume. `MNEMIC_LOG_FILE_MAX_BYTES=0` keeps no file and logs to standard output only. A log directory the server cannot create or write, and a log file setting that is not a valid number, fail startup with exit code `2`. `--check-ready`, `--help` and `--version` write no log.

Each row is one line:

```text
2026-01-15T10:38:56.629Z info ClientConnection [FK_VIOLATION] db=shop conn=4f5af58d user=sa: Request 8 answered with User error: ...
```

| Part | Meaning |
|---|---|
| Timestamp | UTC, with milliseconds. |
| Level | `trce`, `dbug`, `info`, `warn`, `fail` or `crit`. |
| Category | The component that wrote the row. |
| `[CODE]` | The stable error code of the outcome the row reports, the same code a client is answered with; absent on a row that reports none. |
| `db=` | The database the row concerns; absent on a row of the server itself. |
| `conn=` | The first eight hexadecimal digits of the connection's id, which its `Client connected` row carries in full; absent on a row no session caused. |
| `user=` | The session's authenticated user. |
| Message | The text after the colon. An exception follows it as its type, message and stack frames, separated by ` \| `. |

A row never spans lines: line breaks and other control characters are written as escapes, and an over-long message or stack is cut with a count of what was left out.

The level says whose failure a row reports. An error a client is answered with, such as a constraint violation or a syntax error, is one `info` row under its code without a stack, and the engine's own row for it is `dbug`. `warn` and above mean the server needs attention: `fail` is a failure of the server or an engine with its stack, and `crit` means an engine entered quarantine or the server process ended on a failure. Alert on `warn` and above, or on a `[CODE]`.

A failure that would otherwise show only when the server stops is logged when it happens, and the server stops on one it cannot serve past. A connection that ends on a process-terminal failure, such as memory exhaustion, writes one `fail` row under `CONNECTION_TERMINAL_FAILURE` with the failure, naming its session where the session itself ended on it. A server loop that ends on a failure, accepting connections or certificate renewal, writes one `fail` row under `SERVER_LOOP_STOPPED` that names the loop. A background maintenance worker of a database that ends on a failure writes one `fail` row under `BACKGROUND_WORKER_STOPPED` that names the worker, the database's [health](MqlReference/health.md) becomes `Degraded` under the same code, and the worker does not run again until the database reopens. After a connection's or a loop's row, and after a worker's row whose failure is process-terminal, a `warn` row says that the server is stopping on the failure; the server then stops as it does on request and exits with code `3`, so run it under a supervisor that restarts it, as the shipped compose file does with `restart: unless-stopped`. A worker's ordinary failure leaves the server running with that database `Degraded`.

Logging never delays a statement. When rows are logged faster than they can be written, rows below `warn` are dropped first, and a `LOG_ROWS_DROPPED` row counts what was lost. A log file that becomes unwritable while the server runs is reported once as `LOG_FILE_UNWRITABLE` on standard output, which keeps every row, and `LOG_FILE_RESUMED` reports when the file takes rows again.

## Standalone server runtime

The standalone server runs the .NET server garbage collector with background collection and the runtime's dynamic heap-count adaptation, which starts the process on one heap and adds heaps as load grows. The runtime's own environment variables override the shipped configuration and are read once at process start:

| Variable | Effect |
|---|---|
| `DOTNET_gcServer` | `0` selects the workstation collector. |
| `DOTNET_gcConcurrent` | `0` makes full collections blocking. |
| `DOTNET_GCDynamicAdaptationMode` | `0` disables dynamic adaptation, so the server keeps one heap per processor it may use. |
| `DOTNET_GCHeapCount` | Hexadecimal upper bound on the number of heaps. |
| `DOTNET_GCHeapHardLimitPercent` | Hexadecimal share of physical or container memory the heap may commit; inside a memory-limited container the default is 75%. |

An embedded engine runs under the collector its application selects.

## Programmatic server host

`MnemicServerOptions` is the supported server composition object. Authentication is always enabled: the always-on auth plugin owns its options and their defaults (`AuthOptions`, passed as `MnemicServerOptions.Auth`), and their `RootPassword` is mandatory. The secure defaults are loopback, port `5150`, automatic TLS, 1,000 maximum connections, the auth plugin's bounded authentication work, attempt limits, finite authentication deadline and opt-in audit, finite TLS-handshake, keepalive and idle-transaction timeouts, and the process-wide admission controller. The standalone host resolves `MNEMIC_ROOT_PASSWORD` or `MNEMIC_ROOT_PASSWORD_FILE` into `AuthOptions.RootPassword`. Statement and idle-session timeouts are off unless `DefaultStatementTimeout` or `IdleSessionTimeout` sets them. Startup validates combinations before accepting a connection. Per-database `EngineOptions` reach a programmatic server through the `DatabaseManager` options configurator, which receives each database's name and baseline options, master included, and returns the options that database opens with. No `MnemicServerOptions`, `DatabaseManagerOptions` or `EngineOptions` setting selects a spill directory: like an [embedded engine](#embedded-engine), a programmatic host keeps its temporary files under the platform temporary directory and moves them only through the platform's temporary-directory variables.

## Embedded engine

Use `MnemicDatabaseExtensions.OpenLocalAsync(path, configure)` for ordinary embedded use. The overload accepting `EngineOptions` exposes process-local storage configuration such as `WalArchiveUri`, scrub cadence, secure delete, and injected providers. The application owns directory permissions, backup destinations, shutdown, and key/provider lifecycle. Temporary files (sort, hash-join and index-build spill, backup and archive scratch) live in one leased directory per process under the platform temporary directory (`Path.GetTempPath()`: `TMPDIR` on Linux and macOS, `TMP` or `TEMP` on Windows). Those variables are the only way an embedded application moves them, since no `EngineOptions` setting selects a spill directory: point them at a volume sized for the largest sort or index build before the process starts. Page frames and decoded index nodes, which are cached in process memory beside the buffer pool, come from the process's automatic storage-cache budget unless the application sizes them: with `EngineOptions.BufferPoolBytes` and `IndexNodeCacheBytes` left at 0, every database of the process draws its page frames from one shared page-frame budget and the decoded nodes of all its indexes from one shared node budget, together half of the memory available to the process above 1 GiB and at least 256 MiB, split evenly (the default size of WiredTiger's cache; the available memory honours a container's memory limit), and a database takes frames and nodes only as its working set needs them. A database that warms after the others filled the page-frame budget still grows to its fair share of it, the budget divided among the active databases, those that asked for a frame within the last second, while a background pass takes cold, clean frames back from the databases above theirs. A positive `BufferPoolBytes` gives the database a pool of its own of that capacity and, unless `IndexNodeCacheBytes` sets one, a node budget of its own as large, never below 64 MiB. When `NodeCacheLoadCount` in `[Mnemic].[Metrics]` climbs with the write rate, the indexes' working set has outgrown the node budget: set `IndexNodeCacheBytes` to give the database a node budget of its own, or, beside a pool of the database's own, raise `BufferPoolBytes`, which raises both; setting `BufferPoolBytes` on a database whose pool is automatic replaces both shared budgets with ones of its own of that size. The optional Vamana vector-index plugin caches decoded adjacency pages in process memory too, up to `VamanaPluginOptions.PageCacheBytes` shared by every vector index of the database, passed to `new VamanaVectorPlugin(options)`: the pages of all its indexes compete for those bytes, so a hot index takes the share a cold one leaves unused and adding indexes does not add memory. Its default, 0, takes an eighth of a buffer-pool capacity of the database's own, never below 4 MiB, and 32 MiB beside an automatic pool; the standalone server, which constructs the plugin itself, takes the budget from [`MNEMIC_VAMANA_PAGE_CACHE_BYTES`](#standalone-server-environment). An approximate search whose distinct page working set could exceed the cache declines to the exact path, and the `mnemic.vector.ann.page_reads` and `page_cache_hits` counters show how often searches miss it.
