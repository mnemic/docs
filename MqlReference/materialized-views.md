# Materialized views

A materialized view stores the result of a SELECT in a real backing collection. Reads against the materialized view name hit B+ trees directly — no re-running of the underlying query. MANUAL contents are stale until the operator runs `REFRESH`; INCREMENTAL contents are maintained from source mutations and temporal clock transitions.

Every materialized-view backing is engine-owned and read-only to user DML. Internal seed, refresh, and incremental maintenance carry explicit authority. A projected reference, including one chosen as the `PRIMARY KEY`, keeps only its target identity: the backing copies neither the source's `ON REMOVE` action nor its `MATCH FULL` mode and enforces no foreign key of its own, including when the source declared the reference before its target existed. Target removal acts on live source rows and their incrementally maintained results; a MANUAL snapshot keeps its stored values until REFRESH, including after reopening. References from ordinary collections to a materialized view remain enforced and follow the view across REFRESH and ALTER; see [references to a materialized view](#references-to-a-materialized-view). A projected [CreatedBy attribution column](queries.md#createdby-attribution-columns), such as `[Meta].[CreatedBy] AS [Author]`, stays marked as one in the backing, so reads of the view keep the mark, also after reopening.

## DDL

```mql
CREATE MATERIALIZED VIEW [Ns].[Name]
    WITH (
        REFRESH = MANUAL | INCREMENTAL,
        PRIMARY KEY = [Field]
    )
AS <from-query>;

REFRESH MATERIALIZED VIEW [Ns].[Name];
EXPLAIN REFRESH MATERIALIZED VIEW [Ns].[Name];

ALTER MATERIALIZED VIEW [Ns].[Name]
    WITH (
        REFRESH = INCREMENTAL,
        PRIMARY KEY = [Field]
    )
AS <from-query>;

DROP MATERIALIZED VIEW [Ns].[Name];

FROM [Mnemic].[MaterializedViews];  -- catalog
```

### Example

Given a database at schema level 3 (each committed migration advances the level by one):

```mql
BEGIN MIGRATION 3;
CREATE MATERIALIZED VIEW [Sales].[BigOrdersM]
    WITH (REFRESH = MANUAL, PRIMARY KEY = [Id]) AS
    FROM [Sales].[Orders]
    WHERE [Total] >= 5000
    SELECT [Id], [Customer], [Total];
COMMIT MIGRATION;

-- Cheap read — hits the backing collection directly
FROM [Sales].[BigOrdersM] WHERE [Customer] = 'Acme';

-- After source writes, the view is stale until REFRESH
BEGIN MIGRATION 4;
REFRESH MATERIALIZED VIEW [Sales].[BigOrdersM];
COMMIT MIGRATION;
```

## How it differs from a plain view

| | **VIEW** | **MATERIALIZED VIEW** |
|---|---|---|
| Storage | None — re-runs SELECT every time | Real engine collection at the user's name |
| Read cost | Cost of re-running the SELECT | Cost of a regular collection scan / index probe |
| Freshness | Always current | MANUAL is stale between refreshes; INCREMENTAL is maintained |
| Indexes | N/A — inlined into the calling query | `CREATE INDEX` works as on any other collection |
| DDL surface | `CREATE / ALTER / DROP VIEW` | `CREATE / ALTER / REFRESH / DROP MATERIALIZED VIEW` |
| Catalog | `[Mnemic].[Views]` | `[Mnemic].[MaterializedViews]` |

## Refresh modes

- **`REFRESH = MANUAL`** — operator runs `REFRESH MATERIALIZED VIEW` to re-compute the backing. Stale-by-default between refreshes.
- **`REFRESH = INCREMENTAL`** — engine maintains the backing transactionally on every participating source mutation and at temporal activation/expiry boundaries. A mutation counts whether or not a statement made it: the rows a temporal expiry's `ON REMOVE Cascade` or `SetNull` deletes, ends or clears in a referencing collection maintain that collection's INCREMENTAL views too. A boundary whose maintenance fails is retried, and the boundary is not recorded as delivered until its maintenance has succeeded. An explicit `REFRESH` can also rebuild the maintained generation. See *Maintenance execution classes* below for the cost model.

During the initial CREATE seed, source writes remain available. External transactions journal mutated PKs, and their final committed source images are reconciled through the seed transaction immediately before `COMMIT MIGRATION`. Deletes of rows absent from the seed at that point cannot be resurrected by the older seed snapshot; writes that arrive during the final commit handoff wait briefly and then apply to the published backing. An unresolved explicit source transaction can therefore delay COMMIT up to the engine lock-timeout budget, after which the migration aborts rather than publishing an uncertain view.

Incremental CREATE includes earlier INSERT, UPDATE, and DELETE statements from its owning migration, including rows in a materialized backing created earlier in that migration. Preceding source schema or index DDL is also supported. When the migration already owns source writes or a retained DDL fence, CREATE fences every input through commit or abort and seeds the migration's current rows. Competing source operations may wait or be refused within the engine's lock-timeout budget; retry them after the migration settles. Source writes after CREATE share the seed transaction and update its private backing directly. Without prior owner writes or retained fences, the concurrent seed/journal protocol described above remains in use.

MANUAL CREATE / ALTER / REFRESH supports computed source columns and computed SELECT projections. Their evaluated values become ordinary stored fields in the backing, including inside nested objects, array elements, and subtype extensions; backing reads do not re-evaluate them, and the projection need not retain the computed expression's source operands. INCREMENTAL row-local plans support scalar computed SELECT expressions against a computed-enriched copy of the authoritative source row. Interleaving those expressions with source fields preserves written column order during seeding, maintenance, and reopening while retaining single-key maintenance. A body that groups one collection by one field uses per-group maintenance, and other shapes outside the row-local contract use full recomputation.

Materialized backings store evaluated result values. Source defaults and object-relative CHECK constraints stay on the source and are cleared from backing schemas, including nested objects, array elements, and subtype fields. RENAME, EXCLUDE, and REPLACE can therefore change the projected shape without retaining source expressions that refer to its old children. Source writes continue to apply their declared defaults and constraints.

INCREMENTAL bodies accept an omitted SELECT, `SELECT *`, qualified wildcards, and mixtures of wildcards and explicit projections. Wildcards select declared data fields; source `Meta` and private polymorphic envelope fields are not stored as backing data. EXCLUDE, REPLACE, and RENAME work on top-level fields, nested Object children, and Array elements. Scalar replacements read original values. These forms retain one-key maintenance when their expressions are row-local. The declared output primary key must directly project the unchanged source key. `SELECT [Id] AS [Key]` and `SELECT * RENAME ([Id] AS [Key])` work with `PRIMARY KEY = [Key]`, including a projection containing only that alias. Removing the key or replacing its value is rejected for this class. Window and subquery replacements use full recomputation. Source schema changes re-expand the saved wildcard and atomically rebuild affected maintained backings.

## REFRESH transaction scope

`REFRESH MATERIALIZED VIEW` must run inside `BEGIN MIGRATION`; a standalone REFRESH and REFRESH inside `BEGIN TRANSACTION` are rejected. The shadow seed, backing-name swap, secondary indexes, and `LastRefreshedAt` update share the owning migration's single WAL outcome. The owner sees the new backing immediately; other readers keep the committed generation until commit. Source and affected backing writers stay fenced until commit or abort. Earlier owner writes participate in the seed, and later writes maintain INCREMENTAL backings normally. Aborting preserves the old generation and timestamp. A failure after the migration's durable decision recovers the new rows and metadata together.

DDL order within the migration is honored. Multiple REFRESH statements for one name supersede their earlier private generations, so only the last REFRESH can publish. If `DROP MATERIALIZED VIEW` follows, the final identity is absent. Superseded generations retire after publication through the read epoch. DROP followed by CREATE may reuse the name in either refresh mode, including a view created earlier in the same migration. DROP drains and fences every old incremental input, removes its maintenance subscription, and closes any unfinished initial seed. Later owner writes and repeated replacements use the current definition; an old DROP cannot unregister the final replacement at commit. Aborting restores the originally committed view and maintenance, or removes the name if it did not exist before the migration. DROP/CREATE assigns a new catalog identity; ALTER preserves it. Existing dependent-view and trigger checks still apply to DROP.

An ordinary publication-callback failure after the migration is durable makes COMMIT return `CATALOG_PUBLICATION_FAILED_AFTER_COMMIT`, clears migration session state, and records structural-corruption health faults including the underlying `POST_COMMIT_CALLBACK_FAILED`. Direct or nested memory exhaustion and simulated process death instead escape with their original outer exception identity after retained publication owners and migration session state are released. Neither outcome is reported as success or as a retryable rollback. An ordinary failure while recording the callback health fault remains subordinate, but direct or nested simulated process death from that reporter escapes after the same cleanup and stops later publication callbacks.

The collection-name segments `__refresh_`, `__alter_`, and `__retired_` are reserved for materialized-view transient backings. `CREATE COLLECTION`, `CREATE VIEW`, `CREATE MATERIALIZED VIEW`, `ALTER COLLECTION ... RENAME TO`, `ALTER VIEW ... RENAME TO` and `ALTER NAMESPACE ... RENAME TO` reject a relation or namespace name containing any of these segments with `RESERVED_NAME` (`CREATE MATERIALIZED VIEW [App].[Totals__refresh_]` reports `Cannot create materialized view '[App].[Totals__refresh_]': the name segments …`); this keeps user identities distinct from internal replacement generations. The generation a REFRESH, `ALTER MATERIALIZED VIEW` or source rebuild retires is never listed: from the replacing commit until its drop it stays out of [`[Mnemic].[Collections]`](system-collections.md) and every other catalog view, out of `[Mnemic].[Statistics]` and out of [`COMPLETE`](editor-assistance.md), so a client that reads the catalog right after the migration sees only the view.

## ALTER and EXPLAIN REFRESH

