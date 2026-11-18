# Views

A view is a named, persisted SELECT statement. Querying a view inlines its body so the result is exactly what the underlying SELECT would produce — but you write the predicate / projection / join once, in one place, and every caller agrees on it.

## DDL

```mql
CREATE [OR REPLACE] VIEW [Ns].[Name] AS <from-query>;
ALTER VIEW [Ns].[Name] AS <new-from-query>;
ALTER VIEW [Ns].[Name] RENAME TO [Ns].[NewName];
DROP VIEW [Ns].[Name] [CASCADE];
DESCRIBE VIEW [Ns].[Name];

FROM [Mnemic].[Views];  -- catalog of every view in the database
```

### Example

```mql
CREATE VIEW [Sales].[BigOrders] AS
    FROM [Sales].[Orders] WHERE [Total] >= 1000;

-- Use it just like a collection
FROM [Sales].[BigOrders] WHERE [Customer] = 'Acme';

-- View on view
CREATE VIEW [Sales].[HugeOrders] AS
    FROM [Sales].[BigOrders] WHERE [Total] >= 10000;

-- Rename, or move to another namespace; [Sales].[HugeOrders] keeps reading it
ALTER VIEW [Sales].[BigOrders] RENAME TO [Reports].[LargeOrders];
```

`RENAME TO` takes the complete destination name, as `ALTER COLLECTION ... RENAME TO` does: another namespace moves the view there, and that namespace needs no separate creation. The view keeps its ID, body, creation time and every dependent: views, materialized views and trigger bodies that read it bind its identity, so they follow it to the new name without being rewritten, and an INCREMENTAL materialized view keeps maintaining from it. The old name stops resolving and is free for another collection or view, even later in the same migration. Renaming a view to its own name changes nothing; a casing-only change updates the stored spelling. Like every view change, a rename runs in a migration: ABORT restores the old name, and COMMIT publishes the new one to other sessions.

## How inlining works

When a query references a view in `FROM`, `JOIN`, a lateral `CROSS APPLY` / `OUTER APPLY`, or a scalar / `EXISTS` / `IN` subquery inside `WHERE` / `HAVING`, the executor substitutes the view's body before dispatch. The same inlining applies to subqueries inside `UPDATE`, `REPLACE`, `DELETE`, and `CANCEL VERSION` predicates. Set-operation branches share one transitive binding scope, so view references work from either side of `UNION`, `INTERSECT`, and `EXCEPT`. Conceptually:

```mql
-- You write
FROM [Sales].[BigOrders] WHERE [Customer] = 'Acme';

-- Executor runs the equivalent of
WITH [BigOrders] AS (
    FROM [Sales].[Orders] WHERE [Total] >= 1000
)
FROM [BigOrders] WHERE [Customer] = 'Acme';
```

A CTE name is unqualified, so the expansion is shown with a local name; the view itself keeps its qualified identity.

View-on-view chains expand transitively, however long: execution, schema discovery and `EXPLAIN` walk a chain link by link, and the expanded views do not count toward the 9,362 definitions a `WITH` list accepts. Cycles are detected and rejected at query-dispatch time with `User / INVALID_DEFINITION`.

`EXPLAIN` and `EXPLAIN ANALYZE` resolve read queries through transitive plain views. Static EXPLAIN validates schema without reading rows; ANALYZE executes the query. Both honor the current migration's private view definitions.

## Lifecycle and write rules

