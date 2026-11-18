# Connecting

`OpenRemoteAsync` opens a database on a Mnemic server and returns it as a `MnemicDatabase`, which the application disposes when its work is done.

## The connection string

```text
tcps://user:password@host:port/database
```

- `tcps` connects over TLS. `tcp` connects without it, which a server refuses unless its operator turned TLS off and accepted that passwords travel in the clear ([configuration](../../configuration.md#standalone-server-environment)).
- The port is `5150` unless the server listens on another.
- The database is the one the connection opens. A connection as `sa` to a database the server does not have creates it; any other user's is refused with `INSUFFICIENT_PRIVILEGE`.
- A user name, password or database name with a character a URL reserves, such as a space, `@`, `:`, `/` or `#`, is percent-encoded, as `Uri.EscapeDataString` does.

## A certificate an authority issued

```csharp
using Mnemic.Client.Remote;

await using var database = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => { });
```

The client accepts the server's certificate when an authority the operating system trusts issued it for the host name in the address, as a browser does, and it has not been revoked.

## A server's own certificate

A server given no certificate makes a self-signed one on its first start, keeps it with its data and logs its SHA-256 fingerprint at every start. No authority vouches for it, so the client pins it:

```csharp
await using var pinned = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/shop",
    options => { },
    transport => transport.Tls = new ClientTlsOptions {
        PinnedServerCertificateSha256 = "9F86D081884C7D659A2FEAA0C55AD015A3BF4F1B2B0B822CD15D6C15B0F00A08",
    });
```

The pin takes the place of the authority and the host name: the client accepts that certificate and no other, while it is valid. The fingerprint stays the same across restarts until the operator replaces the certificate. A client certificate for mutual TLS is set beside the pin, as `ClientCertificate` ([mutual TLS](../../MqlReference/connecting.md#tls--mutual-auth-mtls)).

## Credentials outside the connection string

A password in the connection string ends up wherever the string is kept. The transport callback takes the credentials instead, and they take the place of any in the string:

```csharp
using System.Net;

await using var signedIn = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://db.example.com:5150/shop",
    options => { },
    transport => transport.Credentials = new NetworkCredential("app", Environment.GetEnvironmentVariable("SHOP_DB_PASSWORD")));
```

## How long to keep a database open

An open `MnemicDatabase` is one session on one connection. The statements sent on it run one after another, in the order they were sent, also when several callers send them at once, so work that should run in parallel opens a database for each. A transaction, a migration, and settings made with `SET` belong to the session.

Open a database for a unit of work, such as a request or a job, and dispose it when the work is done. Disposing it resets the session, which rolls back a transaction it left open and ends its subscriptions, and returns the connection to a pool the process shares: the next open with the same address, database, user and TLS settings takes it without a new handshake or sign-in. The pool keeps up to 16 connections for each such combination, and an open that finds all of them in use waits up to 10 seconds for one before it throws `ConnectionPoolExhaustedException`. [Connection pooling](../../MqlReference/connecting.md#connection-pooling) describes the pool's settings.

Opening a database checks each registered class against its collection, so every open registers the classes it uses. Keeping the registrations in one method lets every unit of work open the database the same way:

```csharp
using Mnemic.Client.Shared;
using Mnemic.Client.Shared.Configuration;

Console.WriteLine(await PriceOfAsync("ELE-2001"));

static async Task<decimal?> PriceOfAsync(string sku) {
    await using var shop = await Shop.OpenAsync();
    var product = await shop.GetCollection<string, Product>("[Catalog].[Products]").GetAsync(sku);

    return product?.Body.Price;
}

static class Shop {
    public static Task<MnemicDatabase> OpenAsync(CancellationToken cancellationToken = default) {
        return MnemicDatabaseExtensions.OpenRemoteAsync(
            "tcps://app:app-password@db.example.com:5150/shop",
            Configure,
            cancellationToken);
    }

    static void Configure(DatabaseOptions options) {
        options.Collection<Product>("[Catalog].[Products]", collection => collection.HasKey(product => product.Sku));
    }
}

public sealed class Product {
    public string Sku { get; set; } = "";
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
}
```

## A connection to the server

A connection string without a database opens no database. It runs the statements that act on the server itself, such as listing its databases:

```csharp
await using var server = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://app:app-password@db.example.com:5150/",
    options => { });

foreach (var name in await server.ListDatabasesAsync())
    Console.WriteLine(name);
```

The server's version is known from the moment the connection opens:

```csharp
var remote = (RemoteClient)database.Client;

Console.WriteLine($"Mnemic {remote.Diagnostics!.ServerSemanticVersion}, protocol {remote.Diagnostics.ProtocolVersion}");
```

## An embedded database

An application that keeps its database to itself can run the database engine in its own process, with no server ([embedded databases](../README.md#embedded-databases)). It installs the in-process client:

```shell
dotnet add package Mnemic.Client.InProcess
```

and opens a database file, which is created when it does not exist:

```csharp
using Mnemic.Client.InProcess;

var path = Path.Combine(AppContext.BaseDirectory, "shop.mnemic");

await using var local = await MnemicDatabaseExtensions.OpenLocalAsync(
    path,
    options => options.Collection<Product>("[Catalog].[Products]", collection => collection.HasKey(product => product.Sku)));

await local.GetCollection<string, Product>("[Catalog].[Products]").InsertAsync(new Product { Sku = "HOM-3001", Name = "Pour-over coffee set", Price = 540m });
```

An embedded database is one session too. `OpenSessionAsync` opens another session over the same engine, for work that runs beside the first; disposing it leaves the engine open, and disposing the database that opened the engine ends every session and closes the file:

```csharp
using Mnemic.Client.Shared.Query;

await using var second = await local.OpenSessionAsync();

var count = await second.GetCollection<string, Product>("[Catalog].[Products]").AsQueryable().CountAsync();
```

Embedded databases are supported on Windows x64.
