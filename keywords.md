# Reserved keywords

All keywords are case-insensitive. A **reserved** word is a keyword wherever it appears. A **contextual** word is a keyword only where the grammar expects it (`NOW` after `AS OF`, `CASCADE` after `DROP VIEW`) and an ordinary identifier everywhere else, such as a function name, a session-parameter key or a grammar option. Neither kind limits user-defined names: fields, aliases and every other user-defined name are always bracketed ([identifiers](syntax.md#identifiers)), so `[Order]`, `[Using]` and `[Now]` are valid names, and an unbracketed relation alias is a `SYNTAX_ERROR` whatever its spelling (`FROM [App].[T] t`, `FROM [App].[T] wal`).

`[Mnemic].[Keywords]` lists the words the connected server reads as keywords, the reserved ones with `IsReserved` true and the contextual ones with `IsReserved` false (see [system collections](system-collections.md)); it is the authoritative list for the server's version and the plugins it loads. A plugin's statement or clause extension declares the words it reads, such as the [authentication](auth-admin.md) statements' `USER`, `ROLE`, `PASSWORD`, `GRANT` and `REVOKE` and the `VAMANA` index options, and they are listed as contextual while the plugin is loaded; a word an extension reads without declaring it is refused, so the list misses none.

## Query & control flow

Reserved: `FROM`, `WHERE`, `AND`, `OR`, `NOT`, `ORDER`, `BY`, `ASC`, `DESC`, `SKIP`, `LIMIT`, `SELECT`, `JOIN`, `INNER`, `LEFT`, `CROSS`, `OUTER`, `APPLY`, `LATERAL`, `ON`, `AS`, `OF`, `IN`, `IS`, `BETWEEN`, `LIKE`, `ESCAPE`, `EXISTS`, `CONTAINS`, `STARTSWITH`, `MATCH`, `WITH`, `RECURSIVE`, `UNION`, `INTERSECT`, `EXCEPT`, `ALL`, `EXPLAIN`, `NEAREST`, `USING`, `OPTION`

Contextual: `ANY` (a trailing element quantifier, as `ALL`), `TOP`, `APPROXIMATE`, `COSINE`, `L2`, `DOT`, and `SEARCH_LIST` (a `NEAREST` clause), `SERIES` and `EVERY` (a generated [series source](queries.md#series) after `FROM` or `JOIN`, and its step), `OMIT` (`OMIT INNER KEY`), `FORMAT`, `TEXT`, and `JSON` (`EXPLAIN (FORMAT …)`), `MAXDOP` and `TIMEOUT` (`OPTION (…)`, [cancellation](cancellation.md#option---per-statement-options))

`COSINE_DISTANCE`, `L2_DISTANCE`, and `DOT_PRODUCT` are function names, not keywords. `EXCLUDE`, `REPLACE`, and `RENAME` are also the wildcard-projection modifiers after `SELECT *` or `SELECT [alias].*`; they may be written in any order, and canonical output prints them in that order. A series' units (`MINUTE` … `YEAR`), `TO`, `IN` and `TIMEZONE` keep their reserved roles. `OMIT INNER KEY` removes the joined relation's key only from wildcard or implicit result projection.

## Conditional expressions

Reserved: `CASE`, `WHEN`, `THEN`, `ELSE`, `END`, `CAST`

Contextual: `TRY_CAST`

## Aggregation

Reserved: `GROUP`, `HAVING`, `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COLLECT_ARRAY`, `JOIN_STRING`, `DISTINCT`, `FILTER`

## Window functions

Reserved: `OVER`, `WINDOW`, `PARTITION`, `RANGE`, `ROWS`, `GROUPS`, `ROW`, `PRECEDING`, `FOLLOWING`, `UNBOUNDED`, `EXCLUDE`, `TIES`, `OTHERS`, `CURRENT`, `NO`

> Window-function names (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, `NTILE`, `PERCENT_RANK`, `CUME_DIST`) are function names, not keywords. The aggregates (`SUM`, `AVG`, `MIN`, `MAX`, `COUNT`) are reserved keywords that also compute a window with `OVER (...)`.

## DML

Reserved: `INSERT`, `INTO`, `UPDATE`, `SET`, `DELETE`, `REPLACE`, `VALID`, `TO`, `RETURNING`, `CONFLICT`, `NOTHING`, `DO`, `EXCLUDED`, `CANCEL`, `VERSION`, `PREVIOUS`, `CREATEDBY`

Contextual: `AT`, `CLEAR`, `CONFIRM`, `FORCE`, `ID`, `WRITE`, `EXPECTED`, `REVISION`, `IF`, `UNCHANGED`, `CREATEDAT`, `COPY`, `APPEND`, `SEAL`, `UNSEAL`

`COPY` follows `CREATEDBY`; `COPY OR <actor>` adds an explicit fallback ([statement attribution](dml.md#createdby-attribution)). `DEFAULT`, reserved for DDL, is also a field's whole value in a written document or SET list: the field's declared default ([DEFAULT](dml.md#default)). `APPEND` names an [array edit](arrays.md#editing-arrays). `SEAL` and `UNSEAL` are statement heads of the reversible record-seal commands; the matching bulk confirmations are `CONFIRM SEAL ALL` and `CONFIRM UNSEAL ALL`.

## DDL

Reserved: `CREATE`, `DROP`, `ALTER`, `COLLECTION`, `DATABASE`, `INDEX`, `UNIQUE`, `INCLUDE`, `FIELD`, `RENAME`, `ADD`, `REMOVE`, `SCHEMA`, `SUBTYPE`, `DISCRIMINATOR`, `NULLABLE`, `NOT`, `NULL`, `PRIMARY`, `KEY`, `DEFAULT`, `AUTOINCREMENT`, `SEQUENTIAL`, `COMPUTED`, `STORED`, `TEMPORAL`, `REFERENCES`, `MATCH`, `PIN`, `PINNED`, `CHECK`, `ENUM`, `VALUES`, `VALUE`, `TYPE`, `EXTENDS`, `NAMESPACE`, `EXTERNAL`, `DESCRIBE`, `LIST`

Contextual: `SKIPUNCHANGEDUPDATES` and `CACHECURRENTVERSION` (collection settings; see [CREATE COLLECTION](ddl.md#create-collection)), `RESTRICT`, `CASCADE`, and `SETNULL` (`ON REMOVE` actions; `CASCADE` also ends `DROP … CASCADE`), `FULL` and `SIMPLE` (after `MATCH` on a foreign key), `ONLINE` and `OFF` (`CREATE INDEX … WITH (ONLINE = …)`), `STATISTICS` (`CREATE` / `DROP STATISTICS`), `MOVE` and `FIRST` ([`MOVE FIELD … FIRST`](ddl.md#move-a-field), whose `LAST`, `BEFORE` and `AFTER` keep their reserved roles), `EXISTING` and `DOCUMENTS` ([`ADD SUBTYPE … FOR EXISTING DOCUMENTS`](ddl.md#make-a-collection-polymorphic), whose `FOR` keeps its reserved role)

Contextual while the optional vector-index plugin is loaded: `VAMANA`, `INDEXES`, `METRIC`, `MAX_DEGREE`, `BUILD_LIST`, `ALPHA` and `FILTER_FIELDS`, the words of `CREATE VECTOR INDEX` and `LIST VECTOR INDEXES` ([vector search](vectors.md)).

## Types

Reserved: `STRING`, `INT`, `INT8`, `INT16`, `INT32`, `INT64`, `LONG`, `BOOL`, `DECIMAL`, `GUID`, `DATETIME`, `DATEONLY`, `TIMEONLY`, `FLOAT`, `FLOAT32`, `FLOAT64`, `DOUBLE`, `SINGLE`, `BYTES`, `VECTOR`, `OBJECT`, `ARRAY`

## Transaction / migration

Reserved: `BEGIN`, `COMMIT`, `ROLLBACK`, `ABORT`, `MIGRATION`, `TRANSACTION`, `ISOLATION`, `SERIALIZABLE`, `SNAPSHOT`, `JOIN`, `MERGE`, `SCRIPT`

## Maintenance / subscriptions / meta

Reserved: `ARCHIVE`, `BACKUP`, `RESTORE`, `VERIFY`, `DEFRAGMENT`, `SHRINK`, `ANALYZE`, `SUBSCRIBE`, `UNSUBSCRIBE`, `PLUGIN`, `CHANNEL`, `DEPENDENCY`, `SHOW` (with `SET` for session parameters — `SET <name> = <value>` / `SHOW <name>` / `SHOW ALL`)

Contextual: `RECLAIM`, `ORPHAN`, and `PAGES` ([`RECLAIM ORPHAN PAGES`](maintenance.md#reclaim-orphan-pages)), `INCREMENTAL` and `BUDGET` (`DEFRAGMENT INCREMENTAL … BUDGET n`), `WAL`, `REQUIRE`, `PITR`, `TIMELINE`, `LATEST`, `CSN`, `TIMESTAMP`, `POINT`, and `REPLAY` (the WAL archive and point-in-time restore forms, whose `USING` is reserved), `PLUGINS` (`LIST PLUGINS`), `REOPEN` ([`REOPEN DATABASE`](databases.md#reopening-a-database)), `DISPLAY` (how the actors of [`SET CREATEDBY`](maintenance.md#createdby-display) read)

## Engine administration

Contextual: `ENGINE`, `RESUME`, `ADMISSION`, `SLOWQUERY`, `THRESHOLD`, `RINGCAPACITY`, `NONE` — used by `ALTER ENGINE RESUME`, `ALTER ENGINE SET ADMISSION` and `ALTER ENGINE SET SLOWQUERY` ([health](health.md)); `NONE` also clears a session statement timeout.

## Temporal & bucketing

Reserved: `LAST`, `PER`, `STARTING`, `TIMEZONE`, `MINUTE`, `HOUR`, `DAY`, `WEEK`, `MONTH`, `YEAR`, `MONDAY`, `TUESDAY`, `WEDNESDAY`, `THURSDAY`, `FRIDAY`, `SATURDAY`, `SUNDAY`

## Auth administration

Reserved: `ENABLE`, `DISABLE`

Contextual: `USER`, `USERS`, `ROLE`, `ROLES`, `GRANT`, `GRANTS`, `REVOKE`, `AUDIT`, `PASSWORD`, and the permission names `READ`, `WRITE`, `DDL`, `USERADMIN` and `SOFT_VETO_BYPASS`. The authentication plugin's statements read them ([users and roles](auth-admin.md)), not the core lexer, and every server loads that plugin.

## Editor assistance

Contextual: `COMPLETE`, `VALIDATE` — statement heads of `COMPLETE @text AT @position` and `VALIDATE @text`, whose `AT` is contextual under DML ([editor assistance](editor-assistance.md)).

## Cancellation

Contextual: `STATEMENT`, `TIMEOUT`, `QUERIES`, `QUERY`, `NONE` — used by `SET STATEMENT TIMEOUT`, `LIST QUERIES`, and `CANCEL QUERY` ([cancellation](cancellation.md)).

## Literals

Reserved: `TRUE`, `FALSE`, `NULL`

Contextual: `NOW`

## Triggers

Reserved: `TRIGGER`, `BEFORE`, `AFTER`, `FOR`, `EACH`, `ROW`

Contextual: `OLD`, `NEW` (the row images a trigger body reads)

Used by `CREATE TRIGGER ... ON [Coll] BEFORE|AFTER INSERT|UPDATE|DELETE FOR EACH ROW AS BEGIN ... END` and `DROP TRIGGER [name] ON [Coll]`. See [Triggers](triggers.md).

## Views

Reserved: `VIEW` — used by `CREATE [OR REPLACE] VIEW [Ns].[Name] AS <select>`, `ALTER VIEW [Ns].[Name] AS <new-select>`, and `DROP VIEW [Ns].[Name]`. See [Views](views.md).

## Materialized views

Reserved: `MATERIALIZED`, `REFRESH`

Contextual: `MANUAL`, `INCREMENTAL` (`REFRESH = MANUAL` / `REFRESH = INCREMENTAL`)

Used by `CREATE MATERIALIZED VIEW [Ns].[Name] WITH (REFRESH = MANUAL, PRIMARY KEY = [F]) AS <select>`, `ALTER MATERIALIZED VIEW ... AS <select>`, `REFRESH MATERIALIZED VIEW [Ns].[Name]`, `EXPLAIN REFRESH MATERIALIZED VIEW [Ns].[Name]`, and `DROP MATERIALIZED VIEW [Ns].[Name]`. See [Materialized views](materialized-views.md).

## Diagrams

Contextual: `DIAGRAM`, `FIELDS`, `KEYS`, with `AT` (a box's position) and `NONE` (a `FIELDS` value), whose other roles are listed above, and the reserved `ALL`

Used by `CREATE DIAGRAM [Name] ([Ns].[Name] AT (x, y), ...) WITH (FIELDS = ALL | KEYS | NONE)`, `ALTER DIAGRAM [Name] (...) WITH (FIELDS = ...) EXPECTED REVISION n`, `ALTER DIAGRAM [Name] RENAME TO [NewName]`, and `DROP DIAGRAM [Name]`. See [Database diagrams](diagrams.md).

## Expiry policies

Reserved: `TTL`, `EXPIRY` — automatic deletion policies in `CREATE INDEX … WITH TTL '7 DAYS'` or `CREATE INDEX … WITH EXPIRY`. See [Expiry indexes](indexes.md#expiry-indexes).
