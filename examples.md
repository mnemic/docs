# Practical examples

> **In this section** — recipe-style examples that combine multiple MQL features. Each example links back to the relevant reference page.

## Paginated product listing

```sql
FROM [Store].[Products]
WHERE [Active] = TRUE AND [Price] > 0
ORDER BY [Name] ASC
SKIP 40 LIMIT 20
SELECT [Name], [Price], [Category];
```

See [Basic queries](queries.md).

## Types, inheritance and subtypes

A schema can say what classes in code say: a currency is one of a fixed set, money is an amount in a currency, a payment is money, and a payment is made either by card or by invoice, each with fields of its own.

### Declare the types and the subtypes

```sql
BEGIN MIGRATION 0;

CREATE ENUM [Shop].[Currency] (INT16)
VALUES (1 = 'EUR', 2 = 'SEK', 3 = 'USD');

CREATE TYPE [Shop].[Money] AS (
  [Amount]   DECIMAL NOT NULL,
  [Currency] [Shop].[Currency] NOT NULL
) WITH CHECK ([Amount] >= 0);

CREATE TYPE [Shop].[Address] AS (
  [Street]  STRING,
  [City]    STRING,
  [Country] STRING
);

-- A payment is an amount of money, made by card or by invoice.
CREATE COLLECTION [Shop].[Payments] EXTENDS [Shop].[Money]
WITH SCHEMA (
  [Id]     INT32 PRIMARY KEY AUTOINCREMENT,
  [PaidAt] DATETIME NOT NULL
)
WITH SUBTYPE [card] ([Last4] STRING NOT NULL)
WITH SUBTYPE [invoice] (
  [DueDays] INT32 NOT NULL,
  [BillTo]  [Shop].[Address]
);

COMMIT MIGRATION;
```

`[Shop].[Payments]` takes `[Amount]` and `[Currency]` from `[Shop].[Money]`, with its check that an amount is never negative, and every payment is a `[card]` or an `[invoice]`.

### Write and query the subtypes

```sql
INSERT INTO [Shop].[Payments] SUBTYPE [card] (
  [Amount]: 690,
  [Currency]: ENUM_VALUE('EUR', '[Shop].[Currency]'),
  [PaidAt]: NOW,
  [Last4]: '4242'
);

INSERT INTO [Shop].[Payments] SUBTYPE [invoice] (
  [Amount]: 2490,
  [Currency]: ENUM_VALUE('SEK', '[Shop].[Currency]'),
  [PaidAt]: NOW,
  [DueDays]: 30,
  [BillTo]: ([Street]: 'Storgatan 12', [City]: 'Uppsala')
);

FROM [Shop].[Payments]
WHERE SUBTYPE IS [invoice] AND [DueDays] > 14
SELECT [Id], [Amount], [BillTo].[City] AS [City],
       ENUM_LABEL([Currency]) AS [CurrencyName];
```

The query returns the invoice, its amount, `SEK` and `Uppsala`. See [Composite types](composite-types.md), [Polymorphism](polymorphism.md) and [Enums](enums.md).

## Temporal audit trail

```sql
-- All versions of an employee's record across time
FROM [HR].[Employees]
IN BETWEEN '2024-01-01T00:00:00Z' AND NOW
WHERE [Id] = 42
ORDER BY [Meta].[ValidFrom] ASC;
```

