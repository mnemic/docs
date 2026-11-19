# Connecting to a Mnemic server

> **In this section** — connection-string schemes, plain TCP vs TLS, pinned-certificate validation, mTLS, **authentication credentials**, and the typed-client surface.

MQL is delivered to a Mnemic server over the TCP wire protocol. This page covers the **transport layer** — choosing TCP vs TLS, pinning a self-signed certificate, sending credentials, and the mTLS knobs. Once a connection is open, every MQL statement on the rest of this reference works the same way regardless of whether the bytes ride plain TCP or TLS.

## Plain TCP

```text
tcp://username:password@host:port/dbname
```

```csharp
await using var db = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcp://alice:p4ss@localhost:5150/mydb",
    configure: opts => { … });
```

Default port is 5150. The `dbname` segment selects the database via the `Connect` opcode after the socket is up. It is a URI path segment, so percent-encode a name that contains a space, `#`, `%`, or non-ASCII letters (`tcp://alice:p4ss@localhost:5150/My%20Shop` opens `My Shop`); the client decodes it, like the userinfo, before connecting. Credentials in the URL userinfo travel in the `Connect` frame's payload and are validated by the server's authentication gate before any MQL is processed.

Connecting to a database that does not exist creates it only for `sa`: `ServerAdmin` is never granted through a role, so it is the same capability `CREATE DATABASE` requires. Any other user's `Connect` is refused with `User / INSUFFICIENT_PRIVILEGE` ("Permission denied.") and nothing is created, so a database-scoped role cannot mint a database by naming it. A server whose master engine is read-only refuses the creation with its role error while still opening existing databases.

The standalone host binds `127.0.0.1` and provisions a persistent self-signed TLS certificate by default. Plaintext requires both `MNEMIC_TLS=off` and `MNEMIC_ACCEPT_PLAINTEXT_CREDENTIALS=true`. A non-loopback plaintext listener also requires the deliberately named `MNEMIC_UNSAFE_ALLOW_REMOTE_PLAINTEXT=true`; changing only the listen address cannot expose a local plaintext configuration.

### The database segment is optional — server-level connections

The trailing `/dbname` is optional. Connecting with an **empty** database segment (`tcp://user:pass@host:port/`, no name) opens an authenticated but **database-less** connection — the SSMS-style "browse the whole server" handshake. No database is bound, so:

- **Server-level statements work** — `LIST DATABASE`, `LIST PLUGINS`, `CREATE DATABASE`, `DROP DATABASE`, `REOPEN DATABASE`, and the [user / role administration](auth-admin.md) statements operate on the server itself and need no bound database; the master engine runs the administration statements.
- **Database-scoped statements are refused** — any `FROM … SELECT`, `INSERT`, DDL, etc. returns the error **"No database selected."** Issue a fresh `Connect` carrying a database name to bind one.