`ALTER MATERIALIZED VIEW` runs in a migration and supports MANUAL/INCREMENTAL mode changes. It streams the new query into a replacement backing, preserving catalog `Id`, `CreatedAt`, and compatible secondary-index definitions. The owner sees the replacement immediately; source and backing mutations are fenced until commit or rollback. The definition, rows, indexes, mode, and maintenance behavior share one durable migration outcome. Later source schema changes and owner writes use the replacement definition. A failed build or abort restores the prior state.

REFRESH and ALTER rebuild downstream INCREMENTAL views in dependency order. Propagation stops at MANUAL snapshots, which retain their rows until explicitly refreshed. Backing triggers remain registered; rebuilding emits no row events, and subsequent DML fires them normally. Saved dependent projections and trigger bodies must remain valid; an incompatible replacement aborts the migration.

`EXPLAIN REFRESH MATERIALIZED VIEW [Ns].[Name]` plans the stored body in either refresh mode through ordinary `EXPLAIN` without creating a shadow or changing `LastRefreshedAt`. `EXPLAIN ANALYZE REFRESH ...` executes the stored SELECT for analysis but still does not publish a replacement.

## References to a materialized view

A field declared `REFERENCES [Ns].[View]` binds the materialized view, not one backing generation. REFRESH, `ALTER MATERIALIZED VIEW`, and a source-schema rebuild move every such reference to the published generation in the same migration outcome. Inserts and updates then validate against the view's current rows, and the replaced generation is reclaimed without touching referencing rows. A view that is the database's [`SET CREATEDBY COLLECTION`](maintenance.md#set-createdby) target is referenced the same way: the policy moves to the published generation, whose rows then validate `CREATEDBY` actors, and a key that leaves the view while a retained version names it as actor fails the REFRESH or ALTER with `CREATEDBY_ACTOR_REFERENCED`.

A key that the old generation contained but the new one does not is removed from the view. Its references receive their declared `ON REMOVE` action inside the replacing migration, exactly as for a deleted target row:

| Action | Effect of a key leaving the view |
|---|---|
| `Restrict` | The REFRESH or ALTER fails with a foreign-key error and the migration rolls back |
| `Cascade` | The referencing document is deleted, or the key is removed from a reference array |
| `SetNull` | The nullable reference is cleared |

Given schema level 5:

```mql
BEGIN MIGRATION 5;
CREATE MATERIALIZED VIEW [Sales].[Active] WITH (REFRESH = MANUAL, PRIMARY KEY = [Id])
AS FROM [Sales].[Customers] WHERE [Active] = TRUE SELECT [Id], [Name];
COMMIT MIGRATION;

BEGIN MIGRATION 6;
CREATE COLLECTION [Sales].[Campaigns] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Customer] INT32 REFERENCES [Sales].[Active] ON REMOVE Cascade
);
COMMIT MIGRATION;

-- After a customer becomes inactive, this REFRESH also deletes its campaigns.
BEGIN MIGRATION 7;
REFRESH MATERIALIZED VIEW [Sales].[Active];
COMMIT MIGRATION;
```

Writers of collections that reference the view are held from the replacement until the migration commits or aborts. A held statement may return a retryable error; a retry validates against the published view. Readers are not blocked. Aborting restores the old generation together with every row an action changed. A replacement cannot change the primary-key type of a view that a collection references or that the CreatedBy policy names, with or without an actor recorded: ALTER MATERIALIZED VIEW, or the source change that rebuilds the view, reports `DEPENDENT_OBJECTS_EXIST` naming the reference or the policy, and the migration rolls back. A replacement keyed by another field of the same type is judged by its keys, for references and the policy alike: a key that stays keeps what names it, and a key that leaves the view is removed as above. A view nothing references may change its key type; seals do not carry over, because a key of another type identifies no prior row.

## Source schema changes

`ALTER COLLECTION` permits source field and subtype changes when every affected saved query and declared output key remains valid. Errors identify the dependent view and invalid field; the whole migration rolls back. Pending definitions, replacements, and removals participate in validation, which repeats against the latest committed dependencies before publication. A concurrent incompatible source or definition change requires retrying the migration.

MANUAL views keep their existing snapshot and schema until explicit REFRESH. INCREMENTAL views rebuild when the projected layout or computed source values change. Rebuilds proceed through dependent INCREMENTAL views in dependency order; propagation stops at a MANUAL snapshot. Source and backing schema changes, rows, secondary indexes (including expiry policies), and refresh timestamps commit atomically. Competing writers remain fenced until settlement. Unused field changes that preserve the projected layout keep the existing backing.

These structural rebuilds do not emit per-row trigger events. Subsequent source DML uses the current backing and ordinary maintenance/trigger ordering. Dependent views and trigger bodies must remain valid against any rebuilt backing schema. Use `ALTER MATERIALIZED VIEW` to change a backing's projection; direct field DDL on the backing remains rejected.

## Lifecycle and dependency rules

Triggers on a backing fire for the view's net row changes, in both maintenance classes: a key that enters the view fires INSERT, a key whose row changes fires UPDATE with the previous row as `OLD`, a key that leaves the view fires DELETE, and a row that stays the same fires nothing. A trigger-maintained copy of the view therefore stays equal to it. REFRESH, ALTER and schema rebuilds emit no row events.

The following rules apply:
- **REFRESH preserves the complete user index set.** Every secondary-index definition is rebuilt on the shadow before publication. An index-build failure preserves the old generation, index set, and refresh timestamp. Vector indexes of the view are rebuilt on the replacement in the same migration too ([vector indexes](vectors.md)); one the replacement cannot satisfy refuses it with `DEPENDENT_OBJECTS_EXIST`.
- **Statistics declarations follow the view.** A [`CREATE STATISTICS`](analyze.md#declaring-column-group-statistics) declaration on a materialized view moves to the generation each REFRESH, `ALTER MATERIALIZED VIEW` or source rebuild publishes, in the same migration outcome; a replacement that no longer stores one of its fields as a scalar drops it, as `DROP FIELD` would. Collected `[Mnemic].[Statistics]` rows belong to the generation they were collected from, so the new generation starts unanalyzed.
- **ALTER is a complete replacement, not field-level editing.** Supply the complete refresh mode, primary key, and read-query body. Compatible INCREMENTAL dependants rebuild and backing triggers survive the replacement.
- **`PRIMARY KEY` is mandatory and explicit** — must name a single field present in the SELECT projection. No auto-derivation.
- **Direct DML against the backing is rejected.** INSERT, UPDATE, REPLACE, DELETE, SEAL, UNSEAL, and CANCEL VERSION return `MATERIALIZED_VIEW_READ_ONLY`. The decision comes from durable catalog identity and remains effective after reopen; internal maintenance alone carries write authority.
- **Seal metadata survives maintenance.** Schema rewrites and refresh preserve the seal flag for surviving backing identities. An IVM write that would logically mutate a sealed backing identity fails with `DOCUMENT_SEALED` in the source transaction.
- **SEAL and UNSEAL maintain views that read row metadata.** A seal toggle on a source fires no source trigger, but it changes `[Meta].[Sealed]`: a row-local view that reads row metadata rewrites the toggled row, and a fully recomputed view recomputes when its body names `Meta` (directly or through a plain view it reads). A view that reads no row metadata does no work for the toggle, so sealing many rows costs it nothing.
- **REFRESH supports both modes and maintained descendants.** Dependent INCREMENTAL backings rebuild in the same migration. MANUAL descendants keep their snapshots. Rebuilding preserves trigger registrations without row events.
- **`DROP MATERIALIZED VIEW` is rejected when an INCREMENTAL mat view depends on the target.** the drop would otherwise leave the downstream's `IvmRegistry` entry pointing at a missing source. Drop the downstream INCREMENTAL view first.
- **`DROP COLLECTION` on a regular collection is rejected when an INCREMENTAL mat view depends on it.** Same reasoning as the mat-view check above — recreating under the same name would silently re-activate IVM maintenance against an unrelated collection. Drop the dependent INCREMENTAL view first, then drop + recreate the collection, then recreate the dependent view.
- **Collection and namespace rename preserve materialized views.** `ALTER COLLECTION ... RENAME TO` can rename a source or the backing itself; `ALTER NAMESPACE ... RENAME TO` moves every affected source and backing. Saved query bindings and incremental registrations retain their identities. Collection identity, rows, indexes, creation time, and refresh time are preserved; a MANUAL view retains its snapshot until REFRESH. Explicit aliases, CTE names, output names, and string literals retain their meaning.
- **Rename composes within one migration.** Later source writes use the new names, and a later REFRESH, ALTER MATERIALIZED VIEW or DROP MATERIALIZED VIEW names a renamed backing by its new name, through a chain of renames or a swap through a free name too, acting on the generation the name holds. A name that differs only in letter case respells the materialized view with its backing. Earlier CREATE, ALTER, REFRESH, and DROP/CREATE remain supported, including repeated renames and reuse of names freed earlier in the block. Pending incremental seeds keep their dirty-key journals. Source-only rename permits foreign writes, which are reconciled before publication. Backing changes retain input/backing fences until commit or abort. Abort restores the original names and maintenance. Compatible concurrent dependencies can commit; missing identities, incompatible schema changes and name collisions require retry.
- **`DROP COLLECTION` rejects a materialized backing.** Use `DROP MATERIALIZED VIEW` so the backing and definition are removed together.
- **`DROP COLLECTION` rejects a collection referenced by a MANUAL view.** References anywhere in the saved query count, including nested subqueries. Drop dependent views before removing their sources. Source field/subtype ALTER follows the schema validation rules above. Plain-view DROP retains `CASCADE` removal semantics; rename preserves its static dependencies.
- **`DROP VIEW` is rejected when a MANUAL mat view's body references the view.** Symmetric with the collection guard — the body inlines the view at REFRESH time.
- **`DROP MATERIALIZED VIEW` is rejected when the target's backing is referenced by plain view(s), by another MANUAL mat view's body, or by the body of a trigger on another target** (`DEPENDENT_OBJECTS_EXIST` naming the trigger). The backing is queryable like any collection, so dropping it out from under a dependent view breaks that view at query / REFRESH time. Drop the dependents first.
- **MANUAL DROP dependency checks apply owner changes to the latest committed definitions.** A new definition protects its sources immediately. Dropping or retargeting it releases its former dependencies for later DDL in the same migration. Dismantle plain, MANUAL, or mixed chains outside-in within one migration. Commit rejects dependencies published concurrently against removed identities, including reused names. MANUAL creation and refresh can read plain views created or altered earlier in that migration; other sessions continue to use committed definitions.
- **Mat-view-on-mat-view chains are bounded at depth 100.** Each cascade step shares the nesting bound of foreign-key cascades and trigger bodies, so a 101st nested IVM application fails with `LIMIT_EXCEEDED` (`Nested write depth exceeded 100: …`) and rolls back the source. A genuine feedback cycle that re-enters the same live `(matview, pk)` is also rejected (`TRIGGERED_DATA_CHANGE_VIOLATION`) and rolls back the shared source/backing transaction. It is never silently skipped: doing so could commit a later source image while leaving the backing at the outer image.
- **A catalog row whose source vanished out-of-band is fatal.** MQL `DROP COLLECTION` rejects dropping an INCREMENTAL source (see above), but an embedded host bypass or an inconsistent restore can still orphan the persisted row. Executor initialization then reports `IVM_SOURCE_MISSING` as catalog corruption instead of disabling maintenance and leaving a stale backing collection queryable. Restore the source from a trusted backup, or remove and recreate the materialized view only after auditing the backing contents.
- **A missing backing or mismatched row identity is fatal.** Executor initialization reports `IVM_BACKING_MISSING` when the catalog definition has no backing collection. It raises `IVM_PRIMARY_KEY_MISMATCH` when the catalog primary key differs from the backing's primary key. A row-local INCREMENTAL plan additionally proves that the declared output slot directly projects its source key, allowing aliases; unsafe persisted projections raise `IVM_DEFINITION_UNSAFE`. A full-recompute plan instead proves uniqueness from the complete result.
- **The persisted refresh mode is an exact domain.** Only `0 = MANUAL` and `1 = INCREMENTAL` are valid. Any other value raises `MATVIEW_REFRESH_MODE_INVALID` during executor initialization rather than being treated as MANUAL and silently disabling incremental maintenance.

Stored sources bind to stable collection or view identities. Collection and namespace renames preserve the stored body, bindings and materialized rows; catalog queries render current names. Reusing an old source name does not redirect REFRESH or incremental maintenance. Both refresh modes expand omitted SELECT and wildcard projections over declared data fields, excluding private metadata.

## Plain views as maintained sources

INCREMENTAL definitions can read plain views, including transitive chains and views created or altered earlier in the same migration. FROM, JOIN, scalar and WHERE subqueries, WITH, and set expressions use the same logical definitions as ordinary queries. SELECT may be omitted when the resulting fields include the declared key. Logical-view sources use full recomputation: every underlying physical input is registered, and its mutations recompute the complete result in the source transaction. The declared output key must be present and unique.

ALTER VIEW and CREATE OR REPLACE VIEW rebuild affected INCREMENTAL backings and their maintained descendants in dependency order. The owner sees the new rows immediately. Old and new input writers remain fenced until commit or abort; other readers retain the committed definitions and rows. MANUAL snapshots retain their prior contents and stop propagation until explicitly refreshed. Replacing a source view can retarget maintenance to another collection without recreating the materialized view. Abort restores the original definitions, rows, and inputs. Renaming a source view (`ALTER VIEW ... RENAME TO`) rebuilds nothing and fences no input: the materialized view keeps its rows and keeps maintaining from the same inputs under the view's new name.

DROP VIEW and plain-view CASCADE reject any remaining INCREMENTAL materialized dependent. Remove or retarget those dependents first in the same migration. Cycles in the maintained source graph, through plain definitions or maintained backings, are rejected. Reopening reconstructs the same physical registrations and rejects cyclic persisted maintenance graphs with `IVM_DEFINITION_UNSAFE` before enabling any registry.

## Maintenance execution classes

Incremental definitions use the cheapest proved class for their stored body:

- **Row-local delta.** One non-system, non-temporal source; source-field projections including the source primary key; an optional row-local `WHERE`; and optional scalar computed SELECT projections. INSERT/UPDATE evaluates membership and the exact projection from the authoritative source after-image, then upserts or deletes that one backing key. DELETE removes the same key. Stored and virtual computed scalar source fields are supported. Maintenance synthesizes virtual values from the changed row before WHERE and SELECT, with the same statement clock as the source write and RETURNING. The backing stores those values as ordinary fields; source virtual slots stay physically NULL. Clock-dependent values change at seeding or source mutation, not merely with elapsed time. OBJECT and ARRAY projections preserve complete containers, including computed descendants. Selecting `[Detail].[Name]` retains the whole `[Detail]` object once, even when multiple children or computed expressions are interleaved in SELECT. Unknown children reject before publication. Polymorphic Object fields and Array Object elements retain their discriminator, active extension, and computed descendants across subtype changes. Nested Object predicates use each row's active field declaration when sibling subtypes reuse a name with different types. Field predicates use the same native comparison and IN domain during initial seeding and subsequent maintenance, including INT8/INT16 unsigned literal spellings. Scalar expressions retain their ordinary mathematical semantics. Collection-level subtypes also use one-key maintenance: base fields, extension fields, and their computed values can be selected together. An extension field absent from a row's subtype becomes NULL. Scalar expressions read nested operands through the active subtype layout. A selected field or whole container must have compatible declarations across subtypes; incompatible output shapes reject before publication. Writes during initial seeding and subsequent reopening preserve the same source/backing agreement. Source aliases and bare or namespace-qualified collection names are accepted in fields, computed SELECT expressions, and WHERE, and `[Body]` is optional shorthand unless the source declares a `Body` field, which then owns its children there as in the body's query. For `SELECT [i].[Id]`, the backing declaration is `PRIMARY KEY = [Id]`; `SELECT [i].[Id] AS [Key]` instead uses `PRIMARY KEY = [Key]`. The source key value and compact encoding remain unchanged across both forms. These bindings and authored output order are preserved during maintenance and after reopening. SELECT may read row metadata as a read of the source does: `[Meta].[CreatedBy] AS [Author]`, `[Meta].[CreatedAt]`, `[Meta].[Revision]`, `[Meta].[Sealed]`, `[Meta].[ValidFrom]` and `[Meta].[ValidTo]`, directly or inside an expression (`[Meta].[Revision] * 10 AS [Rev10]`), alongside source fields or a wildcard. Each changed row's columns hold the metadata of the version its write produced, with the types and values a MANUAL refresh of the same body stores, and a CreatedBy column keeps its [attribution mark](queries.md#createdby-attribution-columns). Name each such column with AS: a backing column cannot be called `Meta` (see *Validation*). The WHERE decides membership from the stored row alone, which carries no row metadata, so a WHERE that reads `[Meta]` (`WHERE [Meta].[Revision] >= 1`) rejects the definition: `INCREMENTAL mat view rejected: Unknown field 'Meta.Revision' in row predicate.` A body maintained by full recomputation (below) may filter on row metadata. A projection the body cannot bind as a read (an undeclared or repeated item) rejects the definition with the diagnostic and code a read of the body reports: `SELECT [Id], [Nope]` reports `UNKNOWN_FIELD` `INCREMENTAL mat view rejected: Column 'Nope' does not exist. Available columns: …`; a migration that alters or revalidates the body reports such a mistake under the view's own context with the same code. Broader relational shapes route to full recomputation.
- **Per-group delta.** One non-system, non-temporal collection without subtypes, read directly or through another materialized view but not through a plain view; `GROUP BY` one stored field, and `PRIMARY KEY` naming that grouping key's output; an optional `WHERE` that compares stored fields with constants (`[Region] != NULL AND [Total] > 0`, `[Status] IN ('paid', 'sent')`); and no DISTINCT, window, ordering, paging, temporal clause or subquery. A source write then changes only the groups its old and new row belong to, so its cost does not grow with the view. Each such group's row changes in place from the view's own columns:
  - `COUNT(*)` and `COUNT([F])` add and subtract the changed row (`COUNT([F])` skipping NULL);
  - `SUM([F])` of an `INT8`–`INT64` field adds and subtracts the value, NULL when the group holds no non-null value;
  - `AVG([F])` of an integer field is derived from a projected `SUM([F])` and the field's non-null count, holding the bits a REFRESH computes;
  - `MIN([F])` and `MAX([F])` of a scalar field take an added value that passes the stored extremum;
  - a group's first member inserts its row, and a group whose row count reaches 0 is deleted.

  A change these columns cannot decide recomputes only the touched group, by running the body for that one grouping key: a removed row when the view projects no `COUNT(*)` (or `COUNT` of a `NOT NULL` field) to tell whether its group emptied, or no count of a `SUM`'s field to tell whether the sum became NULL; a `MIN` or `MAX` whose extremum leaves; a `STRING` key or extremum whose stored spelling leaves with its row. A body with `HAVING`, an expression over its aggregates, a `SUM` or `AVG` of a `FLOAT` or `DECIMAL` field (whose result depends on summation order and scale), or an aggregate with `FILTER`, `DISTINCT` or an expression argument recomputes every group a write touches; without an index on the grouping key that recomputation scans the collection. Project `COUNT(*)`, and `COUNT([F])` beside `SUM([F])` and `AVG([F])` of a nullable field, to keep every change in place:

  ```mql
  CREATE MATERIALIZED VIEW [Sales].[ByCustomer] WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Customer]) AS
  FROM [Sales].[Orders] WHERE [Customer] != NULL GROUP BY [Customer]
  SELECT [Customer], COUNT(*) AS [Orders], SUM([Total]) AS [Revenue], COUNT([Total]) AS [Priced], AVG([Total]) AS [Average]
  ```

  Every group row equals the row a REFRESH of the same body writes for that group. A group total that leaves the `INT64` range fails the source write with `INVALID_VALUE`, and a member whose grouping key is NULL fails it with `NOT_NULL_VIOLATION`, as at seeding and REFRESH. Text that differs only in letter case, a `DECIMAL` that differs only in scale and a `DATETIME` that differs only in offset are one value, so a group's key or extremum is spelled as one current member spells it; which member a REFRESH picks depends on where the rows are stored, so the view may spell such a value differently from a REFRESH, never hold a different value. Concurrent writers of one group take turns on its row. Triggers on the backing fire for each group row that changed: an UPDATE with `OLD` and `NEW`, an INSERT for a new group and a DELETE for an emptied one.
- **Transaction-owned full recomputation.** `JOIN`, grouping outside the per-group class (several grouping keys, a computed, virtual or pinned-reference grouping key, a key other than the `PRIMARY KEY`, a plain-view or subtyped source, a `WHERE` that calls a function or reads `NOW`, row metadata or a virtual computed field), `DISTINCT`, windows, scalar projection and WHERE subqueries, expressions that read another SELECT item's alias, ordering/paging, and temporal clauses evaluate the complete stored query whenever any participating source mutates. Inside that source transaction the result is compared with the backing by primary key: keys that left the result are deleted first, then a key whose row changed is updated and a new key inserted, while an unchanged row is not written. A failure rolls back source and derived changes together, and a result that repeats a primary key fails with `UNIQUE_VIOLATION`, as at seeding and REFRESH. The seed result owns the exact output schema and the declared materialized-view primary key only needs to be a unique projected result field; it need not equal a source key.

All physical sources, including nested scalar-query sources, are registered. Partner reads share the source transaction and one statement clock. Reopen hydrates the same multi-source registration from the durable body.

Temporal sources use full recomputation. The durable temporal event queue invokes maintenance when a future version activates or a current version expires, including scheduled `DELETE AT`; retries remain queue-owned. Source DML also recomputes in its transaction, `CANCEL VERSION` included, so a predecessor whose end it restores or whose body it rewrites never leaves the view behind.

Set expressions, leading WITH, and derived read sources retain their complete query scopes in the persisted body. Both MANUAL refresh and INCREMENTAL recomputation track every physical source. Non-read bodies are rejected during DDL and catalog hydration.

A body that reads a [`SERIES`](queries.md#series) uses full recomputation: a mutation of the body's stored collection recomputes it, and `NOW`-relative series bounds are sampled then. An `INCREMENTAL` body that reads no stored collection at all, such as a series alone, has nothing to maintain it from and is rejected with `INVALID_DEFINITION`; materialize it with `REFRESH = MANUAL`.

## Incremental maintenance ordering and observability

Incremental maintenance for a source mutation completes before user `AFTER` triggers on that mutation. The trigger therefore sees the derived row inside the shared transaction. If it writes the same source row again, that nested mutation is later and its maintenance result remains authoritative. Any trigger or maintenance failure rolls back the source, backing, and trigger effects together.

Successful incremental applies publish these OpenTelemetry-compatible instruments on the `Mnemic.Engine` meter:

| Instrument | Tags | Meaning |
|---|---|---|
| `mnemic.ivm.maintain_count` (`Counter<long>`) | `op`, `matview` | Successful insert, update, or delete applies. |
| `mnemic.ivm.maintain_us` (`Histogram<double>`) | `op` | Wall-clock microseconds for a successful backing apply. |
| `mnemic.ivm.reconverge_count` (`Counter<long>`) | `matview` | INSERT replaced a pre-existing backing key or UPDATE repaired a missing key. |

`EXPLAIN ANALYZE` of source DML emits an `IvmMaintain(<matview> <op>)` stage for each successful apply. Telemetry is best-effort and never changes the statement result. These instruments measure completed maintenance work; the enclosing transaction can still roll back after an observation.

## Validation

`CREATE MATERIALIZED VIEW` rejects:

- a duplicate mat view name;
- an existing real collection or plain view with the same name;
- a name in `[Mnemic]` / `[Mnemic.*]`;
- a `PRIMARY KEY = [F]` whose field is missing from the SELECT projection;
- a `PRIMARY KEY = [F]` whose projected field cannot identify backing rows (`INVALID_DEFINITION`): the key follows the [collection key rule](ddl.md#field-modifiers), so it is an `INT8`, `INT16`, `INT32`, `INT64`, `GUID`, `STRING` or `BYTES` value and not a `PINNED` reference. A `DATETIME`, `DECIMAL`, `ARRAY` or `VECTOR` column is refused whether it is projected directly or as a `GROUP BY` key: `CREATE MATERIALIZED VIEW [App].[ByDay] WITH (REFRESH = MANUAL, PRIMARY KEY = [PlacedAt]) AS FROM [Sales].[Orders] GROUP BY [PlacedAt] SELECT [PlacedAt], COUNT(*) AS [Orders]` reports `INVALID_DEFINITION` when `[PlacedAt]` is a `DATETIME`;
- a body that names a field its sources do not declare anywhere — a projection expression, an aggregate argument, `WHERE` or an ordering — in every refresh mode. The body binds over catalog metadata exactly as a later migration that revalidates it binds it, so the definition is refused now rather than accepted with the field read as NULL: `CREATE MATERIALIZED VIEW [App].[V] WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Id]) AS FROM [App].[Items] SELECT [Id], [Nope] + 1 AS [X]` reports `UNKNOWN_FIELD` `Invalid definition for materialized view '[App].[V]': Unknown field 'Nope' in expression.` `ALTER MATERIALIZED VIEW` and `REFRESH MATERIALIZED VIEW` bind the body the same way;
- a projected column named `Meta` or `Envelope`, in every refresh mode (`RESERVED_NAME`): every read of the view presents its own row metadata under that name. `SELECT [Id], [Meta]` and `SELECT [Id], [Meta].[CreatedBy]` report `Materialized view column 'Meta' has a reserved name: every read of the view presents its own [Meta]. Name the column with AS, as in [Meta].[CreatedBy] AS [Author].`;
- a body that uses an array element path where one value per row is read — a column, a grouping, ordering or window key, an aggregate or function argument — in every refresh mode (`FEATURE_NOT_SUPPORTED`; [element paths read no single value](queries.md#array-element-paths-read-no-single-value)).

A migration that changes a source revalidates every definition that reads it and refuses a change that would leave one unable to bind, naming that definition with the binder's code (`Invalid definition for materialized view '[App].[V]': Unknown field 'Spare' in expression.`). Repair the dependent in the same migration, before the change: `ALTER MATERIALIZED VIEW` to a body that no longer reads the field, or `DROP MATERIALIZED VIEW`. Pending replacements and removals take effect for the check, so the change then commits. A migration that changes no source of the definition never revalidates it.

The projected `PRIMARY KEY` must also be **unique across the SELECT result**. The backing is a real collection with a single-field PK, so two result rows that share the named PK value would collide on the backing's PK index. CREATE, REFRESH and ALTER **fail loud** in that case ("Materialized view '[App].[V]' PRIMARY KEY '…' is not unique across the SELECT result …") and roll back the half-built backing — they do not silently drop the colliding rows. For an aggregating view, name the `GROUP BY` key (or any field whose values are unique) as the `PRIMARY KEY`. A `CREATE UNIQUE INDEX` on the view holds on every generation as well: a REFRESH or ALTER result whose rows have distinct keys but share a value of such an index is refused with `UNIQUE_VIOLATION` naming the index and its fields ("Materialized view '[App].[V]' cannot hold its SELECT result: two projected rows share a value of its unique index 'secondary_Code' on [Code]. …"), and the view keeps its committed rows.

The backing key is `NOT NULL` even when the projected column is nullable, so every result row must carry a key. CREATE, REFRESH and ALTER refuse a row whose key is NULL with `NOT_NULL_VIOLATION` ("Materialized view '[App].[V]' PRIMARY KEY 'K' is NULL in a row of its query; …"), and so does the source write whose incremental recomputation would produce one. Every refusal of a replacement's row names the view as the catalog calls it, never the transient generation the replacement builds before it takes the view's name. Exclude NULL keys in the body; a nullable reference makes a valid key once filtered, and keeps its target identity without the source's removal action:

```mql
CREATE MATERIALIZED VIEW [Sales].[RepTotals] WITH (REFRESH = MANUAL, PRIMARY KEY = [SalesRepId]) AS
FROM [Sales].[Orders] WHERE [SalesRepId] != NULL GROUP BY [SalesRepId]
SELECT [SalesRepId], COUNT(*) AS [Orders], SUM([Total]) AS [Revenue]
```

Here `[SalesRepId]` may be `INT32 NULLABLE REFERENCES [Org].[Employees] ON REMOVE SetNull` in the source. Removing an employee clears the source rows' reference; the snapshot keeps its row for that employee until `REFRESH`.

The backing publishes the canonical structural schema fingerprint from the moment CREATE or REFRESH constructs it; the live generation and the same generation after reopen therefore have identical schema identity. The fingerprint is never a field-count placeholder, so different backing layouts cannot appear to share an identity in metadata or client schema checks.

Migration-owned CREATE / atomic REFRESH backings are removed by the migration transaction's rollback compensation; the failing statement does not issue a second out-of-band DROP that could wait on its own structural drain. If that rollback cleanup fails, the statement returns `ROLLBACK_CLEANUP_FAILED` and records an engine-wide structural-corruption health fault. Inspect `[Mnemic].[Health]` and recover from a trusted state before resuming ordinary statements.

`REFRESH MATERIALIZED VIEW` and `DROP MATERIALIZED VIEW` reject unknown view names.

Programmatically constructed `CREATE`, `ALTER`, `REFRESH`, `EXPLAIN REFRESH`, and `DROP MATERIALIZED VIEW` statements must use the exact canonical `[Ns].[Name]` identity accepted by parsed MQL. Malformed UTF-16 and non-canonical names fail before any dependency, catalog, or backing lookup.

## Persistence

The catalog row in `[Mnemic].[MaterializedViews]`, as a query reads it:

| Field             | Type            | Notes |
|-------------------|-----------------|-------|
| `Id`              | `GUID`          | primary key, synthesized |
| `Name`            | `STRING`        | the view's current qualified name, which is its backing collection's name |
| `NamespaceId`     | `GUID`          | the identity of the backing collection's namespace, stable across a namespace rename |
| `BodyText`        | `STRING`        | the canonical text of the SELECT body, rendered with the current names of the objects it reads, not the text as written |
| `PrimaryKeyField` | `STRING`        | the field declared in `WITH (PRIMARY KEY = …)` |
| `RefreshMode`     | `INT8`          | exactly `0 = Manual` or `1 = Incremental`; every other value is catalog corruption |
| `CreatedAt`       | `DATETIMEOFFSET`| at CREATE; preserved by ALTER |
| `LastRefreshedAt` | `DATETIMEOFFSET`| at the CREATE/ALTER generation or last successfully published REFRESH generation |

The view's name is stored once, as its backing collection's. The stored row holds no name: it names its backing by collection ID, which a REFRESH or ALTER rewrites in the same outcome that publishes the replacement generation, and it is unique on that ID. Renaming the view, moving it to another namespace, relabelling or merging its namespace therefore changes only the backing's catalog entry, and a relation that later takes a name the view once held is never mistaken for it. The backing collection persists like any other; `Engine` open re-discovers it on its own. The mat view is queryable across restart without any extra step. For REFRESH, the timestamp and live backing-name swap share one durable transaction, so reopen cannot pair an old generation with a newly advanced timestamp. Executor initialization validates the persisted refresh-mode domain and all incremental dependencies before a statement can query a backing as maintained truth. A persisted body retains one complete read query, including leading `WITH` bindings and `UNION`/`INTERSECT`/`EXCEPT` expressions. CREATE, executor initialization, `REFRESH`, and dependency checks preserve those wrappers through the shared decoder. Malformed or non-read bodies fail with `DDL_CATALOG_BODY_INVALID`.

## See also

- [Plain views](views.md) — when re-running the SELECT each time is fast enough.
- [System collections](system-collections.md).

## Detailed syntax and execution

Stored computed source fields are durable scalar values and are eligible for row-local incremental projection. Backing fields store the query result as an ordinary field, without inheriting the source computation contract.

### Seed an incremental chain after source writes

Given schema level 1 and an existing `[App].[Items]` collection with `[Id] INT32 PRIMARY KEY, [Value] INT32` (and unused key 3):

```mql
BEGIN MIGRATION 1;
INSERT INTO [App].[Items] ([Id]: 3, [Value]: 30);
CREATE MATERIALIZED VIEW [App].[Stored] WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Id])
AS FROM [App].[Items] SELECT [Id], [Value];
CREATE MATERIALIZED VIEW [App].[Downstream] WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Id])
AS FROM [App].[Stored] ORDER BY [Id] SELECT [Id], [Value];
UPDATE [App].[Items] SET ([Value]: 31) WHERE [Id] = 3;
COMMIT MIGRATION;
```

Both backings contain key 3 with Value = 31. Aborting instead would remove both new views and undo the source insert. Creation retains the participating source fences until the migration settles.

### Refresh an upstream snapshot and its maintained descendants

Given `[App].[Stored]` in MANUAL mode and `[App].[Downstream]` in INCREMENTAL mode over it, with schema level 2:

```mql
BEGIN MIGRATION 2;
REFRESH MATERIALIZED VIEW [App].[Stored];
FROM [App].[Downstream] SELECT [Id], [Value];
COMMIT MIGRATION;
EXPLAIN REFRESH MATERIALIZED VIEW [App].[Downstream];
```

The query inside the migration sees both refreshed generations. Other readers keep their committed generations until COMMIT. Backing triggers remain registered without firing during the rebuild. EXPLAIN inspects the incremental view's stored query without changing its rows or timestamp.

### Replace a materialized view under the same name

Given schema level 2, `[App].[Items]` and `[App].[Other]` with `[Id]` / `[Value]`, and `[App].[Stored]` without dependants or backing triggers:

```mql
BEGIN MIGRATION 2;
DROP MATERIALIZED VIEW [App].[Stored];
CREATE MATERIALIZED VIEW [App].[Stored] WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Id])
AS FROM [App].[Other] SELECT [Id], [Other].[Value] * 2 AS [Value];
UPDATE [App].[Other] SET ([Value]: 15) WHERE [Id] = 1;
COMMIT MIGRATION;
```

The computed item qualifies `[Other].[Value]` because its alias repeats the input name ([alias rules](queries.md#alias-repeating-an-input-field)).

The replacement follows `[App].[Other]` immediately inside the migration and retains that maintenance after commit. Aborting restores the original definition. A view created earlier in this same migration may also be dropped and replaced.

### Retarget a snapshot and remove its former source

Given schema level 3, a plain `[App].[Current]` view over `[App].[Items]`, a MANUAL `[App].[Stored]` materialized view over `[App].[Current]`, and an existing `[App].[Replacement]` collection with `Id INT32 PRIMARY KEY` and `Value INT32`, the whole change fits in one migration:

```mql
BEGIN MIGRATION 3;
ALTER MATERIALIZED VIEW [App].[Stored]
WITH (REFRESH = MANUAL, PRIMARY KEY = [Id])
AS FROM [App].[Replacement] SELECT [Id], [Value];
DROP VIEW [App].[Current];
DROP COLLECTION [App].[Items];
COMMIT MIGRATION;
```

The replacement seeds from the new source and releases the old dependency before either DROP. COMMIT publishes the complete change; ABORT restores the previous snapshot, view, and source. Plain views created or altered earlier in a migration are also available to that owner's queries and MANUAL seeds.

### Maintain a projection through a plain view

Given schema level 4 and `[App].[Items]` with `Id INT32 PRIMARY KEY` and `Value INT32`:

```mql
BEGIN MIGRATION 4;
CREATE VIEW [App].[Current] AS
FROM [App].[Items] WHERE [Value] >= 0 SELECT [Id], [Value];
CREATE MATERIALIZED VIEW [App].[Stored]
WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Id])
AS FROM [App].[Current] SELECT [Id], [Value];
COMMIT MIGRATION;

BEGIN MIGRATION 5;
ALTER VIEW [App].[Current] AS
FROM [App].[Items] WHERE [Value] >= 0 SELECT [Id], [Items].[Value] * 2 AS [Value];
COMMIT MIGRATION;

FROM [App].[Stored];
```

The ALTER rebuilds Stored in the same migration. Later Items writes recompute Stored through the new definition; ABORT would restore the original definition and rows. This uses full recomputation on each source mutation.

### Incremental primary-key aliases

Given schema level 4 and `[App].[Items]` with `Id INT32 PRIMARY KEY` and `Value INT32`:

```mql
BEGIN MIGRATION 4;
CREATE MATERIALIZED VIEW [App].[ByKey]
WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Key])
AS FROM [App].[Items] SELECT [Id] AS [Key], [Value];
CREATE MATERIALIZED VIEW [App].[RenamedKey]
WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Key])
AS FROM [App].[Items] SELECT * RENAME ([Id] AS [Key]);
COMMIT MIGRATION;
```

Both views preserve the source key value and maintain one row per source mutation. The PRIMARY KEY option names the output alias. Queries can select only `[Id] AS [Key]` when the backing needs no other fields.



<a id="syntax-create--alter--refresh--drop-materialized-view"></a>

### CREATE / ALTER / REFRESH / DROP MATERIALIZED VIEW

```sql
CREATE MATERIALIZED VIEW [Ns].[Name]
    WITH (REFRESH = MANUAL, PRIMARY KEY = [Field])
AS <from-query>;

REFRESH MATERIALIZED VIEW [Ns].[Name];
EXPLAIN REFRESH MATERIALIZED VIEW [Ns].[Name];
ALTER MATERIALIZED VIEW [Ns].[Name]
    WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Field])
AS <from-query>;
DROP MATERIALIZED VIEW [Ns].[Name];
FROM [Mnemic].[MaterializedViews];
```

A materialized view is a real backing collection populated by running the SELECT once. `REFRESH MATERIALIZED VIEW` re-runs the SELECT into a shadow backing, atomically publishes that generation, and retires the old backing through the read epoch. The mat view name itself is the backing collection — `FROM [name]` reads it directly, no inlining. Reclaiming a retired backing waits for statements that are already running and never cancels them; a long-running statement can postpone it. Until then the `__retired_` backing stays cataloged but is never listed: `[Mnemic].[Collections]`, the other catalog views, `[Mnemic].[Statistics]` and `COMPLETE` name only the view. Nor does it hold what it stored: once the view and every listed collection stop using an enum, an enum member, a named type or a referenced collection, `DROP ENUM`, `ALTER ENUM ... REMOVE VALUE`, `DROP TYPE` and `DROP COLLECTION` succeed, in the migration that refreshed or altered the view or a later one, and no `ALTER ENUM` or `ALTER TYPE` probes, rewrites or changes the retired backing. The retirement is recorded with that commit: if the database closes or the process stops before reclamation finishes, the next open drops the retired backing and frees its pages before it returns.

REFRESH must run inside `BEGIN MIGRATION`. Its shadow seed, stable-name swap, indexes, and `LastRefreshedAt` update share that migration's single WAL outcome. The owner sees the replacement immediately; ordinary readers retain the committed generation until commit. Source and backing writers stay fenced through settlement. Abort restores the original generation; a post-durable failure recovers all new rows and metadata together. Standalone REFRESH and REFRESH inside `BEGIN TRANSACTION` are rejected.

Within one migration, later REFRESH statements supersede earlier private generations of the same name, so only the last can publish. A later DROP removes the final live identity; superseded generations retire through the read epoch. A later CREATE in either mode may reuse that name, including a view created and dropped earlier in the block. DROP fences the old incremental inputs, removes their subscription, and settles the old initial seed. Later owner writes use the current definition; old DROP callbacks cannot remove its replacement. Abort restores the originally committed view, or leaves the name absent if it was newly created. DROP/CREATE assigns a new catalog identity; ALTER preserves it. Dependent-view and trigger checks still apply before DROP.

`ALTER COLLECTION ... RENAME TO` renames a source or a materialized backing, its letter case alone included; `ALTER NAMESPACE ... RENAME TO` moves all affected physical relations. Saved query references and incremental registrations retain their identities. Rows, collection identity, indexes, creation/refresh times, aliases, CTE names, output names, and literals are preserved. MANUAL snapshots stay unchanged. Renames compose with earlier CREATE/ALTER/REFRESH/DROP+CREATE, with later REFRESH, ALTER and DROP by the new name, repeated rename, and old-name reuse. Pending initial seeds retain their dirty-key journal and reconcile foreign writes across source-only rename. Backing changes retain write fences until settlement. Abort restores the original names and maintenance. Compatible concurrent dependencies can commit; missing identities, incompatible schema changes and name collisions require retry. Plain-view DROP keeps CASCADE removal semantics; rename preserves static dependencies.

INCREMENTAL materialized views can read plain views, including pending and transitive definitions. Every underlying physical input drives full recomputation. SELECT may be omitted; the result must include the declared key and its values must be unique. ALTER VIEW / CREATE OR REPLACE VIEW atomically rebuild affected INCREMENTAL generations and descendants, with old/new input writers fenced through settlement; MANUAL snapshots retain their contents. Abort restores the previous rows, definitions, and subscriptions. DROP VIEW and CASCADE reject live materialized dependents. Retarget or remove those dependents first. Persisted logical or maintenance cycles fail closed as `IVM_DEFINITION_UNSAFE`.

**Refresh modes:**
- `REFRESH = MANUAL` — operator runs `REFRESH MATERIALIZED VIEW` to re-compute the backing.
- `REFRESH = INCREMENTAL` — engine maintains the backing transactionally on every participating source mutation, including the referencing rows a temporal expiry's `ON REMOVE` action writes, and on temporal activation/expiry delivery, which retries a failed maintenance before recording the boundary as delivered. Explicit REFRESH can rebuild the maintained generation. Row-local projection/filter bodies use one-key deltas; a body that groups one collection by its key field changes only the groups a write touches; joins, other grouping, DISTINCT, windows, scalar projection and WHERE subqueries, ordering/paging, and temporal bodies use transaction-owned full recomputation. See the usage pitfalls below for the cost boundary.

MANUAL CREATE / ALTER / REFRESH supports computed source columns and computed SELECT projections. Their evaluated values become ordinary stored fields in the backing, recursively through objects, array elements, and subtype extensions; backing reads do not re-evaluate them, and the projection need not retain the computed expression's source operands. INCREMENTAL row-local maintenance supports scalar computed SELECT projections; other computed/global shapes use the full-recompute class.

During the initial INCREMENTAL seed, concurrent external source transactions journal dirty PKs. `COMMIT MIGRATION` drains their outcomes, re-reads each key's final committed source image, and reconciles it through the seed transaction. A delete-before-seed cannot be resurrected; writes arriving after the close wait for publication and then apply normally. The transaction drain is bounded by the engine lock-timeout policy. Incremental CREATE includes earlier INSERT, UPDATE, and DELETE statements from its owning migration, including rows in a materialized backing created earlier in that migration. Preceding source schema or index DDL is also supported. When the migration already owns source writes or a retained DDL fence, CREATE fences every input through commit or abort and seeds the migration's current rows. Competing source operations may wait or be refused within the engine's lock-timeout budget; retry them after the migration settles. Source writes after CREATE share the seed transaction and update its private backing directly. Without prior owner writes or retained fences, the concurrent seed/journal protocol described above remains in use.

INCREMENTAL bodies accept an omitted SELECT, `SELECT *`, qualified wildcards, and mixtures of wildcards and explicit projections. Wildcards select declared data fields; source `Meta` and private polymorphic envelope fields are not stored as backing data. EXCLUDE, REPLACE, and RENAME work on top-level fields, nested Object children, and Array elements. Scalar replacements read original values. These forms retain one-key maintenance when their expressions are row-local. The declared output primary key must directly project the unchanged source key. `SELECT [Id] AS [Key]` and `SELECT * RENAME ([Id] AS [Key])` work with `PRIMARY KEY = [Key]`, including a projection containing only that alias. Removing the key or replacing its value is rejected for this class. Window and subquery replacements use full recomputation. Source schema changes re-expand the saved wildcard and atomically rebuild affected maintained backings.

**Materialized-view contracts:**
- `PRIMARY KEY = [F]` is mandatory and must name a single field present in the SELECT projection whose values can identify rows ([Validation](#validation)). No auto-derivation. The backing key is NOT NULL and keeps a projected reference's target identity without its removal action. It must also be **unique across the SELECT result** — a non-injective projected PK (two result rows sharing the value) fails loud at CREATE / REFRESH and rolls back, rather than silently dropping the colliding rows. For an aggregating view, use the `GROUP BY` key as the PK.
- The backing carries its canonical structural schema fingerprint immediately at CREATE / REFRESH and preserves that identity across reopen; distinct layouts never share a field-count placeholder fingerprint.
- REFRESH reproduces every user-declared secondary-index definition, including expiry policies, on the shadow before publication. A failed build keeps the old generation and complete index set.
- `CREATE STATISTICS` declarations on the view move to every published replacement generation in its migration outcome; one naming a field the replacement no longer stores as a scalar is dropped.
- Direct DML against any backing is rejected with `MATERIALIZED_VIEW_READ_ONLY`; durable catalog identity drives the decision.
- `ALTER MATERIALIZED VIEW` is an atomic complete replacement and supports MANUAL↔INCREMENTAL transitions. The owner sees the new definition/backing immediately; all old/new sources and the backing stay fenced until settlement. Rows, schema, definition, compatible indexes, and maintenance commit in the same migration, including later source changes and owner writes. It preserves catalog identity/creation time. REFRESH and ALTER rebuild downstream INCREMENTAL views in dependency order, stopping at MANUAL snapshots. Backing triggers survive; rebuilding emits no row events. Invalid dependent projections or trigger bodies abort the migration. EXPLAIN [ANALYZE] REFRESH plans the stored body in either mode without publishing a replacement.
- Row-local INCREMENTAL: one ordinary non-temporal source, optional row-local WHERE, source fields including the source key, and computed SELECT projections. Object/Array containers and nested object paths retain the complete parent once in authored column order. Stored and virtual computed source values work in membership and projection; maintenance enriches only the changed row with the source statement clock. The backing stores ordinary derived fields. Computed siblings in any emitted container are evaluated, as in ordinary nested SELECT projection. Polymorphic Object fields and Array Object elements preserve the discriminator and active extension, including computed values. Nested Object predicates use the active subtype's field type when sibling subtypes declare the same name with different types. Collection-level subtypes retain one-key maintenance for base, extension, and computed fields. Missing extension values become NULL; scalar operands use the active nested layout. Selected fields/containers need compatible subtype declarations. Creation and reopening reject incompatible output shapes, and writes during initial seeding reconcile from the final source envelope. Alias-, collection-, and namespace/collection-qualified source fields bind identically in SELECT, computed expressions, and WHERE. The PRIMARY KEY clause uses the unqualified output name; bindings survive catalog reload. Clock-dependent values are sampled at seed/write time. Row-local-shape diagnostics apply only to this class; broader relational shapes route to full recomputation.
- Per-group INCREMENTAL: one collection without subtypes, `GROUP BY` one stored field that is the `PRIMARY KEY`, a WHERE comparing stored fields with constants, and no DISTINCT, window, ordering, paging, temporal clause or subquery. A write changes only its old and new row's groups: COUNT, integer SUM and AVG, MIN and MAX change the group's row in place from the view's own columns, and a change those columns cannot decide (a removal without a projected `COUNT(*)`, an extremum leaving, HAVING, a FLOAT or DECIMAL sum) recomputes that group alone. Group rows match a REFRESH, a group total outside INT64 fails the write with `INVALID_VALUE`, and triggers see one UPDATE per changed group, an INSERT for a new group and a DELETE for an emptied one.
- Full-recompute INCREMENTAL: multi-source JOIN, grouping outside the per-group class, DISTINCT, windows, scalar projection and WHERE subqueries, expressions that read another SELECT item's alias, ordering/paging, and temporal clauses. Every source is registered; temporal event-queue activation/expiry also triggers maintenance. The recomputed result is applied as its net row changes (deleted, updated and inserted keys; unchanged rows are not written), so backing triggers see exactly what changed in the view.
- Read bodies include leading WITH, set expressions, and derived sources. Hydration, MANUAL refresh, and dependency guards retain all bindings and branches. Malformed or write-bearing catalog bodies report `DDL_CATALOG_BODY_INVALID` before execution.
- Source field/subtype ALTER binds every affected saved query and output key. MANUAL snapshots stay unchanged until REFRESH. INCREMENTAL layout changes and computed-expression changes rebuild affected backings and maintained descendants in the same migration. Unused changes that preserve output keep the backing. Secondary indexes and expiry policies survive rebuild; invalid dependent views or triggers abort the migration. Structural rebuilding emits no row events. Pending replacements/removals participate; commit rechecks current definitions and source generations, rejecting incompatible concurrent publication.
- MANUAL DROP guards apply explicit owner additions, replacements, and removals to the latest committed definitions. Removed or retargeted definitions release their former dependencies immediately. Dismantle plain, MANUAL, or mixed chains outside-in in one migration. Commit rejects conflicting concurrent dependencies even when a removed name is reused. MANUAL creation and refresh can read plain views created or altered earlier in the same migration.

Incremental maintenance completes before user `AFTER` triggers for the same source mutation. A trigger sees the derived row inside the shared transaction; if it rewrites the same source row, that nested write is later and its IVM result remains authoritative. Successful applies emit `mnemic.ivm.maintain_count` (`op`, `matview`) and `mnemic.ivm.maintain_us` (`op`) on the `Mnemic.Engine` meter. INSERT finding an existing backing key or UPDATE finding a missing key emits `mnemic.ivm.reconverge_count` (`matview`), and `EXPLAIN ANALYZE` adds an `IvmMaintain(<matview> <op>)` stage. Telemetry is best-effort and measures completed apply work even if the enclosing transaction later rolls back.

See [`materialized-views.md`](materialized-views.md).

Programmatic CREATE / REFRESH / DROP nodes must carry an exact canonical `[Ns].[Name]`; malformed UTF-16 and non-canonical identities fail before any dependency, catalog, or backing lookup and cannot mutate another materialized view.

## Usage pitfalls


<a id="pitfall-refresh-is-migration-scoped-and-atomically-publishes-a-shadow-backing"></a>

### REFRESH is migration-scoped and atomically publishes a shadow backing

`REFRESH MATERIALIZED VIEW` must run inside `BEGIN MIGRATION`. It builds a shadow backing, atomically republishes it as the view's name (no unknown-collection window and no partial result), streams the SELECT, preserves the old generation on a SELECT error or non-injective PK, and reports `(atomic swap)` in its result message. Aborting the migration preserves the old `LastRefreshedAt` and discards the unpublished shadow. The backing-name swap and timestamp update share one WAL-committed structural transaction: failure before its marker preserves the old generation/timestamp pair, while recovery after its marker restores the new pair. An ordinary failure in the atomic publication callback after the migration is durable returns `CATALOG_PUBLICATION_FAILED_AFTER_COMMIT`, clears migration session state, and records structural-corruption health faults including the underlying `POST_COMMIT_CALLBACK_FAILED`. Direct or nested memory exhaustion and simulated process death instead escape with their original outer exception identity after retained publication owners and migration session state are released. An ordinary failure while recording the callback health fault remains subordinate, but direct or nested simulated process death from that reporter escapes after the same cleanup and stops later publication callbacks. Do not retry either outcome as a rolled-back refresh.

<a id="pitfall-a-non-injective-projected-primary-key-fails-create--refresh-it-does-not-silently-drop-rows"></a>

### A non-injective projected PRIMARY KEY fails CREATE / REFRESH (it does not silently drop rows)

The mat view's backing is a real single-PK collection, so the field named in `WITH (PRIMARY KEY = [F])` must be **unique across the SELECT result**. If two result rows share the value of `[F]`, CREATE (and MANUAL REFRESH) fail loud — "Materialized view '[App].[V]' PRIMARY KEY '…' is not unique across the SELECT result …" — and roll back the half-built backing. They do **not** silently keep the first row and drop the colliding one. Pick a field whose values are unique, or name the `GROUP BY` key of an aggregating view as the PK. (The seed's skip-on-duplicate path exists only for the INCREMENTAL cascade race, never for MANUAL.)

The enclosing migration transaction owns removal of an incomplete CREATE / atomic-REFRESH backing. The statement does not issue a competing out-of-band DROP while that transaction holds the structural drain. If the migration's rollback cleanup itself fails, MQL returns `ROLLBACK_CLEANUP_FAILED` and records an engine-wide structural-corruption health fault. Do not retry blindly: inspect `[Mnemic].[Health]` and recover the transient backing state first.

<a id="pitfall-refresh-rebuilds-every-declared-backing-index-before-publication"></a>

### REFRESH rebuilds every declared backing index before publication

`CREATE INDEX [...] ON [matview] (...)` follows the stable materialized-view identity. `REFRESH MATERIALIZED VIEW` reproduces the complete index definition set on the shadow before swapping it live. If any build fails, the old generation, indexes, and refresh timestamp remain authoritative.

<a id="pitfall-row-local-ivm-preserves-computed-values-and-complete-containers"></a>

### Row-local IVM preserves computed values and complete containers

Virtual computed scalar fields work in WHERE, direct projection, and scalar SELECT expressions. Maintenance enriches the changed source image with the same computed evaluator and statement clock used by query execution and RETURNING. It then updates only that source key's backing row. Source virtual slots stay physically NULL; the backing stores ordinary values without computation flags. Clock-dependent values are sampled at seeding or source mutation, so elapsed time alone does not refresh a view over a non-temporal source.

OBJECT and ARRAY outputs retain their complete container layouts. A nested object path such as `[Detail].[Name]` emits the whole parent once, including computed siblings, even when several selected children surround a computed SELECT expression. Invalid child paths reject before backing publication. Polymorphic Object fields and Array Object elements preserve their discriminator, active extension, and computed descendants through subtype changes. Nested Object predicates resolve each row's active field type, so same-named extension fields with different types do not use another subtype's binary interpretation. Collection-level subtypes also use one-key maintenance, including base and extension computed values. Missing extension fields become NULL. Scalar nested operands use the active subtype's parent layout; selecting the whole parent still requires a compatible result schema across subtypes. Incompatible output shapes reject before the backing or maintenance registration is published.

Source aliases and bare or namespace-qualified collection names work in projection fields, computed expressions, and WHERE, and a declared `Body` field owns its children there as in the body's query. `PRIMARY KEY = [Id]` names the projected key even when SELECT spells it `[i].[Id]`. Creation, maintenance, and reopening use the same bindings and retain authored output names/order.

A body that groups one collection by its key field selects the per-group class; JOIN, other grouping, DISTINCT, window, ordering/paging, and temporal bodies select the full-recompute class. Each class must retain exact seed/write layout agreement.

<a id="pitfall-incremental-seeds-can-follow-migration-writes"></a>

### Incremental seeds can follow migration writes

CREATE INCREMENTAL includes earlier source inserts, updates, and deletes in the same migration. Populated incremental backings can themselves feed another CREATE before COMMIT. Earlier source schema/index DDL is also supported. Such creation retains source fences until commit or abort; other sessions may need to retry source operations after settlement. The ordinary concurrent seed path still applies when the owner has no earlier source writes or retained fences. Ordered scans and indexed predicates read private transaction rows too; do not commit setup DML merely to make those rows queryable.

<a id="pitfall-incremental-create-drains-seed-window-source-transactions-at-commit"></a>

### INCREMENTAL CREATE drains seed-window source transactions at COMMIT

Source writes stay available while an INCREMENTAL backing is initially seeded. Their PKs are journaled and reconciled from final committed source images just before `COMMIT MIGRATION`; this prevents delete-before-seed resurrection and cross-transaction writes into the seed transaction's private pages. If an explicit source transaction that touched the window remains open, migration commit waits for its COMMIT/ROLLBACK up to the engine lock-timeout budget and then aborts rather than publishing uncertain contents. Finish long-running source transactions promptly when an INCREMENTAL CREATE is waiting to commit.

<a id="pitfall-create--rename-collection-rejects-when-the-target-name-collides-with-a-registered-view-or-mat-view"></a>

### CREATE / RENAME COLLECTION rejects when the target name collides with a registered view or mat view

`CREATE COLLECTION [Ns].[X]` and `ALTER COLLECTION [Y] RENAME TO [Ns].[X]` both reject when `[Ns].[X]` is already a registered view or another materialized view; a name that differs from the backing's own only in letter case respells it. Query-source identity must remain unambiguous; otherwise a plain definition and physical collection could compete for the same name. Drop the conflicting view or materialized view first, or pick another name.

<a id="pitfall-alter-collection-is-rejected-when-the-target-is-a-materialized-views-backing"></a>

### ALTER COLLECTION is rejected when the target IS a materialized view's backing

`ALTER COLLECTION ... ADD/DROP/ALTER/RENAME FIELD` against a mat view's backing is rejected at execute time. An independent schema mutation would disagree with the projection-derived definition; MANUAL REFRESH would replace it and INCREMENTAL maintenance could no longer reproduce the stored row shape. Use `ALTER MATERIALIZED VIEW` with the desired projection.

<a id="pitfall-rename-preserves-materialized-definitions-plain-drop-must-use-the-materialized-form"></a>

### Rename preserves materialized definitions; plain DROP must use the materialized form

`ALTER COLLECTION ... RENAME TO` can rename a source or a materialized backing. `ALTER NAMESPACE ... RENAME TO` moves all affected sources and backings together. Saved physical references and maintenance registrations retain their identities; rows, collection identity, indexes, and creation/refresh times stay unchanged. MANUAL snapshots therefore retain their values until REFRESH. Explicit aliases, CTE names, output names, and literals retain their meaning.

Rename composes with earlier CREATE, ALTER, REFRESH, and DROP/CREATE, with later REFRESH, ALTER and DROP by the new name, repeated renames, swaps through a free name, and old-name reuse within one migration. Pending initial seeds retain all dirty keys, including foreign writes across source-only rename. Backing changes retain write fences until settlement. Abort restores the prior names and maintenance. Exclusive migration admission orders definition changes; missing identities, incompatible schema changes and name collisions are rejected. Plain-view DROP keeps CASCADE removal rules; rename preserves static dependencies.

Plain `DROP COLLECTION` against a backing is rejected because it would orphan the durable definition. Use `DROP MATERIALIZED VIEW`.

<a id="pitfall-source-schema-changes-validate-materialized-definitions"></a>

### Source schema changes validate materialized definitions

Source field/subtype ALTER permits compatible edits. Invalid saved queries or missing output keys abort the migration with the dependent view named. MANUAL snapshots keep their prior rows and schema until REFRESH. INCREMENTAL output layout or computed-expression changes rebuild backings and maintained descendants atomically with the source change. Index definitions and expiry policies survive; structural rebuilding emits no per-row trigger events. Current dependent views and trigger bodies must still bind. Competing writers wait or fail within their lock budget until settlement. Unused edits that preserve output retain the backing.

The same rules apply to `ALTER TYPE` changes propagated through EXTENDS and composed fields. All affected source collections change before a shared dependent query is rebound or rebuilt. Virtual and STORED computed-expression replacements refresh maintained values even when the output schema stays the same; abort and reopen preserve the source/backing outcome together. An Object projection of a named type follows its view's refresh mode rather than receiving a separate direct type alteration.

Earlier replacements and removals count. `ALTER MATERIALIZED VIEW` makes its new backing and maintenance behavior available to later statements by the same owner; commit or abort settles the whole migration. Admission orders migrations; publication still validates the exact definitions and source generations.

<a id="pitfall-drop-protects-sources-of-manual-materialized-views"></a>

### DROP protects sources of MANUAL materialized views

The dependency guard checks saved query bodies at DDL time: `DROP COLLECTION` is rejected when any MANUAL body references the collection — anywhere in the body (FROM, JOIN, lateral apply, WHERE / HAVING subquery). `DROP VIEW` is rejected when a MANUAL body reads the view, and `DROP MATERIALIZED VIEW` is rejected when the target's backing is referenced by plain view(s) or another MANUAL body. Allowing any of these operations would persist a body whose next `REFRESH` (or dependent-view query) cannot resolve its source. Drop the dependent materialized view(s) first.

The guard applies explicit owner additions, replacements, and removals to the latest committed definitions. A new MANUAL view protects its current sources; retargeting or dropping it releases former dependencies for later owner DDL. Dismantle plain, MANUAL, or mixed chains outside-in within one migration. Commit rejects a competing dependency on a removed identity even if its name was reused. A MANUAL seed or refresh can read plain views created or altered earlier in the owning migration.

The persisted body retains one complete read query, including leading `WITH` and `UNION`/`INTERSECT`/`EXCEPT` expressions. CREATE, executor initialization, MANUAL `REFRESH`, and dependency checks use the same decoder. Invalid or non-read bodies fail with `DDL_CATALOG_BODY_INVALID`; valid query wrappers retain their bindings and complete operation tree after reopen.

<a id="pitfall-drop-collection-is-rejected-when-an-incremental-mat-view-depends-on-the-source"></a>

### DROP COLLECTION is rejected when an INCREMENTAL mat view depends on the source

If `[A]` is a regular collection and `[B]` is an INCREMENTAL mat view whose source is `[A]`, then `DROP COLLECTION [A]` is rejected at execute time. The drop would strand `[B]`'s maintenance registration, and later reuse of `[A]` could redirect that name-keyed subscription to an unrelated collection. Drop the dependent INCREMENTAL view first.

<a id="pitfall-drop-materialized-view-is-rejected-when-downstream-ivm-mat-views-depend-on-the-target"></a>

### DROP MATERIALIZED VIEW is rejected when downstream IVM mat views depend on the target

If `[B]` is an INCREMENTAL mat view whose source is `[A]`, then `DROP MATERIALIZED VIEW [A]` is rejected at execute time. The drop would strand `[B]`'s maintenance registration, and later reuse of `[A]` could redirect it to an unrelated collection. Run `DROP MATERIALIZED VIEW [B]` first, then drop `[A]`.

<a id="pitfall-refresh-and-alter-preserve-compatible-dependants"></a>

### REFRESH and ALTER preserve compatible dependants

REFRESH works for MANUAL and INCREMENTAL views. REFRESH and ALTER rebuild INCREMENTAL descendants within the same migration and stop propagation at MANUAL snapshots. Backing triggers survive without firing during the structural rebuild; later DML uses ordinary trigger ordering. Invalid saved projections or trigger bodies reject the whole migration. The owner sees replacement rows immediately while other readers see the committed generation. Competing source writes may need to retry after settlement. EXPLAIN [ANALYZE] REFRESH inspects either mode without changing the backing or refresh timestamp.

<a id="pitfall-ivm-mat-view-on-mat-view-chains-are-bounded-at-depth-100"></a>

### IVM mat-view-on-mat-view chains are bounded at depth 100

A chain of INCREMENTAL mat views (each one's source is the previous one's backing) propagates a single source mutation through every stage in the same transaction. The nesting bound shared with foreign-key cascades and trigger bodies limits the chain to 100 levels — a 101st stage fails with `LIMIT_EXCEEDED` and rolls back the source write. A genuine feedback cycle that re-enters the same live `(matview, pk)` pair is rejected (`TRIGGERED_DATA_CHANGE_VIOLATION`) and rolls back the shared source/backing transaction. It is never silently skipped: doing so could commit a later source image while leaving the backing at the outer image.

<a id="pitfall-ivm-mat-view-whose-source-vanished-fails-closed"></a>

### IVM mat view whose source vanished fails closed

If you `DROP COLLECTION [Source]` while an INCREMENTAL mat view's body references `[Source]`, supported MQL rejects the drop. If an embedded host bypass or inconsistent restore nevertheless leaves the catalog row in `[Mnemic].[MaterializedViews]`, the next lazy IVM hydration raises `IVM_SOURCE_MISSING` as catalog corruption. It never disables maintenance while leaving the stale backing collection queryable. Restore the source from trusted backup, or remove and recreate the materialized view only after auditing its backing contents.

Executor initialization also rejects a missing backing with `IVM_BACKING_MISSING`, and rejects a catalog primary key that disagrees with the backing with `IVM_PRIMARY_KEY_MISMATCH`. Row-local INCREMENTAL definitions may alias the source key, but hydration rejects a changed or missing source identity projection with `IVM_DEFINITION_UNSAFE`. Never recreate a missing backing by hand: audit the incident and recreate the materialized view through MQL so row identity is derived consistently.

The persisted `RefreshMode` byte is equally strict: only `0 = MANUAL` and `1 = INCREMENTAL` are accepted. Any other value raises `MATVIEW_REFRESH_MODE_INVALID` while constructing an executor instead of being mistaken for MANUAL and silently suppressing incremental maintenance.

<a id="pitfall-direct-dml-against-a-materialized-view-backing-is-rejected"></a>

### Direct DML against a materialized-view backing is rejected

INSERT, UPDATE, REPLACE, DELETE, SEAL, UNSEAL, and CANCEL VERSION against the user-visible backing return `MATERIALIZED_VIEW_READ_ONLY`. The guard uses the durable materialized-view catalog identity rather than a name pattern and remains effective after reopen. Only seed, refresh, and maintenance paths enter the internal write-authority scope.

<a id="pitfall-materialized-views-store-computed-results-row-local-ivm-evaluates-scalar-projections"></a>

### Materialized views store computed results; row-local IVM evaluates scalar projections

A MANUAL CREATE / REFRESH may select a computed source column or a computed projection. The evaluated result is materialized as ordinary stored backing data, recursively through nested objects, array elements, and subtype extensions; it is not evaluated again when the backing is queried. This also means the projection need not retain the computed expression's source operands.

INCREMENTAL row-local maintenance evaluates scalar computed SELECT projections against each authoritative stored source row, using the same result schema as the seed. Bare virtual computed source fields also work: the changed row is enriched before membership and projection, matching its query-time value.

<a id="pitfall-refresh--incremental-has-row-local-and-full-recompute-cost-classes"></a>

### `REFRESH = INCREMENTAL` has row-local, per-group and full-recompute cost classes

One non-temporal source with an optional row-local WHERE, source fields (including computed values and Object/Array containers) and its primary key, and computed projections uses one-key delta maintenance. A body that groups one collection by the stored field it is keyed by changes only the groups a write touches, in place when the view projects the state the change needs: project `COUNT(*)`, and `COUNT([F])` beside a `SUM([F])` or `AVG([F])` of a nullable field, or every removal recomputes its group; `HAVING`, an expression over aggregates and a `FLOAT` or `DECIMAL` sum recompute every touched group, which scans the collection unless the grouping key is indexed ([maintenance execution classes](#maintenance-execution-classes)). JOIN, other grouping, DISTINCT, windows, scalar projection and WHERE subqueries, expressions that read another SELECT item's alias (`[Qty] + 1 AS [A], [A] * 2 AS [B]`), ORDER BY/paging, and temporal clauses use complete transaction-owned recomputation. Every physical source is registered. These forms are correct but can make one source write pay the complete query cost; inspect IVM telemetry and use MANUAL refresh when that write amplification is unsuitable.

<a id="pitfall-incremental-wildcards-must-preserve-source-identity"></a>

### Incremental wildcards must preserve source identity

Omitted SELECT, `SELECT *`, qualified wildcards, and mixed wildcard lists are supported. Row-local EXCLUDE / REPLACE / RENAME, including nested Object and Array modifiers, keep one-key maintenance. The declared output key must directly project the source key value. An identity alias or wildcard RENAME can change its name: `SELECT [Id] AS [Key]` requires `PRIMARY KEY = [Key]`. Private source metadata is not copied into the backing. Schema ALTER re-expands saved wildcards and rebuilds affected maintained backings; window or subquery replacements use full recomputation.

<a id="pitfall-materialized-view-ddl-is-rejected-inside-an-explicit-transaction"></a>

### Materialized view DDL is rejected inside an explicit transaction

`CREATE / ALTER / REFRESH / DROP MATERIALIZED VIEW` cannot run inside `BEGIN TRANSACTION`. All four run inside a migration; standalone CREATE, ALTER, REFRESH, and DROP are rejected.

Trigger and view DDL is migration-scoped; it is rejected both as a standalone statement and inside `BEGIN TRANSACTION`. The catalog mutation and executable registry change publish at `COMMIT MIGRATION`, and abort publishes neither. Inside the defining migration, queries use the owner's current plain-view definitions, including pending CREATE, ALTER, REPLACE, and DROP. Trigger dispatch continues to use the committed trigger registry. Trigger dependency checks read the migration's own catalog view, so a pending CREATE protects its target and a pending DROP removes that dependency.
