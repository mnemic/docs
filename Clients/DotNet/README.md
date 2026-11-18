# .NET client

The .NET client lets an application on .NET 11 work with Mnemic through classes of its own. A class is mapped to a collection, documents are read and written as instances of it, and LINQ queries over it are translated into MQL, which the client sends to the server like any other statement.

## Install

```shell
dotnet add package Mnemic.Client.Remote
```

`Mnemic.Client.Remote` connects to a Mnemic server and holds no database engine of its own. An application that keeps its database inside its own process installs `Mnemic.Client.InProcess` instead ([an embedded database](connecting.md#an-embedded-database)).

## A first program

```csharp
using Mnemic.Client.Remote;
using Mnemic.Client.Shared.Query;

await using var database = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => options.Collection<Product>(
        "[Catalog].[Products]",
        collection => collection.HasKey(product => product.Sku)));

var products = database.GetCollection<string, Product>("[Catalog].[Products]");

await products.InsertAsync(new Product { Sku = "ELE-2001", Name = "Aurora headphones", Price = 2490m, Tags = ["wireless"] });
await products.InsertAsync(new Product { Sku = "CMP-1102", Name = "Ergonomic mouse", Price = 690m, Tags = ["wireless", "usb-c"] });

var affordable = await products.AsQueryable()
    .Where(product => product.Body.Price < 1000m)
    .OrderBy(product => product.Body.Name)
    .ToListAsync();

foreach (var product in affordable)
    Console.WriteLine($"{product.Id}: {product.Body.Name}, {product.Body.Price}");

public sealed class Product {
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public List<string> Tags { get; set; } = [];
}
```

Opening the database registers `Product` as the class of `[Catalog].[Products]`, keyed by its `Sku`, and creates the collection when the database does not have it yet. The address assumes a server whose certificate a public authority issued; [Connecting](connecting.md) shows how to trust the certificate a server makes for itself.

## What the client is made of

- `MnemicDatabase` is an open database: one session on one connection, which `OpenRemoteAsync` opens and disposing it gives back.
- `IMnemicCollection<TKey, TDocument>` is a collection of the database, typed by its key and its class, from `GetCollection`. It reads and writes documents by key and is the start of a LINQ query.
- `Document<TKey, TDocument>` is a document as the database holds it: its key, its body as an instance of the class, its revision, and the instants its version is valid from and to.
- `QueryAsync` and `ExecuteAsync` on the database send MQL as written, for what LINQ does not express.

Every operation becomes an MQL statement on the way to the server, so the client can do nothing an MQL statement cannot, and a query written in LINQ runs as the same statement written in MQL would.

## Errors

A statement the server refuses throws `MqlException` (`Mnemic.Abstractions.Errors`), whose `Code` names the error, such as `REVISION_MISMATCH` or `INSUFFICIENT_PRIVILEGE`, and whose `Class` says what kind of failure it is. [Common errors](../../MqlReference/errors.md) lists the codes. A class that does not match the collection it is registered for throws `SchemaMismatchException` when the database opens.

## Pages

- [Connecting](connecting.md): connection strings, certificates, credentials, how long to keep a database open, and embedded databases.
- [Mapping classes](collections.md): registering a class, keys, nested objects and arrays, computed fields, subtypes, and schema changes.
- [Reading and writing](documents.md): inserting, reading, replacing, updating, upserting and deleting documents, with revision checks.
- [Queries](queries.md): LINQ, paging, MQL with parameters, references and vector search.
- [Temporal collections](temporal.md): versions valid over time, reading the past, and scheduling a change.
- [Transactions](transactions.md): several writes that commit or roll back as one.
- [Change notifications](subscriptions.md): being told when documents change.
