# Transactions

Each write commits on its own unless a transaction is open. `BeginTransactionAsync` opens one on the database's session: every write on that `MnemicDatabase` from then on, typed or in MQL, belongs to it, until `CommitAsync` commits them together. Disposing the transaction without committing it rolls them back.

```csharp
using Mnemic.Abstractions.Errors;
using Mnemic.Client.Remote;
using Mnemic.Client.Shared;

await using var database = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => {
        options.Collection<Product>("[Catalog].[Products]", collection => collection.HasKey(product => product.Sku));
        options.Collection<Order>("[Shop].[Orders]", collection => collection.HasKey(order => order.Id));
    });

var products = database.GetCollection<string, Product>("[Catalog].[Products]");
var orders = database.GetCollection<Guid, Order>("[Shop].[Orders]");

await products.InsertAsync(new Product { Sku = "ELE-2001", Name = "Aurora headphones", Price = 2490m, Stock = 12 });

await using (var transaction = await database.BeginTransactionAsync()) {
    await orders.InsertAsync(new Order { Id = Guid.NewGuid(), Sku = "ELE-2001", Quantity = 2 });
    await products.UpdateAsync("ELE-2001", update => update.Set(product => product.Stock, product => product.Stock - 2));

    await transaction.CommitAsync();
}

public sealed class Product {
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    public int Stock { get; set; }
}

public sealed class Order {
    public Guid Id { get; set; }
    public string Sku { get; set; } = "";
    public int Quantity { get; set; }
}
```

The order and the lower stock are committed together, or, when anything fails before `CommitAsync`, neither is.

## One transaction at a time

A transaction belongs to the session, so a `MnemicDatabase` has at most one open, and the writes of every caller using that database while it is open belong to it. Work that should commit independently opens a database of its own ([how long to keep a database open](connecting.md#how-long-to-keep-a-database-open)).

## When a statement fails

A statement that fails inside a transaction leaves it able only to roll back: every later statement is refused with `TX_ABORTED`, and the transaction is disposed and begun again.

Two transactions that write the same data at the same time cannot both commit: the second write is refused with an error of class `Conflict`, such as `TX_CONFLICT_FWW`. The whole transaction is then run again, reading what it decides on again:

```csharp
for (var attempt = 1; ; attempt++) {
    try {
        await using var transaction = await database.BeginTransactionAsync();
        var product = await products.GetAsync("ELE-2001");

        if (product!.Body.Stock < 1)
            break;

        await orders.InsertAsync(new Order { Id = Guid.NewGuid(), Sku = "ELE-2001", Quantity = 1 });
        await products.UpdateAsync("ELE-2001", update => update.Set(item => item.Stock, product.Body.Stock - 1), new MutationOptions { ExpectedRevision = product.Revision });
        await transaction.CommitAsync();

        break;
    } catch (MqlException failure) when (failure.Class == EngineErrorClass.Conflict && attempt < 5) {
        await Task.Delay(TimeSpan.FromMilliseconds(20 * attempt));
    }
}
```

A transaction begun this way runs with snapshot isolation. [Transactions](../../MqlReference/transactions.md) in the MQL reference describes the isolation levels, `ISOLATION SERIALIZABLE` among them, and a transaction that several connections share.