See [Temporal queries](temporal.md) and the [Meta fields](system-collections.md#meta-fields).

## Finalize and later reopen a historical record

```mql
-- Optimistic check happens before the idempotent/no-op decision.
SEAL [Archive].[Records]
WHERE [Id] = @id
EXPECTED REVISION @revision
RETURNING [Id], [Meta].[Sealed];

-- Ordinary updates now fail with DOCUMENT_SEALED.
UPDATE [Archive].[Records]
SET ([Summary]: 'corrected')
WHERE [Id] = @id;

-- Reopen the workflow explicitly, then apply the correction.
UNSEAL [Archive].[Records] WHERE [Id] = @id;
UPDATE [Archive].[Records]
SET ([Summary]: 'corrected')
WHERE [Id] = @id;
```

Use `CONFIRM SEAL ALL` only for an intentional collection-wide operation. Sealing is reversible workflow protection, not an immutable audit/WORM policy. See [DML](dml.md#seal-and-unseal).

## Hierarchical query (org chart)

```sql
WITH RECURSIVE [Team] AS (
    FROM [Org].[Employees] WHERE [Id] = 1
    UNION ALL
    FROM [Org].[Employees] [e]
    JOIN [Team] [t] ON [e].[ManagerId] = [t].[Id]
)
FROM [Team]
ORDER BY [Name] ASC
SELECT [Id], [Name], [Title];
```

See [Common Table Expressions](cte.md).

## Join with temporal propagation

```sql
FROM [Orders].[Orders] [o]
AS OF '2024-06-15T00:00:00Z'
JOIN [Store].[Products] [p] ON [o].[ProductId] = [p].[Id]
LEFT JOIN [Store].[Customers] [c] ON [o].[CustomerId] = [c].[Id]
WHERE [o].[Total] > 100
ORDER BY [o].[Total] DESC
LIMIT 50;
```

See [JOIN](queries.md#join) and [Temporal queries](temporal.md).

## Join without the duplicate inner key

```sql
FROM [Orders].[Orders] [o]
JOIN [Store].[Customers] [c]
    ON [o].[CustomerId] = [c].[Id]
    OMIT INNER KEY
SELECT [c].*
RENAME ([Name] AS [CustomerName]);
```

The joined customer `Id` is absent from this wildcard result, while `o.CustomerId` remains available and `SELECT [c].[Id]` can explicitly restore the key when required. `WHERE`, `ORDER BY`, and later JOIN clauses still use the original source schemas.

See [JOIN output key omission](queries.md#join).

## Polymorphic sub-document filtering

```sql
-- Find devices where the Solar component has high power
FROM [App].[Devices]
WHERE [Component].[Power] > 200;

-- Find plants where any array element is a high-power Solar component
FROM [App].[Plants]
WHERE [Components][].[Power] > 200;

-- Correlated: same element must satisfy both conditions
FROM [App].[Plants]
WHERE [Components][] MATCH ([Manufacturer] = 'SunCorp' AND [Power] > 200);
```

See [Polymorphism](polymorphism.md) and [Array queries](arrays.md).

## Transaction with FK cascade

```sql
BEGIN TRANSACTION;

INSERT INTO [Store].[Categories] ([Id]: @catId, [Name]: 'Electronics');
INSERT INTO [Store].[Products] (
  [Id]: @prodId,
  [Name]: 'Widget',
  [CategoryId]: @catId
);

COMMIT;
```

See [Transactions](transactions.md).

## Migration with multiple DDL steps

```sql
BEGIN MIGRATION 2;

ALTER COLLECTION [Store].[Products] ADD FIELD [Weight] DECIMAL DEFAULT 0;
CREATE ENUM [Priority] (INT32) VALUES (1 = 'Low', 2 = 'Medium', 3 = 'High');
ALTER COLLECTION [Store].[Products] ADD FIELD [Priority] [Priority] DEFAULT 2;

CREATE INDEX ON [Store].[Products] ([Category]);
CREATE INDEX ON [Store].[Products] ([Priority]);
COMMIT MIGRATION;
```

See [Migrations](migrations.md).

## Correct the past (backdate a fix)

Temporal collections let you write a version whose validity starts in the past, fixing historical queries without touching today's data:

```sql
-- The salary recorded from January was wrong from April 1. The update supersedes
-- the version valid on April 1: the correction runs to that version's end, and the
-- versions after it are untouched.
UPDATE [HR].[Employees]
SET ([Salary]: 78000)
WHERE [Id] = 42
VALID FROM '2024-04-01T00:00:00Z';

-- Verify the correction appears in a historical AS-OF query
FROM [HR].[Employees] AS OF '2024-04-15T00:00:00Z' WHERE [Id] = 42 SELECT [Salary];

-- To correct only April, overlay the bounded interval instead: the rest of the
-- version it splits keeps its body. INSERT writes the whole document.
INSERT INTO [HR].[Employees] ([Id]: 42, [Name]: 'Ada', [Salary]: 78000)
VALID FROM '2024-04-01T00:00:00Z' TO '2024-05-01T00:00:00Z';
```

`WHERE` reads each document's version valid at `VALID FROM`, so a filter on a field the correction changes matches the historical value, not the current one. A `VALID FROM` before the document's first version, or in a gap of its timeline, matches nothing.

## Scheduled pricing

Two scheduled UPDATEs build a temporary campaign price on top of the current price. Each UPDATE supersedes the version valid at its `VALID FROM` and keeps that version's end, so the second one splits the campaign version the first one scheduled, and running them in the other order gives the same timeline:

```sql
-- Current: 2690 from 2026-05-01, open-ended. Campaign price from Black Friday...
UPDATE [Catalog].[Products] SET ([Price]: 2150) WHERE [Sku] = 'ELE-2001' VALID FROM '2026-11-27T00:00:00Z';
-- ...and back to the regular price from December 1
UPDATE [Catalog].[Products] SET ([Price]: 2690) WHERE [Sku] = 'ELE-2001' VALID FROM '2026-12-01T00:00:00Z';

FROM [Catalog].[Products] IN BETWEEN '2026-05-01T00:00:00Z' AND '2027-01-01T00:00:00Z'
WHERE [Sku] = 'ELE-2001'
ORDER BY [Meta].[ValidFrom]
SELECT [Price], [Meta].[ValidFrom] AS [From], [Meta].[ValidTo] AS [To];
```

| Price | From | To |
|---|---|---|
| 2690 | 2026-05-01 | 2026-11-27 |
| 2150 | 2026-11-27 | 2026-12-01 |
| 2690 | 2026-12-01 | NULL (open) |

Dropping the campaign before it starts is a cancellation of the December version followed by one of the campaign version, each restoring its predecessor's end (`CANCEL VERSION AT '2026-12-01T00:00:00Z' ... UPDATE PREVIOUS VALID TO NULL`, then the same at `'2026-11-27T00:00:00Z'`).

## Top-N per group via CTE + sort

```sql
-- Most recent failed login per user (assumes a successful-or-failed [Status] column)
WITH [Failed] AS (
    FROM [Auth].[Logins] WHERE [Status] = 'failed'
)
FROM [Failed] [f]
ORDER BY [f].[UserId] ASC, [f].[At] DESC
SELECT [f].[UserId], [f].[At], [f].[IpAddress];
```

## Monthly billing — last-active tier per day per customer

The classic "we bill based on the end-of-day state in local time" report. `LAST PER DAY` with an IANA zone gives you one row per span where the tier was stable — three tier changes on March 15 produce one row (the last change), four consecutive days with no changes produce one row (the carry-forward).

```sql
-- One row per (customer, tier-span) for March 2026, buckets aligned to Swedish midnight
FROM [Sales].[Subscriptions]
IN BETWEEN '2026-03-01T00:00:00+01:00' AND '2026-04-01T00:00:00+02:00'
WHERE [CustomerId] = @cid
LAST PER DAY IN TIMEZONE 'Europe/Stockholm'
SELECT [Tier], [Price];
```

If a version was active since February, it carries forward into the first March bucket. If a new version starts on March 12, buckets March 1–11 produce no row for that customer (document-didn't-exist semantics) — match with your billing cycle accordingly.

See [`LAST PER`](temporal.md#last-per-temporal-downsampling).

## Highest salary per month

A month [`SERIES`](queries.md#series) drives an APPLY body over the temporal `[HR].[Employees]`, whose instant is an expression over each month's `[Start]` and `[End]` ([per-period reads](temporal.md#per-period-reads)). The forms differ only in that instant, and each answers a different question.

```sql
-- At the start of each month: a raise effective at 00:00 on the 1st counts for that month
FROM SERIES '2026-01-01T00:00:00Z' TO '2027-01-01T00:00:00Z' EVERY 1 MONTH [m]
CROSS APPLY (
    FROM [HR].[Employees] AS OF [m].[Start]
    ORDER BY [Salary] DESC, [Name]
    LIMIT 1
    SELECT [Name], [Salary]
) AS [top]
SELECT CAST([outer].[Start] AS DATEONLY) AS [Month], [inner].[Name], [inner].[Salary];

-- On each month's last day, at 00:00
FROM SERIES '2026-01-01T00:00:00Z' TO '2027-01-01T00:00:00Z' EVERY 1 MONTH [m]
CROSS APPLY (
    FROM [HR].[Employees] AS OF DATEADD('DAY', -1, [m].[End])
    ORDER BY [Salary] DESC, [Name]
    LIMIT 1
    SELECT [Name], [Salary]
) AS [top]
SELECT CAST(DATEADD('DAY', -1, [outer].[End]) AS DATEONLY) AS [MonthEnd], [inner].[Name], [inner].[Salary];

-- The closing state, one second before the next month: a raise at 00:00 on
-- the 1st belongs to the new month, and a version starting inside the final
-- second also falls to the next month (DATEADD's smallest unit is SECOND)
FROM SERIES '2026-01-01T00:00:00Z' TO '2027-01-01T00:00:00Z' EVERY 1 MONTH [m]
CROSS APPLY (
    FROM [HR].[Employees] AS OF DATEADD('SECOND', -1, [m].[End])
    ORDER BY [Salary] DESC, [Name]
    LIMIT 1
    SELECT [Name], [Salary]
) AS [top]
SELECT CAST([outer].[Start] AS DATEONLY) AS [Month], [inner].[Name], [inner].[Salary];

-- Anyone during the month: the highest salary any version held at any point of it
FROM SERIES '2026-01-01T00:00:00Z' TO '2027-01-01T00:00:00Z' EVERY 1 MONTH [m]
CROSS APPLY (
    FROM [HR].[Employees] IN BETWEEN [m].[Start] AND [m].[End]
    ORDER BY [Salary] DESC, [Name]
    LIMIT 1
    SELECT [Name], [Salary]
) AS [top]
SELECT CAST([outer].[Start] AS DATEONLY) AS [Month], [inner].[Name], [inner].[Salary];

-- Every leader, ties kept: rank inside a CTE and keep rank 1
WITH [ranked] AS (
    FROM SERIES '2026-01-01T00:00:00Z' TO '2027-01-01T00:00:00Z' EVERY 1 MONTH [m]
    CROSS APPLY (FROM [HR].[Employees] AS OF [m].[Start] SELECT [Name], [Salary]) AS [e]
    SELECT CAST([outer].[Start] AS DATEONLY) AS [Month], [inner].[Name] AS [Name], [inner].[Salary] AS [Salary],
        RANK() OVER (PARTITION BY [outer].[Start] ORDER BY [inner].[Salary] DESC) AS [Rank]
)
FROM [ranked] WHERE [Rank] = 1 ORDER BY [Month], [Name] SELECT [Month], [Name], [Salary];

-- Any other instant: compute it in a derived source, here the 15th of each month
FROM (
    FROM SERIES '2026-01-01T00:00:00Z' TO '2027-01-01T00:00:00Z' EVERY 1 MONTH [s]
    SELECT [Start], DATEADD('DAY', 14, [Start]) AS [At]
) [m]
CROSS APPLY (
    FROM [HR].[Employees] AS OF [m].[At]
    ORDER BY [Salary] DESC, [Name]
    LIMIT 1
    SELECT [Name], [Salary]
) AS [top]
SELECT CAST([outer].[At] AS DATEONLY) AS [Day], [inner].[Name], [inner].[Salary];
```

For months in a zone, add `IN TIMEZONE 'Europe/Stockholm'` and write the start in the zone's offset (`'2026-01-01T00:00:00+01:00'`) so each month starts at local midnight.

## Unscheduling a mistakenly-scheduled change

Say a tier change was scheduled for May 1, but you realize it was wrong before the change took effect:

```sql
-- Schedule: tier bumps to 'Platinum' on May 1
INSERT INTO [Sales].[Subscriptions] ([Id]: @id, [Tier]: 'Platinum') VALID FROM '2030-05-01T00:00:00Z';

-- Oh no, typo — meant 'Gold'. Cancel the scheduled version and reopen the prior one
-- in one statement, so no observable state has a gap in it:
CANCEL VERSION AT '2030-05-01T00:00:00Z' FROM [Sales].[Subscriptions]
WHERE [Id] = @id
UPDATE PREVIOUS VALID TO NULL;

-- Then schedule the intended tier:
INSERT INTO [Sales].[Subscriptions] ([Id]: @id, [Tier]: 'Gold') VALID FROM '2030-05-01T00:00:00Z';
```

After the cancellation the May 1 version is physically gone and the prior version, with its own tier, is open-ended again; any existing FKs still resolve. `UPDATE PREVIOUS SET (...)` would instead rewrite the prior version's body over its whole valid interval, so an `AS OF` read of a past instant that version covers would see the new values too: use it to correct the prior version itself, not to schedule a change.

See [`CANCEL VERSION`](dml.md#cancel-version).

## Batch upsert with optimistic concurrency

Client reads rows, computes changes, then writes with `EXPECTED REVISION` to detect concurrent modifications:

```sql
-- Step 1: read with revisions
FROM [Catalog].[Items] WHERE [Id] IN (@id1, @id2, @id3)
SELECT [Id], [Price], [Meta].[Revision];

-- Step 2: one statement per row, per-row revision check. EXPECTED REVISION
-- accepts @parameters so the observed revision can be bound at execution time.
UPDATE [Catalog].[Items] SET ([Price]: @p1) WHERE [Id] = @id1 EXPECTED REVISION @r1 SKIP IF UNCHANGED;
UPDATE [Catalog].[Items] SET ([Price]: @p2) WHERE [Id] = @id2 EXPECTED REVISION @r2 SKIP IF UNCHANGED;
UPDATE [Catalog].[Items] SET ([Price]: @p3) WHERE [Id] = @id3 EXPECTED REVISION @r3 SKIP IF UNCHANGED;
```

Any concurrent writer's revision bump causes the matching `UPDATE` to fail and the client can re-read and retry that row.

## Polymorphic query with subtype fan-out

```sql
-- List the subtype tag of every vehicle (the client aggregates if needed)
FROM [Fleet].[Vehicles] SELECT SUBTYPE, [Make], [Year];

-- Cars with 4+ doors made after 2020
FROM [Fleet].[Vehicles] WHERE SUBTYPE IS [car] AND [Doors] >= 4 AND [Year] > 2020;

-- Trucks above a payload threshold, paginated
FROM [Fleet].[Vehicles]
WHERE SUBTYPE IS [truck] AND [PayloadTons] > 10
ORDER BY [PayloadTons] DESC
SKIP 0 LIMIT 20
SELECT SUBTYPE, [Make], [Year], [PayloadTons], [Axles];
```

See [Polymorphism](polymorphism.md).

## Polymorphic ARRAY OBJECT with correlated MATCH

```sql
-- Tariffs whose Components list has a DelegatedSubscription component
-- with an amount above a threshold AND a specific BenefactorId.
-- SUBTYPE IS works inside MATCH, tested per element.
FROM [Core.Billing.Tariff].[Tariff]
WHERE [Components][] MATCH (
    SUBTYPE IS [DelegatedSubscription]
    AND [Amount] > 100
    AND [BenefactorId] = @benefactorId
)
SELECT [TariffId], [Name];
```

## Soft-delete + temporal resurrection

```sql
-- End a contract (temporal soft delete)
DELETE FROM [HR].[Contracts] WHERE [Id] = @id AT NOW;

-- Resurrect it by inserting a new version starting now
INSERT INTO [HR].[Contracts]
  ([Id]: @id, [EmployeeId]: @empId, [Salary]: 82000)
  VALID FROM NOW;
```

## Introspect the schema via system virtual collections

Every per-field detail lives inside the `Fields` ARRAY OBJECT column on `[Mnemic].[Collections]`. Clients walk the nested elements to extract field-level metadata. Similarly, per-value enum details live inside the `Values` ARRAY OBJECT on `[Mnemic].[Enums]`.

```sql
-- One row per collection, with nested Fields + SubTypes
FROM [Mnemic].[Collections] SELECT [Name], [Fields], [SubTypes];

-- One row per enum, with a nested Values array of (Value, Description)
FROM [Mnemic].[Enums] SELECT [Name], [UnderlyingType], [Values];

-- Migration history on this database
FROM [Mnemic].[Migrations] ORDER BY [Level] ASC SELECT [Level], [Statements], [Timestamp];

-- Indexes, constraints, and foreign keys (PostgreSQL pg_indexes / pg_constraint)
FROM [Mnemic].[Indexes] WHERE [CollectionName] = '[Shop].[Products]';
FROM [Mnemic].[Constraints] WHERE [ConstraintType] = 'ForeignKey';
FROM [Mnemic].[ForeignKeys];

-- Per-collection capacity + freshness, server build, and capabilities
FROM [Mnemic].[CollectionStats] ORDER BY [DocumentCount] DESC;
FROM [Mnemic].[Version];
FROM [Mnemic].[Capabilities];

-- Live operational state (pg_stat_activity / pg_locks)
FROM [Mnemic].[Sessions];
FROM [Mnemic].[Transactions];
FROM [Mnemic].[Locks];
```

See [System virtual collections](system-collections.md).

## Live dashboard wiring

Combine a collection-wide SUBSCRIBE for the live feed with a per-document DEPENDENCY subscription for the detail pane so FK-linked labels refresh without polling:

```sql
-- Client opens the alerts feed
SUBSCRIBE TO [Ops].[Alerts];

-- User clicks alert → open detail pane. SUBSCRIBE WHERE and UNSUBSCRIBE both
-- accept @parameters so the client can bind the id at execution time.
SUBSCRIBE TO [Ops].[Alerts] WHERE [Id] = @alertId;
SUBSCRIBE TO DEPENDENCY [Ops].[Alerts] WHERE [Id] = @alertId;

-- When the pane closes
UNSUBSCRIBE @detailSubId;
UNSUBSCRIBE @dependencySubId;
```

See [Subscriptions](subscriptions.md).

## Schema-level sync with `BEGIN MIGRATION <fromLevel>`

Application code sets an expected schema level on startup; the migration fails fast if the database is on a different level, preventing silent drift:

```sql
-- App expects level 7. If the database is at 7 the migration runs cleanly to 8.
-- If the database is ahead (e.g. 9), the BEGIN fails — the app is out of date.
BEGIN MIGRATION 7;

ALTER COLLECTION [Catalog].[Items] ADD FIELD [Slug] STRING DEFAULT '';

COMMIT MIGRATION;
```

See [Migrations](migrations.md).

## Full DB maintenance cycle

```sql
BACKUP TO '/backups/snapshot-before-compaction.mnemic';
DEFRAGMENT;
SHRINK;
```

Each statement is sent as its own request — all three require no open transaction or migration.

See [Maintenance](maintenance.md).

## Practical query patterns


<a id="example-pinned-string-references"></a>

### Pinned String references

Pins bind a retained revision. String identifiers use the same invariant lowercase equality as the target primary key; byte identifiers use Base64 literals and exact byte equality.

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Policies] (Temporal: TRUE)
WITH SCHEMA ([Id] STRING PRIMARY KEY, [Text] STRING);
CREATE COLLECTION [App].[Approvals]
WITH SCHEMA ([Id] INT32 PRIMARY KEY,
  [Policies] ARRAY STRING PINNED REFERENCES [App].[Policies] ON REMOVE Restrict);
CREATE INDEX ON [App].[Approvals] ([Policies][]);
COMMIT MIGRATION;
BEGIN TRANSACTION;
INSERT INTO [App].[Policies] ([Id]: 'Policy:A', [Text]: 'Approved text')
  VALID FROM '2024-01-01T00:00:00Z';
INSERT INTO [App].[Approvals] ([Id]: 1, [Policies]: [PIN('policy:a', 1)]);
COMMIT;
FROM [App].[Approvals]
WHERE [Policies][] MATCH ([Id] = 'POLICY:A' AND [Revision] = 1)
SELECT [Id];
```

<a id="example-vector-similarity-search"></a>

### Vector similarity search

```mql
BEGIN MIGRATION 1;
CREATE COLLECTION [Search].[Documents] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [Active] BOOL NOT NULL,
  [TenantId] STRING NOT NULL,
  [Embedding] VECTOR(1536) NULLABLE
);
COMMIT MIGRATION;

-- Exact top-k works before any ANN index is installed.
FROM [Search].[Documents]
WHERE [Active] = TRUE
NEAREST [Embedding] TO @query USING COSINE TOP 10
LIMIT 10
SELECT [Id];

-- Optional Vamana acceleration. SEARCH_LIST tunes recall/latency per query.
CREATE VECTOR INDEX [documents_embedding]
ON [Search].[Documents] ([Embedding]) USING VAMANA
WITH (METRIC = COSINE, MAX_DEGREE = 64, BUILD_LIST = 100, ALPHA = 1.2,
      FILTER_FIELDS = ([TenantId]));

FROM [Search].[Documents]
WHERE [Active] = TRUE AND [TenantId] = @tenant
NEAREST APPROXIMATE [Embedding] TO @query USING COSINE TOP 10 WITH (SEARCH_LIST = 200)
LIMIT 10
SELECT [Id];
```

Bind `@query` as a finite `float[]` with exactly 1,536 components. Use `L2_DISTANCE([Embedding], @query)` or `DOT_PRODUCT([Embedding], @query)` when you need an exact scalar predicate or projection rather than top-k syntax.

<a id="example-0-natural-string-and-bytes-identities"></a>

### Natural String and Bytes identities

```sql
BEGIN MIGRATION 0;
CREATE COLLECTION [Catalog].[Countries] WITH SCHEMA (
  [Code] STRING PRIMARY KEY,
  [Name] STRING NOT NULL
);
CREATE COLLECTION [Content].[Blobs] WITH SCHEMA (
  [Digest] BYTES PRIMARY KEY,
  [Label] STRING
);
COMMIT MIGRATION;

INSERT INTO [Catalog].[Countries] ([Code]: 'SE', [Name]: 'Sweden');
FROM [Catalog].[Countries] WHERE [Code] = 'se';

-- BYTES literals are Base64; '' is the valid empty byte-sequence identity.
INSERT INTO [Content].[Blobs] ([Digest]: 'AQID', [Label]: 'three bytes');
FROM [Content].[Blobs] WHERE [Digest] = 'AQID';
-- Unsigned raw-byte order; strict prefixes precede their extensions.
FROM [Content].[Blobs] ORDER BY [Digest] ASC;
-- Parameters are byte arrays; this range uses the same raw-byte comparison.
FROM [Content].[Blobs] WHERE [Digest] >= @start AND [Digest] < @end
ORDER BY [Digest];
```

String identity is invariant-lowercase while preserving the stored spelling. Bytes identity is exact. Variable identities must fit on the row's home page.

<a id="example-1-paginated-list-with-stable-sort"></a>

### Paginated list with stable sort

```sql
FROM [Shop].[Orders]
WHERE [CustomerId] = @customer
ORDER BY [PlacedAt] DESC, [Id] ASC
SKIP @offset LIMIT @pageSize
SELECT [Id], [Total], [Status], [PlacedAt];
```

Always include the PK as a tiebreaker in `ORDER BY` — without it two rows with the same `PlacedAt` could swap positions across page boundaries.

<a id="example-2-count--page-in-one-round-trip"></a>

### Count + page in one round-trip

Use `EXPLAIN ANALYZE` if you suspect the count is double-counting, otherwise issue two queries:

```sql
FROM [Shop].[Orders] WHERE [CustomerId] = @c SELECT COUNT(*) AS [Total];

FROM [Shop].[Orders] WHERE [CustomerId] = @c
ORDER BY [PlacedAt] DESC, [Id]
SKIP @offset LIMIT @pageSize
SELECT [Id], [Total], [PlacedAt];
```

(The typed client offers `IncludeTotalCount = true` to bundle them on the wire.)

<a id="example-3-joined-query-with-aggregate"></a>

### Joined query with aggregate

```sql
FROM [Shop].[Customers] [c]
JOIN [Shop].[Orders] [o] ON [c].[Id] = [o].[CustomerId]
WHERE [c].[Region] = 'EU' AND [o].[Status] = 'paid'
GROUP BY [c].[Id], [c].[Name]
ORDER BY [Revenue] DESC
LIMIT 25
SELECT [c].[Id], [c].[Name],
       COUNT(*) AS [OrderCount],
       SUM([o].[Total]) AS [Revenue];
```

<a id="example-distinct-aggregate-values"></a>

### Distinct aggregate values

```sql
-- Return each surviving revenue value once, even when regions share a total.
FROM [Shop].[Orders]
GROUP BY [Region]
HAVING [Revenue] > 10000
ORDER BY [Revenue] DESC
LIMIT 10
SELECT DISTINCT SUM([Amount]) AS [Revenue];
```

The region is a grouping key but is absent from the final tuple, so it does not keep equal revenues separate. HAVING filters groups first; DISTINCT then compares selected values before final ordering and paging.

<a id="example-3a-cyclic-join-fieldfield-where-edge--flat-as-output"></a>

### Cyclic join (field=field WHERE edge) + flat AS output

```sql
-- Orders whose customer region matches the fulfilling warehouse region.
-- The [c].[Region] = [w].[Region] edge closes the triangle (o–c, o–w, c–w),
-- making the join graph cyclic; the planner may run it as a bushy plan.
-- AS aliases flatten the result to top-level [OrderId] / [Customer] / [Warehouse].
FROM [Sales].[Orders] [o]
JOIN [Sales].[Customers]  [c] ON [o].[CustomerId]  = [c].[Id]
JOIN [Sales].[Warehouses] [w] ON [o].[WarehouseId] = [w].[Id]
WHERE [c].[Region] = [w].[Region]
SELECT [o].[Id] AS [OrderId], [c].[Name] AS [Customer], [w].[Name] AS [Warehouse];
```

Join order is automatic — write the `JOIN`s in any order; the planner reorders for cost (run `ANALYZE` first). `WHERE [c].[Region] = [w].[Region]` is an extra equi-join edge, not a row filter.

<a id="example-3b-omit-a-duplicate-joined-key-from-output"></a>

### Omit a duplicate joined key from output

```sql
FROM [Sales].[Orders] [o]
JOIN [Sales].[Customers] [c]
    ON [o].[CustomerId] = [c].[Id]
    OMIT INNER KEY
SELECT [c].*
RENAME ([Name] AS [CustomerName]);
```

The customer `Id` is omitted from the qualified wildcard, while `o.CustomerId` remains. The omitted key can still be used by later JOIN, WHERE, and ORDER BY clauses, or restored with an explicit `SELECT [c].[Id]` projection.

<a id="example-3c-cross-apply--top-n-rows-per-outer"></a>

### CROSS APPLY — top-N rows per outer

```sql
-- Latest 3 shipments per order (per-outer top-N — JOIN can't express this)
FROM [Orders].[Orders] [o]
CROSS APPLY (
    FROM [Logistics].[Shipments]
    WHERE [OrderId] = [o].[Id]
    ORDER BY [ShippedAt] DESC
    LIMIT 3
    SELECT [Id], [ShippedAt], [Carrier]
) AS [s];
```

INNER semantics: orders with zero shipments are dropped. For LEFT-OUTER semantics (orders survive with `inner=null` when no shipments match), swap `CROSS APPLY` for `OUTER APPLY`. The merged result lands in `{outer:Object, inner:Object?}` — same shape as conventional joins. The statement aliases do not rename those envelope fields. JOIN and APPLY may be chained, with each APPLY referencing earlier aliases. Final filtering, grouping, windows, projection, sorting, and paging consume the merged rows. Root temporal clauses select the physical outer source. If the outer order source is empty, the inner shipment query is not executed to obtain column metadata. Preparing the result makes no extra calls to functions used inside the shipment projection.

<a id="example-4-as-of-temporal-snapshot"></a>

### AS OF temporal snapshot

```sql
FROM [HR].[Contracts]
AS OF '2024-06-01T00:00:00Z'
WHERE [EmployeeId] = @id
SELECT *;
```

Compare with current state by running the same query without `AS OF`. To track changes between two timestamps:

```sql
FROM [HR].[Contracts]
IN BETWEEN '2024-01-01T00:00:00Z' AND '2024-12-31T23:59:59Z'
WHERE [EmployeeId] = @id
ORDER BY [Meta].[ValidFrom] ASC
SELECT [Meta].[ValidFrom], [Meta].[ValidTo], [Salary], [Title];
```

<a id="example-5-audit-trail-across-all-of-yesterday"></a>

### Audit trail across all of yesterday

```sql
FROM [Audit].[Events]
WHERE [Meta].[ValidFrom] >= @yesterdayStart
  AND [Meta].[ValidFrom] <  @yesterdayEnd
ORDER BY [Meta].[ValidFrom] ASC
SELECT [Meta].[ValidFrom], [User], [Action], [Target];
```

<a id="example-6-hierarchy-traversal-recursive-cte"></a>

### Hierarchy traversal (recursive CTE)

```sql
WITH RECURSIVE [Subtree] AS (
    FROM [Org].[Employees] WHERE [Id] = @rootId
    UNION ALL
    FROM [Org].[Employees] [e]
    JOIN [Subtree] [s] ON [e].[ManagerId] = [s].[Id]
)
FROM [Subtree]
ORDER BY [Name]
SELECT [Id], [Name], [Title];
```

Recursion is capped at 100 levels — set a depth column in the seed and a guard in the recursive WHERE for deeper trees.

<a id="example-7-upsert--update-on-conflict"></a>

### UPSERT — update on conflict

```sql
INSERT INTO [App].[Settings] ([Key]: 'theme', [Value]: 'dark', [UpdatedAt]: NOW)
ON CONFLICT ([Key]) DO UPDATE SET ([Value]: 'dark', [UpdatedAt]: NOW);
```

PK target: `ON CONFLICT DO ...` (target implied) or `ON CONFLICT ([Id]) DO ...`. Secondary UNIQUE target: target list must align exactly with a registered `UNIQUE` index.

<a id="example-8-atomic-compare-and-swap-with-optimistic-concurrency"></a>

### Atomic compare-and-swap with optimistic concurrency

```sql
UPDATE [Account].[Balances]
SET ([Balance]: [Balance] + @delta)
WHERE [AccountId] = @id
EXPECTED REVISION @rev
RETURNING [Balance], [Meta].[Revision];
```

On revision mismatch the engine throws — the caller refreshes `@rev` from a fresh SELECT and retries.

<a id="example-8a-finalize-and-later-reopen-a-historical-record"></a>

### Finalize and later reopen a historical record

```mql
SEAL [Archive].[Records]
WHERE [Id] = @id
EXPECTED REVISION @rev
RETURNING [Id], [Meta].[Sealed];

-- Later, when an intentional correction is authorized:
UNSEAL [Archive].[Records] WHERE [Id] = @id;
UPDATE [Archive].[Records]
SET ([Summary]: 'corrected')
WHERE [Id] = @id;
```

Use `CONFIRM SEAL ALL` only for an intentional sweep. SEAL is reversible workflow protection; it is not immutable/WORM retention.

<a id="example-9-bulk-update-with-a-confirmed-sweep"></a>

### Bulk update with a confirmed sweep

```sql
UPDATE [App].[Items]
SET ([Status]: 'archived', [ArchivedAt]: NOW)
CONFIRM UPDATE ALL;
```

Required when no `WHERE` is given. Without `CONFIRM UPDATE ALL` the parser refuses.

<a id="example-10-delete-with-a-cascade-aware-cleanup"></a>

### Delete with a cascade-aware cleanup

```sql
-- Will throw if any [Order] still REFERENCES this customer with ON REMOVE Restrict
DELETE FROM [Shop].[Customers] WHERE [Id] = @id;
```

To survive the cascade, schema must declare `ON REMOVE Cascade` (or `SetNull` on the dependent FK).

<a id="example-11-polymorphic-insert"></a>

### Polymorphic INSERT

```sql
INSERT INTO [Fleet].[Vehicles] SUBTYPE [car] (
  [Id]: '...', [Make]: 'Toyota', [Year]: 2024, [Doors]: 4, [Sunroof]: TRUE
);
INSERT INTO [Fleet].[Vehicles] SUBTYPE [truck] (
  [Id]: '...', [Make]: 'Ford', [Year]: 2024, [PayloadTons]: 1.5, [Axles]: 2
);
```

Filter by subtype: `WHERE SUBTYPE IS [car]`, `SUBTYPE IS NOT [car]`, `SUBTYPE IN ([car], [truck])`, `SUBTYPE IS @t` — unknown names error. Inside MATCH: `[Components][] MATCH (SUBTYPE IS [battery] AND ...)`. Field-level subtype on nested object: `[Component]: SUBTYPE [Solar] ([Power]: 100)`.

<a id="example-12-pinned-fk--bind-to-a-specific-version"></a>

### Pinned FK — bind to a specific version

```sql
INSERT INTO [App].[Snapshots] (
  [Id]: '...',
  [ContractRef]: PIN(@contractId, @contractRevision)
);
```

Schema for the FK: `[ContractRef] GUID PINNED REFERENCES [HR].[Contracts]`.

<a id="example-pinned-array-fk--multiple-bound-versions-per-row"></a>

### Pinned-array FK — multiple bound versions per row

```sql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Invoices] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [TariffRefs] ARRAY GUID PINNED REFERENCES [App].[Tariffs]
    ON REMOVE Restrict
);
COMMIT MIGRATION;

INSERT INTO [App].[Invoices] (
  [Id]: '...',
  [TariffRefs]: [PIN(@t1, 3), PIN(@t2, 7)]
);
```

Each array element binds to its own `(id, revision)` pair — laid out on disk as the per-element extended-width tail. `Restrict` blocks `CANCEL VERSION AT` when any element binds the cancelled revision; `SetNull` drops the matching elements (the rest of the array is preserved).

<a id="example-pinned-fk-in-a-polymorphic-extension-subtype"></a>

### Pinned FK in a polymorphic-extension subtype

```sql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Invoices] WITH SCHEMA (
  [Id] GUID PRIMARY KEY, [Kind] STRING
)
WITH SUBTYPE [Paid] (
  [TariffRef] GUID PINNED REFERENCES [App].[Tariffs] ON REMOVE Restrict,
  [PaidAmount] INT32
)
WITH SUBTYPE [Free] ([Note] STRING);
COMMIT MIGRATION;

INSERT INTO [App].[Invoices] SUBTYPE [Paid] (
  [Id]: '...', [Kind]: 'Paid',
  [TariffRef]: PIN('a1b2-...-...', 1), [PaidAmount]: 500
);
```

Same shape inside a field-level polymorphic subtype:

```sql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Lines] WITH SCHEMA (
  [Id] GUID PRIMARY KEY,
  [Component] OBJECT ([Amount] INT32 NOT NULL)
    WITH SUBTYPE [EnergyFixed] ([Watts] INT32)
    WITH SUBTYPE [StationFee] ([TariffRef] GUID PINNED REFERENCES [App].[Tariffs] ON REMOVE Restrict)
);
COMMIT MIGRATION;

INSERT INTO [App].[Lines] (
  [Id]: '...',
  [Component]: SUBTYPE [StationFee] ([Amount]: 50, [TariffRef]: PIN('...', 1))
);
```

The pinned slot's extended `[id][rev:8]` width rides on the extension binary; cascade enforcement (`CANCEL VERSION AT` Restrict / SetNull) and `.Include` hydration both work uniformly across base- and extension-resident pinned FKs.

Filter by per-element revision via the `[].[Revision]` pseudo-field:

```sql
-- "Find invoices that pin some tariff at revision 7."
FROM [App].[Invoices] WHERE [TariffRefs][].[Revision] = 7;

-- Range / set forms work the same way.
FROM [App].[Invoices] WHERE [TariffRefs][].[Revision] >= 5;
FROM [App].[Invoices] WHERE [TariffRefs][].[Revision] IN (1, 2, 3);
```

Use MATCH to correlate id and revision on the *same* element:

```sql
-- "Find invoices that pin tariff @t at revision 7." Independent ANY
-- predicates would over-match (any element with id @t plus any element
-- at rev 7); MATCH keeps the correlation tight.
FROM [App].[Invoices]
WHERE [TariffRefs][] MATCH ([Id] = @t AND [Revision] = 7);
```

<a id="example-124b-array-element-gin-index-for-membership-queries"></a>

### Array-element (GIN) index for membership queries

Index a scalar-element array so `WHERE [Field][] = value` (ANY) seeks instead of scanning. Created inside a migration like any DDL:

```sql
BEGIN MIGRATION 5;
CREATE INDEX ON [App].[Posts] ([Tags][]);     -- [Tags] is ARRAY STRING
-- Use CREATE UNIQUE INDEX instead to reserve each tag for one document.
-- Repeated tags in that document are allowed; a duplicate in another is rejected.
COMMIT MIGRATION;

-- Membership seek (EXPLAIN → Strategy = ArrayMembership). Case-folded.
FROM [App].[Posts] WHERE [Tags][] = 'urgent' SELECT [Id];

-- IN-list membership: ANY element in the set. Unions the per-value posting-list
-- seeks (each matching doc once) — also Strategy = ArrayMembership.
FROM [App].[Posts] WHERE [Tags][] IN ('urgent', 'blocker') SELECT [Id];

-- Range membership: ANY element >= the bound. Scans the ordered element keys
-- and de-duplicates pointers — also Strategy = ArrayMembership.
FROM [App].[Scores] WHERE [Vals][] >= 90 SELECT [Id];

-- Predicates are independent existentials: these may be satisfied by different
-- elements. One side can use the index while the other remains a residual.
FROM [App].[Scores] WHERE [Vals][] > 5 AND [Vals][] < 10 SELECT [Id];

-- Object-element: index a scalar sub-field within each ARRAY OBJECT element.
BEGIN MIGRATION 6;
CREATE INDEX ON [App].[Orders] ([Items][].[Sku]);   -- [Items] is ARRAY OBJECT ([Sku] STRING, ...)
COMMIT MIGRATION;
FROM [App].[Orders] WHERE [Items][].[Sku] = 'A1' SELECT [Id];   -- ANY element's Sku = 'A1'
```

The ANY membership shapes use the index — equality, `IN`, and ranges, on a scalar array (`[Field][]`) or a nested scalar of an object element (`[Items][].[Sub]`); `ALL`, `!=`, `NOT IN`, and explicit NULL comparisons use value/coverage candidates with residual checks. For example, `[Items][].[Sub] NOT IN (7, NULL)` matches a reached non-null leaf other than 7, but no null leaf ([NULL equality](queries.md#where-operators)); it still uses the array index. Repeated explicit boundaries such as `[Groups][].[Values][]` and `[Matrix][][]` are indexable too. Uncorrelated subqueries also use the array index, for example:

```mql
FROM [App].[Rows]
WHERE [Data][].[Value] IN (FROM [App].[Probes] SELECT [Value])
SELECT [Id];
```

The runtime set retains its spill budget; the index scans distinct values and rechecks candidate rows. Use nested `MATCH` predicates to express correlation across arrays as shown below.

<a id="example-124c-arrays-owned-by-collection-subtypes"></a>

### Arrays owned by collection subtypes

This example uses a fresh database. All declaring subtypes share one scalar key type; a subtype without the array contributes coverage rather than a scalar-value posting.

```sql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Assets] WITH SCHEMA ([Id] INT32 PRIMARY KEY)
WITH SUBTYPE [Tagged] ([Tags] ARRAY STRING)
WITH SUBTYPE [Archived] ([Note] STRING, [Tags] ARRAY STRING)
WITH SUBTYPE [Other] ([Note] STRING);
CREATE INDEX ON [App].[Assets] ([Tags][]);
COMMIT MIGRATION;

INSERT INTO [App].[Assets] SUBTYPE [Tagged] ([Id]: 1, [Tags]: ['urgent']);
EXPLAIN FROM [App].[Assets] WHERE [Tags][] = 'urgent' SELECT [Id];
```

The plan uses `ArrayMembership`. Arrays inside an extension's Object containers and scalar leaves inside its Object arrays use the same syntax and ownership rule.

<a id="example-124d-indexing-through-repeated-array-boundaries"></a>

### Indexing through repeated array boundaries

This example uses a fresh database. Every `[]` names one array dimension.

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Batches] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Groups] ARRAY OBJECT ([Values] ARRAY INT32)
);
CREATE INDEX ON [App].[Batches] ([Groups][].[Values][]);
COMMIT MIGRATION;
INSERT INTO [App].[Batches] ([Id]: 1, [Groups]: [([Values]: [3]), ([Values]: [12])]);
FROM [App].[Batches] WHERE [Groups][].[Values][] > 5 AND [Groups][].[Values][] < 10 SELECT [Id];
```

The row matches: different inner values satisfy the independent comparisons. The same index can serve negative and universal predicates:

```mql
INSERT INTO [App].[Batches] ([Id]: 2, [Groups]: []);
FROM [App].[Batches] WHERE [Groups][].[Values][] NOT IN (3) SELECT [Id]; -- Id 1: value 12 qualifies
FROM [App].[Batches] WHERE [Groups][].[Values][] > 0 ALL SELECT [Id];    -- Ids 1 and 2; empty is vacuously true
```

Use `[Groups][] MATCH (...)` to require the same Group, and another nested `MATCH` when several comparisons must hold on one inner object element.

A compound array index can scope membership by a stored scalar field. In a fresh database:

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[ScopedValues] WITH SCHEMA ([Id] INT32 PRIMARY KEY, [Tenant] INT32 NOT NULL, [Values] ARRAY INT32);
CREATE UNIQUE INDEX ON [App].[ScopedValues] ([Tenant], [Values][]);
COMMIT MIGRATION;
BEGIN TRANSACTION;
INSERT INTO [App].[ScopedValues] ([Id]: 1, [Tenant]: 10, [Values]: [7, 9]);
INSERT INTO [App].[ScopedValues] ([Id]: 2, [Tenant]: 20, [Values]: [7]);
COMMIT;
FROM [App].[ScopedValues] WHERE [Tenant] = 10 AND [Values][] = 7 SELECT [Id];
FROM [App].[ScopedValues] WHERE [Tenant] IN (10) AND [Values][] IN (7, 7) AND [Values][] = 9 SELECT [Id];
FROM [App].[ScopedValues] WHERE [Tenant] = 10 AND [Values][] > 6 ALL SELECT [Id];
FROM [App].[ScopedValues] WHERE [Tenant] = 10 AND [Values][] NOT IN (7) SELECT [Id];
```

These queries return Id 1. The second intersects independent membership predicates. Reusing value 7 in another tenant is allowed; another row containing value 9 in tenant 10 conflicts with the unique index.

<a id="example-125-nested-array-element-predicate--countpredicate-shapes"></a>

### Nested array-element predicate / `.Count(predicate)` shapes

Use a nested array-element predicate inside `MATCH` for the "some outer element has some inner element …" shape (LINQ `xs.Any(o => o.Inner.Any(i => …))`):

```sql
-- Hubs with at least one station whose Tags array carries 'urgent'.
FROM [Facility].[Hubs]
WHERE [Stations][] MATCH ([Tags][] = 'urgent');
```

Use `LENGTH(predicate)` to filter by *count* of matching elements (LINQ `xs.Count(predicate)` on an array field):

```sql
-- Items with more than two 'urgent' tags.
FROM [App].[Items]
WHERE LENGTH([Tags][] = 'urgent') > 2;

-- Hubs with at least one high-power station (equivalent to `[Stations][].[Power] > 100`,
-- but reads naturally when paired with other counts).
FROM [Facility].[Hubs]
WHERE LENGTH([Stations][].[Power] > 100) > 0;
```

<a id="example-13-window-function--running-total-per-group"></a>

### Window function — running total per group

```sql
FROM [Shop].[Orders]
SELECT [Id], [CustomerId], [Total], [PlacedAt],
       SUM([Total]) OVER (PARTITION BY [CustomerId] ORDER BY [PlacedAt]) AS [RunningTotal],
       ROW_NUMBER() OVER (PARTITION BY [CustomerId] ORDER BY [PlacedAt]) AS [OrderSeq];
```

The default frame for an aggregate-with-`ORDER BY` is `UNBOUNDED PRECEDING TO CURRENT ROW`.

<a id="example-aggregate-over-window-with-filter-where-"></a>

### Aggregate-over-window with `FILTER (WHERE …)`

```sql
-- Running total of paid orders only — other statuses don't move the
-- accumulator but still appear in the result set.
FROM [Shop].[Orders]
SELECT [Id], [CustomerId], [Status], [Total],
       SUM([Total]) FILTER (WHERE [Status] = 'paid')
         OVER (PARTITION BY [CustomerId] ORDER BY [PlacedAt]) AS [PaidRunning];
```

<a id="example-groups-frame--count-distinct-peer-groups"></a>

### `GROUPS` frame — count distinct peer-groups

```sql
-- For each row, COUNT rows whose ORDER BY tuple sits in the current
-- peer-group, the one before it, or the one after — useful when ties
-- on the order-by column should be treated as a single "step".
FROM [Sales].[Orders]
SELECT [Customer], [Date], [Amount],
       COUNT(*) OVER (
         PARTITION BY [Customer]
         ORDER BY [Date] ASC
         GROUPS BETWEEN 1 PRECEDING AND 1 FOLLOWING) AS [AdjacentDays];
```

<a id="example-percentile-aggregates"></a>

### Percentile aggregates

```sql
-- Per-region median and 90th percentile. PERCENTILE_CONT interpolates
-- between adjacent ranked values; PERCENTILE_DISC returns the exact
-- ranked value at the requested probability.
FROM [Sales].[Orders]
SELECT [Region], [Amount],
       PERCENTILE_CONT(0.5) OVER (PARTITION BY [Region] ORDER BY [Amount] ASC) AS [Median],
       PERCENTILE_DISC(0.9) OVER (PARTITION BY [Region] ORDER BY [Amount] ASC) AS [P90];
```

<a id="example-14-top-n-per-group-with-last-per"></a>

### Top-N per group with `LAST PER`

```sql
FROM [Sensors].[Readings] IN BETWEEN @from AND @to
WHERE [SensorId] = @id
LAST PER MINUTE IN TIMEZONE 'Europe/Stockholm'
SELECT [Meta].[ValidFrom], [Value];
```

Yields the last row per minute bucket, downsampling a high-rate stream to the chart resolution you actually need.

<a id="example-15-schema-discovery-on-an-unknown-collection"></a>

### Schema discovery on an unknown collection

```sql
DESCRIBE COLLECTION [Shop].[Orders];
```

Returns the schema (fields, types, NOT NULL / NULLABLE / DEFAULT, FK targets, CHECK constraints, indexes). Use this **before** authoring queries against an unfamiliar collection.

For a system-wide view:

```sql
FROM [Mnemic].[Collections] SELECT [Name], [IsTemporal], [PrimaryKey];
FROM [Mnemic].[Enums]       SELECT [Name], [UnderlyingType];
FROM [Mnemic].[Migrations]  ORDER BY [Level] DESC SELECT *;
```

Catalog + operational introspection (PostgreSQL `pg_*` parity):

```sql
-- Every index on a collection (pg_indexes)
FROM [Mnemic].[Indexes] WHERE [CollectionName] = '[Shop].[Products]'
    SELECT [IndexName], [Kind], [KeyFields], [IsUnique], [FilterText];

-- All constraints / foreign keys (pg_constraint)
FROM [Mnemic].[Constraints] WHERE [ConstraintType] = 'Check';
FROM [Mnemic].[ForeignKeys] WHERE [ReferencedCollection] = '[Shop].[Categories]';

-- Per-collection capacity + freshness (pg_stat_user_tables)
FROM [Mnemic].[CollectionStats] ORDER BY [DocumentCount] DESC;

-- Server metadata + capabilities
FROM [Mnemic].[Version];
FROM [Mnemic].[Capabilities] WHERE [Category] = 'Transactions';

-- Live operational state (pg_stat_activity / pg_locks)
FROM [Mnemic].[Sessions];        -- server-only; empty in-process
FROM [Mnemic].[Transactions];    -- explicit in-flight transactions
FROM [Mnemic].[Locks];           -- participating locks; Granted = false means contention
```

<a id="example-16-migration-block--the-canonical-shape"></a>

### Migration block — the canonical shape

```sql
BEGIN MIGRATION 1;
CREATE COLLECTION [Shop].[Orders] (Temporal: TRUE) WITH SCHEMA (
    [Id]         GUID    PRIMARY KEY,
    [CustomerId] GUID    REFERENCES [Shop].[Customers] ON REMOVE Restrict,
    [Total]      DECIMAL NOT NULL,
    [Status]     STRING  NOT NULL,
    [PlacedAt]   DATETIME
);
CREATE INDEX ON [Shop].[Orders] ([CustomerId]);
CREATE INDEX ON [Shop].[Orders] ([Status]) WHERE [Status] != 'archived';
COMMIT MIGRATION;
```

Each statement ends at its top-level `;`; line breaks and indentation are only formatting ([multi-statement execution](syntax.md#multi-statement-execution)).

Adding an index to a large, live collection while keeping writers available during the long scan — **standalone, NOT in a migration** (`WITH (ONLINE = ON)` builds while DML continues between brief registration/publication cuts, then swaps in atomically):

```sql
CREATE INDEX ON [Shop].[Orders] ([Status]) WITH (ONLINE = ON);
```

<a id="example-17-explain-analyze-workflow"></a>

### EXPLAIN ANALYZE workflow

Inspect a combined set and its final sort:

```sql
EXPLAIN FROM [Shop].[Customers] SELECT [Email]
UNION ALL FROM [Shop].[Subscribers] SELECT [Email]
ORDER BY [Email];
```

The Sort consumes Concatenation, with both source access plans retained as children. ORDER BY names the combined output; an unknown name fails planning.

Static EXPLAIN validates ORDER BY, HAVING, and window contracts against their input schemas without invoking query functions. Use EXPLAIN ANALYZE to execute value-dependent checks, including percentile probability values.

Inspect the consumer above a reusable producer:

```sql
EXPLAIN WITH [PaidOrders] AS (
    FROM [Shop].[Orders] WHERE [Status] = 'paid'
)
FROM [PaidOrders] SELECT COUNT(*) AS [Count];
```

The consumer uses `HashAggregation` with an estimate of one output row, even when PaidOrders is empty. The source access remains a child. Adding a HAVING predicate makes the surviving output estimate unknown until execution.

Inspect a CTE used as a join input:

```sql
EXPLAIN WITH [PaidOrders] AS (
    FROM [Shop].[Orders] WHERE [Status] = 'paid'
)
FROM [Shop].[Customers] [c]
JOIN [PaidOrders] [o] ON [c].[Id] = [o].[CustomerId]
SELECT [c].[Id];
```

EXPLAIN checks both ON keys before rendering the plan. A misspelled `[o].[CustomerId]` is an error even when PaidOrders is empty; valid mixed numeric key types retain runtime's equality semantics.

`DeferredJoin` retains the filtered producer plan. Add ANALYZE to see the physical join tree chosen after the paid orders have been materialized.

Static EXPLAIN also validates a write predicate without changing rows:

```sql
EXPLAIN DELETE FROM [Shop].[Orders]
WHERE [CustomerId] IN (
    FROM [Shop].[Customers] WHERE [Region] = 'EU' SELECT [Id]
);
```

This reports `DeferredPredicate`: matching values must be read before the write access path can be selected. A plain view can replace the inner collection. Adding ANALYZE performs the deletion and reports its affected count.

Start with static EXPLAIN to validate projection fields without reading query rows. This resolves the derived output before checking the outer selection:

```sql
EXPLAIN FROM (
    FROM [Shop].[Orders] SELECT [Id], [Total]
) [o] SELECT [Id];
```

Selecting `[Missing]` instead of `[Id]` fails during schema binding, even when Orders is empty. Add ANALYZE when actual execution and measurements are wanted:

```sql
EXPLAIN ANALYZE
FROM [Shop].[Orders] [o]
JOIN [Shop].[Customers] [c] ON [o].[CustomerId] = [c].[Id]
WHERE [c].[Region] = 'EU' AND [o].[Status] = 'paid'
ORDER BY [o].[PlacedAt] DESC
LIMIT 100
SELECT [o].[Id], [c].[Name], [o].[Total];
```

Look at:

- **`Strategy`** — the outer-FROM access path (`SecondaryEquality`, `FullScan`, `IndexOrderScan`, etc.). A broad filtered query can be cheaper as one sequential scan even when an index exists. Read `Notes` to distinguish a cost-selected scan from a missing index; `EstimatedRows` describes output.
- **`JoinPlan:join:<sorted-aliases>` rows** — the selected physical join tree. `Strategy` distinguishes scan leaves, streaming left-deep nodes, and materializing bushy hash nodes; `Notes` gives aliases and equality edges. A node's `EstimatedRows` is `null` where nothing defends a number: over a filtered, temporal, or snapshot-read input, for a semi or anti join or an outer join, and for an equality join without fresh `ANALYZE` statistics on its keys. A `LEFT JOIN` on the joined relation's primary key is the exception among outer joins: it keeps its outer estimate, even when that relation is temporal, because each outer row yields exactly one row. Under ANALYZE, the same nodes, in the same tree, add `RowsIn`, `RowsProduced`, spill, elapsed time, and the exact streaming join algorithm that ran.
- **`Notes`** — also surfaces missing-index suggestions (`WHERE [X] has no index; CREATE INDEX ...`) and the cost-based ranking summary.
- **`EstimatedRows`** — `null` wherever no evidence defends a number: statistics-dependent strategies without fresh statistics, and full or ordered scans that apply a predicate no statistic estimates or read a temporal collection. An unfiltered scan of a standard collection reports its document count, point PK / temporal seeks report `1` (`PK IS NULL` reports `0`), and `BitmapOr` sums value-aware input estimates, discounts proven repeated cached probes, and caps them at collection cardinality. `LIMIT N` caps a known value; the row shows the expected output count, not the input scan count.
- **`StreamingMode`** — `Buffered` means the final result was exposed as a materialised row list; inspect the child stages and spill fields to identify the blocking operator. `Streaming` means the result was drained from `RowsAsync`; `n/a` is used for non-row statements.
- **`SpilledBytes > 0`** — the memory cap (`EngineOptions.MaxInMemoryStreamingState`) was hit; either raise the host's cap (the [standalone server](https://mnemic.io/docs/configuration/#standalone-server-environment) keeps the default) or add an index to shrink the working set.

<a id="example-18-subscribe-to-changes"></a>

### Subscribe to changes

```sql
SUBSCRIBE TO [Shop].[Orders] WHERE [Status] = 'paid';
```

Returns a subscription id. Pushed `ChangeNotification` frames carry the affected collection/document identity, change kind and timestamp. The server sends one change if its before or after image has paid status, so moving an order out of paid status is also visible. To stop:

```sql
UNSUBSCRIBE '<subscription-id>';
```

To filter changed orders against currently eligible customers:

```sql
SUBSCRIBE TO [Shop].[Orders]
WHERE [CustomerId] IN (FROM [Shop].[Customers] WHERE [Active] = TRUE SELECT [Id]);
```

The reference query runs in an independent validated observation. Changing only a customer's active status does not emit an order event.

<a id="example-19-check-constraint-at-create-time"></a>

### CHECK constraint at create-time

```sql
BEGIN MIGRATION 0;
CREATE COLLECTION [Shop].[Orders] WITH SCHEMA (
    [Id]         GUID PRIMARY KEY,
    [CustomerId] INT32 NOT NULL,
    [Total]      DECIMAL NOT NULL,
    [Status]     STRING NOT NULL,
    [Tags]       ARRAY STRING
)
WITH CHECK [TotalNonNegative] ([Total] >= 0)
WITH CHECK ([Status] IN ('pending', 'paid', 'shipped', 'cancelled'))
WITH CHECK [TagsNonEmpty] (LENGTH([Tags]) > 0);
COMMIT MIGRATION;
```

The constraint name (`[TotalNonNegative]`) is optional. CHECK fires on every INSERT / UPDATE / REPLACE. `LENGTH` counts STRING UTF-16 units, ARRAY elements, or raw BYTES. `OCTET_LENGTH` counts UTF-8 bytes for STRING and raw bytes for BYTES; both return NULL for NULL.

<a id="example-correlated-scalar-projection"></a>

### Correlated scalar projection

```mql
FROM [Sales].[Customers] [c]
SELECT [Id],
    (FROM [Sales].[Orders] WHERE [CustomerId] = [c].[Id]
     SELECT COUNT(*)) AS [OrderCount];
```

A scalar subquery has one column and at most one row. An empty result is NULL; an aggregate such as COUNT produces its ordinary single result. Use an ordered `LIMIT 1` only when choosing one row is the intended meaning.

<a id="example-20-auth-admin-minimal-sequence"></a>

### Auth admin minimal sequence

```sql
CREATE USER [alice] WITH PASSWORD 'secret';
GRANT  [Reader] TO [alice];
GRANT  [Writer] TO [alice] ON DATABASE [shop];

LIST USERS;
LIST GRANTS;

REVOKE [Writer] FROM [alice] ON DATABASE [shop];
ALTER  USER     [alice] DISABLE;
DROP   USER     [alice];
```

<a id="example-21-pin-a-column-level-default-on-an-existing-field"></a>

### Pin a column-level default on an existing field

```sql
BEGIN MIGRATION 7;
ALTER COLLECTION [Shop].[Items] ALTER FIELD [Currency] SET DEFAULT 'USD';
ALTER COLLECTION [Shop].[Items] ALTER FIELD [CreatedAt] SET DEFAULT NOW;
COMMIT MIGRATION;

-- After this migration:
INSERT INTO [Shop].[Items] ([Id]: 1, [Name]: 'Widget');
-- Currency and CreatedAt populate from the persisted defaults.

-- To clear:
BEGIN MIGRATION 8;
ALTER COLLECTION [Shop].[Items] ALTER FIELD [Currency] DROP DEFAULT;
COMMIT MIGRATION;
```

Schema-only mutation — existing rows are not rewritten. A default fills the field where an inserted document, or an object or list of objects a later write assigns, leaves it out, and where a write assigns it `DEFAULT`; the top-level fields a REPLACE leaves out stay NULL, and explicit values (including NULL) override ([defaults](dml.md#defaults)).

<a id="example-21b-virtual-computed-columns"></a>

### Virtual computed columns

Express derivations once on the schema; the engine evaluates them on every read. Value is **not** stored on disk — INSERT/UPDATE/REPLACE must NOT supply a value for a computed field.

```sql
-- Declare at create time
BEGIN MIGRATION 8;
CREATE COLLECTION [Shop].[Lines] WITH SCHEMA (
  [Id]    INT32 PRIMARY KEY,
  [Price] DECIMAL,
  [Qty]   INT32,
  [Total] DECIMAL AS ([Price] * [Qty])
);
COMMIT MIGRATION;

-- Add another to the existing collection
BEGIN MIGRATION 9;
ALTER COLLECTION [Shop].[Lines] ADD FIELD [Doubled] INT64 AS ([Qty] * 2);
COMMIT MIGRATION;

-- INSERT supplies Price + Qty only; Total synthesises on read.
INSERT INTO [Shop].[Lines] ([Id]: 1, [Price]: 9.99, [Qty]: 3);
FROM [Shop].[Lines] SELECT [Total];    -- → 29.97

-- WHERE / ORDER BY work but trigger a full scan (no index pushdown).
FROM [Shop].[Lines] WHERE [Total] >= 30 ORDER BY [Total] DESC;

-- Nested and ARRAY OBJECT expressions bind to their containing object.
CREATE COLLECTION [Shop].[Bundles] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Price] OBJECT (
    [Net] DECIMAL,
    [Tax] DECIMAL,
    [Gross] DECIMAL AS ([Net] + [Tax])
  ),
  [Lines] ARRAY OBJECT (
    [Qty] INT32,
    [Each] DECIMAL,
    [LineTotal] DECIMAL AS ([Qty] * [Each])
  )
);
```

Computed values support scalar and compatible OBJECT, ARRAY, and VECTOR shapes. They are nullable and cannot be PRIMARY KEY, AUTOINCREMENT, or REFERENCES. Acyclic computed dependencies evaluate in dependency order. Collection-level base and active extension fields share that order; a referenced field absent from the active subtype reads typed NULL when another subtype declares it. JOINs materialize an enriched outer or inner only when a consumer can observe that side's computed values; unobserved computed slots leave the ordinary physical/index path eligible.

<a id="example-21b-add-a-foreign-key-or-array-to-an-existing-collection"></a>

### Add a foreign key (or array) to an existing collection

```sql
BEGIN MIGRATION 9;
-- Target must already exist (no forward reference at ALTER). Nullable by
-- default; existing rows read NULL.
ALTER COLLECTION [App].[Posts] ADD FIELD [AuthorId] GUID REFERENCES [App].[Users] ON REMOVE SetNull;
-- DEFAULT backfills every existing row and is validated against the target.
ALTER COLLECTION [App].[Posts] ADD FIELD [TeamId] GUID REFERENCES [App].[Teams] DEFAULT '7b9...e1';
-- Pinned FK (temporal target); DEFAULT must be a PIN(id, rev) literal.
ALTER COLLECTION [App].[Invoices] ADD FIELD [TariffRef] GUID PINNED REFERENCES [App].[Tariffs] DEFAULT PIN('a1...', 3);
-- ARRAY of a scalar element (incl. array FKs). DEFAULT supplies a backfill.
ALTER COLLECTION [App].[Posts] ADD FIELD [TagIds] ARRAY GUID REFERENCES [App].[Tags] ON REMOVE Cascade;
COMMIT MIGRATION;
```

`ALTER COLLECTION ... ADD FIELD` supports both scalar arrays and `ARRAY OBJECT` element schemas, including nested FKs and polymorphic `WITH SUBTYPE` elements. FKs declared at `CREATE` (rather than `ALTER`) may forward-reference a collection created later in the same migration; those targets are validated at `COMMIT MIGRATION`.

<a id="example-22-correlated-subqueries--scalar--exists"></a>

### Correlated subqueries — scalar + EXISTS

```sql
-- Scalar correlated: customers whose Target equals their own MAX order total
FROM [Sales].[Customers]
WHERE [Target] = (FROM [Sales].[Orders]
                  WHERE [CustomerId] = [Sales].[Customers].[Id]
                  SELECT MAX([Total]));

