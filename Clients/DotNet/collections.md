# Mapping classes

A collection's documents are read and written as instances of a class. Each `MnemicDatabase` learns the class of a collection when it opens, from `options.Collection<T>`, which names the collection and says which property is its key.

## Registering a class

```csharp
using Mnemic.Client.Remote;

await using var database = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => {
        options.Collection<Customer>("[Shop].[Customers]", collection => collection.HasKey(customer => customer.Id).AutoIncrement());
        options.Collection<Order>("[Shop].[Orders]", collection => collection.HasKey(order => order.Id));
    });

var customers = database.GetCollection<int, Customer>("[Shop].[Customers]");
var orders = database.GetCollection<Guid, Order>("[Shop].[Orders]");

var astrid = await customers.InsertAsync(new Customer {
    Name = "Astrid Lindqvist",
    Email = "astrid@example.com",
    Tier = Tier.Gold,
    Since = new DateOnly(2024, 3, 1),
    Address = new Address { Street = "Storgatan 12", City = "Uppsala", Country = "SE" },
});

await orders.InsertAsync(new Order {
    Id = Guid.NewGuid(),
    CustomerId = astrid.Id,
    PlacedAt = DateTimeOffset.UtcNow,
    Lines = [new OrderLine { Sku = "ELE-2001", Quantity = 1, UnitPrice = 2490m }],
});

Console.WriteLine($"Customer {astrid.Id} placed an order");

public sealed class Customer {
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string? Email { get; set; }
    public Tier Tier { get; set; }
    public DateOnly Since { get; set; }
    public Address Address { get; set; } = new();
}

public sealed class Address {
    public string Street { get; set; } = "";
    public string City { get; set; } = "";
    public string Country { get; set; } = "";
}

public enum Tier {
    Standard,
    Gold,
}

public sealed class Order {
    public Guid Id { get; set; }
    public int CustomerId { get; set; }
    public DateTimeOffset PlacedAt { get; set; }
    public List<OrderLine> Lines { get; set; } = [];
}

public sealed class OrderLine {
    public string Sku { get; set; } = "";
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}
```

`GetCollection<TKey, T>` takes the type of the key and the class, and every operation on the collection that names a document takes its key.

## How properties become fields

Every public property with both a getter and a setter is a field of the collection, in the order the class declares them, under the property's name:

| .NET type | Field type |
|---|---|
| `string` | `STRING` |
| `bool` | `BOOL` |
| `sbyte`, `short`, `int`, `long` | `INT8`, `INT16`, `INT32`, `INT64` |
| `byte`, `ushort`, `uint` | `INT16`, `INT32`, `INT64`, the next wider signed integer |
| `ulong` | `DECIMAL` |
| `float`, `double` | `FLOAT32`, `FLOAT64` |
| `decimal` | `DECIMAL` |
| `Guid` | `GUID` |
| `DateTimeOffset`, `DateTime` | `DATETIME` |
| `DateOnly`, `TimeOnly` | `DATEONLY`, `TIMEONLY` |
| `byte[]` | `BYTES` |
| an enum | the integer field that holds every value of its underlying type |
| a class | `OBJECT`, whose fields are the class's properties |
| `T[]`, `List<T>`, `IList<T>`, `IReadOnlyList<T>`, `ICollection<T>`, `IEnumerable<T>` | `ARRAY` of `T` |
| `float[]` marked `[MnemicVector(n)]` | `VECTOR(n)` |

A `string`, a `byte[]`, a class, an array and a nullable value type such as `int?` may hold `NULL`; any other value type may not, and neither may the key. A property of a type the table does not name stops the database from opening with an `InvalidOperationException` that names it. A property that holds a document another collection refers to is marked `[MnemicResolve]` and is not stored ([references](queries.md#references)). [Field types](../../MqlReference/ddl.md#field-types) describes each type.

## Keys

`HasKey` names the property that is the collection's primary key: an `sbyte`, `short`, `int`, `long`, `Guid`, `string` or `byte[]`. A document's key is the value of that property, so the application chooses it, as with the order's `Guid` above, unless the collection numbers its documents: `AutoIncrement` on an `int` or `long` key gives each inserted document the next number, which the inserted document returned by `InsertAsync` carries.

## Computed fields

A property marked `[MnemicComputed]` holds a value the database computes from the document's other fields with an MQL expression, and is never written by the application:

```csharp
using Mnemic.Client.Shared.Attributes;

await using var catalog = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => options.Collection<Product>("[Catalog].[Products]", collection => collection.HasKey(product => product.Sku)));

var products = catalog.GetCollection<string, Product>("[Catalog].[Products]");

await products.InsertAsync(new Product { Sku = "CMP-1102", Name = "Ergonomic mouse", Price = 690m });

var mouse = await products.GetAsync("CMP-1102");

Console.WriteLine(mouse!.Body.PriceWithVat);

public sealed class Product {
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";
    public decimal Price { get; set; }

    [MnemicComputed("[Price] * 1.25")]
    public decimal? PriceWithVat { get; set; }
}
```

The value is computed when the document is read, unless `Stored = true` keeps it with the document, where an index can use it ([computed columns](../../MqlReference/ddl.md#computed-columns)). A computed property is nullable.

## Subtypes

A collection can hold documents of several classes that derive from one base, told apart by a discriminator the database keeps with each document:

```csharp
await using var billing = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => options.Collection<Payment>(
        "[Billing].[Payments]",
        collection => collection.HasKey(payment => payment.Id).SubType<CardPayment>("card").SubType<InvoicePayment>("invoice")));

var payments = billing.GetCollection<Guid, Payment>("[Billing].[Payments]");
var invoiceId = Guid.NewGuid();

await payments.InsertAsync(new CardPayment { Id = Guid.NewGuid(), Amount = 690m, Last4 = "4242" });
await payments.InsertAsync(new InvoicePayment { Id = invoiceId, Amount = 2490m, DueDays = 30 });

var invoice = await payments.GetAsync(invoiceId);

if (invoice?.Body is InvoicePayment { DueDays: var days })
    Console.WriteLine($"Due in {days} days");

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

A document is read back as the class its discriminator names, and a query can ask for one subtype ([subtypes in queries](queries.md#subtypes)). [Polymorphism](../../MqlReference/polymorphism.md) describes how the collection keeps them, and `PolymorphicField` maps a field whose objects are of several classes in the same way.

## New and existing collections

Opening a database checks every class it registers against the database:

- A collection that does not exist is created, in a migration the open runs, so the user who opens it needs a role that may change the schema, such as `Admin`.
- A collection that exists must have the same fields, in the same order and of the same types, and the same settings, such as whether it is temporal. A collection that does not match stops the open with `SchemaMismatchException`, and nothing is changed.

Registering a class never changes a collection that exists. A schema change is a [migration](../../MqlReference/migrations.md) in MQL, which `ExecuteAsync` runs; a migration that adds, removes or changes a field goes out with the class that matches it:

```csharp
await database.ExecuteAsync($"""
    BEGIN MIGRATION {database.SchemaLevel};
    CREATE INDEX ON [Shop].[Orders] ([CustomerId]);
    COMMIT MIGRATION;
    """);
```

Each migration moves the database to its next schema level. An application built for one schema level sets `options.SchemaLevel` to it, and opening a database at another level then throws `SchemaMismatchException`; `options.SchemaLevelCheck = SchemaLevelCheckMode.Minimum` accepts that level or a later one.