- **Views are read-only.** `INSERT`, `UPDATE`, `REPLACE`, `DELETE`, `SEAL`, `UNSEAL`, and `CANCEL VERSION` reject a view target with the classified `User / VIEW_READ_ONLY` diagnostic. Target the underlying collection, or use a materialized view when the query result itself must be stored.
- **Every supported query-source position is inlined.** This includes top-level and nested `FROM`, `JOIN`, lateral APPLY, set-operation branches, query or DML `WHERE` / `HAVING` subqueries, and scalar subqueries inside SELECT expressions, CASE arms, and window arguments.
- **The migration owner reads its own view changes.** Queries, DML subqueries, `DESCRIBE VIEW`, and MANUAL materialized-view seeding use earlier CREATE, ALTER, REPLACE, and DROP operations in the same migration. Other sessions see committed definitions until COMMIT; ABORT restores the prior definitions. Read-only checks and relation name checks follow the owner's current state.
- **Relation removal uses Restrict by default.** `DROP COLLECTION` and `DROP VIEW` reject a direct or transitive dependent view, and a view (or a view in the `CASCADE` closure) that a trigger body reads, naming the trigger. Pending additions protect their sources; pending removals and replacements release their former dependencies. Remove a chain outside-in in one migration. Commit checks concurrent dependencies and name conflicts.
- **Renames preserve static dependencies.** Collection, namespace and view renames retain bound collection and view IDs. Stored view bodies and binding payloads remain unchanged; current query text is rendered from those identities. Reusing an old name cannot redirect a stored reference to a new object.
- **Collection schema changes validate dependent plain views.** ADD, DROP, RENAME FIELD, and other schema changes succeed when every affected view still binds against the migration's resulting schema. Wildcard views derive new fields automatically. Checks include join conditions, ordering, aggregate filters, windows, and nested queries without executing the view or requiring its runtime parameter values. An invalid direct or transitive definition names the offending view and aborts the migration. Earlier view replacements or removals in the same migration participate in this schema check. Commit rechecks the latest dependencies: a source generation that standalone structural work changed after the migration read it rolls the migration back with the transient `MIGRATION_CONFLICT`, and running it again reads the published definitions. Materialized-view and trigger protections also apply.
- **Explicit `CASCADE` removes the complete plain-view closure for DROP.** `DROP VIEW` and `DROP COLLECTION` preflight the dependency graph, reject a reachable cycle without mutation (`INVALID_DEFINITION`), and remove dependent views deepest-first. Collection and namespace rename still accept `CASCADE`; their bound static dependencies remain intact and no dependent views are dropped.
- **No materialised column list.** `[Mnemic].[Views]` stores the SELECT body but not its output schema. `DESCRIBE VIEW` plans that body against current source generations, abandons its unconsumed row source, and returns the derived fields in the same one-row `[Mnemic].[Collections]` shape as `DESCRIBE COLLECTION`. A view has no key, so `PrimaryKey` is empty and no field is `IsPrimaryKey` or `IsAutoIncrement`, even when the body projects its source's key; and `Fields` lists only data columns, never the private `Meta` and `Envelope` slots a query appends, which a collection's row omits too.

## Maintained dependents

A plain view can feed an INCREMENTAL materialized view, including through other plain views. Its physical inputs drive full recomputation. ALTER VIEW and CREATE OR REPLACE VIEW validate and rebuild affected maintained generations in the same migration; MANUAL snapshots keep their prior contents. A rename rebuilds nothing: maintenance continues from the same inputs under the view's new name. Invalid output fields, keys, or dependency cycles abort the migration. DROP and CASCADE require removing or retargeting all materialized dependents first. See [materialized views](materialized-views.md#plain-views-as-maintained-sources).

## Validation rules

`CREATE VIEW` rejects:

