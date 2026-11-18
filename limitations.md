# Limitations

This page records current product boundaries. Query-specific syntax and workarounds remain authoritative in the [MQL reference](MqlReference/README.md).

## Deployment

- One engine/server owns one writable database file. Built-in replication, high availability, sharding, and multi-writer clustering are absent.
- The production target is a trusted single-node deployment, not hostile multi-tenant isolation within one server process.
- Tier-1 targets are Linux x64 server/container and Windows x64 embedded/Studio on .NET 11.
- The standalone `mnemic-server` executable and its container image resolve only the settings in [configuration](configuration.md#standalone-server-environment) and open every database with the default engine options plus the plugins they load. Engine options such as storage encryption, WAL archiving, snapshot isolation and memory budgets, and the server's audit, client-certificate, statement-timeout and connection-limit settings, are set only by an embedded application or a [programmatic server host](configuration.md#programmatic-server-host).
- Storage encryption is opt-in through `EncryptionKey` or `EncryptionKeyProvider` of an engine an application hosts. It protects durable pages, WAL, backups, plugin snapshots, and temporary spill; operators supply and retain the keys. Existing plaintext databases require explicit migration into newly encrypted storage. The standalone server and its container image do not configure it ([data at rest](security.md#data-at-rest)).
- Database-page compression is absent. Application payload compression remains an application concern.
- The built-in WAL archive provider accepts durable absolute `file:` destinations. `WalArchiveUri` cannot be combined with authenticated storage encryption: the active segmented archive log does not use the keyed WAL framing required by encrypted databases. Encryption of sealed archive objects does not remove this restriction.

## Schema and storage

- Primary keys are single-field `INT8`, `INT16`, `INT32`, `INT64`, `GUID`, `STRING`, or `BYTES`. Composite primary keys are absent.
- Encoded keys must fit the B+ tree key bound. Oversized variable keys fail before mutation.
- `AUTOINCREMENT` applies to `INT32`/`INT64`; `SEQUENTIAL` applies to `GUID`.
- References of every shape (scalar, packed array, inside array objects, and pinned) support all seven primary-key types. Pinned references require a temporal target and cannot `Cascade`; an unpinned packed reference array rejects `SetNull`, and `Cascade` removes its individual references instead.
- Logical dump/import is absent. Backup/restore is physical and format-bound.
- Full-text and spatial indexes are absent. Secondary B+ tree indexes support compound keys, covering fields, and deterministic partial predicates. Vector search supports exact top-k and the optional Vamana approximate index provider.

## Queries

- Prepared-statement handles and wire compression are absent. A connection can carry multiple outstanding requests with correlated responses, control frames, and notifications; ordinary statements execute in order through result delivery. See [request cancellation](MqlReference/cancellation.md) for queue bounds.

## Triggers

- Triggers are `BEFORE` or `AFTER` and per-row. They support combined events, ordered statement lists, and an optional `WHEN` condition.
- `NEW`/`OLD` are read-only and exposed only for the event combinations in the [trigger reference](MqlReference/triggers.md).
- Writable `NEW`, statement-level triggers, transition tables, and deferred trigger execution are absent.

## Views

- Ordinary and materialized views are read-only to user DML.
- Complex incremental views use transaction-owned full recomputation for joins, windows, subqueries, temporal shapes and grouping outside the per-group class; a view grouped by its key field recomputes a touched group, rather than changing it in place, when its own columns cannot decide the change.
- Materialized views use an explicit single-field primary key.

## Server and identity

- Authentication is built in; federated identity providers are absent.
- Authorization is granted per database: per-collection grants, row-level security, and column masking are absent ([users and roles](MqlReference/auth-admin.md)).
- Automatic TLS manages a persistent self-signed certificate. Public-CA automation such as ACME is external to Mnemic.
- Plugins are trusted in-process extensions, not sandboxed workloads.
