# Database diagrams

> **In this section** — `CREATE` / `ALTER` / `DROP DIAGRAM`, box positions and the `FIELDS` choice, `EXPECTED REVISION`, how boxes follow their relations, and the `[Mnemic].[Diagrams]` catalog.

A database diagram is a named layout stored in the database, so every user and client of it sees the same one: a box for each collection or materialized view it draws, placed at integer coordinates, and how much of each relation's schema its boxes show. The engine stores the layout and names each box's relation as the catalog names it now; a client such as Studio draws the boxes and the references between them, which it reads from [`[Mnemic].[Collections]`](system-collections.md#virtual-collections).

## Creating a diagram

```sql
CREATE DIAGRAM [Order flow] ([Shop].[Orders] AT (40, 60), [Shop].[Customers] AT (360, -20)) WITH (FIELDS = KEYS);
CREATE DIAGRAM [Empty] ();
```

A diagram's name is one bracketed name, unique in the database regardless of letter case, as a [trigger](triggers.md)'s is; a name another diagram holds is `DUPLICATE_NAME`. The parentheses hold its boxes, in the order the diagram keeps them, and may hold none. Each box names a collection or a [materialized view](materialized-views.md) and places its top-left corner `AT (x, y)`, two signed INT32 literals; a coordinate INT32 cannot hold, a fraction among them, is `INVALID_VALUE`. A diagram holds at most 1024 boxes (`LIMIT_EXCEEDED`), and names each relation in one box at most (`DUPLICATE_NAME`).

A box cannot draw a relation of the reserved `[Mnemic]` or `[Mnemic.*]` namespaces (`RESERVED_NAMESPACE`) or a plain [view](views.md), which has no stored schema of its own to draw (`WRONG_OBJECT_TYPE`). A name no collection or materialized view holds is `COLLECTION_NOT_FOUND`, and so is the name of a materialized view's replaced generation awaiting reclamation, which no listing shows.

`WITH (FIELDS = …)` stores how much of each relation the boxes show: `ALL`, every field; `KEYS`, the primary key and the fields that reference another relation; `NONE`, the relation's name alone. Without `WITH` a diagram shows `ALL`. A new diagram is at revision 1.

## Changing a diagram

```sql
ALTER DIAGRAM [Order flow] ([Shop].[Orders] AT (40, 80), [Shop].[Customers] AT (360, -20)) WITH (FIELDS = ALL) EXPECTED REVISION @revision;
ALTER DIAGRAM [Order flow] WITH (FIELDS = NONE);
ALTER DIAGRAM [Order flow] () EXPECTED REVISION 4;
ALTER DIAGRAM [Order flow] RENAME TO [Orders] EXPECTED REVISION 5;
```

A box list replaces the diagram's boxes, and `()` clears them; `WITH (FIELDS = …)` replaces what they show. A statement may change either or both, and one that changes neither is a `SYNTAX_ERROR`. The boxes are checked as `CREATE DIAGRAM` checks them. `RENAME TO` gives the diagram another name, which no other diagram may hold; a name that differs from its own only in letter case respells it. A diagram keeps its identity and its creation time through every change.

Each change raises the diagram's revision by one. `EXPECTED REVISION`, a non-negative literal or an `@parameter`, applies the change only while the diagram is at that revision; otherwise the statement is refused with `REVISION_MISMATCH` and changes nothing, so a client that read the diagram at one revision cannot overwrite a change made since. Without it, concurrent changes apply one after the other on the stored diagram, none is lost, and the revision rises by one for each. A parameter that holds no revision is `UNKNOWN_PARAMETER` when absent and `INVALID_VALUE` otherwise, as for [DML](dml.md#concurrency-control).

## Dropping a diagram

```sql
DROP DIAGRAM [Order flow];
```

A name no diagram holds is `UNKNOWN_OBJECT`, as it is for `ALTER DIAGRAM`. There is no `IF EXISTS`: a client that only needs the diagram gone treats `UNKNOWN_OBJECT` as done.

## Reading diagrams

```sql
FROM [Mnemic].[Diagrams] SELECT [Id], [Name], [Revision], [Fields], [Boxes];
FROM [Mnemic].[Diagrams] WHERE [Id] = @id SELECT [Id], [Name], [Revision], [Fields], [Boxes];
```

`[Mnemic].[Diagrams]` has one row per diagram: `Id` (GUID), `Name`, `Revision` (INT64), `Fields` (`All`, `Keys` or `None`), `Boxes`, `CreatedAt` and `LastAlteredAt` ([system collections](system-collections.md#virtual-collections)). `Boxes` is an ARRAY OBJECT of the boxes in their stored order, each with `Collection`, the name the relation has now, the identity the box is bound to, `CollectionId` (INT32) for a collection or `MaterializedViewId` (GUID) for a materialized view, the other zero or empty, and its position `X` and `Y` (INT32). Before the first `CREATE DIAGRAM` the database holds no catalog of diagrams, and the read returns no rows with the same schema.

## Boxes follow their relations

A box is bound to its relation's identity, never its name, so renaming a collection, a materialized view or a namespace moves no box and changes no revision: the diagram reads the new name in the renaming [migration](migrations.md) at once and in every other session once it commits. A `REFRESH MATERIALIZED VIEW` or `ALTER MATERIALIZED VIEW` keeps the view's box on the generation it publishes.

`DROP COLLECTION` and `DROP MATERIALIZED VIEW` remove the dropped relation's boxes from every diagram in the dropping migration's transaction, raising each changed diagram's revision: `ABORT MIGRATION` restores the boxes and the revisions with the relation. A box placed on a relation while another session's migration is dropping it, which the drop could not see, is not read once that migration commits; identities are never reused, so it can never draw another relation.

## Diagram statements stand alone

Each DIAGRAM statement commits on its own. It is refused inside an explicit transaction and inside a migration (`INVALID_TRANSACTION_STATE`), as [`ANALYZE`](analyze.md) is: neither could roll it back, and a diagram is not a schema change, so it records no [migration history](migrations.md), is never replayed by `SCRIPT MIGRATION` and leaves the schema level unchanged. The WAL, backups and point-in-time recovery carry diagrams as they carry rows. The statements require the `Ddl` permission, as view definitions do; reading `[Mnemic].[Diagrams]` requires `Read`. Like any statement, one that runs while a migration commits, or while the first `CREATE DIAGRAM` creates the catalog, can be asked to yield with the transient `CATALOG_PUBLICATION_RETRY`; it changed nothing, and running it again applies it. A statement whose catalog work meets a transient conflict with a concurrent statement, such as two first `CREATE DIAGRAM`s building the catalog, runs that work again itself, up to eight times with a jittered pause, and each such retry counts in the server-side retry totals of [`[Mnemic].[Metrics]`](system-collections.md#virtual-collections).

## Errors

| Code | When |
|------|------|
| `DUPLICATE_NAME` | The name, or a `RENAME TO` name, belongs to another diagram; a box list names a relation twice. |
| `UNKNOWN_OBJECT` | `ALTER` or `DROP DIAGRAM` names no diagram. |
| `REVISION_MISMATCH` | `EXPECTED REVISION` no longer holds. |
| `RESERVED_NAMESPACE` | A box names a `[Mnemic]` or `[Mnemic.*]` relation. |
| `WRONG_OBJECT_TYPE` | A box names a plain view. |
| `COLLECTION_NOT_FOUND` | A box names no collection or materialized view. |
| `INVALID_VALUE` | A coordinate is outside INT32; an `EXPECTED REVISION` parameter holds no revision. |
| `LIMIT_EXCEEDED` | A box list holds more than 1024 boxes. |
| `INVALID_TRANSACTION_STATE` | The statement runs inside an explicit transaction or a migration. |
