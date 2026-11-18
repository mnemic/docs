# Queries

A collection is the start of a LINQ query, which the client translates into one MQL statement and the server runs. What LINQ does not express is written in MQL and sent as it is.

The examples on this page use these collections:

```csharp
using Mnemic.Client.Remote;
using Mnemic.Client.Shared;
using Mnemic.Client.Shared.Attributes;
using Mnemic.Client.Shared.Query;

await using var database = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => {
        options.Collection<Category>("[Catalog].[Categories]", collection => collection.HasKey(category => category.Id));
        options.Collection<Product>("[Catalog].[Products]", collection => collection.HasKey(product => product.Sku));
    });

var categories = database.GetCollection<int, Category>("[Catalog].[Categories]");
var products = database.GetCollection<string, Product>("[Catalog].[Products]");

await categories.InsertManyAsync([new Category { Id = 1, Name = "Audio" }, new Category { Id = 2, Name = "Computing" }]);
await products.InsertManyAsync([
    new Product { Sku = "ELE-2001", Name = "Aurora headphones", CategoryId = 1, Price = 2490m, Stock = 12, Embedding = [0.9f, 0.1f, 0.0f] },
    new Product { Sku = "ELE-2002", Name = "Aurora earbuds", CategoryId = 1, Price = 1290m, Stock = 40, Embedding = [0.8f, 0.2f, 0.1f] },
    new Product { Sku = "CMP-1101", Name = "Mechanical keyboard", CategoryId = 2, Price = 1490m, Stock = 30, Embedding = [0.1f, 0.9f, 0.2f] },
    new Product { Sku = "CMP-1102", Name = "Ergonomic mouse", CategoryId = 2, Price = 690m, Stock = 45, Embedding = [0.2f, 0.8f, 0.3f] },
]);

public sealed class Category {
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

public sealed class Product {
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";
    public int CategoryId { get; set; }
    public decimal Price { get; set; }
    public int Stock { get; set; }

    [MnemicVector(3)]
    public float[]? Embedding { get; set; }

    [MnemicResolve(nameof(CategoryId))]
    public Category? Category { get; set; }
}
```

## LINQ

`AsQueryable` returns the collection's documents as `Document<TKey, TDocument>`, so a query reaches the class's fields through `Body`. Running a query is asynchronous: `ToListAsync`, `FirstOrDefaultAsync`, `CountAsync`, `AnyAsync`, `SumAsync` and the other `Async` methods of `Mnemic.Client.Shared.Query`, or `await foreach` over the query.

```csharp
var inStock = await products.AsQueryable()
    .Where(product => product.Body.Stock > 20 && product.Body.Price < 1500m)
    .OrderByDescending(product => product.Body.Price)
    .Take(10)
    .ToListAsync();

var cheapest = await products.AsQueryable()
    .OrderBy(product => product.Body.Price)
    .FirstOrDefaultAsync();

var audio = await products.AsQueryable().Where(product => product.Body.CategoryId == 1).CountAsync();
```

A query is translated as a whole: a method the client cannot translate, or a synchronous one such as `ToList`, throws `NotSupportedException` instead of reading the collection into memory to finish the query there. A value the query captures from the program, such as a variable, is sent as a parameter.

## Projections and groups

`Select` asks for the fields it names and nothing else, and `GroupBy` groups and aggregates on the server:

```csharp
var names = await products.AsQueryable()
    .Where(product => product.Body.CategoryId == 2)
    .Select(product => new { product.Body.Name, product.Body.Price })
    .ToListAsync();

var perCategory = await products.AsQueryable()
    .GroupBy(product => product.Body.CategoryId)
    .Select(group => new { Category = group.Key, Products = group.Count(), Units = group.Sum(product => product.Body.Stock) })
    .ToListAsync();

foreach (var row in perCategory)
    Console.WriteLine($"Category {row.Category}: {row.Products} products, {row.Units} units");
```

## Paging

