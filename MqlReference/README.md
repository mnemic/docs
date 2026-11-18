# MQL Reference

**Mnemic Query Language (MQL)** is the text-based query and command language for Mnemic. Every operation — queries, inserts, updates, deletes, schema management, transactions, maintenance — flows through MQL. Any language that can send MQL text over TCP can do everything the .NET typed client can do.

This reference is split into focused pages by topic. Start with [Syntax](syntax.md) for the lexical conventions, then jump to whichever area you need.

## Getting started

- [Syntax conventions](syntax.md) — identifiers, strings, comments, multi-statement scripts
- [Document literals](document-literals.md) — the `(...)` / `[...]` value syntax used across INSERT / UPDATE / REPLACE

## Queries

- [Basic queries](queries.md) — `FROM`, `SERIES` row sources, `WHERE`, `JOIN`, `ORDER BY`, `SKIP` / `LIMIT`, a page of a statement's result, `SELECT`, `EXPLAIN`
- [Aggregation](aggregation.md) — `GROUP BY`, `HAVING`, `COUNT` / `SUM` / `AVG` / `MIN` / `MAX`
- [Window functions](windows.md) — `OVER (...)`, `PARTITION BY`, `ORDER BY`, frames, named `WINDOW`
- [Common Table Expressions](cte.md) — `WITH` and `WITH RECURSIVE`
- [Temporal queries](temporal.md) — `AS OF`, `IN BETWEEN`, instant expressions, per-period reads over a `SERIES`, `LAST PER`
- [Array queries](arrays.md) — `[]` element semantics and correlated `MATCH`
- [Vector search](vectors.md) — `VECTOR(n)`, distance functions, `NEAREST`, and optional Vamana ANN indexes
- [Parameterized queries](parameters.md) — `@parameters`
- [Query metadata](system-collections.md#meta-fields) — the virtual `Meta` field on every result
- [Read sources](read-sources.md) — a request option: which read each column of a result came from, how each read chose its versions, and where its version bounds lie in the rows

## Mutations

- [DML](dml.md) — `INSERT`, `UPDATE`, `REPLACE`, `DELETE`, `SEAL`, `UNSEAL`, `CANCEL VERSION`, `INSERT … ON CONFLICT` (UPSERT), `RETURNING`
- [Concurrency control](dml.md#concurrency-control) — `EXPECTED REVISION`, `SKIP IF UNCHANGED`, `FORCE WRITE`
- [Transactions](transactions.md) — `BEGIN TRANSACTION` / `COMMIT` / `ROLLBACK`, distributed transactions

## Schema

- [Schema definition (DDL)](ddl.md) — `CREATE` / `ALTER` / `DROP COLLECTION`, field types, modifiers, foreign keys (incl. `PINNED REFERENCES`), `WITH CHECK` constraints, `DESCRIBE`
- [Polymorphism](polymorphism.md) — collection-level and field-level subtypes
- [Enums](enums.md) — `CREATE` / `ALTER` / `DROP ENUM`
- [Composite types](composite-types.md) — `CREATE` / `DROP TYPE`, field-of-type composition, `EXTENDS` inheritance
- [Indexes](indexes.md) — `CREATE` / `DROP` / `ALTER INDEX`, `UNIQUE`, covering, partial / filtered, vector ANN
- [Column-group statistics](analyze.md#declaring-column-group-statistics) — `CREATE` / `DROP STATISTICS` declare correlated field groups inside a migration; `ANALYZE` collects their distributions after it commits
- [Migrations](migrations.md) — `BEGIN MIGRATION` / `COMMIT MIGRATION`, `MERGE` / `SCRIPT`
- [Triggers](triggers.md) — `CREATE` / `DROP TRIGGER`, `[Mnemic].[Triggers]` catalog (row images, event timing, and conditions)
- [Views](views.md) — `CREATE` / `ALTER` / `DROP VIEW`, parse-time inlining, `[Mnemic].[Views]` catalog
- [Materialized views](materialized-views.md) — `CREATE` / `ALTER` / `REFRESH` / `DROP MATERIALIZED VIEW`, maintained refresh, `[Mnemic].[MaterializedViews]` catalog
- [Database diagrams](diagrams.md) — `CREATE` / `ALTER` / `DROP DIAGRAM`, box positions bound to their relations by identity, `EXPECTED REVISION`, `[Mnemic].[Diagrams]` catalog

## Server

- [Connecting](connecting.md) — `tcp://` vs `tcps://`, SHA-256 certificate pins, mTLS, the request options beside a statement's text
- [Database management](databases.md) — `CREATE` / `DROP` / `REOPEN` / `LIST DATABASE`, `LIST PLUGINS`
- [User and role administration](auth-admin.md) — `CREATE` / `ALTER` / `DROP USER`, `GRANT` / `REVOKE`, `LIST USERS / ROLES / GRANTS`
- [Subscriptions](subscriptions.md) — `SUBSCRIBE` / `UNSUBSCRIBE` (TCP push)
- [Maintenance](maintenance.md) — `BACKUP`, WAL archiving, point-in-time recovery, verified `RESTORE`, `VERIFY DATABASE`, `DEFRAGMENT`, `SHRINK`, `RECLAIM ORPHAN PAGES`, `SET CREATEDBY`
- [Statistics + cost-based planner](analyze.md) — `ANALYZE`, `[Mnemic].[Statistics]`, `EstimatedRows` in `EXPLAIN`
- [Timeouts and cancellation](cancellation.md) — `SET STATEMENT TIMEOUT`, `LIST QUERIES`, `CANCEL QUERY`, `[Mnemic].[ActiveQueries]`, FIN detection
- [Engine health](health.md) — `[Mnemic].[Health]` virtual collection, `Quarantined` admission gate, fault domains
- [Editor assistance](editor-assistance.md) — `COMPLETE` (what fits at a position) and `VALIDATE` (what does not parse), plugins' statements included

## Reference

- [System virtual collections](system-collections.md) — `[Mnemic].[Collections]`, `[Mnemic].[Enums]`, `[Mnemic].[Metrics]`, `[Mnemic].[Migrations]`, `[Mnemic].[Triggers]`, `[Mnemic].[Views]`, `[Mnemic].[MaterializedViews]`, `[Mnemic].[Diagrams]`
- [Practical examples](examples.md) — paginated lists, audit trails, hierarchies, joins, FK cascades, billing reports, etc.
- [Reserved keywords](keywords.md) — the full keyword list grouped by area
- [Common errors](errors.md) — error messages and what triggers them

## Conventions across this reference

- User-defined names require brackets, including `[Namespace].[Collection]`, `[Namespace].[TypeName]`, `[FieldName]`, database names, aliases, and named windows; brackets are always accepted, and the `ON DATABASE` scope of the user and role administration statements also takes a plain identifier ([authentication](auth-admin.md)). Built-in type keywords, function names, session-parameter keys, and grammar options remain unbracketed; see [Syntax conventions](syntax.md).
- Single-quoted strings, doubled to escape (`'O''Brien'`). Line comments use `--`; block comments use `/* ... */` and may nest up to 100 levels. Unterminated block comments reject; quoted delimiters remain literal content.
- All keywords are case-insensitive.
- All `STRING` value comparisons are case-insensitive under the active .NET invariant simple-lowercase mapping, followed by ordinal comparison (`'café' = 'CAFÉ'` matches). This is neither linguistic collation nor Unicode Default Case Folding: ignorable characters stay significant and no normalization occurs. The header fingerprints the exact effective mapping and policy. If a runtime or globalization-backend change alters it, open durably invalidates and rebuilds every derived index before publishing the new profile; an interruption repeats that work on reopen.
- `NOW` resolves at execute time, not parse time.
