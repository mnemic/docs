# Temporal collections

A temporal collection keeps every version of each document. A version is valid from one instant until the next version takes over, so the collection answers what a document was at any instant, and holds changes that are scheduled to take effect later. [Temporal queries](../../MqlReference/temporal.md) describes the same in MQL.

`Temporal()` on a registration makes the collection temporal when the open creates it, and requires it of a collection that exists:

```csharp
using Mnemic.Client.Remote;
using Mnemic.Client.Shared;
using Mnemic.Client.Shared.Query;

await using var database = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => options.Collection<Product>("[Catalog].[Products]", collection => collection.HasKey(product => product.Sku).Temporal()));

var products = database.GetCollection<string, Product>("[Catalog].[Products]");

public sealed class Product {
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
}
```

## Writing versions

A document inserted without an instant is valid from the moment it is written. `InsertAsync` takes the instant a document's first version starts at, which may lie in the past:

```csharp
var launched = DateTimeOffset.UtcNow.AddMonths(-6);

await products.InsertAsync(new Product { Sku = "ELE-2001", Name = "Aurora headphones", Price = 2490m }, validFrom: launched);
```

Every update or replace writes a new version, valid from the moment it is written, and the version before it ends there.

## Scheduling a change

An update given `ValidFrom` takes effect at that instant instead. The version valid then ends at it, and the new one lasts as long as that one would have:

```csharp
var nextMonth = DateTimeOffset.UtcNow.AddMonths(1);

await products.UpdateAsync(
    "ELE-2001",
    update => update.Set(product => product.Price, 2290m),
    new MutationOptions { ValidFrom = nextMonth });
```

Until next month, the headphones cost 2490, and from then on 2290. An instant in the past corrects what was valid from then on in the same way.

## Reading the past and the future

`GetAsync` and `AsQueryable` take the instant to read at, and read the versions valid then. Without one they read the versions valid now:

```csharp
var now = await products.GetAsync("ELE-2001");
var then = await products.GetAsync("ELE-2001", asOf: nextMonth.AddDays(1));

Console.WriteLine($"{now!.Body.Price} now, {then!.Body.Price} next month");

var onSale = await products.AsQueryable(asOf: nextMonth.AddDays(1))
    .Where(product => product.Body.Price < 2400m)
    .ToListAsync();
```

`GetAllVersionsAsync` returns every version of a document, oldest first, each with the instants it is valid between:

```csharp
foreach (var version in await products.GetAllVersionsAsync("ELE-2001"))
    Console.WriteLine($"{version.Body.Price} from {version.ValidFrom:d} to {version.ValidTo?.ToString("d") ?? "open end"}");
```

In MQL, `AS OF` reads a collection at an instant, and `IN BETWEEN` lists the versions that overlap a range:

```csharp
var history = await database.QueryAsync<PricePoint>(
    "FROM [Catalog].[Products] IN BETWEEN @from AND @to WHERE [Sku] = @sku SELECT [Price], [Meta].[ValidFrom] AS [From]",
    new { from = launched, to = nextMonth.AddDays(1), sku = "ELE-2001" });

public sealed class PricePoint {
    public decimal Price { get; set; }
    public DateTimeOffset From { get; set; }
}
```

## Ending a document

`DeleteAsync` ends the document's current version at the moment it runs and keeps its history: a read at an earlier instant still finds it.

```csharp
await products.InsertAsync(new Product { Sku = "OUT-4002", Name = "Insulated bottle", Price = 320m });

var beforeDelete = DateTimeOffset.UtcNow;

await products.DeleteAsync("OUT-4002");

var gone = await products.GetAsync("OUT-4002", asOf: DateTimeOffset.UtcNow.AddSeconds(1));
var kept = await products.GetAsync("OUT-4002", asOf: beforeDelete);

Console.WriteLine($"{gone is null} {kept?.Body.Name}");
```
