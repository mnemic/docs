# Change notifications

A client can ask to be told when the documents of a collection change. The server sends a notification for each change that commits, with the document's key, the kind of change and when it happened, but not the document itself, which the application reads when it needs it. Notifications are a signal, not a log: one sent while a connection is down is not sent again.

```csharp
using System.Threading.Channels;
using Mnemic.Client.Remote;
using Mnemic.Client.Shared;

await using var database = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => options.Collection<Product>("[Catalog].[Products]", collection => collection.HasKey(product => product.Sku)));

var products = database.GetCollection<string, Product>("[Catalog].[Products]");
var changes = Channel.CreateUnbounded<DocumentChangeEvent<string>>();

await using var subscription = await products.OnChangeAsync(change => changes.Writer.TryWrite(change));

await products.InsertAsync(new Product { Sku = "ELE-2001", Name = "Aurora headphones", Price = 2490m });

var change = await changes.Reader.ReadAsync();

Console.WriteLine($"{change.ChangeType} {change.DocumentId} at {change.Timestamp:T}");

public sealed class Product {
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
}
```

`ChangeType` says what happened: `Insert`, `Update` or `Delete`, and in a [temporal collection](temporal.md) also `InsertVersion`, `TemporalEnd`, and `VersionActivated` or `VersionExpired` when a scheduled version takes effect or a version's end is reached.

## Handlers

The client runs every handler on the connection's own reader, so the next notification and the answers to the database's statements wait until the handler returns. A handler hands the change on, as the channel above does, and the work it calls for is done elsewhere. Disposing the subscription stops its handler, once a handler already running has returned.

A subscription belongs to the session of the database it was made on and ends when that database is disposed, so a database that delivers notifications stays open for as long as they are wanted. The server also ends a subscription when its collection is dropped; `Completion` then fails with `COLLECTION_NOT_FOUND`.

## One document, or the documents that match

`OnChangeAsync` with a key is told about one document only:

```csharp
await using var headphones = await products.OnChangeAsync("ELE-2001", change => changes.Writer.TryWrite(change));
```

`SubscribeWhereAsync` on the remote client is told about the documents that match a condition in MQL, before or after the change:

```csharp
var remote = (RemoteClient)database.Client;

await using var cheap = await remote.SubscribeWhereAsync(
    "[Catalog].[Products]",
    "[Price] < @limit",
    change => Console.WriteLine($"{change.ChangeType} in {change.CollectionName}"),
    new Dictionary<string, object?> { ["limit"] = 1000m });
```

[Subscriptions](../../MqlReference/subscriptions.md) in the MQL reference describes the conditions a subscription takes and how the server delivers notifications.
