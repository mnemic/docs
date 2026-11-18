# User and role administration

> **In this section** — the MQL surface for managing users, role grants, and the built-in superuser. Everything here runs over the standard `ExecuteMql` opcode — no privileged client paths.

Authentication is mandatory on every Mnemic server (see [Connecting](connecting.md)). Credentials, roles, grants, role permissions, and audit evidence live in hidden engine-owned master-database catalogs; userland MQL clients never reach them directly. The statements on this page are how operators read and mutate that state.

Every statement on this page runs on a server-level connection, one that names no database (`tcp://host:port/`), as well as on a connection bound to a database. Either way it acts on the master database's user directory; on a server-level connection the master engine runs it.

> **Authorisation.** Every statement on this page requires `MqlPermission.UserAdmin`. Statements without an `ON DATABASE` scope (`CREATE / ALTER / DROP USER`, `CREATE / ALTER / DROP ROLE`, `LIST USERS`, `LIST ROLES`, `LIST AUDIT`, server-wide `GRANT` / `REVOKE`, per-user `LIST GRANTS` and `LIST GRANTS` without scope) require server-wide UserAdmin — `sa` or a server-wide grant of a role with UserAdmin, such as `Owner`. Statements with an `ON DATABASE foo` scope (`GRANT … ON DATABASE foo`, `REVOKE … ON DATABASE foo`, `LIST GRANTS ON DATABASE foo`) require UserAdmin on that specific database — `sa`, a server-wide UserAdmin grant, or a UserAdmin grant on `foo`, such as its `Owner`.

> **Engine role.** Authorization does not override role admission. `LIST` forms are non-mutating and remain available in `ReadOnly` and `Draining`. Every form that changes users, roles, or grants commits independently and requires the engine it runs on to be `Writable` — the bound database's engine, or the master engine on a server-level connection; an open transaction does not make those changes admissible while draining.

> **Database scope.** `ON DATABASE` takes a database name either bare, when it is a plain identifier (`ON DATABASE myshop`), or in brackets, which any name allows (`ON DATABASE [my-shop]`). Statements print it in brackets.

## Users

### `CREATE USER`

```sql
CREATE USER [alice] WITH PASSWORD 'p4ss'
CREATE USER [alice] WITH PASSWORD @password
```

Inserts a fresh user record with the hashed password (PBKDF2-SHA256, 16-byte salt). The user is enabled by default. The reserved name `sa` is rejected.

**Password as a parameter.** `WITH PASSWORD` takes a string literal or a [parameter](parameters.md) that binds a `STRING`. A parameter keeps the credential out of the statement's text altogether, so no audit record, log or diagnostic that holds the text can hold it; the statement prints, and is audited, as `WITH PASSWORD @password`. A parameter the caller did not bind is `UNKNOWN_PARAMETER`, and one that binds anything else, `NULL` included, is `INVALID_VALUE`; neither refusal quotes the value. The same holds for `ALTER USER … WITH PASSWORD`.

**Username case.** Usernames are stored lower-cased — `CREATE USER [Alice]` and `CREATE USER [alice]` refer to the same account, and `LIST USERS` returns the lower-cased form. The folded storage backs a unique secondary index on `[Mnemic.Plugin.Auth.Users].Username` that closes the concurrent-create race (two parallel `CREATE USER [bob]` calls no longer produce two rows). Same fold applies to `ALTER USER`, `DROP USER`, and `GRANT`/`REVOKE` targets.

### `ALTER USER`

```sql
ALTER USER [alice] WITH PASSWORD 'newp';
ALTER USER [alice] WITH PASSWORD @password;
ALTER USER [alice] ENABLE;
ALTER USER [alice] DISABLE;
```

`WITH PASSWORD` re-hashes and replaces the credential. `ENABLE` / `DISABLE` flip the `IsEnabled` flag — `Connect` against a disabled user fails with the same `"Authentication failed."` response a wrong-password attempt would produce, so the disabled-vs-unknown distinction does not leak. Re-`ENABLE` to restore login.

