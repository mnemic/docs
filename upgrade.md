# Upgrade

Mnemic 1.0 is the first release, so there is no earlier release to upgrade from. Every format and protocol it uses is at version `1` ([compatibility](compatibility.md)).

## Evolving your own schema

Your database's schema changes through [migrations](MqlReference/migrations.md): each one moves the database to its next schema level, and an application that is built for a schema level can require it when it opens the database ([mapping classes](Clients/DotNet/collections.md#new-and-existing-collections)). Schema levels are your application's own and are independent of Mnemic's versions.

## Keeping a database whole

A database, its WAL and its backups belong together, and Mnemic reads each only as its own version `1` shape. Never copy WAL files from one database beside another, edit a version marker, delete a file beside the database to make it open, or use a physical backup to convert a database: Mnemic refuses what it cannot read exactly, because reading it anyway could produce a database that looks right and is not.

A user's password hash is strengthened to the current work factor when the user next signs in, with nothing to do on your side.