-- Numeric Target and Total let SUM use the set-wise rewrite and exact numeric
-- equality even when the aggregate result is wider than Target. A cross-domain
-- comparison (for example STRING Target) keeps per-row MQL coercion instead.
FROM [Sales].[Customers]
WHERE [Target] = (FROM [Sales].[Orders]
                  WHERE [CustomerId] = [Sales].[Customers].[Id]
                  SELECT SUM([Total]));

-- EXISTS correlated: customers who placed at least one order
FROM [Sales].[Customers]
WHERE EXISTS (FROM [Sales].[Orders]
              WHERE [CustomerId] = [Sales].[Customers].[Id]);

-- NOT EXISTS: customers with zero orders
FROM [Sales].[Customers]
WHERE NOT EXISTS (FROM [Sales].[Orders]
                  WHERE [CustomerId] = [Sales].[Customers].[Id]);
```

The bracketed RHS `[Sales].[Customers].[Id]` parses as an outer field reference and is resolved against the outer row at execute time (rewritten internally to `__corr_<n>` parameters).

```sql
-- Correlated EXISTS against a temporal target: invoices whose tariff
-- has a covering version at this statement's pinned NOW. The outer
-- TariffRef changes per row, but every inner dispatch shares the same
-- instant, so the result is one coherent current-time snapshot.
FROM [App].[Invoices]
WHERE EXISTS (FROM [App].[Tariffs] AS OF NOW
              WHERE [Id] = [App].[Invoices].[TariffRef]);