The `sa` superuser cannot be disabled. Its password **can** be rotated online — `ALTER USER [sa] WITH PASSWORD '…'` — but only by `sa` itself: a server-wide `Owner` (who otherwise clears the UserAdmin gate) is rejected with `"Permission denied."`, because `Owner` sits below `sa` and rotating `sa`'s credential would be a privilege escalation. An online rotation survives server restarts: the configured `AuthOptions.RootPassword` (`MnemicServerOptions.Auth`) is reconciled declaratively and only re-applied to `sa` when its value *changed* since the last boot (which doubles as the break-glass recovery path if the online-rotated password is lost).

### `DROP USER`

```sql
DROP USER [alice]
```

Removes the user and every grant they hold. The `sa` superuser is not droppable.

### `LIST USERS`

```sql
LIST USERS
```

Streams every user. Result columns: `Username`, `IsEnabled`, `CreatedAt`, `LastLoginAt` (nullable). `LastLoginAt` is stamped on successful authentication (at most once per minute per user — dormancy triage doesn't need sub-minute precision); failed attempts never stamp it, so a brute-force probe can't make a dormant account look active.

## Roles

Two flavours of role exist:

- **Built-in roles** — `Reader`, `Writer`, `Admin`, `Owner`. Hierarchical: Writer ≥ Reader, Admin ≥ Writer, Owner ≥ Admin. Defined by Mnemic and unkillable. See [Connecting → Roles and per-database authorization](connecting.md#roles-and-per-database-authorization) for the canonical permission matrix.
- **Custom roles** — operator-defined via `CREATE ROLE`. Per-permission semantics: each of the five flags (`READ`, `WRITE`, `DDL`, `USERADMIN`, `SOFT_VETO_BYPASS`) is independent. Granting a custom role with `WRITE: TRUE` does **not** imply `READ: TRUE`; set both flags when both are wanted.

Multiple grants compose additively: a user holding both `Reader` and a custom `WriteOnly` role for the same database gets read + write on that database (the bitmaps OR together). `ALTER ROLE` changes a custom role's flags in place.

### `CREATE ROLE`

```sql
CREATE ROLE [Auditor] WITH (READ: TRUE, WRITE: FALSE, DDL: FALSE, USERADMIN: FALSE, SOFT_VETO_BYPASS: FALSE);
CREATE ROLE [Reporter] WITH (READ: TRUE);        -- omitted flags = FALSE
CREATE ROLE [Inert];                             -- WITH clause optional, every flag = FALSE
CREATE ROLE [BreakGlass] WITH (SOFT_VETO_BYPASS: TRUE); -- bypass-only, no other privileges
```

The four built-in role names (`Reader`, `Writer`, `Admin`, `Owner`) are reserved — the executor rejects them. Duplicate role names (case-insensitive) are rejected. Permission keys are case-insensitive; duplicate keys (`READ: TRUE, READ: FALSE`) are a parse error so a typo doesn't silently overwrite the prior value.

`USERADMIN: TRUE` lets holders run user / role administration statements within the granted scope — the same gate that `Owner` clears.

`SOFT_VETO_BYPASS: TRUE` is an additive grant that suppresses `VetoPriority.Soft` fragments from plugin `IWriteGuard` / `IReadGate` dispatch for sessions carrying the bit. `Normal` and `Critical` fragments are not suppressed for anyone — only the advisory layer is overridable. The built-in `Owner` role already carries the bit automatically; custom roles must opt in explicitly. The grant can stand alone (a break-glass operator whose only privilege is overriding advisory vetoes) or compose with any other flag.

### `ALTER ROLE`

```sql
ALTER ROLE [Auditor] WITH (WRITE: TRUE);                 -- add write, keep the rest
ALTER ROLE [Auditor] WITH (READ: FALSE, DDL: TRUE);      -- change two flags
```

Changes the flags of a custom role that the statement names; every flag it does not name keeps its value. The list names at least one flag, each once, with the keys `CREATE ROLE` takes. Built-in roles keep their fixed permissions and are rejected with `RESERVED_NAME`; an unknown role fails with `UNKNOWN_OBJECT`. Sessions holding the role feel the change at their next statement, as they feel a `GRANT` or `REVOKE`. The role keeps its name, identity and grants.

### `DROP ROLE`

```sql
DROP ROLE [Auditor]
```

Removes the custom role, its permission row, and every grant referencing it (mirrors `DROP USER`'s grant-cleanup pattern, so a dropped role doesn't leave dangling assignments). Built-in role names are rejected.

## Grants

Both built-in and custom roles can be granted. Custom roles must be created via `CREATE ROLE` first.

### `GRANT`

```sql
GRANT [Reader] TO [alice];                      -- server-wide
GRANT [Writer] TO [alice] ON DATABASE myshop;   -- per-database
GRANT [Writer] TO [alice] ON DATABASE [my-shop]; -- a name that is no plain identifier
```

Grants are idempotent — re-granting an existing assignment changes nothing. `GRANT` reports the grants it adds: `AFFECTED 1` for a new assignment, `AFFECTED 0` when the user already holds the role in that scope. Server-wide grants apply to every database; per-database grants apply only to the named database. The user's effective permission for a given database is the union of every matching grant: the permission flags of their roles OR together.

A per-database grant is bound to the database itself, not to its name. `ON DATABASE` must name a database that exists: a name no database holds fails with `UNKNOWN_OBJECT`, so a grant never waits for a database to be created under that name. `DROP DATABASE` revokes every grant on the database together with the drop, also across a crash or restart, and a database created again under the same name is a new database that starts with no grants — grant again after re-creating it. A `GRANT` whose database is dropped while the statement runs fails with `UNKNOWN_OBJECT` and grants nothing; it never applies to a database that takes the name meanwhile.

### `REVOKE`

```sql
REVOKE [Writer] FROM [alice] ON DATABASE myshop;
REVOKE [Reader] FROM [alice];                   -- server-wide
```

`REVOKE` removes the grant of exactly the scope it names — a server-wide `REVOKE` leaves a per-database grant of the same role in place — and reports the grants it removes: `AFFECTED 1`, or `AFFECTED 0` when the user holds no such grant in that scope. Revoking from an unknown user fails with `UNKNOWN_OBJECT`, as granting to one does, and so does revoking on a database that does not exist.

Grant changes (`GRANT`, `REVOKE`, `ALTER ROLE`, `DROP USER`, `DROP ROLE`, and the grants `DROP DATABASE` revokes) take effect on already-connected sessions at their **next statement** — no reconnect is required because privileges resolve before each statement. `ALTER USER … DISABLE` follows the opposite convention: it blocks new logins but does not terminate existing sessions. The process-lifetime invalidation generation cannot wrap and alias an old session snapshot: at its terminal `INT64` value, live sessions conservatively reload grants (and their derived permission bitmap) on every statement.

### `LIST ROLES`

```sql
LIST ROLES
```

Streams every role row. Result columns: `Name`, `Description` (nullable; a built-in role's names the statements its permission tier adds, a custom role has none), `IsBuiltIn`, and one `BOOL` per permission flag — `Read`, `Write`, `Ddl`, `UserAdmin`, `SoftVetoBypass` — saying what the role permits, for built-in and custom roles alike. Custom roles created via `CREATE ROLE` appear with `IsBuiltIn = false`.

### `LIST AUDIT`

```sql
LIST AUDIT
```

Streams every persisted authorisation-audit row through `RowsAsync`; the server does not first accumulate the retained log in memory. Result columns: `Timestamp`, `Username`, `Statement`, `Decision` (`Allow` / `Deny`), `Reason`, `TargetDatabase` (the only nullable column; null = server-wide).

Before scanning, `LIST AUDIT` waits for every audit event already accepted by the server's bounded writer channel to be processed. A successful statement is therefore visible to an immediately following `LIST AUDIT` unless persistence failed through the configured, observable audit-failure policy. `DropWrite` only requires enqueue before the protected operation continues; `FailOperation` and `Shutdown` wait for durable audit persistence first. The server persists the events queued together as one batch, in acceptance order and in one master transaction, so concurrent statements under a strict mode share one durable write, and each is answered only once its whole batch is durable. When an event cannot be admitted or persisted, `Shutdown` also quarantines every engine the server hosts with `AUDIT_FAILSTOP` — the master and every tenant database, including one opened while the master still carries the fault — so no database keeps running audited work without its trail ([health](health.md)). The accepted/processed barrier rebases at `INT64` maximum only at quiescence; with live backlog it rejects the new event through `AuditFailureMode` instead of wrapping and allowing an early audit read. Persisted audit `Sequence` maximum is likewise terminal: it may be exported and verified after reopen, but no later event can wrap negative or reuse the identity. Credentials are always masked as `'***'` in the stored statement, in parse-error server logs, and in every statement text the engine prints for diagnostics: `[Mnemic].[ActiveQueries]`, `[Mnemic].[SlowQueries]` and the slow-query log, and `[Mnemic].[StatementStats]` fingerprints. The mask follows the statement's shape rather than its literal, so it also covers a credential the parser rejects — unquoted, bracketed, followed by more tokens, after a misspelled `PASSWORD` or a missing `WITH`, or inside MQL written as the string-literal text of [`COMPLETE` or `VALIDATE`](editor-assistance.md), which those statements refuse — and comments between `PASSWORD` and the value, or after it, cannot place a cleartext password in any of them. A password bound as a parameter is no text to mask: the parameter stays as written. A user statement that carries a credential but does not parse keeps its `SYNTAX_ERROR` code with a withheld message instead of the parser's diagnostic, which could quote the credential.

Audit rows are only written when the server is started with the auth plugin's `AuthOptions.AuditMode` (`MnemicServerOptions.Auth`) ≠ `Off`. `DenialsOnly` records every `"Permission denied."` event; `AllowsAndDenials` adds every successful authorisation for a full query-by-actor history. Every enabled mode also requires `AuthOptions.AuditAnchor`: an absolute append-only anchor-file path, an independently provisioned HMAC key of at least 32 bytes, and a stable non-empty deployment `BindingId`. Startup authenticates that external high-water state before accepting queries and rejects a missing, modified, mismatched, or database-ahead/behind committed tail. Keep the anchor and key outside the database trust boundary; colocating either with the database defeats tail-rollback detection. The anchor retains one credential-redacted recovery copy per audit event and is not shortened by database `AuditRetention`; protect and rotate that external audit archive under the operator's evidence-retention policy. Database retention is bounded by `AuthOptions.AuditRetention` (default 90 days, not a claim of compliance with any particular retention regime); a background loop the auth plugin runs prunes rows older than the threshold every `AuthOptions.AuditPruneInterval` (default 1 hour).

The audit surface lives in the hidden master-DB collection `[Mnemic.Plugin.Auth.AuthorizationAudit]` — userland MQL only reaches it through `LIST AUDIT`; direct `FROM [Mnemic.Plugin.Auth.AuthorizationAudit]` queries are filtered by the engine-internal hidden-collection guard.

### `LIST GRANTS`

```sql
LIST GRANTS;                         -- every grant on the server
LIST GRANTS [alice];                 -- everything alice holds
LIST GRANTS ON DATABASE myshop;      -- everyone with access to myshop
```

Without a scope, `LIST GRANTS` streams every grant of every user, server-wide and per database.

The `[user]` form streams every grant the named user holds, each row naming the user as the directory stores it (lower-cased), whatever spelling the statement used; it returns no rows for a user that does not exist.

The `ON DATABASE name` form streams every grant whose scope applies to the named database — i.e. the grants on the database that holds `name` plus every server-wide grant; for a name no database holds, only the server-wide grants. This is the operator's view of "who can do what on this database".

Result columns (every form): `Username`, `RoleName`, `DatabaseName` (nullable; null = server-wide), `GrantedAt`. `DatabaseName` is the database's current name, read from the server's database registry. Rows come in no guaranteed order.

## Errors

A statement the directory refuses fails as the caller's error (class `User`) with one of these codes:

| Code | When |
|------|------|
| `RESERVED_NAME` | `CREATE USER [sa]`, `ALTER USER [sa] DISABLE`, `DROP USER [sa]`; `CREATE ROLE`, `ALTER ROLE` or `DROP ROLE` naming a built-in role. |
| `DUPLICATE_NAME` | `CREATE USER` or `CREATE ROLE` naming one that exists (user names compare lower-cased, role names case-insensitively). |
| `UNKNOWN_OBJECT` | `ALTER USER` or `DROP USER` of a user that does not exist; `ALTER ROLE` or `DROP ROLE` of a role that does not exist; `GRANT` or `REVOKE` naming an unknown role; `GRANT` to or `REVOKE` from an unknown user; `GRANT` or `REVOKE` `ON DATABASE` naming a database that does not exist, or one dropped while the statement ran. |
| `INSUFFICIENT_PRIVILEGE` | The caller lacks UserAdmin at the statement's scope, or someone other than `sa` changes `sa`'s password. |

`REVOKE` of a grant the user does not hold, and a repeated `GRANT`, succeed with `AFFECTED 0`.

## Examples

### Bootstrap a new database and its owner

```sql
CREATE DATABASE [inventory];
CREATE USER [warehouse] WITH PASSWORD 's3cret';
GRANT [Owner] TO [warehouse] ON DATABASE inventory;
```

### Rotate a user's password

```sql
ALTER USER [alice] WITH PASSWORD 'newp4ss'
```

### Audit who has access to a database

```sql
LIST GRANTS ON DATABASE myshop
```

### Disable a user without losing their grants

```sql
ALTER USER [alice] DISABLE;
-- ... later ...
ALTER USER [alice] ENABLE;
```

## See also

- [Connecting](connecting.md) — credentials transport, `tcps://`, SHA-256 certificate pins, mTLS.

## Detailed syntax and execution


<a id="syntax-auth-admin"></a>

### Auth admin

```sql
CREATE USER [alice] WITH PASSWORD 'secret';
ALTER  USER [alice] WITH PASSWORD 'newsecret';  -- rotate password
ALTER  USER [alice] DISABLE;                    -- block sign-in (keeps the row)
ALTER  USER [alice] ENABLE;                     -- re-enable
DROP   USER [alice];

CREATE ROLE [Auditor] WITH (READ: TRUE, WRITE: FALSE, DDL: FALSE, USERADMIN: FALSE, SOFT_VETO_BYPASS: FALSE);
CREATE ROLE [BreakGlass] WITH (SOFT_VETO_BYPASS: TRUE);  -- override-only role
ALTER  ROLE [Auditor] WITH (WRITE: TRUE);                -- change the named flags only
DROP   ROLE [Auditor];

GRANT  [Reader] TO [alice];                     -- server-wide
GRANT  [Writer] TO [alice] ON DATABASE [shop];  -- per-database
REVOKE [Reader] FROM [alice];

LIST USERS;
LIST ROLES;
LIST GRANTS;                                    -- every grant
LIST GRANTS [alice];
LIST GRANTS ON DATABASE [shop];
LIST AUDIT;                                     -- when audit-mode is on
```

All forms require `USERADMIN` and run on a server-level connection as well as on one bound to a database; authorization does not override engine-role admission. The `LIST` forms are non-mutating and remain available in `ReadOnly` and `Draining`. Forms that change users, roles, or grants commit independently and require the engine they run on to be `Writable` — the bound database's, or the master engine on a server-level connection; an open transaction does not make them admissible while draining.

`LIST AUDIT` takes an accepted-event read barrier before scanning: every audit event the server accepted before the list began has been processed, so a successful statement is visible to an immediately following audit query unless its persistence followed the configured audit-failure path. Rows stream through `RowsAsync` with non-null `Timestamp`, `Username`, `Statement`, `Decision`, and `Reason`; only `TargetDatabase` is nullable. `DropWrite` requires enqueue before the protected operation continues, while `FailOperation` and `Shutdown` wait for durable audit persistence first. An event `Shutdown` cannot admit or persist quarantines every engine the server hosts with `AUDIT_FAILSTOP`, including a tenant opened while the master still carries the fault. The accepted/processed barrier rebases at `INT64` maximum only when no queued event or waiter can still name the old epoch; otherwise admission rejects via the configured audit-failure policy. Persisted audit `Sequence` maximum is a terminal record retained across reopen for export/verification, and later events are rejected rather than wrapping or reusing an identity. Every credential is masked as `'***'` in audit statements, parse-error logs, and the diagnostic statement text of `[Mnemic].[ActiveQueries]`, `[Mnemic].[SlowQueries]`, the slow-query log and `[Mnemic].[StatementStats]`, in any spelling (quoted or not, with comments around it, after a misspelled or missing keyword); a user statement that carries one but does not parse reports `SYNTAX_ERROR` with a withheld message. An enabled `AuthOptions.AuditMode` requires `AuthOptions.AuditAnchor` (absolute append-only file path, independent HMAC key of at least 32 bytes, and stable non-empty deployment `BindingId`). Startup authenticates and reconciles this external high-water boundary before serving MQL; a missing/mismatched anchor or committed database-tail rollback fails closed. Keep the file and key outside the database trust boundary. The append-only anchor retains one credential-redacted recovery row per event and is not shortened by database `AuditRetention`; it has its own operator-managed evidence-retention/rotation policy.

**Grants are bound to the database, not its name.** `ON DATABASE` names a database that exists, or the statement fails with `UNKNOWN_OBJECT`. `DROP DATABASE` revokes every grant on the database with the drop, durably, and a database created again under the name starts with none.

**Grant changes bite live sessions.** `GRANT` / `REVOKE` / `ALTER ROLE` / `DROP USER` / `DROP ROLE` and the grants `DROP DATABASE` revokes take effect on already-connected sessions at their next statement — no reconnect needed because privileges resolve per statement. `ALTER USER … DISABLE` follows the opposite convention: it blocks new logins but does not terminate existing sessions. The process-lifetime invalidation generation cannot wrap and alias an old session snapshot: at terminal `INT64`, sessions conservatively reload grants and their derived permission bitmap on every statement.

**Disabled vs unknown users.** `ALTER USER … DISABLE` sets a flag on the user row; sign-in attempts return a deliberately indistinguishable error from "user not found" so an attacker can't enumerate accounts via timing or wording. Re-enable with `ALTER USER … ENABLE`. `DROP USER` removes the row entirely (and all assignments to it).

**Rotating the `sa` superuser.** `ALTER USER [sa] WITH PASSWORD '…'` rotates the built-in superuser's password online (no restart), but **only `sa` itself may run it** — a server-wide `Owner` is denied with `INSUFFICIENT_PRIVILEGE` (`Owner` is below `sa`, so this would be privilege escalation). `sa` cannot be dropped or disabled. An online rotation persists across restarts; the configured `RootPassword` only re-applies when its value changes between boots (the break-glass reset path).

**Username case.** Usernames are stored lower-cased — `CREATE USER [Alice]` and `CREATE USER [alice]` refer to the same account, and `LIST USERS` returns the lower-cased form. Same fold applies to `ALTER USER`, `DROP USER`, and the user side of `GRANT` / `REVOKE`. Backed by a unique index on `[Mnemic.Plugin.Auth.Users].Username` so concurrent `CREATE USER` on the same name never produces two rows.

**Custom roles.** Built-in roles (`Reader` / `Writer` / `Admin` / `Owner`) keep their hierarchical interpretation. `CREATE ROLE` mints a custom role with five independent boolean flags:

| Flag | Permits |
|------|---------|
| `READ` | every read-only MQL statement (`FROM`, `EXPLAIN`, `LIST` of read-only system collections, `DESCRIBE COLLECTION`, `DESCRIBE VIEW`), `SUBSCRIBE` / `UNSUBSCRIBE`, and transaction control (`BEGIN` / `JOIN TRANSACTION`, `COMMIT`, `ROLLBACK`) — each statement inside a transaction still needs its own flag. |
| `WRITE` | DML — `INSERT`, `UPDATE`, `REPLACE`, `DELETE`, `SEAL`, `UNSEAL`, `CANCEL VERSION`, `ON CONFLICT`. |
| `DDL` | schema management — `CREATE/ALTER/DROP COLLECTION/INDEX/ENUM/TYPE/VIEW/MATERIALIZED VIEW/TRIGGER/STATISTICS`, `REFRESH MATERIALIZED VIEW`, migrations, `BACKUP`, `ARCHIVE WAL`, `CREATE RESTORE POINT`, `DEFRAGMENT`, `SHRINK`, `ANALYZE`. |
| `USERADMIN` | user / role / grant management + `LIST AUDIT`. |
| `SOFT_VETO_BYPASS` | suppress `VetoPriority.Soft` fragments from plugin `IWriteGuard` / `IReadGate` dispatch. `Normal` and `Critical` fragments still apply to everyone — only the advisory layer is overridable. Built-in `Owner` carries the bit automatically; custom roles must opt in. |

Flags compose via OR across grants — a user with two roles gets the union of their permissions. The five-flag breakdown persists in the hidden `[Mnemic.Plugin.Auth.RolePermissions]` collection.

See [`auth-admin.md`](auth-admin.md).