- a duplicate name (use `CREATE OR REPLACE` to overwrite);
- a body that is not a read query (FROM, WITH, and set expressions are accepted): `CREATE VIEW [App].[V] AS DELETE FROM [App].[Items]` reports `SYNTAX_ERROR` `CREATE VIEW body must be a read query (FROM, WITH or a set operation), not DELETE.`;
- a name in the `[Mnemic]` / `[Mnemic.*]` namespace;
- a name that collides with an existing real collection;
- a body that does not bind: a field its sources do not declare, anywhere in the body (a projection expression, an aggregate, `WHERE`, an ordering, a nested query), or an array element path where one value per row is read (a column, a grouping, ordering or window key, an aggregate or function argument; [element paths read no single value](queries.md#array-element-paths-read-no-single-value)). The body binds over catalog metadata, without reading rows, exactly as a later migration that revalidates the view binds it, so the definition is refused now instead of blocking every later change to its sources: `CREATE VIEW [App].[V] AS FROM [App].[Items] SELECT [Id], [Nope] + 1 AS [X]` reports `UNKNOWN_FIELD` `Invalid definition for view '[App].[V]': Unknown field 'Nope' in expression.` `CREATE OR REPLACE VIEW` and `ALTER VIEW` bind the new body the same way, against the other definitions in effect before the replacement; a reference cycle the new body closes is still rejected at dispatch time.

`ALTER VIEW`, its `RENAME TO` form included, rejects an unknown view name with `UNKNOWN_OBJECT` (use `CREATE VIEW`). `ALTER VIEW ... RENAME TO` applies the name rules of `CREATE VIEW` to its destination: a name a collection, another view or a materialized view already holds is `DUPLICATE_NAME`, a `[Mnemic]` / `[Mnemic.*]` destination is `RESERVED_NAMESPACE`, and a transient-backing segment is `RESERVED_NAME`. `DROP VIEW` rejects an unknown view name, and without `CASCADE` rejects a view that other views reference (see *Lifecycle and write rules* above).

Programmatically constructed `CREATE`, `ALTER`, `ALTER ... RENAME`, and `DROP VIEW` statements must use the same exact canonical `[Ns].[Name]` identity accepted by parsed MQL. Malformed UTF-16 and non-canonical names fail before any dependency or catalog lookup, so they cannot alias a different persisted view.

## Persistence

Views persist to the catalog collection `[Mnemic].[Views]`:

| Field           | Type            | Notes |
|-----------------|-----------------|-------|
| `Id`            | `GUID`          | primary key, synthesized at CREATE |
| `Name`          | `STRING`        | the view's bracketed name. UNIQUE secondary index (case-insensitive) — duplicates are rejected at the catalog level. |
| `NamespaceId`   | `GUID`          | the identity of the view's namespace, stable across a namespace rename |
| `BodyText`      | `STRING`        | the canonical text of the SELECT body, rendered with the current names of the objects it reads (a rename shows the new name), not the text as written |
| `CreatedAt`     | `DATETIMEOFFSET`| at CREATE |
| `LastAlteredAt` | `DATETIMEOFFSET`| at last CREATE / ALTER, a rename included |

Definitions survive engine restart: on first use after reopen, the catalog is scanned and every read body is re-parsed with its complete CTE and set scopes. Malformed or non-read bodies and duplicate persisted names fail closed on first use; Mnemic neither disables the definition nor chooses/deletes a duplicate.

## Comparison with CTEs

Both name a query. Differences:

| | **View** | **CTE** |
|---|---|---|
| Surface | MQL DDL (`CREATE VIEW`) | Inline `WITH [name] AS (...)` |
| Scope | Database-wide, every query sees it | One query only |
| Persistence | Survives engine restart | Lives for one statement |
| Multi-call use | One definition, many callers | Each caller redefines |

Use a view when the same SELECT will be reused across many queries or callers. Use a CTE when the named subquery is local to one statement.

## See also

- [System collections](system-collections.md) — the catalog views, including `[Mnemic].[Views]`.
- [CTEs](cte.md) — the intra-query naming primitive.

## Detailed syntax and execution


<a id="syntax-create--alter--drop--describe-view"></a>

### CREATE / ALTER / DROP / DESCRIBE VIEW

```sql
CREATE [OR REPLACE] VIEW [Ns].[Name] AS <from-query>;
ALTER VIEW [Ns].[Name] AS <new-from-query>;
ALTER VIEW [Ns].[Name] RENAME TO [Ns].[NewName];
DROP VIEW [Ns].[Name] [CASCADE];
DESCRIBE VIEW [Ns].[Name];

FROM [Mnemic].[Views];  -- catalog
```

Views are named SELECTs. Inlined at query dispatch via a synthesized CTE wrap, so a `FROM [Sales].[BigOrders]` runs as if the user had pasted the view's body inline.

**Current rules:**
- Views reject every DML target form (`INSERT`, `UPDATE`, `REPLACE`, `DELETE`, `SEAL`, `UNSEAL`, `CANCEL VERSION`) with `User / VIEW_READ_ONLY`.
- Views auto-inline from top-level/nested FROM, JOIN, lateral APPLY, and scalar / IN / EXISTS subqueries in query or DML predicates.
- `UNION`, `INTERSECT`, and `EXCEPT` branches share one complete transitive view-binding scope.
- Scalar subqueries in SELECT expressions participate in the same view expansion and dependency discovery, including CASE arms and window arguments.
- The migration owner reads its private view definitions in queries, DML subqueries, DESCRIBE VIEW, and MANUAL materialized-view seeding. Earlier CREATE/ALTER/REPLACE/DROP statements take effect immediately for that owner; other sessions see committed definitions until COMMIT. ABORT restores them.
- View names cannot collide with real collections, including pending owner definitions and competing publications. Dropping or renaming a view frees its name for an owner collection, view or materialized view in the same migration, which a rename may give it: the definitions that read either relation stay bound to its identity, and a later drop by the name is judged by the definitions that read the relation the name now holds.
- `ALTER VIEW ... RENAME TO` renames or moves a view to a complete destination name, keeping its ID, body, bindings and creation time; dependent views, materialized views and triggers follow its identity. An identical name changes nothing, and a casing-only change updates the stored spelling.
- Programmatic CREATE / ALTER / DROP nodes must carry an exact canonical `[Ns].[Name]`; malformed UTF-16 and non-canonical identities fail before dependency or catalog lookup and cannot alias another persisted view.
- View bodies are read queries, including leading `WITH`, set expressions, and derived sources. Canonical persistence retains all bindings and branches.
- View-on-view chains expand transitively, however long; cycles are rejected at dispatch time (`INVALID_DEFINITION`).
- Restrict is the default: DROP VIEW is rejected while other views reference the target (transitively); DROP COLLECTION likewise rejects dependent views. Collection and namespace rename preserve their static identity bindings. Pending additions protect their sources; pending removals or replacements release former dependencies. Remove the chain outside-in within one migration. Commit rechecks concurrent dependencies and names.
- Schema-only ALTER COLLECTION permits changes when all affected plain views still bind, including join conditions, ordering, aggregate filters, windows, and nested queries. Validation does not execute the view or require runtime parameter values. Wildcard outputs follow the new schema. Earlier view replacements and removals participate in the check; invalid direct or transitive bodies abort the migration. Commit rechecks the latest view definitions and source generations, rejecting conflicting concurrent changes. Trigger and materialized-view protections still apply.
- `DROP VIEW` and `DROP COLLECTION` accept `CASCADE`; they reject reachable cycles and remove the dependent plain-view closure deepest-first in the same migration outcome. `CASCADE` on collection or namespace rename preserves views.
- `DESCRIBE VIEW` plans the stored body against live source schemas and returns the derived fields in the `DESCRIBE COLLECTION` row shape: an empty `PrimaryKey`, no key or auto-increment field, and data columns only (no `Meta` or `Envelope`). View columns are not persisted separately.
- `[Mnemic]` / `[Mnemic.*]` namespace is rejected.

See [`views.md`](views.md).

## Usage pitfalls


<a id="pitfall-view-references-are-inlined-from-every-supported-query-source-position"></a>

### View references are inlined from every supported query-source position

The view inliner walks top-level and nested `FROM`, every `JOIN` target, lateral `CROSS APPLY` / `OUTER APPLY`, and scalar / `EXISTS` / `IN` subqueries inside query or DML `WHERE` / `HAVING`. View-on-view chains are collected transitively. Scalar subqueries in SELECT expressions, including CASE arms and window arguments, participate in the same traversal. Every scalar subquery must select one column and return at most one row; zero rows produces NULL and a second row raises an error.

`UNION`, `INTERSECT`, and `EXCEPT` give the complete set operation one transitive binding scope. Both branches therefore resolve the same immutable view generation.

<a id="pitfall-views-are-read-only"></a>

### Views are read-only

Every DML form against a view name fails with the stable `User / VIEW_READ_ONLY` diagnostic, including `SEAL`, `UNSEAL`, and `CANCEL VERSION`. If you need stored snapshot semantics, use a materialized view; otherwise write the underlying collection directly.

<a id="pitfall-view-names-cannot-collide-with-collections"></a>

### View names cannot collide with collections

`CREATE VIEW [Sales].[Orders] AS ...` is rejected if `[Sales].[Orders]` already exists as a real collection. FROM resolution would otherwise silently pick one or the other.

<a id="pitfall-referencing-views-restrict-relation-ddl-unless-cascade-is-explicit"></a>

### Referencing views Restrict relation DDL unless CASCADE is explicit

`DROP COLLECTION [X]` rejects references from plain views, including nested queries and transitive view chains; `ALTER COLLECTION [X] RENAME TO …` keeps those views bound to the renamed collection. Schema-only `ALTER COLLECTION` instead binds affected views against the private resulting schema. Unused-field changes and valid additions succeed; removing a field that a dependent view needs names that view and aborts the migration. The checks cover join conditions, ordering, aggregate filters, windows, and nested queries without executing the view or requiring runtime parameters. Earlier view replacements and removals count in this schema validation. Commit rechecks current dependencies and source generations. Exclusive migration admission orders definition changes between sessions. A view created or altered earlier in the active migration protects its new source immediately, so the same migration cannot commit a definition whose source was removed or made incompatible. `DROP VIEW` and `DROP COLLECTION` accept `CASCADE` to remove the complete dependent plain-view closure. A reachable dependency cycle rejects the statement before mutation; an acyclic closure is removed deepest-first and publishes with the relation DDL at commit.

<a id="pitfall-temporal-incremental-views-recompute-on-durable-boundary-delivery"></a>

### Temporal INCREMENTAL views recompute on durable boundary delivery

A temporal source selects the full-recompute maintenance class. Source DML recomputes in its transaction. Future-version activation and scheduled expiry are delivered by the durable temporal event queue, which opens an internal transaction and recomputes the stored query at that boundary. A failed delivery stays retryable in the queue; it is not acknowledged as current derived state.

<a id="pitfall-drop-view-restricts-by-default-and-cascade-removes-an-acyclic-closure"></a>

### DROP VIEW Restricts by default and CASCADE removes an acyclic closure

View-on-view chains are protected the same way collections are: a plain `DROP VIEW` whose target appears in another view's body (FROM, JOIN, or subquery — transitively) is rejected with the dependent view names. The guards apply owner changes to the latest committed registry, so a newly created or altered dependent protects its current sources while a pending removal or retargeting releases former dependencies. Owner queries, DML subqueries, and DESCRIBE VIEW use the same private definitions; other sessions see committed ones until COMMIT. Use `DROP VIEW [Ns].[Name] CASCADE` to preflight and remove an acyclic transitive closure in one migration; cycles reject without mutation.

<a id="pitfall-rename-namespace-is-rejected-when-any-collection-in-it-is-referenced-by-a-view"></a>

### Namespace renames preserve view identities and dependencies

A namespace rename changes the namespace label; its types, enums and plain views keep their namespace ID and local name. View references follow their bound identities, and source-owned triggers follow their targets. `CASCADE` is accepted without removing dependent views. Merging into an existing namespace moves membership transactionally and rejects destination collisions.
<a id="pitfall-plain-view-sources-are-maintained-through-their-physical-inputs"></a>

### Plain-view sources are maintained through their physical inputs

INCREMENTAL materialized views can read plain views, including pending and transitive definitions. Every underlying physical input drives full recomputation. SELECT may be omitted; the result must include the declared key and its values must be unique. ALTER VIEW / CREATE OR REPLACE VIEW atomically rebuild affected INCREMENTAL generations and descendants, with old/new input writers fenced through settlement; MANUAL snapshots retain their contents. Abort restores the previous rows, definitions, and subscriptions. DROP VIEW and CASCADE reject live materialized dependents. Retarget or remove those dependents first. Persisted logical or maintenance cycles fail closed as `IVM_DEFINITION_UNSAFE`.
