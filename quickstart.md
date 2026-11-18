# Quickstarts

Mnemic runs as a server that applications connect to. This page starts a server in a container on your machine and connects to it from a .NET application and from Studio. An application that keeps its database to itself can [embed it](#embedded-net) instead.

## Server container

Choose a password for `sa`, the server's built-in administrator, and start the server:

```shell
docker run -d --name mnemic -p 5150:5150 -v mnemic-data:/data -e MNEMIC_ROOT_PASSWORD=change-this-local-password ghcr.io/mnemic/mnemic
```

The server keeps its databases in the `mnemic-data` volume, listens on port 5150, and encrypts every connection with TLS, using a certificate it makes for itself on its first start. Its log names the certificate's SHA-256 fingerprint, which clients pin, on the line that begins `TLS enabled — certificate SHA-256`:

```shell
docker logs mnemic
```

`docker stop mnemic` stops the server and the volume keeps its databases. [Installation](installation.md#server-container) covers a compose file, a password kept in a secret, and what the container needs.

## Remote .NET

Create a console application and add the remote client:

```shell
dotnet new console --framework net11.0 -n MnemicQuickstart
cd MnemicQuickstart
dotnet add package Mnemic.Client.Remote
```

Replace `Program.cs` with the following, and put the fingerprint from the server's log in place of the placeholder:

```csharp
using Mnemic.Client.Remote;
using Mnemic.Client.Shared.Query;

await using var database = await MnemicDatabaseExtensions.OpenRemoteAsync(
    "tcps://sa:change-this-local-password@localhost:5150/quickstart",
    options => options.Collection<TodoItem>(
        "[App].[Todos]",
        collection => collection.HasKey(item => item.Id)),
    transport => transport.Tls = new ClientTlsOptions {
        PinnedServerCertificateSha256 = "the-fingerprint-from-the-log",
    });

var todos = database.GetCollection<Guid, TodoItem>("[App].[Todos]");

await todos.InsertAsync(new TodoItem { Id = Guid.NewGuid(), Title = "Try Mnemic" });
await todos.InsertAsync(new TodoItem { Id = Guid.NewGuid(), Title = "Read the docs", Done = true });

var open = await todos.AsQueryable()
    .Where(item => !item.Body.Done)
    .ToListAsync();

foreach (var item in open)
    Console.WriteLine(item.Body.Title);

public sealed class TodoItem {
    public Guid Id { get; set; }
    public string Title { get; set; } = "";
    public bool Done { get; set; }
}
```

Run it with `dotnet run`. A connection as `sa` to a database the server does not have creates it, and registering `TodoItem` creates the collection `[App].[Todos]`. Every typed operation is translated to MQL and takes the same path through the engine as a statement written in MQL.

An application connects as a user of its own, with only the roles it needs ([users and roles](MqlReference/auth-admin.md)). The [.NET client](Clients/DotNet/README.md) covers connecting, mapping classes, queries, transactions and change notifications.

## Studio

Studio connects to the same server: give it the host `localhost`, the port `5150`, the user `sa` with its password, and the certificate's fingerprint. It shows the database `quickstart` and its collection, runs MQL against it, and opens its documents. The [MQL reference](MqlReference/README.md) covers the statements to run.

## Embedded .NET

An application whose database belongs to it alone, such as a test suite, a tool or a desktop application, can run the database engine in its own process, on a file, with no server ([embedded databases](Clients/README.md#embedded-databases)). Add the in-process client instead of the remote one:

```shell
dotnet add package Mnemic.Client.InProcess
```

and open the file, with the same `TodoItem` class:

```csharp
using Mnemic.Client.InProcess;

var path = Path.Combine(AppContext.BaseDirectory, "quickstart.mnemic");
await using var local = await MnemicDatabaseExtensions.OpenLocalAsync(
    path,
    options => options.Collection<TodoItem>(
        "[App].[Todos]",
        collection => collection.HasKey(item => item.Id)));

var localTodos = local.GetCollection<Guid, TodoItem>("[App].[Todos]");
var id = Guid.NewGuid();

await localTodos.InsertAsync(new TodoItem { Id = id, Title = "Try Mnemic", Done = false });
var saved = await localTodos.GetAsync(id);
Console.WriteLine(saved?.Body.Title);
```

The file is created when it does not exist, and everything else works as it does against a server.
