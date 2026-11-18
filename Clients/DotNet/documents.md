# Reading and writing

A collection reads and writes its documents by key. Each write is one MQL statement, which commits on its own unless a [transaction](transactions.md) is open.

The examples on this page use this collection:

```csharp
using Mnemic.Abstractions.Errors;
using Mnemic.Client.Remote;
using Mnemic.Client.Shared;

await using var database = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => options.Collection<Product>("[Catalog].[Products]", collection => collection.HasKey(product => product.Sku)));

var products = database.GetCollection<string, Product>("[Catalog].[Products]");

public sealed class Product {
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public List<string> Tags { get; set; } = [];
}
```

## Inserting

```csharp
var inserted = await products.InsertAsync(new Product { Sku = "ELE-2001", Name = "Aurora headphones", Price = 2490m, Stock = 12, Tags = ["wireless"] });

await products.InsertManyAsync([
    new Product { Sku = "CMP-1101", Name = "Mechanical keyboard", Price = 1490m, Stock = 30, Tags = ["usb-c"] },
    new Product { Sku = "CMP-1102", Name = "Ergonomic mouse", Price = 690m, Stock = 45, Tags = ["wireless", "usb-c"] },
]);
```

`InsertAsync` returns the document as the database stored it, with its key and its first revision. A document whose key the collection already holds is refused with `UNIQUE_VIOLATION`. `InsertManyAsync` sends its documents as one statement; when one of them fails, `InsertManyPartialException` says how many were inserted before it and which one failed.

## Reading

```csharp
var headphones = await products.GetAsync("ELE-2001");

if (headphones is not null)
    Console.WriteLine($"{headphones.Body.Name} at revision {headphones.Revision}");
```

`GetAsync` returns `null` for a key the collection does not hold. A `Document<TKey, TDocument>` carries the body and what the database keeps beside it: `Id`, `Revision`, which counts the document's writes, `CreatedAt`, and `ValidFrom` and `ValidTo`, the instants its version is valid between ([temporal collections](temporal.md)).

## Replacing

`ReplaceAsync` writes a whole new body for a key. Given the revision it was read at, it writes only when the document is still at that revision, so a change another writer made in between is never lost without notice:

```csharp
var read = await products.GetAsync("CMP-1102");

try {
    await products.ReplaceAsync("CMP-1102", new Product { Sku = "CMP-1102", Name = "Ergonomic mouse", Price = 649m, Stock = 45, Tags = ["wireless", "usb-c"] }, read!.Revision);
} catch (MqlException failure) when (failure.Code == "REVISION_MISMATCH") {
    Console.WriteLine("Someone changed the mouse first; read it again and decide.");
}
```

The new body carries the key it replaces.

## Updating fields

`UpdateAsync` changes the fields it names and leaves the rest as they are. A value is set as it is, or computed from the document's own fields by an expression the database evaluates:

```csharp
await products.UpdateAsync("ELE-2001", update => update
    .Set(product => product.Price, 2290m)
    .Set(product => product.Stock, product => product.Stock - 1));
```

An array is edited in place, element by element:

```csharp
await products.UpdateAsync("CMP-1102", update => update
    .Array(product => product.Tags, tags => tags.Append("ergonomic").DeleteWhere(tag => tag == "usb-c")));
```

`UpdateWhereAsync` updates every document that matches a condition and returns how many it changed:

```csharp
var restocked = await products.UpdateWhereAsync(
    product => product.Stock < 20,
    update => update.Set(product => product.Stock, product => product.Stock + 50));

Console.WriteLine($"{restocked} restocked");
```

`MutationOptions` gives an update the revision it expects, as `ReplaceAsync` takes one, or the instant a temporal change takes effect ([scheduling a change](temporal.md#scheduling-a-change)).

## Inserting or updating

`UpsertAsync` inserts a document when the collection does not hold its key, and otherwise applies the update to the document it holds:

```csharp
await products.UpsertAsync(
    new Product { Sku = "HOM-3001", Name = "Pour-over coffee set", Price = 540m, Stock = 10 },
    update => update.Set(product => product.Stock, product => product.Stock + 10));
```

Run twice, the first call inserts the set with 10 in stock and the second adds 10 to it.

## Deleting

```csharp
await products.DeleteAsync("HOM-3001");
```

A delete in a [temporal collection](temporal.md) ends the document's current version and keeps its history.

## Statements in MQL

`ExecuteAsync` runs MQL that writes and returns how many documents it changed, and `QueryAsync` runs a query ([queries in MQL](queries.md#queries-in-mql)):

```csharp
var discounted = await database.ExecuteAsync(
    "UPDATE [Catalog].[Products] SET ([Price]: [Price] * 0.9) WHERE [Stock] > @stock",
    new { stock = 40 });
```