-- IN BETWEEN and LAST PER work too — the bound outer Guid flows
-- into the engine's overlap scan / bucketing executor.
FROM [App].[Invoices]
WHERE EXISTS (FROM [App].[Tariffs]
              IN BETWEEN '2024-01-01T00:00:00Z' AND '2024-12-31T00:00:00Z'
              WHERE [Id] = [App].[Invoices].[TariffRef]);
```

<a id="example-23-upsert-on-a-secondary-unique-index"></a>

### UPSERT on a secondary UNIQUE index

```sql
-- Schema declares: CREATE UNIQUE INDEX ON [Shop].[Users] ([Email])
INSERT INTO [Shop].[Users] ([Email]: 'a@x.com', [Name]: 'A')
ON CONFLICT ([Email]) DO UPDATE SET ([Name]: 'A');

-- Compound UNIQUE — order matters; must align with the index declaration:
INSERT INTO [Shop].[Tickets] ([Tenant]: 't1', [Slot]: 'A', [Qty]: 1)
ON CONFLICT ([Tenant], [Slot]) DO UPDATE SET ([Qty]: 999);
```

Auto-increment PKs work fine — the conflict probes the unique target rather than the PK, so the engine assigns a fresh PK on the no-conflict path and an UPDATE-by-PK on the hit path.

<a id="example-24-aggregate-filter--variable-width-aggregates"></a>

### Aggregate FILTER + variable-width aggregates

```sql
-- Per-region open / closed counts plus revenue, all from one scan
FROM [Shop].[Orders] GROUP BY [Region]
SELECT [Region],
       COUNT(*) FILTER (WHERE [Status] = 'open')   AS [Open],
       COUNT(*) FILTER (WHERE [Status] = 'closed') AS [Closed],
       SUM([Total]) FILTER (WHERE [Status] = 'paid') AS [PaidRevenue];

