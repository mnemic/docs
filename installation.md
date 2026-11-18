# Installation

What Mnemic runs on, the server container and how to run it, the .NET packages, and Studio.

## Requirements

- Linux x64 for the server, which runs in its container image
- .NET 11 for applications that use the .NET client
- Windows x64 for Studio's desktop application and for embedded databases
- A writable database volume, encrypted by the operating system, the filesystem or the volume when the data must be encrypted at rest ([data at rest](security.md#data-at-rest))

## Server container

The server image is `ghcr.io/mnemic/mnemic`, tagged with each release's version: `ghcr.io/mnemic/mnemic:<version>` names one release. The container:

- runs as UID/GID `1654`;
- stores database state under `/data`;
- listens for database traffic on port `5150`;
- keeps its diagnostics listener on loopback inside the container;
- supports a read-only root filesystem with writable `/data` and `/tmp`;
- keeps its temporary files, such as sort and index-build spill, in `/data/spill` on the data volume rather than the small `/tmp` tmpfs ([`--spill-dir`](configuration.md#standalone-server-arguments) moves them);
- keeps its log in `/data/logs` on the data volume as well as on standard output, because Docker deletes a container's log with the container ([server log](configuration.md#standalone-server-log));
- loads the Vamana vector-index plugin from `/app/plugins`, the default plugin folder (`MNEMIC_PLUGIN_DIR` points elsewhere);
- runs the .NET server garbage collector within the container's processor and memory limits, from which the runtime derives its heap limit and the server its default [admission budget](configuration.md#standalone-server-environment) ([runtime settings](configuration.md#standalone-server-runtime));
- handles `SIGTERM` with a graceful checkpoint and shutdown.

The server requires exactly one of `MNEMIC_ROOT_PASSWORD` or `MNEMIC_ROOT_PASSWORD_FILE`, the password of the built-in `sa` user, and makes a self-signed TLS certificate on its first start unless it is given one ([configuration](configuration.md#standalone-server-environment)). A compose file that runs it with a read-only root filesystem, no added privileges and a bounded log:

```yaml
services:
  mnemic:
    image: ghcr.io/mnemic/mnemic:<version>
    ports:
      - "5150:5150"
    volumes:
      - mnemic-data:/data
    read_only: true
    tmpfs:
      - /tmp:size=64m,mode=1777
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    environment:
      - MNEMIC_TLS=auto
      - MNEMIC_ROOT_PASSWORD_FILE=/run/secrets/mnemic_root_password
    secrets:
      - mnemic_root_password
    restart: unless-stopped
    stop_grace_period: 30s
    mem_limit: 8g
    logging:
      driver: json-file
      options:
        max-size: "16m"
        max-file: "4"

secrets:
  mnemic_root_password:
    file: ./mnemic-root-password.txt

volumes:
  mnemic-data:
```

The password file holds the password alone, and a line break at its end is ignored; the container's user, UID `1654`, must be able to read it. `docker compose down` stops the server and keeps the `mnemic-data` volume; `docker compose down --volumes` deletes it, and the databases with it.

## .NET packages

Applications that connect to a server install:

```shell
dotnet add package Mnemic.Client.Remote --version <version>
```

Applications that run an embedded database in their own process install:

```shell
dotnet add package Mnemic.Client.InProcess --version <version>
```

These entry packages bring their required Mnemic contracts, and the in-process client its language engine, storage engine and production providers, transitively. Do not add internal provider packages unless the application is deliberately composing the engine. The [.NET client](Clients/DotNet/README.md) describes both.

## Studio

Studio runs as a desktop application on Windows x64, or in a browser, served by a server that loads the Studio plugin. It connects through the same client contracts as any application and does not bypass server authorization.