- **Master health is inspectable and resumable** — `FROM [Mnemic].[Health]` and `ALTER ENGINE RESUME '<domain>'` run against the master engine, for `sa` only ([health](health.md#a-fault-belongs-to-the-engine-that-owns-it)). A database's own faults are read and resumed on a connection bound to it.

The handshake reply reports schema level `0` (there is no bound engine to report a level for). Use this to enumerate databases before choosing one, the way a SQL Server connection at the *server* (not a specific database) lets you `LIST` first and `USE`/reconnect second. All successful `Ok` replies carry a non-negative schema level; a negative value is a protocol fault on both encoder and decoder, never an initialization sentinel.

### A database that cannot be opened

A `Connect` to a database whose files cannot be opened is answered with an error frame carrying the detector's classified pair, for example `StructuralCorruption / PAGE_CHECKSUM` for a page whose checksum does not verify, or `StructuralCorruption / WAL_DATABASE_MISMATCH` for a recovery log that belongs to another database; the connection stays open. The database is not left registered half-open: every later `Connect` or `REOPEN DATABASE` opens its files again and reports the same pair until the files are repaired, and `DROP DATABASE` removes it like any other. Only that database is affected; the fault is not recorded against the master engine, so other databases, `CREATE DATABASE`, `CREATE USER` and `/health/ready` are unaffected.

### Programmatic credentials

If you'd rather not put secrets in URLs, set `TcpTransport.Credentials` in the configure-transport callback. This overrides any URL userinfo:

```csharp
await using var db = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcp://localhost:5150/mydb",
    configure: opts => { … },
    configureTransport: t => {
        t.Credentials = new NetworkCredential("alice", "p4ss");
    });
```

## TLS — `tcps://` scheme

The `tcps://` scheme turns on TLS with default OS-trust-store validation — the right shape for **CA-issued certificates**:

```text
tcps://host:port/dbname
```

```csharp
await using var db = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://prod.example.com:5150/mydb",
    configure: opts => { … });
```

The handshake completes before any MQL frame is sent. There is no STARTTLS / opportunistic upgrade — a server is either TLS-on or plain. Connection-string scheme determines which client behaviour you get.

Clients accept only TLS 1.2 and TLS 1.3. Platform-selected defaults, older protocol flags, and unknown flags are rejected before opening a socket. The client handshake timeout must be positive and no greater than the runtime timer ceiling (approximately 49.7 days); the default is 10 seconds.

Server endpoints accept only TLS 1.2 and TLS 1.3. An explicit server protocol selection may choose either version or both; startup rejects platform-default selection and any older protocol instead of silently weakening the transport policy. The server applies the same positive, approximately 49.7-day maximum to its TLS handshake timeout and rejects invalid values before listening, so silent peers cannot retain connection slots indefinitely.

## TLS — certificate pinning (self-signed servers)

When the server runs auto-generated self-signed certificates, the cert isn't in the OS trust store. Operators get the active cert's **SHA-256 hash** from the server's startup log (or the typed `MnemicServer.CertificateSha256` property) and pin it on the client side:

```csharp
await using var db = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://localhost:5150/mydb",
    configure: opts => { … },
    configureTransport: t => {
        t.Tls = new ClientTlsOptions {
            PinnedServerCertificateSha256 = "AB12CD…64-hex-chars",
            TargetHost = "localhost"
        };
    });
```

The SHA-256 pin must contain exactly 64 hexadecimal digits. Colons and spaces may be used as separators; malformed or wrong-width pins are rejected before a socket is opened.

The server will reuse the same cert across restarts (it's stored in the master database), so the pin stays valid until the operator rotates the cert. Auto-generated certificates backdate their validity start for clock-skew tolerance without shortening the configured remaining lifetime. Operator-supplied certificates must be inside their validity window at server startup and rotation; expired or future-dated replacements are rejected before they are published. The built-in pin replaces CA-chain and hostname validation with exact certificate identity. It still rejects an expired or future-dated certificate, and a definitive revocation result, even when its SHA-256 pin matches. SHA-1 thumbprint pins are not accepted or persisted.

Validator precedence is explicit `ServerCertificateValidator` > SHA-256 pin > `AcceptAnyServerCertificate` > OS trust store.

Custom validator callbacks are part of connection-pool identity by instance. Reuse the same callback instance to share pooled connections. Different callbacks, and custom validation versus OS trust, never share a transport because the validator runs only during the original TLS handshake. The `configureTransport` callback itself runs exactly once per open; its captured TLS, credentials, and heartbeat settings are the same snapshot used for both pool identity and new-connection setup.

The client builds the server-certificate chain with online revocation checking by default. OS-trust validation rejects resulting chain errors, custom callbacks receive the chain state, and built-in pin validators reject a definitive revoked status while still permitting an explicitly pinned self-signed root. Set `ServerCertificateRevocationMode` to `Offline` when revocation data is operator-cached, or explicitly to `NoCheck` only for a PKI with no revocation mechanism. Connections using different revocation policies do not share a pooled transport.

## TLS — mutual auth (mTLS)

Server-side: set `ServerTlsOptions.ClientCertificateValidator` to a callback that validates the client cert. Non-null = mTLS required. The server checks client-certificate revocation online by default and reports chain failures to that callback. Set `ClientCertificateRevocationMode` to `Offline` when revocation data is operator-cached, or explicitly to `NoCheck` only for a private PKI with no revocation mechanism.

Client-side: set `ClientTlsOptions.ClientCertificate` to a cert with a private key.

```csharp
configureTransport: t => {
    t.Tls = new ClientTlsOptions {
        PinnedServerCertificateSha256 = "...",
        ClientCertificate = X509CertificateLoader.LoadPkcs12(
            File.ReadAllBytes("client.pfx"), password: null)
    };
}
```

## TLS — development "accept anything" knob

For local prototyping against a server whose cert you haven't pinned yet:

```csharp
t.Tls = new ClientTlsOptions {
    AcceptAnyServerCertificate = true
};
```

**Don't use this in production.** It's equivalent in MITM exposure to running plain TCP.

## Roles and per-database authorization

Every Mnemic user holds one or more **role grants** that determine which MQL statements they may execute. Built-in roles:

| Role | Can read | Can write (DML) | Can DDL | Can manage users |
|---|---|---|---|---|
| `Reader` | yes | no | no | no |
| `Writer` | yes | yes | no | no |
| `Admin` | yes | yes | yes | no |
| `Owner` | yes | yes | yes | yes |

`SUBSCRIBE` / `UNSUBSCRIBE` sit at the `Read` tier: subscriptions observe rows or plugin events and do not mutate stored data. Collection, document, and dependency subscriptions still pass the registered read gates before a callback is installed.

`BEGIN TRANSACTION`, `JOIN TRANSACTION`, `COMMIT`, and `ROLLBACK` also sit at the `Read` tier: transaction control opens and ends the session's own scope and touches no stored data, so every role, a `Reader` included, can run a multi-statement read in one snapshot or serializable transaction. Each statement inside the transaction still needs its own tier — a `Reader`'s `INSERT` in a transaction is refused with `INSUFFICIENT_PRIVILEGE` — so `COMMIT` publishes only writes the gate already admitted. Migration control (`BEGIN` / `COMMIT` / `ABORT MIGRATION`) stays at the `DDL` tier.

`LIST QUERIES` / `FROM [Mnemic].[ActiveQueries]` is at the `Read` tier but the result is filtered: a non-`UserAdmin` (and non-`sa`) caller sees only their own active queries. `UserAdmin` callers and `sa` see every connection's queries. The same gate determines who may target another user's query with `CANCEL QUERY`.

Grants are scoped to a database, or **server-wide** when the scope is null (effective on every database). A database-scoped grant is bound to the database itself, not its name: dropping the database revokes it, and a database created again under the name starts with no grants ([auth administration](auth-admin.md#grant)). The user's effective permissions for a given database are the union of every matching grant: the permission flags of their roles OR together.

`sa` is the built-in superuser — always authenticated by `RootPassword`, always authorised for every operation.

Authentication failures return a generic `"Authentication failed."` error with `User / AUTHENTICATION_FAILED` on the wire (no enumeration of unknown users vs wrong passwords). The handler runs a full PBKDF2 round on the no-such-user and disabled-user paths against a process-private dummy hash before returning the failure, so an attacker can't time the wire response to probe whether a candidate username exists. Authorisation failures return `"Permission denied."` with `User / INSUFFICIENT_PRIVILEGE`. A statement sent before Connect reports `AUTHENTICATION_REQUIRED`, and a database-scoped statement on a server-level connection reports `NO_DATABASE_SELECTED`.

Failed attempts are admitted before PBKDF2 through bounded token buckets for both the source address and the source-address + username pair. Defaults allow bursts of 32 per source and 8 per source/user, then replenish one token every 2 seconds. A verified login refunds its tokens; a rejected attempt returns the same generic authentication error and performs no password derivation. Configure the policy with the auth plugin's `AuthOptions.AuthenticationSourceAttemptBurst`, `AuthenticationIdentityAttemptBurst`, and `AuthenticationAttemptRefillPeriod`. The separate `MaxConcurrentAuthentications` ceiling (default 4) bounds PBKDF2 work that does pass admission, and `AuthenticationTimeout` (default 30 seconds) is the absolute deadline for a connection to authenticate.

Concurrent successful logins do not require client retries merely because another login persists its last-login timestamp. Identity lookup is synchronized with that update; password hashing remains concurrent within the configured work limit, and credentials are rechecked before a session is accepted.

## Connection pooling

Every `OpenRemoteAsync` call goes through a process-wide `TcpTransportPool.Default`. Two opens with matching `(host, port, db, user, password, TLS-config)` reuse the same authenticated transport — no fresh TCP setup, no fresh TLS handshake, no fresh PBKDF2 password verification on the second open. For a typical web-app workload (`MnemicDatabase` per request) this turns the 100–350 ms of per-call setup overhead (TCP + TLS + 600k-iteration PBKDF2 verification) into zero on the hot path.

Lease semantics are exclusive: each `MnemicDatabase` owns the underlying transport for its lifetime; on `DisposeAsync` the transport returns to the pool. From the moment its disposal begins, a remote client refuses every operation — a statement, a batch, a transaction's commit or rollback, a subscription, a typed query or page — with `ObjectDisposedException` before it sends anything, and a disposal that finds one of the client's operations still running closes the connection instead of returning it, so nothing a disposed client runs reaches the next lease. A session keeps state between statements only through the statements that begin a transaction or a migration, join a transaction, set a session setting or subscribe; it binds one database for its life and keeps no prepared statements. The client marks a transport "tainted" whenever an answer reports that its session holds any of that state, raw MQL and the typed transaction and subscription APIs alike, since the next caller must not inherit it; the server decides, so how a statement is written cannot hide it. Before deciding whether to dispose, `RemoteClient.DisposeAsync` issues a `ResetSession` round-trip, which rolls back any active transaction or migration, clears subscriptions, clears what `SET` left (`SET STATEMENT TIMEOUT` and every session parameter), and ends a transaction the session lost, which would otherwise refuse the next caller's statements with `TX_ABORTED`, so the session runs as a newly authenticated one does. Reset success returns the transport to the pool clean; reset failure or timeout drops it. Reset clears only the state that existed when it began. A newer concurrent session taint, request-ID exhaustion, or any protocol/read-loop fault keeps the transport ineligible for reuse.

An idle pooled transport whose session the server ended while it sat in the pool — the server closed the connection or terminated the session because its database was dropped or closed — is discarded at the next acquire, whether or not `HealthCheckOnAcquire` probes it, and at the next eviction tick whatever its age. The next `OpenRemoteAsync` therefore connects afresh and binds the database that then holds the name. With `HealthCheckOnAcquire` disabled, or inside `HealthCheckBypassWindow`, a transport whose termination notice is still in flight can still be leased; its first statement then fails with `SESSION_TERMINATED`. The probe closes that gap because a notice the server already sent arrives ahead of its `Pong` on the same connection.

Tuning lives on `TcpTransportPoolOptions`:

```csharp
public sealed class TcpTransportPoolOptions {
    public int MaxConnectionsPerKey { get; init; } = 16;
    public TimeSpan IdleTimeout { get; init; } = TimeSpan.FromMinutes(5);
    public TimeSpan AcquireTimeout { get; init; } = TimeSpan.FromSeconds(10);
    public bool HealthCheckOnAcquire { get; init; } = true;
    public TimeSpan HealthCheckBypassWindow { get; init; } = TimeSpan.FromMilliseconds(500);
    public TimeSpan EvictionInterval { get; init; } = TimeSpan.FromSeconds(60);
}
```

`HealthCheckOnAcquire` sends a reused transport a `Ping` before the open returns it and replaces one that does not answer within the probe's deadline, unless the transport received a frame from the server — an answer, a `Pong` or a push — less than `HealthCheckBypassWindow` ago: that frame already proves the connection alive, so a busy pool pays no round trip per open. A connection that dies without closing, or whose session termination notice has not arrived yet, inside the window is leased unprobed and fails its first statement. `HealthCheckBypassWindow = TimeSpan.Zero` probes every reused transport; a negative window is rejected.

An open that finds every connection to its key leased waits up to `AcquireTimeout` for one to be returned, or not at all when it is zero, and then throws `ConnectionPoolExhaustedException` (`Mnemic.Client.Shared`), which names the host, port, database, capacity and wait. Nothing reached the server, so it is neither a timeout of the server's nor an MQL error; an application reports it as connections it holds, not as a server that did not answer. `IdleTimeout` and `AcquireTimeout` accept zero for their documented disable or fail-fast behavior and otherwise must be positive. `AcquireTimeout` and `EvictionInterval` cannot exceed the runtime timer ceiling of approximately 49.7 days. `EvictionInterval` must be positive; use `IdleTimeout = TimeSpan.Zero` to disable idle eviction.

For tests or multi-tenant deployments that need an isolated pool, the overload `OpenRemoteAsync(connectionString, configure, configureTransport, pool)` accepts an explicit pool instance.

## Request options

An `ExecuteMql` request (opcode `0xA0`) carries, beside the statement's text and parameters, how its result is delivered: whether it reports its total count, whether it reports its [read sources](read-sources.md), and which [page](queries.md#paging-a-statements-result) of it to return. None of them changes what the statement computes, and the text is sent as written. Its payload is `[4 requestId][1 flags]`, then, with the page flag, `[4 offset][4 limit]`, then `[4 mqlLength][mql UTF-8][2 parameterCount][parameters…]`, every integer little-endian. The flags are `0x01` the total count, `0x02` the read sources and `0x04` a page, whose offset is 0 to 2147483647 and limit 1 to 2147483647, the domains of `SKIP` and `LIMIT`; any other bit, and a page value outside its domain, makes the frame malformed. Read sources and a page around a statement that reads no rows are refused with `FEATURE_NOT_SUPPORTED` before it runs. The count arrives in the terminal `ResultEnd` frame (`-1` when it was not asked for), the read sources in the result schema frame. The .NET clients take the same three as `MqlRequestOptions` on `IMnemicClient.ExecuteMqlAsync`, in process and over TCP alike:

```csharp
var page = await db.Client.ExecuteMqlAsync("FROM [App].[Orders] ORDER BY [Id]", null,
    new MqlRequestOptions { IncludeTotalCount = true, ReadSources = true, Page = new MqlResultPage(200, 100) });
```

The typed client's `MnemicDatabase.QueryPageAsync<T>(mql, skip, take)` sends its statement the same way, as written, with the page and the total count, so its page and its `TotalCount` are the statement's own result's.

## Session execution ownership

One connection — or one embedded session, which has an `MqlExecutor` of its own — is one session. An embedded database opens further sessions over its engine with `OpenSessionAsync` (on the `MnemicDatabase` or on its `InProcessClient`): statements on different sessions execute and commit concurrently, as they do on separate connections, each with its own transaction and settings, and disposing a session refuses its later statements, rolls back what it left open once its running statement returns, ends its subscriptions and leaves the engine open, while disposing the database that opened the engine ends its own session and every session still open, then closes the engine, which waits for any statement still running and rolls back what it left open ([an embedded database](../Clients/DotNet/connecting.md#an-embedded-database)); a disposal called from inside a statement of a session it would end, such as by a hook that statement dispatched, would wait for itself and is refused at once with `TRANSACTION_COMPLETION_INSIDE_OPERATION`. Top-level statements submitted concurrently on that same session are executed one at a time so transaction state, migration state, statement cancellation, snapshots, and `SET` parameters cannot cross-contaminate. A caller-cancelled statement that is still waiting for session ownership returns `Statement cancelled.` without entering the engine or changing session state.

The embedded executor's ownership window ends when the statement returns its `MqlResult`. Its streaming result has captured its query state, so an undrained result does not prevent that executor from accepting the next statement. The TCP server additionally serializes ordinary requests through complete row delivery. Use separate connections/sessions for independent parallel statement execution.

On an authenticated TCP connection, carrier controls remain readable during an ordinary request. `CancelRequest` uses opcode `0x62` and exactly four little-endian bytes naming the target `ExecuteMql` request ID. It can cancel active or queued work only on that connection; unknown, completed, or session-control targets are idempotent no-ops. It has no separate acknowledgement: the original request's terminal response proves settlement. Anonymous or malformed cancellation frames close the connection. See [cancellation](cancellation.md) for outcome rules.

The server ends a session when the database it is bound to is dropped, closed for maintenance or a restore, or shut down ([database management](databases.md)). It cancels the session's running statement with that reason, runs none of its queued requests, and sends one final unsolicited `SessionTerminated` frame (`0x75`) before it closes the connection. The frame carries the Error body without a request identifier: a message, class and code, `Cancellation / SESSION_TERMINATED`. The remote client fails every request that has not received its terminal response, and every later request on that connection, with exactly that `MqlException`, and the transport is never pooled again; open a new connection to continue. The session's change subscriptions end with it, and the client faults each subscription handle's completion with the same failure. A malformed `SessionTerminated` payload faults the connection like any other invalid push frame.

Whenever the server ends a connection — after that notice, an idle or liveness reap, or a protocol fault — it closes it without a reset: it stops sending behind its last frame, reads and discards whatever the client still sends until the client closes its side, resets the connection, or two seconds pass, and only then releases the socket. A request that crossed the end therefore never makes the client's TCP stack discard the notice before the client read it; nothing sent after the end is answered. Close your side once you read the notice or the end of the stream: the .NET client does, which lets the server release the connection at once.

A change subscription can also end while its session continues: when a committed drop removes the collection it watches or a relation its filter reads, the server sends an unsolicited `SubscriptionEnded` frame (`0x76`) carrying the subscription ID followed by the Error body without a request identifier, `User / COLLECTION_NOT_FOUND` with a message naming the dropped relation. The frame is the last one for that subscription, and the server then holds nothing for its ID ([subscriptions](subscriptions.md#a-committed-drop-ends-the-subscription)). Like every push, a malformed `SubscriptionEnded` payload faults the connection.

The ordered pending queue accepts at most 64 requests and 16 MiB of payload, independently of the active frame. Excess requests receive `ResourcePressure / REQUEST_QUEUE_FULL` before execution. That answer leaves at once and can arrive before the answers of requests queued ahead of it, but its effect is ordered: once every request accepted before it ran, an open explicit transaction becomes abort-only and an open migration rolls back, so a `COMMIT` pipelined behind a refused statement commits nothing. Every other request the session does not run — cancelled before it started, refused by authorization, or malformed — ends the open work the same way in its place, and so does a transaction the session loses without its own `COMMIT` or `ROLLBACK`: after an idle rollback, or once the creator of a joined transaction ended it, every statement is refused with `TX_ABORTED` until `ROLLBACK` ([transactions](transactions.md#a-failed-statement-aborts-the-transaction)). A refused or cancelled `BEGIN TRANSACTION` opens no transaction, and the statements queued behind it run in autocommit, so pipeline a transaction's statements only behind a `BEGIN` whose answer succeeded. A migration ends at its first failed or unrun statement and nothing resumes it: the statements queued behind that one run outside it, against the committed schema, where its DDL is refused but its DML runs in autocommit and persists, so send a migration's statements one at a time, each behind the answer of the one before ([migrations](migrations.md#a-failed-statement-ends-the-migration)). Heartbeat and cancel controls still run when this queue is full. A duplicate in-flight request ID closes the connection instead of assigning cancellation to an ambiguous owner. Socket health checks confirm actual EOF without consuming frame bytes; query and heartbeat traffic draining concurrently is not treated as disconnection. Connection shutdown joins writer cancellation callbacks before releasing their transport resources.

The .NET client keeps every connection within that budget: a request takes room in a per-connection window of 64 requests and 16 MiB of payload (`BinaryProtocol.MaxPendingRequests`, `BinaryProtocol.MaxPendingRequestBytes`) before it is written, waiting in arrival order for answers to free it, and gives the room back with its terminal answer, so any number of concurrent calls on one connection never meet `REQUEST_QUEUE_FULL`. `IMnemicClient.ExecuteBatchAsync` runs a list of statements as one transaction: it sends `BEGIN TRANSACTION` and waits for its answer, writes as many statements as the window holds with one flush, refills the window as answers free it, and sends `COMMIT` only once every statement succeeded, so a batch that fits the window costs three round trips. The first statement that fails stops the batch with `MqlBatchException`, which names its position and carries the answers before it and the failure — the server's error, or what the client raised for a statement it could not encode or send or whose connection failed before its answer; every statement still unanswered gets one `CancelRequest`, written with one flush, their answers are awaited for one cancellation grace, and the transaction is rolled back, so `COMMIT` is never sent and nothing commits — also when the failure was a read, which leaves a transaction committable ([transactions](transactions.md#a-failed-statement-aborts-the-transaction)). Cancelling the call stops the batch the same way and throws `OperationCanceledException`, unless its `COMMIT` already succeeded. A grace that runs out, or a `ROLLBACK` the client cannot send or whose answer never arrives, closes the connection, whose end rolls the session back. An embedded session runs a batch's statements one after another under the same contract. A client in another language keeps the same guarantees by keeping within the budget, pipelining only behind a `BEGIN` whose answer succeeded, reading the answers in order, and on the first failure cancelling the rest and rolling back.

Remote request identifiers are nonzero and unique for the lifetime of a TCP transport. A transport that has issued all `0xFFFFFFFF` identifiers fails the next request and becomes ineligible for pool reuse; reconnect to obtain a fresh identifier space. Request-ID exhaustion and protocol/read-loop faults are permanent transport taints; `ResetSession` cannot make such a transport pool-eligible. The client rejects a duplicate active identifier before writing, so concurrent single-response and streaming operations cannot claim the same response correlation. Responses for never-issued IDs and duplicate responses for already-completed IDs fault the connection; the client never silently preserves a desynchronized transport. A correlated server response with a truncated four-byte identifier faults the connection and fails pending statements rather than waiting indefinitely for a terminal frame. Error, result-schema, and terminal response bodies are fully validated before delivery; truncation, invalid UTF-8, unknown markers/classes, and trailing bytes also fault the connection instead of returning partial data or preserving it for pool reuse. Outbound frames are canonical too: invalid UTF-16 is never replacement-encoded into different query text, credentials, parameter names/values, or notification identities, and unknown `ExecuteMql` flag bits are rejected locally. Every `ExecuteMql` request includes its parameter count, including zero; missing or trailing fields are rejected before statement execution. Incoming parameter counts must fit the remaining bytes before allocation, and a temporal parameter's UTC ticks plus offset must describe a representable local timestamp. The parameter frame has explicit Int8 and Int16 tags; unsigned CLR widths are losslessly promoted to the next signed MQL width (or Decimal for a large `ulong`), and unsupported formattable values use invariant-culture strings. These are the same canonical values used by in-process execution. EOF is clean only between complete frames; residual bytes are a protocol fault. Both peers validate the signed 32-bit frame length as soon as its four-byte prefix is available. A non-positive or oversized length faults the connection without waiting for the opcode or body, so an invalid prefix cannot retain a connection slot by withholding the next byte. After frame bytes are advanced, a canceled flush or completed peer is an indeterminate delivery failure and permanently faults the writer, so retry cannot duplicate or reorder that frame. Response opcodes are request-class-specific: Connect/reset accepts only Ok/Error, while streamed MQL accepts only Error, schema, row, and terminal responses. Connect-Ok carries the established exact protocol identity; Reset-Ok keeps the fixed schema-level shape, and each request performs its narrower parse after correlation. Unknown or cross-class opcodes fault the connection, even when their payload bytes happen to have a valid shape for a different response. Streaming responses are ordered: queries send exactly one schema, then zero or more rows, then `ResultEnd`; schema-less statements terminate with `MqlOk`. `Error` is terminal from either active state. Rows/end before schema, duplicate schemas, and `MqlOk` after schema fault the connection before the invalid frame can alter the result. `MqlOk` carries an explicit `Ok` / `AffectedCount` result kind, an exact non-negative Int64 affected count, message, a mandatory transaction-id presence marker and a mandatory subscription-id presence marker; a `SUBSCRIBE` acknowledgement names its subscription in that field, never only in its message. Every terminal answer to a request — `MqlOk`, `ResultEnd` and `Error` — carries, right after its request identifier, one byte naming what the session holds once that request settled: `1` a transaction it began, abort-only or not, or one it lost — the server rolled an idle one back, or the creator of the transaction it joined ended it — which refuses its statements until `ROLLBACK`; `2` a transaction it joined, which only its creator ends; `4` a migration; `8` settings `SET` changed (`SET STATEMENT TIMEOUT`, a session parameter); `16` subscriptions the server has not ended; `0` nothing. At most one of `1`, `2` and `4` is set and no other bit is defined; any other byte faults the connection. A successful `Connect` or `ResetSession` leaves a session holding nothing. `ResultEnd` then carries one byte saying what its rows are, followed by the Int64 total count (`-1` when it was not requested): `0` a query's rows, an incomplete prefix a client discards when it cancels the request; `1` the rows of a statement that is not a query and ran to completion — a write's `RETURNING`, matched rows or none, or a `LIST` — which, like `MqlOk`, are that statement's outcome. No other value is defined, and any other byte faults the connection. A cancel, or a statement timeout, that reaches the server after a write completed — after it committed, or staged its changes in an open transaction — does not change its answer: the server sends that write's `MqlOk`, or all of its `RETURNING` rows and their `ResultEnd`, as it would have without the cancel, and answers `CANCELLED` only for a statement the cancel stopped. A `LIST` that reads its rows as it sends them, such as `LIST AUDIT`, or replays rows that outgrew memory, still stops at the cancel; it changed nothing, so nothing is lost. A client that cancelled a request reads on to its terminal answer and reports an `MqlOk`, or rows whose `ResultEnd` marks them as a statement's outcome, as the statement's result, so a committed write's `RETURNING` — `INSERT`, `UPDATE`, `REPLACE`, `DELETE`, `SEAL` or `UNSEAL` — is never reported cancelled; a client in another language follows the same rule to keep that guarantee. The .NET client exposes the last report as `RemoteClient.SessionState`. A negative count, invalid kind/marker, missing field, or trailing byte faults the connection. Consequently, zero-row DML remains `AffectedCount` and large mutations never wrap in remote clients. Request registration is atomic with transport shutdown. If EOF, a protocol fault, read-loop cancellation, or disposal wins first, new single, streaming, and health-check requests are rejected before writing. If registration wins, shutdown drains it with a deterministic failure; no request remains waiting after the response reader exits.

> **Authentication is mandatory.** Every Mnemic server requires credentials on Connect. The built-in `sa` superuser is bootstrapped at server start with the password supplied via the auth plugin's options, `MnemicServerOptions.Auth.RootPassword`; client connections include credentials either in the URL userinfo (`tcp://user:pass@host:port/db`) or via `TcpTransport.Credentials`.

Before authentication succeeds, each inbound frame is capped at 16 KiB. The normal 16 MiB MQL frame allowance is enabled only after `Connect` authenticates; a larger pre-authentication length prefix closes the connection before its body is buffered. Keep the combined Connect payload (database name, username, and password) within the handshake cap. The first successful authentication must also complete within the server's absolute authentication deadline (30 seconds by default). Pre-authentication `Ping` frames do not extend it, so opening a socket and sending heartbeats is not enough to retain a connection slot.

## Exact protocol version and diagnostics

`Connect` validates the remote wire contract before authentication or database binding. Both peers must use exactly protocol version `1`; every other value is rejected with the stable typed error code `PROTOCOL_VERSION_UNSUPPORTED`. Protocol v1 has one canonical layout and all current operations and fields are mandatory. There is no major/minor or capability negotiation.

A successful response includes the server semantic version and protocol version. The .NET remote client exposes both without another round trip:

```csharp
var remote = (RemoteClient)db.Client;
RemoteServerDiagnostics diagnostics = remote.Diagnostics!;

Console.WriteLine(diagnostics.ServerSemanticVersion);
Console.WriteLine(diagnostics.ProtocolVersion);
```

The client retires a transport if the server returns any other protocol version. Connect and Connect-Ok use the canonical protocol-v1 frame shapes; version 0, version 2, incomplete development handshakes, and superseded frame layouts are rejected.

## Heartbeat survives TLS

The application-level `Ping` / `Pong` heartbeat operates above the transport layer — it works identically over TLS. `HeartbeatInterval` and `KeepAliveTimeout` need no adjustment. A zero `HeartbeatInterval` disables client pings; negative intervals are rejected.

## Server-side configuration

The standalone executable accepts only named options:

```text
mnemic-server --data-dir ./data --spill-dir ./data/spill --port 5150 --diagnostics-port 5151 --listen-address 127.0.0.1
```

`--spill-dir` names the directory for the server's temporary files (sort, hash-join and index-build spill, backup and archive scratch) and defaults to `spill` inside the data directory, so a container keeps them on its data volume instead of a small `/tmp` ([configuration](../configuration.md#standalone-server-arguments)). Use `--help` or `--version` without initializing storage. Unknown, positional, duplicate, missing, and malformed arguments fail with exit code `2`. The root credential is mandatory: configure exactly one of `MNEMIC_ROOT_PASSWORD` or `MNEMIC_ROOT_PASSWORD_FILE`. Secret-file loading requires strict UTF-8 and strips terminal CR/LF characters used by mounted secret conventions. There is no built-in or development fallback password.

The host also serves process-only diagnostics on IPv4 loopback. The address is fixed and `--diagnostics-port` defaults to `5151`; it must differ from the MQL port:

| Path | Result |
|---|---|
| `/health/live` | `200` while the diagnostics listener is serving. |
| `/health/ready` | `200` while the MQL listener accepts connections and the master engine is not quarantined; otherwise `503`. A user database's quarantine does not change it. |
| `/metrics` | Prometheus text containing process-wide metrics without labels or database identities. |

`--check-ready` probes the readiness path without loading secrets, plugins, certificates, or storage. It exits `0` for ready and `1` for every non-ready, connection, timeout, or protocol result. It is the container image's built-in health check.

Authentication is mandatory; the always-on auth plugin owns its options (`AuthOptions`, in `Mnemic.Server.Auth`), passed as `MnemicServerOptions.Auth`, and its `RootPassword` for the built-in `sa` superuser is required:

```csharp
var server = new MnemicServer(
    dbm,
    new MnemicServerOptions {
        Port = 5150,
        Auth = new AuthOptions { RootPassword = builtFromEnvOrSecrets },
        Tls = new ServerTlsOptions()
    },
    loggerFactory);

await server.StartAsync(ct);

// Pin this SHA-256 value client-side (PinnedServerCertificateSha256).
Console.WriteLine($"TLS cert SHA-256: {server.CertificateSha256}");
```

When TLS is off, the server requires explicit acknowledgement that credentials transmit in cleartext:

```csharp
new MnemicServerOptions {
    Port = 5150,
    Auth = new AuthOptions { RootPassword = "..." },
    Tls = null,
    AcceptPlainTextCredentials = true   // loopback / dev only
}
```

`MnemicServerOptions` defaults to `IPAddress.Loopback` and automatic TLS. Setting a non-loopback `ListenAddress` while TLS is null also requires `AllowUnsafeRemotePlaintext = true`.

Restarting the server with a changed `RootPassword` rotates the `sa` credential. If the configured value is unchanged, an online `ALTER USER [sa] WITH PASSWORD ...` rotation survives restart. Non-`sa` users are created and managed through the MQL [user-administration surface](auth-admin.md) (`CREATE USER`, `ALTER USER`, `DROP USER`, `GRANT`, `REVOKE`, `LIST USERS / ROLES / GRANTS`).

### Server certificate lifecycle

With no further configuration, the server auto-generates a self-signed certificate on first start and persists it to the master database (`_master.mnemic`); subsequent starts reuse the same cert. The certificate's SHA-256 identity is logged at Information level so operators can wire pinning into clients — pin that value (`MnemicServer.CertificateSha256`).

To supply a CA-issued cert directly:

```csharp
Tls = new ServerTlsOptions {
    ServerCertificate = X509CertificateLoader.LoadPkcs12(
        File.ReadAllBytes("server.pfx"), password: pfxPassword)
}
```

`AutoGenerateCertificate = false` disables generation when neither an operator-supplied certificate nor a reusable server-owned certificate exists. It does not prevent reuse of a certificate already persisted in the master database.

## See also

- [Subscriptions](subscriptions.md) — TLS-aware push notifications.
