# Compatibility

What Mnemic 1.0 runs on, and the versions of the formats and protocols it reads and writes.

## Platforms

- The server runs on Linux x64, in its [container image](installation.md#server-container).
- Applications use the [.NET client](Clients/DotNet/README.md) on .NET 11.
- Studio's desktop application and embedded databases run on Windows x64.

## One version for each format and protocol

Each format and protocol Mnemic owns carries a version of its own, and Mnemic 1.0 reads and writes version `1` of each: the wire protocol, the plugin API, the database file and its storage layout, the WAL and its archive, backups, storage encryption, the catalog, vector-index data, the migration history, authentication records, audit chains, and Studio's settings. The versions are independent, so one boundary can change without the others.

A file, a record or a message of any other version is refused rather than interpreted, and so is one whose version is missing or malformed. Mnemic never guesses at bytes another layout wrote, since reading them as its own could produce a database that looks right and is not.

## Clients, servers and plugins

A client and a server both speak protocol version `1`. Every field and operation of the protocol is mandatory, and a connection does not negotiate versions or capabilities: a server answers a client of another protocol version with `PROTOCOL_VERSION_UNSUPPORTED` ([connecting](MqlReference/connecting.md)).

A plugin declares API version `1` and implements the whole plugin interface. The server loads no plugin written against another version.

## Versions that are not Mnemic's

TLS 1.2 and 1.3, the package versions of the NuGet packages and the container image, the versions an extension gives its own payloads, and the counters Mnemic keeps (commit sequence numbers, log positions, temporal revisions, key generations and archive generations) are not format versions. Neither are the schema and migration levels of your own database, which your application owns ([migrations](MqlReference/migrations.md)).

A user's password hash is strengthened to the current work factor when the user next signs in. That is security maintenance inside version `1` of the authentication record, not a change of format.