-- Tag fan-out per group with stable ordering
FROM [App].[Articles] GROUP BY [Topic]
SELECT [Topic], JOIN_STRING([Tag], ', ' ORDER BY [Tag] ASC) AS [Tags];

-- Collect per-customer ordered amounts into an ARRAY
FROM [Shop].[Orders] GROUP BY [CustomerId]
SELECT [CustomerId],
       COLLECT_ARRAY([Total] ORDER BY [PlacedAt] ASC) AS [Timeline];
```

`FILTER` composes with every aggregate kind; sibling aggregates in the same SELECT see the unfiltered row stream.

Grouped expressions can use the completed aggregate alias and rename a group key. The source Amount is reduced first; Doubled is evaluated once per group:

```sql
FROM [App].[Orders]
GROUP BY [Region]
ORDER BY [Label]
SELECT [Region] AS [Label], SUM([Amount]) AS [Total],
       [Total] * 2 AS [Doubled];
```

<a id="example-25-add-a-subtype-to-an-existing-polymorphic-collection"></a>

### Add a subtype to an existing polymorphic collection

```sql
BEGIN MIGRATION 5;

-- Already polymorphic by 'Car' / 'Truck'; introduce a new 'Bike' variant
-- with its own extension fields. Existing Car / Truck rows are untouched;
-- INSERT ... SUBTYPE [Bike] (...) succeeds immediately after the commit.
ALTER COLLECTION [Fleet].[Vehicles] ADD SUBTYPE [Bike] (
  [Gears]    INT32,
  [Electric] BOOL NOT NULL,
  [OwnerId]  INT32 REFERENCES [Fleet].[Owners] ON REMOVE Restrict
);

COMMIT MIGRATION;

INSERT INTO [Fleet].[Vehicles] SUBTYPE [Bike] (
  [Name]: 'Roadbike', [Gears]: 21, [Electric]: FALSE, [OwnerId]: 42
);