`QueryPageAsync` reads one page of a collection with the number of documents in all of it, and a query pages with `Skip` and `Take`:

```csharp
var page = await products.QueryPageAsync(skip: 0, take: 2);

Console.WriteLine($"{page.Items.Count} of {page.TotalCount}");

var second = await products.AsQueryable().OrderBy(product => product.Body.Sku).Skip(2).Take(2).ToListAsync();
```

## References

A property marked `[MnemicResolve]` names the field that holds the key of a document in another registered collection, and `Include` fills it in. The referenced documents are read in batches after the query, one statement for many keys:

```csharp
var withCategories = await products.AsQueryable()
    .Where(product => product.Body.Price > 1000m)
    .Include(product => product.Category)
    .ToListAsync();

foreach (var product in withCategories)
    Console.WriteLine($"{product.Body.Name} in {product.Body.Category?.Name}");
```

`ThenInclude` follows a reference of the referenced document in turn.

## Subtypes

In a collection of [subtypes](collections.md#subtypes), `OfType` asks for the documents of one class:

```csharp
await using var billing = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => options.Collection<Payment>(
        "[Billing].[Payments]",
        collection => collection.HasKey(payment => payment.Id).SubType<CardPayment>("card").SubType<InvoicePayment>("invoice")));

var payments = billing.GetCollection<Guid, Payment>("[Billing].[Payments]");

await payments.InsertAsync(new CardPayment { Id = Guid.NewGuid(), Amount = 690m, Last4 = "4242" });
await payments.InsertAsync(new InvoicePayment { Id = Guid.NewGuid(), Amount = 2490m, DueDays = 30 });

var invoices = await payments.AsQueryable()
    .OfType<Document<Guid, InvoicePayment>>()
    .Where(payment => payment.Body.DueDays > 14)
    .ToListAsync();

public abstract class Payment {
    public Guid Id { get; set; }
    public decimal Amount { get; set; }
}

public sealed class CardPayment : Payment {
    public string Last4 { get; set; } = "";
}

public sealed class InvoicePayment : Payment {
    public int DueDays { get; set; }
}
```

## Vector search

`NearestBy` orders documents by their distance from a vector and takes the nearest. It compares every document exactly, or, with `approximate: true`, uses a vector index on the field ([vector search](../../MqlReference/vectors.md)):

```csharp
var similar = await products.AsQueryable()
    .NearestBy(product => product.Body.Embedding!, [0.85f, 0.15f, 0.05f], k: 2)
    .ToListAsync();
```

`VectorFunctions` holds the distance functions for use in a query's other clauses.

## Queries in MQL

`QueryAsync<T>` runs a query written in MQL and reads each row into a `T`, matching columns to properties by name. Values travel as parameters, named in the statement with `@` and given as the properties of an object:

```csharp
var totals = await database.QueryAsync<CategoryTotal>("""
    FROM [Catalog].[Products] [p]
    JOIN [Catalog].[Categories] [c] ON [c].[Id] = [p].[CategoryId]
    WHERE [p].[Price] >= @minimum
    GROUP BY [c].[Name]
    ORDER BY [Units] DESC
    SELECT [c].[Name] AS [Category], COUNT(*) AS [Products], SUM([p].[Stock]) AS [Units]
    """,
    new { minimum = 1000m });

foreach (var total in totals)
    Console.WriteLine($"{total.Category}: {total.Products} products, {total.Units} units");

public sealed class CategoryTotal {
    public string Category { get; set; } = "";
    public long Products { get; set; }
    public long Units { get; set; }
}
```

`QueryPageAsync<T>` on the database reads one page of such a query with the number of rows in all of it, and `ExecuteAsync` runs statements that change data or schema. Parameters keep values out of the statement's text, so a value can never change what the statement means, and statements that differ only in their values share one plan on the server. The [MQL reference](../../MqlReference/README.md) describes the language, and [parameters](../../MqlReference/parameters.md) the values a parameter takes.
