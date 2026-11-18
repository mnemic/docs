# Clients

An application works with Mnemic through a client. A client connects to a Mnemic server, sends it statements in [MQL](../MqlReference/README.md) and reads their answers. It has no capability beyond the statements it sends: whatever a client does is a statement any other client can send, so every feature of the server is open to every client.

| Client | What it is for |
|---|---|
| [.NET](DotNet/README.md) | Applications on .NET 11: classes mapped to collections, LINQ queries, transactions and change notifications. |
| [Studio](../installation.md#studio) | Working with a database by hand: writing and running queries, reading and editing documents and their versions, changing the schema, drawing diagrams, and administering users and roles. |

## Connecting to a server

A client connects to one server as one user and opens one database on it. The connection is encrypted with TLS: a server makes a self-signed certificate for itself on its first start and logs its SHA-256 fingerprint, which a client pins, or it is given a certificate that a public authority issued, which a client trusts as it is. Every user signs in with a password, and the roles granted to the user decide which statements it may run ([users and roles](../MqlReference/auth-admin.md)).

[Connecting](../MqlReference/connecting.md) in the MQL reference describes the connection itself: the schemes, TLS and certificate pinning, mutual TLS, the request options beside a statement, and the rules a client follows to keep a connection consistent. It is the contract a client in another language implements.

## Embedded databases

The .NET client can also run the database engine inside the application's own process, on a database file that process opens, with no server. That suits a test suite, a command-line tool or a desktop application that keeps its data to itself. The database then belongs to that one process: other processes and hosts cannot reach it, and the server's users, roles and TLS play no part. [An embedded database](DotNet/connecting.md#an-embedded-database) shows how to open one; everything else in the .NET pages works the same on either.