-- ADD SUBTYPE is additive: naming an existing subtype merges new fields
-- onto it. This is the only way to add a field to an existing subtype
-- (there is no ADD FIELD into a subtype). Existing Bike rows read
-- [FrameSize] as null until rewritten; a field that already exists in
-- the subtype is rejected.
BEGIN MIGRATION 6;
ALTER COLLECTION [Fleet].[Vehicles] ADD SUBTYPE [Bike] ([FrameSize] STRING);
COMMIT MIGRATION;
```

Symmetric pair of `ALTER COLLECTION ... DROP SUBTYPE [value]`. Rejected if the collection is not polymorphic, if a supplied field name already exists in the subtype, or if any FK target doesn't exist in the live catalog (ALTER does not tolerate forward references the way CREATE does). Inside a `BEGIN MIGRATION` the subtype-list change rides the migration transaction — restored on `ABORT MIGRATION`, WAL-enrolled atomically with the commit marker on `COMMIT MIGRATION`.

<a id="example-26-modeling-partition-vs-overlap--subtype-or-separate-collection"></a>

### Modeling: partition vs overlap — subtype or separate collection?

When migrating a relational schema with multiple "extension" tables hanging off a parent, the structural decision is whether each parent row is in **exactly one** extension (mutually exclusive — a discriminator) or **independently in zero or more** (overlapping facets). Getting it wrong leads to either a polymorphic collection that should have been a set of separate facets, or a flat constellation of collections that should have been one polymorphic collection.

The heuristic from row counts:

- Sum the extension-table row counts.
- If **sum ≈ parent rowcount** → each parent row appears in exactly one extension → **partition** → model the parent as a polymorphic collection with `WITH SUBTYPE [X] (...)` per extension.
- If **sum ≫ parent rowcount** → parents appear in multiple extensions concurrently → **overlap** → keep each extension as its own collection with a normal `REFERENCES` (or `PINNED REFERENCES`) back to the parent.

<a id="example-worked-example--partition-subtype"></a>

### Worked example — partition (subtype)

Parent `Zone` (660 rows) has integration-kind extension tables in separate schemas: `Access.Zone.ZoneVersion` (407), `Affinity.Zone` (48), `Ocpi.Zone` (191), `Ocpp.Zone` (14). 407 + 48 + 191 + 14 = 660, exactly the parent count → every zone is in exactly one integration kind → polymorphic:

```sql
CREATE COLLECTION [App].[Zone] (Temporal: TRUE, CacheCurrentVersion: TRUE)
  WITH SCHEMA ([Id] INT32 PRIMARY KEY AUTOINCREMENT, [Name] STRING)
  WITH SUBTYPE [Access]   ()
  WITH SUBTYPE [Affinity] ([BillableId] INT32, [NoticePeriod] INT32)
  WITH SUBTYPE [Ocpi]     ([Providers] ARRAY OBJECT ([LegalEntityId] INT32))
  WITH SUBTYPE [Ocpp]     ([BackendId] INT32);
```

<a id="example-worked-example--overlap-separate-facet-collections"></a>

### Worked example — overlap (separate facet collections)

Parent `Session` (28370 rows) has channel facets: `ChargeHubSession` (28161), `OcpiSession` (13942), `OcppSession` (4183), `AffinitySession` (851), `AccessSession` (9394). Sum = 56531 ≫ 28370 → many sessions appear in two or more facets at once (a chargehub session also exposed via OCPI, etc.) → independent facets:

```sql
CREATE COLLECTION [App].[Session] WITH SCHEMA (...);

CREATE COLLECTION [App].[OcpiSession] WITH SCHEMA (
  [SessionId] INT32 PRIMARY KEY REFERENCES [App].[Session] ON REMOVE Cascade,
  [PlatformId] INT32 NOT NULL REFERENCES [App].[Platform] ON REMOVE Restrict
);
-- repeat for each independent channel
```

<a id="example-subtlety-a-parent-can-have-both-axes"></a>

### Subtlety: a parent can have BOTH axes

A real-world schema sometimes has one partitioning axis AND an orthogonal axis at the same time. `Session` above also has a `SessionType` (Charging / Heating) attribute that partitions (every session is exactly one) — but that axis carries no per-kind fields, so it stays a plain enum, not a second discriminator. Pick the axis that carries the structural fields as the collection-level discriminator; the field-less axis stays an enum (or, if it carries its own fields, an embedded polymorphic `OBJECT` — the field-level form). Mnemic supports exactly one collection-level discriminator per collection.

<a id="example-27-wildcard-exclude-replace-and-rename"></a>

### Wildcard `EXCLUDE`, `REPLACE`, and `RENAME`

```sql
-- PII masking
FROM [Sec].[Users]
SELECT * REPLACE (NULL AS [Password], NULL AS [Salt]);

-- Normalisation in one place
FROM [Shop].[Orders]
SELECT * REPLACE (UPPER([Region]) AS [Region]);

-- Unit conversion preserving every other field
FROM [Sec].[Audit]
SELECT * REPLACE ([AmountCents] / 100.0 AS [AmountCents]);

-- Compose with EXCLUDE: drop one field, substitute another
FROM [Sec].[Users]
SELECT * EXCLUDE ([Salt]) REPLACE (UPPER([Name]) AS [Name], NULL AS [Password]);

-- Rename several columns from one relation after a join
FROM [Sales].[Orders] [o]
JOIN [Core].[Customers] [c] ON [o].[CustomerId] = [c].[Id]
SELECT [c].*
RENAME ([Id] AS [CustomerId], [Name] AS [CustomerName]);

-- Replace a value in its source position, then advertise a new result name
FROM [Sec].[Users] [u]
SELECT [u].*
REPLACE (UPPER([Name]) AS [Name])
RENAME ([Name] AS [DisplayName]);

-- Rename mappings are simultaneous, so swaps are valid
FROM [App].[Pairs]
SELECT * RENAME ([A] AS [B], [B] AS [A]);
```

Each `REPLACE` alias names an existing source field; its expression reads the original full row and changes that slot's value and inferred type in place.

```mql
FROM [App].[Profiles]
SELECT * EXCLUDE ([Address].[Zip])
  REPLACE (UPPER([Address].[City]) AS [Address].[City])
  RENAME ([Address].[City] AS [Town]);
```

`RENAME` maps an existing source name to a result-only output name without changing the value, type, or position. `EXCLUDE`, `REPLACE`, and `RENAME` may be written in any order, but always execute in that order. Nested Object targets are supported, including inside Arrays with explicit `[]` traversal, preserving null parents and untouched siblings. Renaming an excluded field is invalid, while replacing and renaming the same retained field is supported. For a qualified wildcard after `OMIT INNER KEY`, the omitted key behaves like an implicit exclusion: repeating `EXCLUDE` is safe, but targeting it with `REPLACE` or `RENAME` is an error.

Each Object element can use its own original Price and the document's Factor:

```mql
FROM [App].[Baskets]
SELECT * EXCLUDE ([Items][].[Secret])
  REPLACE ([Items][].[Price] * [Factor] AS [Items][].[Price])
  RENAME ([Items][].[Price] AS [Cost]);
```

A window replacement uses one value per source row:

```mql
FROM [App].[Baskets]
SELECT * REPLACE (ROW_NUMBER() OVER [w] AS [Items][].[Price])
WINDOW [w] AS (ORDER BY [Id]);
```

Scalar elements can change value or type while the Array keeps its shape:

```mql
FROM [App].[Arrays]
SELECT * REPLACE (CAST([Values][] * [Factor] AS STRING) AS [Values][]);
```

For `[Values] ARRAY INT32`, `[2, 4]` with Factor 3 becomes `['6', '12']`. Empty Arrays stay empty and null Arrays stay null. A nested Array target uses one `[]` per dimension, for example `[Matrix][][]`.

<a id="example-28-enum_value--label-string--underlying-value"></a>

### `ENUM_VALUE` — label string → underlying value

Inverse of `ENUM_LABEL([Field])`: takes a label string + enum name, returns the declared underlying numeric / `GUID` value.

```sql
BEGIN MIGRATION 0;
CREATE ENUM [Priority] (INT8) VALUES (1 = 'Low', 2 = 'Medium', 3 = 'High');
CREATE COLLECTION [App].[Tasks] WITH SCHEMA ([Id] INT32 PRIMARY KEY, [Pri] [Priority]);

-- Computed column that derives an enum value from a free-form label
CREATE COLLECTION [App].[Requests] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [LabelStr] STRING,
  [Pri] INT64 AS (ENUM_VALUE([LabelStr], 'Priority'))
);
COMMIT MIGRATION;

-- INSERT from a human-readable string
INSERT INTO [App].[Tasks]
  ([Id]: 1, [Pri]: ENUM_VALUE('High', 'Priority'));

-- Project a label-derived value
FROM [App].[Tasks]
SELECT [Id], ENUM_VALUE('High', 'Priority') AS [High];
```

Namespaced enums: the name as `[Mnemic].[Enums]` lists it, `'[Core.Facility].[ChargeState]'`, and the same name with unbracketed segments (`'[Core.Facility].ChargeState'`, `'Core.Facility.ChargeState'`) resolve to the same registered enum. Label match is `OrdinalIgnoreCase` — `'high'` and `'HIGH'` both match `'High'`. NULL label → NULL; unknown label → NULL; unknown enum name → typed error so a typo (`'Priroty'`) is caught when the call is first evaluated rather than silently returning NULL (see [ENUM_VALUE](enums.md#enum_value--label--value-scalar-function)).

<a id="example-29-create-type--reuse-a-field-set-as-a-field-of-type-or-a-collection-base"></a>

### `CREATE TYPE` — reuse a field set as a field-of-type or a collection base

A composite type is a PK-less, reusable field set. Use it as a field's type (composition, stored as a nested `OBJECT`) or flatten it into a collection's top-level fields (`EXTENDS`). DDL is migration-only.

```sql
BEGIN MIGRATION 0;
CREATE TYPE [App].[Money] AS (
  [Amount] DECIMAL NOT NULL,
  [Currency] STRING NOT NULL,
  [Display] STRING AS ([Currency])
) WITH CHECK ([Amount] >= 0);

-- Composition: address the type's fields as a nested object.
CREATE COLLECTION [App].[Orders] WITH SCHEMA ([Id] INT32 PRIMARY KEY, [Total] [App].[Money]);
-- Inheritance: the type's fields become TOP-LEVEL collection fields, and its CHECK applies.
CREATE COLLECTION [App].[Refunds] EXTENDS [App].[Money] WITH SCHEMA ([Id] INT32 PRIMARY KEY);
COMMIT MIGRATION;

INSERT INTO [App].[Orders] ([Id]: 1, [Total]: ([Amount]: 42.00, [Currency]: 'USD'));
INSERT INTO [App].[Refunds] ([Id]: 1, [Amount]: 5.00, [Currency]: 'USD');
-- Both writes synthesize [Display]; a negative [Amount] is rejected in either form.

FROM [App].[Orders] WHERE [Total].[Currency] = 'USD';    -- composition → nested path
FROM [App].[Refunds] WHERE [Currency] = 'USD';           -- EXTENDS → top-level
```

A type cannot declare `PRIMARY KEY`; the using collection owns its PK. `DROP TYPE` errors while any collection or type references the type, including Object, Array, and subtype uses. `ALTER TYPE` propagates to every use in the same migration — `EXTENDS` users, Object and Array fields, collection- and field-level subtype extensions, and derived types. A type may itself `EXTENDS` other types, and a type or collection may list several bases (`CREATE TYPE [App].[TimedPrice] EXTENDS [App].[Priced], [App].[Schedule] AS ()`); see [EXTENDS](composite-types.md#extends--type-and-collection-inheritance) for the inheritance rules.

<a id="example-30-transactional-audit-trigger-with-old-and-new-row-images"></a>

### Transactional audit trigger with OLD and NEW row images

```sql
BEGIN MIGRATION 7;
CREATE TRIGGER [audit_order_status] ON [Sales].[Orders] AFTER UPDATE AS BEGIN
  INSERT INTO [Sales].[OrderStatusAudit] (
    [OrderId]: NEW.[Id],
    [OldStatus]: OLD.[Status],
    [NewStatus]: NEW.[Status],
    [Region]: NEW.[Shipping].[Region],
    [ChangedAt]: NOW
  )
END;
COMMIT MIGRATION;
```

The audit insert and source update commit or roll back together. INSERT triggers can use only `NEW`; DELETE triggers can use only `OLD`; UPDATE triggers can use both. Invalid aliases and paths fail at trigger creation. Collection and namespace renames retarget the trigger automatically and preserve WHEN across reopen. Compatible schema changes preserve it; invalid image, query, or assignment references abort the migration with the trigger name. Drop or replace the dependent trigger before removing a field it needs.

<a id="example-31-merge-and-replay-exact-migration-history"></a>

### Merge and replay exact migration history

```sql
MERGE MIGRATION 0 TO 5;
SCRIPT MIGRATION 0 TO 5;
```

The merged history keeps each original migration transaction boundary. The returned script is NUL-framed; split on `\0` and execute each non-empty frame. Persisted history itself must use exact `MQLH1` NUL framing with explicit group and statement counts. Mnemic rejects unframed, count-free, truncated, or alternate history shapes.

<a id="example-explicit-attribution"></a>

### Explicit attribution

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [Auth].[Actors] WITH SCHEMA ([Id] INT64 PRIMARY KEY);
SET CREATEDBY COLLECTION [Auth].[Actors];
CREATE COLLECTION [App].[Items] (CreatedBy: TRUE)
WITH SCHEMA ([Id] INT64 PRIMARY KEY, [Name] STRING);
COMMIT MIGRATION;

BEGIN TRANSACTION;
INSERT INTO [Auth].[Actors] ([Id]: 7);
INSERT INTO [App].[Items] ([Id]: 1, [Name]: 'Example')
CREATEDBY 7 RETURNING [Id], [Meta].[CreatedBy];
COMMIT;

UPDATE [App].[Items] SET ([Name]: 'Changed') WHERE [Id] = 1 CREATEDBY COPY;
INSERT INTO [App].[Items] ([Id]: 2, [Name]: 'Upserted')
ON CONFLICT DO UPDATE SET ([Name]: 'Upserted') CREATEDBY COPY OR 7;
```

The actor collection defaults to disabled, allowing initial identities to be seeded. The source collection opts in explicitly. Actors use native INT64 keys; Meta.CreatedBy returns those eight bytes. COPY OR uses 7 only when the selected existing row has no actor or the upsert inserts a new document.

<a id="example-validate-before-a-row-changes"></a>

### Validate before a row changes

```sql
-- Inside a migration; Audit already has an auto-increment primary key.
CREATE TRIGGER [record_pending] ON [App].[Items] BEFORE UPDATE AS BEGIN
  INSERT INTO [App].[Audit] ([OldValue]: OLD.[Value], [NewValue]: NEW.[Value])
END;
```

The audit write shares the source transaction and rolls back if any trigger or source validation fails. Row images are immutable at both trigger timings.

<a id="example-multiple-inheritance-and-an-explicit-row-lifetime"></a>

### Multiple inheritance and an explicit row lifetime

```sql
BEGIN MIGRATION 0;
CREATE TYPE [App].[Timestamped] AS ([CreatedAt] DATETIME);
CREATE TYPE [App].[Owned] AS ([Owner] STRING);
CREATE TYPE [App].[SessionData] EXTENDS [App].[Timestamped], [App].[Owned] AS ();
CREATE COLLECTION [App].[Sessions] EXTENDS [App].[SessionData]
WITH SCHEMA ([Id] GUID PRIMARY KEY);
CREATE INDEX ON [App].[Sessions] ([CreatedAt]) WITH TTL '7 DAYS';
COMMIT MIGRATION;
```

