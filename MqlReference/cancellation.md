# Statement timeouts and cancellation

> **In this section** — `SET STATEMENT TIMEOUT`, `LIST QUERIES`, `CANCEL QUERY`, the `[Mnemic].[ActiveQueries]` virtual collection, and the per-connection socket-health monitor that cancels in-flight statements when a client connection drops.

An in-flight statement can be cancelled by its caller or stream consumer, its effective deadline, `CANCEL QUERY`, connection loss, graceful server shutdown, the terminal outcome of a distributed-transaction creator, or an atomic catalog publication that retires the generation it bound. These sources converge on one cooperative statement lifetime; the first attributed reason wins, so a statement already cancelled when a publication drains it reports that cancellation, not the publication's transient `CATALOG_PUBLICATION_RETRY` ([error codes](errors.md)).

The effective deadline can come from `OPTION(TIMEOUT ...)`, `SET STATEMENT TIMEOUT`, `MnemicServerOptions.DefaultStatementTimeout`, or `EngineOptions.DefaultStatementTimeout`. A TCP connection retains the server value as its fallback, so `SET STATEMENT TIMEOUT NONE` clears only the session override and does not disable host policy. Server startup rejects zero, negative, and over-`4294967294ms` defaults.

## `SET STATEMENT TIMEOUT`

Sets a per-connection deadline applied to every subsequent statement on this session. It overrides the TCP host or engine fallback until cleared.

```sql
SET STATEMENT TIMEOUT '30s';       -- 30 seconds
SET STATEMENT TIMEOUT '500ms';     -- 500 milliseconds
SET STATEMENT TIMEOUT '5m';        -- 5 minutes
SET STATEMENT TIMEOUT '2h';        -- 2 hours
SET STATEMENT TIMEOUT '1d';        -- 1 day
SET STATEMENT TIMEOUT 30;          -- bare integer = seconds
SET STATEMENT TIMEOUT NONE;        -- clear; host/engine default applies
```

Permission tier: `Read`. Setting a timeout only constrains the session — it never grants new capability.

A bare integer is seconds, not milliseconds as PostgreSQL's `statement_timeout` reads one: `SET STATEMENT TIMEOUT 1000` sets 1,000 seconds (16.67 minutes); write `'1000ms'` for one second. Durations must be positive and no greater than `4294967294ms` (about 49.7 days). Larger values are rejected at parse time rather than failing when the executor arms the deadline.