The base fields are inherited in declaration order. The expiry index deletes sessions once CreatedAt is at least seven elapsed days old, subject to ordinary DELETE triggers and constraints. NULL CreatedAt values have no deadline.

<a id="example-composable-relations"></a>

### Composable relations

Window a JOIN directly; partitioning happens before final paging:

```sql
FROM [Sales].[Orders] [o]
JOIN [Sales].[Lines] [l] ON [o].[Id] = [l].[OrderId]
ORDER BY [l].[Id] LIMIT 20
SELECT [l].[Id], ROW_NUMBER() OVER (PARTITION BY [o].[Id] ORDER BY [l].[Price] DESC) AS [Position];
```

Keep branch-local paging inside a derived source:

```sql
FROM (FROM [Sales].[Customers] ORDER BY [Email] LIMIT 10 SELECT [Email])
UNION
FROM [Sales].[Subscribers] SELECT [Email]
ORDER BY [Email];
```

<a id="example-index-a-derived-value"></a>

### Index a derived value

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [Sales].[Lines] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY, [Price] DECIMAL, [Qty] INT32,
  [Total] DECIMAL AS ([Price] * [Qty]) STORED
);
CREATE INDEX ON [Sales].[Lines] ([Total]);
COMMIT MIGRATION;
```

Writes omit `Total`; the engine derives and indexes it before publishing the row.

<a id="example-structured-computed-results"></a>

### Structured computed results

```sql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Directions] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Source] OBJECT ([Label] STRING),
  [Copy] OBJECT ([Label] STRING) AS ([Source]) STORED,
  [Direction] VECTOR(3) AS (VECTOR[1, 0, 0]) STORED,
  [Label] STRING AS ([Copy].[Label])
);
COMMIT MIGRATION;
```

`ALTER COLLECTION ... ADD FIELD` accepts the same inline OBJECT, ARRAY, and VECTOR computed shapes and preserves their definitions through migration replay.

<a id="example-add-backfill-and-index-in-one-migration"></a>

### Add, backfill, and index in one migration

```sql
BEGIN MIGRATION 2;
ALTER COLLECTION [App].[Orders] ADD FIELD [Priority] INT32 DEFAULT 0;
UPDATE [App].[Orders] SET ([Priority]: 1) WHERE [Total] > 100;
ALTER COLLECTION [App].[Orders] ADD FIELD [PriorityLabel] STRING AS (CONCAT('P', [Priority])) STORED;
CREATE INDEX ON [App].[Orders] ([Priority]) INCLUDE ([PriorityLabel]);
COMMIT MIGRATION;
```

<a id="example-order-collected-values-with-deterministic-ties"></a>

### Order collected values with deterministic ties

```sql
FROM [App].[Tags] GROUP BY [Bucket]
SELECT [Bucket],
       COLLECT_ARRAY([Tag] ORDER BY [Score] DESC, [Tag] ASC, [Id] ASC) AS [OrderedTags];
```

<a id="example-base-computed-values-from-subtype-fields"></a>

### Base computed values from subtype fields

```mql
CREATE COLLECTION [App].[Prices] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Qty] INT32,
  [Total] INT32 AS ([Adjusted] + 1) STORED
)
WITH SUBTYPE [Priced] (
  [Raw] INT32,
  [Adjusted] INT32 AS ([Raw] * [Qty]) STORED
)
WITH SUBTYPE [Unpriced] ([Note] STRING);
```

`[Total]` uses `[Adjusted]` from the active subtype. Unpriced rows get NULL. Declare this collection inside a migration; dependency order follows expressions, including extension-to-base chains, rather than field declaration order.

<a id="example-multiple-trigger-statements-and-events"></a>

### Multiple trigger statements and events

Inside a migration, this body handles both inserts and updates with one NEW image:

```mql
CREATE TRIGGER [derive_audit] ON [App].[Items] AFTER INSERT OR UPDATE AS BEGIN
  INSERT INTO [App].[Audit] ([Id]: NEW.[Id], [Total]: NEW.[Value])
  ON CONFLICT DO UPDATE SET ([Total]: NEW.[Value]);
  UPDATE [App].[Audit] SET ([Total]: [Total] + 1) WHERE [Id] = NEW.[Id];
END;
```

The source and both body statements have one transaction outcome. A failure in the second statement also rolls back the first statement's effect.

<a id="example-conditional-trigger-using-both-row-images"></a>

### Conditional trigger using both row images

```sql
BEGIN MIGRATION 1;
CREATE TRIGGER [audit_increase] ON [Sales].[Orders] AFTER UPDATE
WHEN (NEW.[Total] > OLD.[Total]) AS BEGIN
    INSERT INTO [Sales].[Audit] ([Note]: 'Order total increased', [At]: NOW)
END;
COMMIT MIGRATION;
```

Assumes Orders has a stored Total field and Audit has an auto-generated primary key plus Note and At fields. An unknown comparison (for example an absent old value) skips this body; use COALESCE or explicit IS NULL when needed.

<a id="example-composed-projection-and-lateral-matching"></a>

### Composed projection and lateral matching

```mql
FROM [App].[Orders] [o]
LEFT JOIN LATERAL (
    FROM [App].[Lines] WHERE [OrderId] = [o].[Id]
    ORDER BY [Id] LIMIT 1
) AS [line] ON [line].[Id] > [o].[Id]
SELECT [o].*, [line].[Sku] AS [FirstMatchingSku];
```

ON tests the selected first line. An order remains with a null SKU if that line fails the condition. To return multiple relation wildcards, rename or exclude overlapping output names:

```mql
FROM [App].[Orders] [o]
LEFT JOIN [App].[Lines] [l] ON [o].[Id] = [l].[OrderId]
SELECT [o].* RENAME ([Id] AS [OrderId]),
       [l].* EXCLUDE ([OrderId]) RENAME ([Id] AS [LineId]);
```

<a id="example-parameterized-plugin-channel-subscription"></a>

### Parameterized plugin-channel subscription

Bind `pluginId` as the plugin's GUID, as `LIST PLUGINS` lists it, and `channel` as a non-empty string. The subscription retains that routing pair until it is removed or the connection ends.

```mql
SUBSCRIBE TO PLUGIN @pluginId CHANNEL @channel;
```

After receiving its subscription id, bind it as `subscriptionId`:

```mql
UNSUBSCRIBE @subscriptionId;
```

<a id="example-page-a-nearest-neighborhood-from-a-derived-source"></a>

### Page a nearest neighborhood from a derived source

```mql
FROM (FROM [Search].[Documents] WHERE [Active] = TRUE SELECT [Id], [Embedding]) [d]
NEAREST [d].[Embedding] TO @query USING COSINE TOP 20
SKIP 10 LIMIT 10
SELECT [Id];
```

Ranking retains the nearest 20 source rows, then returns rows 11 through 20 in distance order. Add ORDER BY before SKIP to sort those 20 rows by another field before taking the page. Derived rows use exact ranking.

<a id="example-evolve-a-source-used-by-an-incremental-materialized-view"></a>

### Evolve a source used by an incremental materialized view

Given schema level 1 and a source `[App].[Items]` with `[Id] INT32 PRIMARY KEY` and `[Details] OBJECT ([Amount] INT32)`:

```mql
BEGIN MIGRATION 1;
CREATE MATERIALIZED VIEW [App].[Stored] WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Id])
AS FROM [App].[Items] SELECT [Id], [Details];
COMMIT MIGRATION;
BEGIN MIGRATION 2;
ALTER COLLECTION [App].[Items] ADD FIELD [Details].[Extra] INT32 DEFAULT 7;
COMMIT MIGRATION;
FROM [App].[Stored] SELECT [Id], [Details];
```

The existing non-null Details objects gain Extra = 7 in the source and maintained backing together. A MANUAL view keeps its old snapshot until REFRESH. Removing a field named by a saved projection fails atomically; change the view definition first if that projection should change.

<a id="example-materialized-view-recipes"></a>

### Materialized view recipes

The [materialized views](materialized-views.md) page owns the worked materialized-view migrations, each with the schema level and objects it starts from:

- [Seed an incremental chain after source writes](materialized-views.md#seed-an-incremental-chain-after-source-writes)
- [Refresh an upstream snapshot and its maintained descendants](materialized-views.md#refresh-an-upstream-snapshot-and-its-maintained-descendants)
- [Replace a materialized view under the same name](materialized-views.md#replace-a-materialized-view-under-the-same-name)
- [Retarget a snapshot and remove its former source](materialized-views.md#retarget-a-snapshot-and-remove-its-former-source)
- [Maintain a projection through a plain view](materialized-views.md#maintain-a-projection-through-a-plain-view)
- [Incremental primary-key aliases](materialized-views.md#incremental-primary-key-aliases)

<a id="example-rename-materialized-sources-and-backings-in-one-migration"></a>

### Rename materialized sources and backings in one migration

```mql
BEGIN MIGRATION 9;
ALTER COLLECTION [Sales].[Orders] RENAME TO [Archive].[Orders];
ALTER COLLECTION [Sales].[OrderTotals] RENAME TO [Reports].[OrderTotals];
COMMIT MIGRATION;
```

Assuming schema level 9 and an existing materialized `[Sales].[OrderTotals]`, rename preserves its rows, identity, and refresh time, retargets saved source references and incremental maintenance, and leaves MANUAL snapshots unchanged until REFRESH. Plain views that depend on either renamed relation still use their documented rename rules.

<a id="example-incremental-wildcard-and-array-element-projection"></a>

### Incremental wildcard and array-element projection

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [Demo].[Readings] WITH SCHEMA (
    [Id] INT32 PRIMARY KEY,
    [Factor] INT32,
    [Values] ARRAY INT32
);
CREATE MATERIALIZED VIEW [Demo].[Scaled]
    WITH (REFRESH = INCREMENTAL, PRIMARY KEY = [Id])
    AS FROM [Demo].[Readings]
    SELECT * REPLACE ([Values][] * [Factor] AS [Values][])
             RENAME ([Values] AS [ScaledValues]);
COMMIT MIGRATION;
```

Each source change updates one backing key. Replacements read the original array element and Factor. Id keeps its name and value. An omitted SELECT or unmodified wildcard stores all declared data fields without source metadata.

<a id="example-collection-qualified-wildcards-and-plain-view-plans"></a>

### Collection-qualified wildcards and plain-view plans

Given `[App].[Items]` with `Id` and `Value`, and a plain view `[App].[Current]`:

```mql
FROM [App].[Items] SELECT [Items].* RENAME ([Value] AS [Amount]);
WITH [R] AS (FROM [App].[Items]) FROM [R] SELECT [R].*;
EXPLAIN FROM [App].[Current] SELECT [Current].*;
EXPLAIN ANALYZE FROM [App].[Current] SELECT [Current].*;
```

The relation name supplies the wildcard qualifier without an explicit alias. Static EXPLAIN resolves the stored view body without running it; ANALYZE measures its execution against the current visible definition.

<a id="example-inspect-one-identitys-retained-history"></a>

### Inspect one identity's retained history

```mql
EXPLAIN FROM [HR].[Employees]
IN BETWEEN '2020-01-01T00:00:00Z' AND '2030-01-01T00:00:00Z'
WHERE [Id] = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY [Meta].[ValidFrom];
```

For a temporal collection with declared GUID primary key `Id`, the access strategy is `TemporalRange`. Run `ANALYZE [HR].[Employees]` to populate its retained-history estimate. Sorting still orders the filtered history; index batch order is not an implicit user-visible ORDER BY contract.

<a id="example-measure-correlated-fields-without-a-compound-index"></a>

### Measure correlated fields without a compound index

Given `[App].[Places]` with stored scalar fields `City` and `Zip`, and current schema level 1:

```sql
BEGIN MIGRATION 1;
CREATE STATISTICS [App].[Geography] ON ([City], [Zip]) FROM [App].[Places];
COMMIT MIGRATION;
ANALYZE [App].[Places];
FROM [Mnemic].[StatisticsDefinitions];
EXPLAIN FROM [App].[Places] WHERE [City] = 3 AND [Zip] = 3;
```

The declaration supplies joint selectivity for eligible equality combinations. It does not force an access path. To remove it at the resulting schema level:

```sql
BEGIN MIGRATION 2;
DROP STATISTICS [App].[Geography];
COMMIT MIGRATION;
```

<a id="example-index-a-scalar-shared-by-array-element-subtypes"></a>

### Index a scalar shared by array element subtypes

The same path may have different offsets in each subtype. Each subtype declaring it must supply a stored scalar type; different types use separate domains, and an absent field supplies no scalar-value key.

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Rows] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Items] ARRAY OBJECT ([Marker] INT32)
    WITH SUBTYPE [A] ([Value] STRING, [Extra] INT32)
    WITH SUBTYPE [B] ([Extra] INT64, [Value] STRING)
    WITH SUBTYPE [Absent] ([Extra] INT32)
);
CREATE INDEX ON [App].[Rows] ([Items][].[Value]);
COMMIT MIGRATION;

INSERT INTO [App].[Rows] ([Id]: 1, [Items]: [
  SUBTYPE [A] ([Value]: 'urgent'),
  SUBTYPE [B] ([Value]: 'URGENT'),
  SUBTYPE [Absent] ([Extra]: 3)
]);
FROM [App].[Rows] WHERE [Items][].[Value] = 'urgent' SELECT [Id];
EXPLAIN FROM [App].[Rows] WHERE [Items][].[Value] = 'urgent' SELECT [Id];
```

The query returns Id 1 once and EXPLAIN reports `ArrayMembership`. Unique and partial indexes use the same active-subtype extraction and value deduplication.

<a id="example-index-an-array-inside-a-polymorphic-object"></a>

### Index an array inside a polymorphic Object

Arrays in different Object subtypes can use the same membership index when their scalar element types agree. Missing containers and subtype fields add no key.

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Profiles] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Profile] OBJECT ([Marker] INT32)
    WITH SUBTYPE [Personal] ([Tags] ARRAY STRING, [Extra] INT32)
    WITH SUBTYPE [Business] ([Extra] INT64, [Tags] ARRAY STRING)
    WITH SUBTYPE [Unclassified] ([Extra] INT32)
);
CREATE INDEX ON [App].[Profiles] ([Profile].[Tags][]);
COMMIT MIGRATION;

INSERT INTO [App].[Profiles] ([Id]: 1,
  [Profile]: SUBTYPE [Business] ([Tags]: ['urgent', 'URGENT']));
FROM [App].[Profiles] WHERE [Profile].[Tags][] = 'urgent' SELECT [Id];
EXPLAIN FROM [App].[Profiles] WHERE [Profile].[Tags][] = 'urgent' SELECT [Id];
```

The query returns Id 1 once and EXPLAIN reports `ArrayMembership`. Use the same `[Profile].[Tags][]` spelling to drop the index.

<a id="example-index-a-pinned-reference-inside-array-objects"></a>

### Index a pinned reference inside array objects

The reference must be a stored leaf in an object element; compatible polymorphic leaves also work. Different declared scalar types use separate ordered domains with per-type UNIQUE ownership. Nested Object fields and repeated explicit array boundaries retain these scalar-key rules. Bare references compare by identifier; `.Revision` compares the bound revision.

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Targets] (Temporal: TRUE)
  WITH SCHEMA ([Id] STRING PRIMARY KEY);
CREATE COLLECTION [App].[Sources] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Items] ARRAY OBJECT ([Sub] OBJECT (
    [Ref] STRING PINNED REFERENCES [App].[Targets] ON REMOVE SetNull
  ))
);
CREATE INDEX ON [App].[Sources] ([Items][].[Sub].[Ref]);
CREATE INDEX ON [App].[Sources] ([Items][].[Sub].[Ref].[Revision]);
COMMIT MIGRATION;

BEGIN TRANSACTION;
INSERT INTO [App].[Targets] ([Id]: 'A') VALID FROM '2024-01-01T00:00:00Z';
INSERT INTO [App].[Targets] ([Id]: 'A') VALID FROM '2090-01-01T00:00:00Z';
INSERT INTO [App].[Sources] ([Id]: 1, [Items]: [
  ([Sub]: ([Ref]: PIN('a', 1))), ([Sub]: ([Ref]: PIN('A', 2)))
]);
COMMIT;

FROM [App].[Sources] WHERE [Items][].[Sub].[Ref] = 'a';
CANCEL VERSION AT '2090-01-01T00:00:00Z' FROM [App].[Targets] WHERE [Id] = 'A';
FROM [App].[Sources] WHERE [Items][].[Sub].[Ref].[Revision] = 1;
```

Cancellation clears only the second reference. The first remains bound to revision 1; the object/array structure and index postings stay consistent.

<a id="example-independent-relationships-at-a-shared-subtype-path"></a>

### Independent relationships at a shared subtype path

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Suppliers] WITH SCHEMA ([Id] STRING PRIMARY KEY);
CREATE COLLECTION [App].[Customers] WITH SCHEMA ([Id] STRING PRIMARY KEY);
CREATE COLLECTION [App].[Documents] WITH SCHEMA ([Id] INT32 PRIMARY KEY)
WITH SUBTYPE [Purchase] (
  [Owner] STRING REFERENCES [App].[Suppliers] ON REMOVE Restrict
)
WITH SUBTYPE [Sale] (
  [Owner] STRING REFERENCES [App].[Customers] ON REMOVE SetNull
)
WITH SUBTYPE [Note] ([Owner] STRING);
COMMIT MIGRATION;
```

Both target collections have STRING primary keys and are declared in the same migration. A Purchase validates its supplier, a Sale validates its customer, and a Note's Owner is unconstrained. Removing a customer clears only matching Sale owners; a supplier reference does not become a customer reference merely because its field has the same name.

<a id="example-count-correlated-children-inside-a-matching-group"></a>

### Count correlated children inside a matching group

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Cases] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY,
  [Groups] ARRAY OBJECT (
    [Enabled] BOOL,
    [Children] ARRAY OBJECT ([Score] INT32, [Kind] STRING)
  )
);
COMMIT MIGRATION;

FROM [App].[Cases]
WHERE [Groups][] MATCH (
  [Enabled] = true AND
  LENGTH([Children][] MATCH ([Score] >= 10 AND [Kind] = 'good')) >= 2
);
```

The query requires two qualifying children in the same enabled group. A high score in one child and the desired kind in another do not combine into a match.

<a id="example-cover-row-fields-for-array-membership"></a>

### Cover row fields for array membership

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[TaggedRows] WITH SCHEMA (
    [Id] INT32 PRIMARY KEY, [Tags] ARRAY STRING, [Title] STRING
);
CREATE INDEX ON [App].[TaggedRows] ([Tags][]) INCLUDE ([Id], [Title]);
COMMIT MIGRATION;
FROM [App].[TaggedRows] WHERE [Tags][] = 'urgent' SELECT [Id], [Title];
ALTER INDEX ON [App].[TaggedRows] ([Tags][]) DROP INCLUDE ([Title]) WITH (ONLINE = ON);
```

The equality projection can use slot-validated coverage. Each posting carries row fields; its element key does not reconstruct the complete Tags array. Online ALTER runs outside a migration and preserves array membership semantics.

<a id="example-compound-array-payloads-and-posting-statistics"></a>

### Compound-array payloads and posting statistics

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[CompoundTags] WITH SCHEMA (
    [Id] INT32 PRIMARY KEY, [Tenant] INT32 NOT NULL, [Tags] ARRAY STRING
);
CREATE INDEX ON [App].[CompoundTags] ([Tenant], [Tags][]) INCLUDE ([Tenant]);
COMMIT MIGRATION;
BEGIN TRANSACTION;
INSERT INTO [App].[CompoundTags] ([Id]: 1, [Tenant]: 10, [Tags]: ['urgent', 'urgent', 'open']);
INSERT INTO [App].[CompoundTags] ([Id]: 2, [Tenant]: 10, [Tags]: []);
COMMIT;
ANALYZE [App].[CompoundTags];
FROM [Mnemic].[Statistics]
WHERE [CollectionName] = '[App].[CompoundTags]' AND [IsArrayComponent] = TRUE
SELECT [FieldPath], [TotalCount], [CollectionDocCountAtAnalyze];
EXPLAIN FROM [App].[CompoundTags]
WHERE [Tenant] = 10 AND [Tags][] = 'urgent'
SELECT [Id], [Tenant];
```

Tags contributes three postings: two distinct values from row 1 and empty-array coverage from row 2. Tenant contributes two component postings. Both statistics rows have a source count of two. The explicit INCLUDE copies Tenant onto every posting. EXPLAIN chooses among single probes, intersections, and a scan using the collected costs, including spill I/O under the streaming-state budget; all predicates remain authoritative row checks.

<a id="example-partial-index-excluding-listed-values"></a>

### Partial index excluding listed values

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Tickets] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY, [Code] STRING NOT NULL, [State] STRING NULLABLE
);
CREATE UNIQUE INDEX ON [App].[Tickets] ([Code]) WHERE [State] NOT IN ('Closed', 'Archived');
COMMIT MIGRATION;

INSERT INTO [App].[Tickets] ([Id]: 1, [Code]: 'A', [State]: NULL);
INSERT INTO [App].[Tickets] ([Id]: 2, [Code]: 'A', [State]: 'Closed');
FROM [App].[Tickets] WHERE [Code] = 'A' AND [State] IS NULL SELECT [Id];
```

The query returns Id 1 through the eligible partial index. Null State values participate in uniqueness; closed and archived tickets do not. Another ticket outside those states with Code A would conflict, including one whose State is null. Adding NULL to the exclusion list exempts null rows from this constraint. A query with `[State] LIKE 'Open%'` also implies the non-null exclusion list: neither 'Closed' nor 'Archived' matches that pattern. The index can therefore serve that query even though it spells membership differently.

<a id="example-partial-index-for-a-string-subset"></a>

### Partial index for a string subset

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Reviews] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY, [Code] STRING NOT NULL, [Label] STRING NULLABLE
);
CREATE UNIQUE INDEX ON [App].[Reviews] ([Code])
WHERE [Label] STARTSWITH 'Open' AND [Label] CONTAINS 'Review';
COMMIT MIGRATION;
INSERT INTO [App].[Reviews] ([Id]: 1, [Code]: 'A', [Label]: NULL);
INSERT INTO [App].[Reviews] ([Id]: 2, [Code]: 'A', [Label]: 'Open review');
FROM [App].[Reviews] WHERE [Code] = 'A' AND [Label] = 'OPEN REVIEW' SELECT [Id];
```

Equality implies both case-insensitive predicates. Replacing the equality with `[Label] LIKE 'Open%Review'` also implies both filters, despite using a different operator. The query returns Id 2; null labels do not participate in this partial uniqueness constraint.

<a id="example-boolean-partial-index"></a>

### Boolean partial index

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[ReviewQueue] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY, [Code] STRING NOT NULL,
  [State] STRING NULLABLE, [Priority] INT32 NULLABLE
);
CREATE INDEX ON [App].[ReviewQueue] ([Code])
WHERE [State] = 'Open' OR [Priority] = 1;
COMMIT MIGRATION;
INSERT INTO [App].[ReviewQueue] ([Id]: 1, [Code]: 'A', [State]: NULL, [Priority]: 1);
FROM [App].[ReviewQueue]
WHERE [Code] = 'A' AND ([State] = 'Open' OR [Priority] = 1)
SELECT [Id];
```

The OR predicate admits Id 1 despite its null State. The shared Code equality can drive the partial-index seek. `NOT ([State] = 'Closed')` would also admit that row, while `NOT ([State] LIKE 'Closed%')` would exclude it.

<a id="example-partial-index-for-a-like-pattern"></a>

### Partial index for a LIKE pattern

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[PatternReviews] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY, [Code] STRING NOT NULL, [Label] STRING NULLABLE
);
CREATE INDEX ON [App].[PatternReviews] ([Code])
WHERE [Label] LIKE 'Open!_%' ESCAPE '!';
COMMIT MIGRATION;
INSERT INTO [App].[PatternReviews] ([Id]: 1, [Code]: 'A', [Label]: 'Open_review');
FROM [App].[PatternReviews]
WHERE [Code] = 'A' AND [Label] LIKE 'Open!_rev%' ESCAPE '!'
SELECT [Id];
```

The underscore is literal. The narrower query pattern implies the index filter and returns Id 1. NOT LIKE filters use the same ESCAPE syntax and also exclude null labels.

<a id="example-unicode-like-wildcards"></a>

### Unicode LIKE wildcards

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[UnicodeText] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY, [Text] STRING NOT NULL
);
COMMIT MIGRATION;
BEGIN TRANSACTION;
INSERT INTO [App].[UnicodeText] ([Id]: 1, [Text]: 'é😀');
INSERT INTO [App].[UnicodeText] ([Id]: 2, [Text]: 'é😀x');
COMMIT;
FROM [App].[UnicodeText] WHERE [Text] LIKE 'é__' SELECT [Id];
```

Only Id 2 matches. The two underscores require two Unicode characters after é; one emoji supplies only one, including on the non-ASCII pattern path.

<a id="example-text-matching-inside-case-when"></a>

### Text matching inside CASE WHEN

```mql
FROM [App].[ReviewQueue]
SELECT [Id], CASE WHEN [Code] STARTSWITH 'A' THEN 'A group' ELSE 'Other' END AS [Group];
```

Both STARTSWITH and CONTAINS also accept parameters and scalar string expressions. NULL conditions select ELSE.

<a id="example-add-an-all-or-none-array-object-foreign-key"></a>

### Add an all-or-none array-object foreign key

```mql
-- Inside a migration; Contributors already contains object elements.
ALTER COLLECTION [App].[Books] ADD FIELD [Contributors][].[AuthorId]
INT32 REFERENCES [App].[Authors] MATCH FULL;
```

Existing elements receive NULL. Subsequent writes must supply AuthorId in every element or leave it NULL in every element.

<a id="example-change-reusable-type-nullability"></a>

### Change reusable-type nullability

```mql
-- Inside a migration; changes every consumer of the declaring type.
ALTER TYPE [App].[Value] ALTER FIELD [Name] SET NOT NULL;
ALTER TYPE [App].[Value] ALTER FIELD [Name] DROP NOT NULL;
```

<a id="example-change-a-named-type-insertion-default"></a>

### Change a named-type default

```mql
BEGIN MIGRATION 4;
ALTER TYPE [App].[Value] ALTER FIELD [Name] SET DEFAULT 'fallback';
COMMIT MIGRATION;
-- Later inserts into consumers, and the objects later writes assign, default omitted Name fields.
BEGIN MIGRATION 5;
ALTER TYPE [App].[Value] ALTER FIELD [Name] DROP DEFAULT;
COMMIT MIGRATION;
```

The type must already exist; use the database's current migration version. Existing rows and explicit NULL values are unchanged.

<a id="example-backfill-and-change-a-vector-insertion-default"></a>

### Backfill and change a vector default

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Vectors] WITH SCHEMA ([Id] INT32 PRIMARY KEY);
COMMIT MIGRATION;
BEGIN MIGRATION 1;
ALTER COLLECTION [App].[Vectors] ADD FIELD [V] VECTOR(2) DEFAULT VECTOR[1, 0];
ALTER COLLECTION [App].[Vectors] ALTER FIELD [V] SET DEFAULT VECTOR[0, 1];
COMMIT MIGRATION;
INSERT INTO [App].[Vectors] ([Id]: 1);
```

The inserted vector is [0, 1]; any pre-existing rows would retain the backfill [1, 0].

<a id="example-default-an-object-or-array-on-insertion"></a>

### Default an object or array on insertion

```mql
BEGIN MIGRATION 0;
CREATE COLLECTION [App].[Defaults] WITH SCHEMA (
  [Id] INT32 PRIMARY KEY, [Tags] ARRAY STRING, [Details] OBJECT ([Count] INT32)
);
ALTER COLLECTION [App].[Defaults] ALTER FIELD [Tags] SET DEFAULT ['new'];
ALTER COLLECTION [App].[Defaults] ALTER FIELD [Details] SET DEFAULT ([Count]: 1);
COMMIT MIGRATION;
INSERT INTO [App].[Defaults] ([Id]: 1);
```

<a id="example-evolve-an-owned-nested-field-in-a-named-type"></a>

### Evolve an owned nested field in a named type

```mql
BEGIN MIGRATION 0;
CREATE TYPE [App].[Counters] AS ([Details] OBJECT ([Count] INT32));
COMMIT MIGRATION;
BEGIN MIGRATION 1;
ALTER TYPE [App].[Counters] ALTER FIELD [Details].[Count] SET DEFAULT 7;
ALTER TYPE [App].[Counters] RENAME FIELD [Details].[Count] TO [Saved];
COMMIT MIGRATION;
```

RENAME retains the containing object and its field defaults. For ARRAY OBJECT parents, traverse with `[Items][].[Count]`.