Inside an explicit transaction, cancelling an ordinary read that has started leaves the transaction usable when no transactional state was staged. A TCP request cancelled before the session took its statement — while it was still queued, or still being parsed and authorized — never runs, and since the server never ran it, it marks the transaction abort-only in its place whatever it was ([transactions](transactions.md#a-failed-statement-aborts-the-transaction)). A client cannot tell from its side whether its cancel reached a read before or after the session took it, so after cancelling a read inside a transaction it should expect `TX_ABORTED` and end the transaction with `ROLLBACK`. Cancelling DML marks the transaction abort-only because partial statement work has no independent undo scope: `ROLLBACK` is the clean exit, while `COMMIT` rolls back and reports the refusal. Cancellation inside a migration automatically rolls back the whole migration.

When the deadline fires the executor returns:

```text
Statement cancelled: timeout exceeded after 30s (elapsed 30.04s).
```

Durations in SET results, SHOW values, and cancellation messages always use an invariant decimal point (`1.5s`), regardless of the server's host culture.

## `OPTION (...)` — per-statement options

A top-level statement may end with one `OPTION (...)` list of comma-separated keys, in any order and each at most once: `TIMEOUT` sets its deadline and `MAXDOP` its parallelism. How the result is delivered — its total count, its [read sources](read-sources.md) and a [page](queries.md#paging-a-statements-result) of it — is no key: the request carries it beside the text. A plugin may add keys of its own; any other key is refused at parse time with a message naming the built-in keys. None of them changes the session.

A read the list decorates — a query, a `WITH` statement or a set operation, asked for its read sources or a page or not — streams and is cancelled exactly as the same read without it: over TCP its rows follow the schema frame as a query's rows, which a cancel ends with `CANCELLED` and a cancelled request discards, the deadline its `TIMEOUT` sets stays armed until the last row is read, and its `MAXDOP` covers every row it produces, the rows read after its execution returned included.

## `OPTION(TIMEOUT '<duration>')` — per-statement override

A trailing `OPTION(TIMEOUT '...')` suffix sets the deadline for a single execution. It overrides `SET STATEMENT TIMEOUT` and the engine default for this one statement; the session's previous setting is unchanged after the statement returns.

```sql
FROM [App].[Orders] WHERE [Status] = 'open' OPTION(TIMEOUT '5s');
FROM [Mnemic].[Statistics] SELECT * OPTION(TIMEOUT '500ms');
WITH [Recent] AS (FROM [App].[Orders]) FROM [Recent] OPTION(TIMEOUT '10s');
```

The duration grammar matches `SET STATEMENT TIMEOUT` exactly (`'30s'` / `'5m'` / `'2h'` / `'1d'` / `'500ms'` / bare positive integer seconds). A quoted duration requires its unit suffix (`'30'` is rejected; write bare `30` for seconds). `NONE` is **not** accepted — `OPTION` exists to set a deadline; clearing one is `SET STATEMENT TIMEOUT NONE`'s job. The same `4294967294ms` maximum applies.

**Scope.** The OPTION is a top-level trailing hint — it can only appear at the very end of a top-level statement, never inside a subquery, CTE body, or `EXISTS (...)`. The deadline covers the **entire** statement, including every CTE iteration, recursive expansion, and correlated-subquery probe. A query that wants different per-subquery deadlines must decompose into multiple statements.

**Errors at parse time:** zero / negative / over-maximum durations, unknown option keys (typos like `OPTION(TIMOUT '5s')` fail loudly rather than being silently ignored), and duplicate keys (`OPTION(TIMEOUT '5s', TIMEOUT '10s')`).

**Precedence (nearest scope wins):**

```text
OPTION(TIMEOUT ...) > SET STATEMENT TIMEOUT > MnemicServerOptions.DefaultStatementTimeout (TCP) > EngineOptions.DefaultStatementTimeout > unlimited
```

This is replacement precedence, not a minimum. A longer `OPTION(TIMEOUT ...)` can intentionally extend a shorter session or host fallback for that one statement.

When the OPTION-imposed deadline fires the error message is the same as the SET path.

## `OPTION(MAXDOP <n>)` — per-statement parallelism override

A trailing `OPTION(MAXDOP <n>)` suffix sets the intra-query degree of parallelism for a single execution, overriding the engine-wide `EngineOptions.MaxQueryParallelism` for that statement only.

```sql
FROM [App].[Events] WHERE [Region] = 'eu' SELECT * OPTION(MAXDOP 8);
FROM [App].[Sales] GROUP BY [Day] SELECT [Day], SUM([Total]) OPTION(MAXDOP 4);
FROM [App].[Audit] ORDER BY [At] SELECT * OPTION(MAXDOP 1);
```

`<n>` is a positive integer naming the maximum worker count for each eligible parallel stage this statement reaches (full scan + `WHERE`, `GROUP BY`, `ORDER BY` sort, the hash-join build, and the dedup-bearing set operations — `UNION` / `INTERSECT` / `EXCEPT`):

- `MAXDOP 1` forces **serial** execution for this statement even on an engine configured for parallelism; use it to cap a statement's CPU footprint on a busy box. `LIMIT` / `SKIP` scans without `ORDER BY` already stay serial so a parallel plan cannot change the selected subset, but MQL still does not guarantee their result order. An `EXISTS` body reads at most one row and runs serially too, so a correlated `EXISTS` evaluated per outer row never starts a parallel scan per row.
- `MAXDOP <n>` (n > 1) opts a single statement **into** parallelism on an otherwise-serial engine (default `MaxQueryParallelism = 1`).

The positive integer is accepted and remains observable exactly, including values past `Environment.ProcessorCount`. Execution treats it as a maximum hint: each operator uses at most 256 workers, and available work (page or partition count) can reduce the actual count further. This prevents extreme legal values from overflowing channel arithmetic or requesting unbounded task and worker-state allocations. The bound is per eligible stage, not a process-wide worker limit.

`OPTION(MAXDOP ...)` composes with `TIMEOUT` in one block (`OPTION(TIMEOUT '5s', MAXDOP 4)`); the same top-level-only scope and duplicate-key / unknown-key parse-time rejections apply. Zero / negative values (`MAXDOP -1` as `MAXDOP 0`) are rejected with `INVALID_VALUE` at parse time, and in a programmatically built statement by execution and the canonical printer.

## `[Mnemic].[ActiveQueries]`

Virtual system collection — each row is one in-flight statement on the server. Queryable with standard MQL:

```sql
FROM [Mnemic].[ActiveQueries];
FROM [Mnemic].[ActiveQueries] WHERE [Username] = 'alice';
FROM [Mnemic].[ActiveQueries] WHERE [DatabaseName] = 'shop' ORDER BY [ElapsedMs] DESC;
```

Columns: `QueryId`, `ConnectionId`, `Username`, `DatabaseName`, `Mql`, `StartedAtUtc`, `ElapsedMs`.

The `LIST QUERIES` MQL statement is shorthand for `FROM [Mnemic].[ActiveQueries]`. Note that `LIST QUERIES` (and any query against `[Mnemic].[ActiveQueries]`) sees its own row while it executes — matches Postgres `pg_stat_activity` behaviour.

Permission tier: `Read`. Without `UserAdmin` the operator-facing filter narrows to the caller's own queries; with `UserAdmin` the caller sees every connection's queries. Rows with null attribution are hidden from authenticated non-admin callers.

## `CANCEL QUERY '<guid>'`

Trips the named query's CTS. Idempotent — cancelling a finished or unknown id returns Ok.

```sql
CANCEL QUERY '8f3c7e6a-1d4b-4e2c-9a3f-1234567890ab';
```

Permission tier: `Read`. An authenticated non-admin can cancel only a query whose non-null username matches the caller. Cross-user and unattributed targets require `UserAdmin`. The error message on the cancelled statement reflects the caller:

```text
Statement cancelled by user 'sa'.
```

## Connection-loss cancellation

The `SocketHealthMonitor` runs alongside the keepalive watchdog on every connection. While the connection has a request queued or running, it polls on `SocketHealthPollInterval` and cancels the connection's active queries when the operating system reports peer closure or socket failure; an idle connection is not polled, since its pending read already ends at the peer's closure. The keepalive watchdog remains the complementary safety net for silent half-open links.

Operators can tune the poll interval to trade detection cadence against polling work during each request. The interval is not a detection SLA: kernel TCP state and scheduler delay determine when a failed peer becomes observable.

The error message on the cancelled statement:

```text
Statement cancelled: client connection lost.
```

## Cancellation reasons

Every cancelled statement carries an attribution (settable on the `ActiveQuery` registry entry) that maps to a specific error string:

| Reason | Set by | Error message |
|---|---|---|
| `Timeout` | deadline timer | `Statement cancelled: timeout exceeded after 30s (elapsed 30.04s).` |
| `ConnectionLost` | `SocketHealthMonitor` / connection teardown | `Statement cancelled: client connection lost.` |
| `UserCancel` | `CANCEL QUERY` | `Statement cancelled by user 'sa'.` |
| `ServerShutdown` | `MnemicServer.StopAsync` (graceful stop), or the database manager terminating a bound session as it shuts down | `Statement cancelled: server shutting down.` |
| `DatabaseDropped` | `DROP DATABASE` terminating the sessions bound to the dropped database | `Statement cancelled: database dropped.` |
| `DatabaseClosed` | A database close, such as the one `REOPEN DATABASE` runs to promote a staged restore, terminating the sessions bound to it | `Statement cancelled: database closed.` |
| _(none / generic)_ | caller's own `CancellationToken` | `Statement cancelled.` |

On a graceful `MnemicServer.StopAsync` every in-flight statement is cancelled with `ServerShutdown` **before** the connection drain begins, so a runaway query does not hold the shutdown open for the full drain window — matching PostgreSQL fast shutdown, which aborts the running query with `terminating connection due to administrator command`. A database drop or close likewise cancels the running statement of every session it terminates, with `DatabaseDropped` or `DatabaseClosed`, before it closes the engine, so the retirement does not wait for that statement to finish; the session then ends with `SESSION_TERMINATED` ([database management](databases.md)).

## Observability

Every aborted statement increments the `mnemic.statement.cancellations` counter on the `Mnemic.Engine` meter, tagged by `reason` (`timeout` / `connection_lost` / `user_cancel` / `server_shutdown` / `database_dropped` / `database_closed` / `none`). Chart it to see the cancellation rate without scraping logs: a climbing `timeout` rate can flag runaway work or an undersized deadline, `connection_lost` can flag client or network instability, and `user_cancel` tracks operator intervention.

## Tunables

```csharp
new EngineOptions {
    FilePath = path,
    DefaultStatementTimeout = TimeSpan.FromMinutes(2)
}

new MnemicServerOptions {
    Port = 5150,
    Auth = new AuthOptions { RootPassword = "..." },
    DefaultStatementTimeout = TimeSpan.FromMinutes(2),
    SocketHealthPollInterval = TimeSpan.FromMilliseconds(100)
}
```

## See also

- [System collections](system-collections.md) — the `[Mnemic].[ActiveQueries]` row layout.
- [Connecting](connecting.md#heartbeat-survives-tls) — the client's `HeartbeatInterval` pings, which keep an idle connection inside the server's `KeepAliveTimeout`.

## Detailed syntax and execution

Cancellation is cooperative. Engine waits and CPU loops observe the statement token at safe points, but socket notification, scheduler progress, and operator check cadence mean there is no fixed wall-clock cancellation guarantee. Streaming compound-array index preparation observes the same combined statement and consumer token as its row reader, including posting scans, spill work, and merge pulls. A cancelled consumer cannot start that deferred preparation, and an empty pointer source cannot turn observed cancellation into successful EOF. Indexed OR queries also observe statement cancellation while preparing their candidate union, before returning rows. Cancellation during a branch, including an empty branch, stops preparation without probing later branches. Single-array indexed `IN` queries also check while parsing members and combining their postings. Cancellation at a singleton or exact-list provider return, including an empty result, stops preparation before later probes or row reads. Quantified and compound-array literal lists also observe cancellation during member parsing, key preparation, sorting, and range construction. A later unprobeable member cannot hide cancellation as an ordinary index-plan decline, and chooser fallback cannot publish a source after cancellation. External index-build sorting checks every 4096 comparisons in resident and spill runs, then again before publishing the sorted result or run. Native comparison cancellation preserves the original token; other comparison failures retain their stronger causes. Ordered input checks each resident or spilled row pull, including the pull after the final row and successful spill-reader cleanup. A cancelled non-unique bulk build stops leaf construction and reclaims unpublished pages. Hash-join builds carry their statement token through spill writes and run compaction, checking between keys and payloads and before run publication. Cancelled build work releases its temporary files and admission before the statement settles. Hash-join probes check before resident or spill work and between payloads, including skipped duplicate-key buckets and full-run scratch authentication. Cancelled scratch verification is never reused as a complete integrity check. Buffered `ORDER BY` preparation, including primary-key tie-break extraction and index-provided leading-key runs, checks every 256 rows and at phase boundaries. Comparison work checks every 4096 comparisons. Small sorts also observe cancellation before returning results; pooled scratch is released before the statement settles. Cancellation during comparison retains the same `Cancellation` / `CANCELLED` outcome and source attribution in serial, parallel, and spill-run sorting. A comparison failure carrying a stronger cause still reaches that cause's failure policy. Terminal keyed aggregation sorting uses the same 4096-comparison cadence and phase checks. Its resident and spilled output also checks each row pull, including the pull after the final row, before reporting successful EOF. Native HAVING checks between groups. Aggregate output remains cancellable when it is empty or the next pull would finish the result, including a LIMIT boundary; its retained buffers and spill files are released before cancellation settles. Buffered window ordering checks each row-context lookup, each row's sort-key evaluation and each native comparison, with phase checks for short inputs and 256-row checks during index preparation and partition walks. Its scalar-subquery path combines statement and consumer cancellation before preparing windows, and releases decoded-state admission before reporting cancellation. Buffered computed SELECT output remains cancellable after source and window evaluation. It checks every 256 output rows and before publishing the result, including short and empty inputs; cancelled staged output is discarded.

Cancellable archive metadata protection and verification also observe the token independently of key-provider cooperation. Cancellation still joins a running key-provider call and closes any returned key handle. It cannot accept cancelled metadata validation as complete or replace an authentication failure already detected by comparison.

Cancellation is never a successful end-of-stream signal. Streaming operators, including temporal `LAST PER` bucketing, fail the stream when the token is observed; a caller must discard rows already received from that statement rather than treating a partial prefix as a complete query result. This remains true when cancellation arrives while the stream is disposing its inner source at end-of-input: terminal cleanup completes, then the row request reports cancellation instead of successful EOF. Terminal cleanup completes before the stream reports cancellation. A stronger failure such as structural corruption or process-memory exhaustion is not flattened into successful EOF or an ordinary cancellation result.

For a remote `ExecuteMql` call, caller cancellation sends an authenticated `CancelRequest` carrier control targeting that request on the same connection. The client continues reading until the original request's terminal response confirms completion. A cancelled statement reports class `Cancellation`, code `CANCELLED`; the .NET caller receives `OperationCanceledException` when its own token caused cancellation. Partial query rows are discarded. A successful `MqlOk` preserves the completed operation's actual outcome, including a durable commit that won the cancellation race, and so do rows whose terminal `ResultEnd` marks them as a completed statement's outcome — the `RETURNING` of a write that committed, which the server sends whole whatever cancel or deadline arrives after the write — which the .NET caller receives as the statement's result instead of `OperationCanceledException` ([connecting](connecting.md#session-execution-ownership)). Stronger classified errors remain errors.

The .NET client allows five seconds for cancellation settlement. If no terminal response arrives, or cancellation delivery fails, it closes the transport and reports an I/O failure because the operation's outcome could not be confirmed. Shutdown interrupts and joins pending frame writes before completing their pipes. A confirmed cancellation keeps the connection usable; it does not release response ownership early or make a dirty transaction eligible for pooling.

Delivered single-response session controls such as `Connect` and session reset still close the transport on caller cancellation: they can change authentication or session binding, and `CancelRequest` targets only `ExecuteMql`. Cancellation before request delivery does not require a close.

For a lazy `RowsAsync` result, the statement deadline, linked caller token, active-query registration and cancellation attribution remain owned until enumeration completes or the enumerator is disposed, except for a write's `RETURNING`, below; slow-query and statement-stats timing cover the enumeration of every lazy result, a write's `RETURNING` included. A cancel that arrives after `ExecuteAsync` returned but before (or during) enumeration therefore still fails that statement. Consumer-side cancellation passed through `WithCancellation(...)` is linked into the same stream. Callers must either drain the stream or dispose its enumerator. If a result will not be read, obtain the enumerator and dispose it immediately; abandoning an execution handle without either action deliberately leaves the statement active so it can still be cancelled and observed.

The TCP server owns this cleanup when it consumes the result. A failure while publishing the schema releases the unconsumed stream without reading its rows. A failure while sending rows also releases the stream, preserving both the transmission error and any cleanup error. This applies to queries and row-producing commands; a successful end-of-result frame is sent only after cleanup succeeds.

Capturing mutation targets or buffered JOIN rows preserves a source failure or cancellation if cleanup also fails. An optimistic-concurrency cardinality check has the same failure contract even though it reads at most two rows. Adding query stages does not grant a failed source another automatic cleanup retry; explicit disposal can retry the retained cursor.

A DML `RETURNING` stream is the exception. Its rows are the detached pre/post-image snapshots of a write that already ran to completion — committed, or staged in an open transaction or migration — so its statement ends when `ExecuteAsync` returns: the statement deadline and active-query registration retire then, and a later cancel of the caller's token, `CANCEL QUERY` or deadline does not stop the replay. Only consumer cancellation through `WithCancellation(...)` does, and the replay does not block schema migration, so a committed write is never reported cancelled, in process or over TCP. The .NET client's raw-MQL helpers (`MnemicDatabase.QueryAsync`, `QueryBinaryAsync`, `QueryPageAsync`) read a statement's rows with no consumer token of their own, so a caller who cancels after a write returned still receives its `RETURNING` rows whole, embedded as over TCP. Consumer cancellation arriving during successful replay cleanup still fails the final row request after that cleanup finishes.

Distributed-transaction gate waits, nested statement work, DML, structural changes, and maintenance statements share the same cancellation lifetime as ordinary queries. A statement nested in a running statement, such as a trigger body or incremental materialized-view maintenance, is part of that statement: it runs under the statement's deadline and cancellation, `LIST QUERIES` and `[Mnemic].[ActiveQueries]` show only the statement that set it off, and `CANCEL QUERY` of that statement cancels it. Once cancellation begins, the executor still joins owned child work and completes rollback or cleanup under independent authority.

Structural DDL and `COMMIT MIGRATION` also observe the token while waiting to close and drain statement/direct-storage admission before the durable commit marker. Cancellation at that point reopens the acquired fence and rolls back. Once the marker is durable, catalog/registry publication and cleanup complete under independent authority; cancellation cannot turn a committed outcome into an indeterminate one.

Target discovery and materialising a blocking operator's input are not cancellation boundaries. Mutation and result-delivery work continues to check the statement token until it reaches a durable outcome or completes.


<a id="syntax-11-cancellation"></a>

### Cancellation

```sql
SET STATEMENT TIMEOUT '5s';            -- per-connection default
SET STATEMENT TIMEOUT '500ms';         -- millisecond precision
SET STATEMENT TIMEOUT '5m';            -- minutes
SET STATEMENT TIMEOUT '2h';            -- hours
SET STATEMENT TIMEOUT '1d';            -- days
SET STATEMENT TIMEOUT 30;              -- bare integer = seconds
SET STATEMENT TIMEOUT NONE;            -- clear the session override
SET slow_query_threshold = '250ms';    -- session param override (= / TO; NONE; DEFAULT to clear)
SET slow_query_threshold = 0;          -- observe every statement
SHOW slow_query_threshold;             -- effective value: Name/Value/Source/Description
SHOW ALL;                              -- every settable session parameter
LIST QUERIES;                          -- active queries on this server
CANCEL QUERY '<id>';                   -- abort by id
FROM [App].[X] WHERE ... SELECT * OPTION(TIMEOUT '5s');  -- per-statement deadline
FROM [App].[X] WHERE ... SELECT * OPTION(MAXDOP 8);      -- per-statement parallelism
FROM [App].[X] SELECT * OPTION(TIMEOUT '5s', MAXDOP 4);  -- both, one block
```

Duration suffixes: `ms`, `s`, `m`, `h`, `d`. Quoted duration strings require one of those suffixes; bare integers are treated as seconds. `NONE` clears the session override; the host or engine fallback (if configured) still applies. Statement deadlines must not exceed `4294967294ms` (about 49.7 days); the parser rejects larger `SET STATEMENT TIMEOUT` and `OPTION(TIMEOUT ...)` values. For TCP sessions, `MnemicServerOptions.DefaultStatementTimeout` is that process-level fallback on every connection; `SET STATEMENT TIMEOUT NONE` restores it. Server startup rejects the same zero, negative, and over-`4294967294ms` values before opening the listener. SET results, SHOW duration values, and cancellation errors render fractional durations with an invariant decimal point (`1.5s`) on every host culture.

**Session parameters (`SET <name> = <value>` / `SHOW`).** A general PostgreSQL-shaped surface for overriding *registered* parameters for the current connection only. Use `=` or `TO`; `= DEFAULT` clears the override; `SHOW <name>` returns the effective value (`Source` = `session` or `engine-default`); `SHOW ALL` lists every settable parameter. Inaugural parameter: `slow_query_threshold` — overrides the current engine default for the session (non-negative duration / non-negative integer seconds / `NONE`). Zero observes every statement; `NONE` disables observation. If the session enables observation while the engine default is off, enabled structured-log and elapsed-meter sinks still receive events. The engine-wide `[Mnemic].[SlowQueries]` ring receives them when host options or `ALTER ENGINE SET SLOWQUERY` enabled history; a session override never allocates it. Unknown names and invalid values are rejected. Distinct from the dedicated `SET STATEMENT TIMEOUT` verb, which is not part of this registry.

`OPTION(TIMEOUT '...')` is a trailing hint suffix that sets the deadline for a single execution only — it overrides `SET STATEMENT TIMEOUT` for this one statement and does not mutate the session. Top-level only (never inside a subquery / CTE body); the deadline covers every CTE iteration and subquery the statement contains. `NONE` is not accepted here — `OPTION` exists to set a deadline, clearing one is `SET STATEMENT TIMEOUT NONE`'s job. The list's built-in keys are `TIMEOUT` and `MAXDOP`; unknown option keys (typos like `OPTION(TIMOUT '5s')`) are rejected at parse time rather than silently ignored. Zero, negative, and over-`4294967294ms` durations are also rejected. A read an OPTION list decorates streams and is cancelled exactly as the same read without it, and its `MAXDOP` covers every row it produces, streamed rows included.

Deadline precedence is nearest-scope replacement: `OPTION(TIMEOUT ...)` > `SET STATEMENT TIMEOUT` > TCP host fallback > engine fallback > unlimited. It is not a minimum; a longer statement override can intentionally extend a shorter session or host fallback for that execution.

`OPTION(MAXDOP <n>)` overrides `EngineOptions.MaxQueryParallelism` (the engine-wide intra-query parallelism cap) for one statement: `MAXDOP 1` forces serial execution even on a parallel engine; `MAXDOP n` (n > 1) opts a single statement into parallelism on an otherwise-serial engine. `<n>` is a positive integer (zero / negative rejected at parse time) and remains observable exactly. Operators treat it as a maximum hint and use at most 256 workers per eligible stage, with available work able to reduce the count further. It is not a process-wide worker limit. It composes with `TIMEOUT` in one `OPTION(...)` block and is top-level only, same as `TIMEOUT`. Affects the parallel scan + `WHERE`, `GROUP BY`, `ORDER BY` sort, hash-join build, and the dedup-bearing set operations (`UNION` / `INTERSECT` / `EXCEPT`).

The virtual `[Mnemic].[ActiveQueries]` collection lists every in-flight statement (`UserAdmin` sees all; other users see only rows attributed to their own username, and null-attribution rows fail closed). An authenticated non-admin can `CANCEL QUERY` only when the target has the same non-null username; cross-user and unattributed targets require `UserAdmin`. Unknown or already-finished ids succeed without effect. Cancellation never appears as ordinary end-of-stream: streaming operators (including `LAST PER`) fail the stream, and any rows already received from the cancelled statement are an incomplete prefix that must be discarded. If cancellation arrives while a stream is disposing its inner source at end-of-input, terminal cleanup completes and the row request reports cancellation rather than successful EOF. An ordinary transient EOF-cleanup failure is retried once. Corruption or cancellation reported by cleanup remains a stream failure even when a later explicit disposal succeeds. Direct or wrapped memory exhaustion on either attempt is not retried into successful EOF or hidden inside an ordinary cleanup aggregate: the exact exception escapes and the enumerator retains its failed owner for an explicit disposal retry. This rule holds through external plugin-row capture, structural-corruption supervision, migration-drain registration, and migration-overlay binding. Explicit row consumers use the same precedence: operation memory exhaustion remains exact after ordinary cleanup, while cleanup memory exhaustion supersedes an ordinary operation failure without aggregate wrapping. Set-operation branch cleanup additionally continues independent sibling owners before surfacing that exact terminal memory failure; only the failed owner is left for an explicit retry. Remote `ExecuteMql` caller cancellation sends an authenticated, same-connection `CancelRequest` control and drains the original response to its terminal outcome. Confirmed cancellation uses class `Cancellation`, code `CANCELLED`; it preserves the connection and discards incomplete query rows. A successful `MqlOk`, including a durable commit, wins a cancellation race, as do a completed write's `RETURNING` rows, which the server sends whole whatever cancel arrives after the write. A stronger error retains its class. The .NET client closes the transport and reports an unconfirmed-outcome I/O failure if cancellation delivery fails or settlement exceeds five seconds. Connect/reset retain their post-delivery cancellation close rule because their session changes are outside request cancellation. Before-delivery cancellation does not require a close. Ordinary TCP requests execute in order through row delivery while heartbeat/cancel controls remain readable. Pending requests are bounded by 64 frames and 16 MiB; excess requests fail before execution with `ResourcePressure / REQUEST_QUEUE_FULL`, and inside an explicit transaction make it abort-only once the requests queued before them ran. Socket health checks confirm actual EOF without consuming frame bytes; concurrent request traffic does not imply disconnection. Connection shutdown joins writer cancellation callbacks before releasing their transport resources. The published row-result wrapper checks cancellation before and after every source move, so plugin/custom sources cannot opt out by ignoring their supplied token; cancellation still retires the exact row owner through bounded cleanup. For lazy `RowsAsync` other than a write's `RETURNING`, the linked token/deadline, active-query registration and cancellation attribution remain alive until enumeration completes or is disposed; slow-query and statement-stats timing last until then for every lazy result, a write's `RETURNING` included. Cancellation after `ExecuteAsync` but before the first `MoveNextAsync`, and consumer `WithCancellation(...)`, therefore still fail the stream. Detached DML `RETURNING` replay is a completed write's outcome: its statement leaves active-query observation and its deadline when `ExecuteAsync` returns, only consumer cancellation stops the replay, and it does not block schema migration. Dispose abandoned enumerators. The TCP server owns that disposal even when schema publication fails before the first row. Query and non-query row results share failure-preserving cleanup; successful end-of-result is sent only after the stream has been released. Inside an explicit transaction, cancelling an ordinary read that has started leaves the transaction usable when it staged no transaction state, while a TCP request cancelled before the session took its statement, still queued or still being parsed and authorized, marks it abort-only. Cancelling DML marks the transaction abort-only: use `ROLLBACK`; `COMMIT` rolls it back and reports the refusal. Cancellation inside a migration automatically rolls back the whole migration. The statement token also covers DML scans/conflict probes, DDL waits and rewrite loops, materialized-view seeding, BACKUP, ANALYZE, DEFRAGMENT, SHRINK, and persisted admission changes. Cancellation never interrupts compensating rollback/cleanup; the engine restores the pre-statement state or fails closed. Pre-marker structural publication waits also observe that token while closing statement and direct-storage admission. Cancellation reopens any acquired fence and rolls the unpublished transaction back; after the marker, publication and cleanup finish under independent authority. Target discovery does not end cancellation: UPDATE, DELETE, and `CANCEL VERSION` recheck after target resolution and before writes; enum rewrites recheck after target materialisation; materialized-view seeds check each write. Scalar/hash/vectorized/parallel aggregation and its keyed spill sorter retain the same token through input, run publication, compaction, merge, framing, and checksum verification. Keyed run sorting checks every 4096 comparisons and at phase boundaries; its output checks each row pull and before successful EOF. Quantified and compound-array literal preparation retains cancellation through parsing, key sorting, range construction, and chooser fallback. A declined or empty index plan cannot hide cancellation before result publication. External index-build input also checks every resumed row pull in resident and spilled paths, and successful reader cleanup checks cancellation before EOF. Buffered `ORDER BY` also checks during key extraction, run detection, and permutation preparation, with checks at phase boundaries for small sorts. Row and metadata permutations finish together before another cancellation check. Cancellation-only comparison failures retain the canonical `Cancellation` / `CANCELLED` outcome and operator attribution across buffered and spill-run sorting. Buffered windows also retain the token through context preparation and cached ordering, checking every native comparison. Deferred scalar-subquery windows combine statement and consumer cancellation before preparing their orderings. Buffered computed SELECT construction also checks every 256 output rows and before publishing even a short or empty result.

## Usage pitfalls


<a id="pitfall-cancellation-remains-live-after-target-or-input-materialisation"></a>

### Cancellation remains live after target or input materialisation

Do not treat target resolution or a blocking operator's input snapshot as a point of no return. UPDATE, DELETE, `CANCEL VERSION`, enum rewrites, and materialized-view seeding keep checking the statement token before writes; aggregation and keyed spill sorting keep checking through CPU-only grouping, run merge, and authenticated replay. A cancelled statement may therefore fail after its scan completed but before mutation or result delivery. Discard any rows already received; compensating rollback continues independently. Nested EXPLAIN/trigger/query helpers inherit the parent cancellation flow even when sibling helpers queue concurrently; one completed child's private cancellation source cannot detach the next helper from caller cancellation. A helper started without being awaited remains structurally owned: the parent waits for every registered child and descendant before resuming built-in transaction/session work. Nested `EXPLAIN ANALYZE` stage collection is flow-local as well, so a child cannot replace or clear the parent's stage sink.

<a id="pitfall-structural-publication-cancellation-stops-only-before-durability"></a>

### Structural publication cancellation stops only before durability

Structural DDL and `COMMIT MIGRATION` may briefly wait for statement and direct storage readers to leave before publishing metadata. That pre-marker wait is bounded and observes statement cancellation; cancellation reopens the fence and rolls back. Once the commit marker is durable, publication/cleanup continues without caller cancellation so storage and the reported outcome cannot diverge.
