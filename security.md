# Security

What Mnemic protects and what it leaves to the deployment: credentials, transport, data at rest, plugins and backups, and how to report a vulnerability.

## Deployment model

Mnemic is a single-node database for trusted deployments. The server provides mandatory authentication, role-based authorization, TLS, bounded unauthenticated work, audit facilities, and fail-closed integrity checks. It is not an isolation boundary for hostile tenants sharing one server process. Plugins run in-process and are fully trusted code.

## Credentials

The `sa` bootstrap password has no compiled default. The standalone host requires exactly one of `MNEMIC_ROOT_PASSWORD` or `MNEMIC_ROOT_PASSWORD_FILE`. Prefer a mounted UTF-8 secret file, restrict its permissions, rotate credentials through MQL, and avoid credentials in command histories, process arguments, or URLs.

Authentication is always enabled. Rate limits and a process-wide PBKDF2 concurrency bound protect the pre-authentication path. Authentication records must use the exact current v1 shape. A successful login may raise an outdated password work factor; this is current security maintenance, not a record-format upgrade path.

## Transport security

TLS mode `auto` is the standalone default. It creates and persists a self-signed certificate; clients pin the SHA-256 hash printed at startup. TLS mode `file` loads an operator-owned PFX suitable for a CA-issued certificate. Clients accept TLS 1.2 and 1.3. Mnemic exposes no SHA-1 certificate-pin API.

Plain TCP transmits credentials in cleartext. It requires `MNEMIC_TLS=off` plus `MNEMIC_ACCEPT_PLAINTEXT_CREDENTIALS=true`. A listener beyond loopback additionally requires `MNEMIC_UNSAFE_ALLOW_REMOTE_PLAINTEXT=true`. Treat these as isolated development controls. A client that accepts any server certificate (`ClientTlsOptions.AcceptAnyServerCertificate`) must likewise remain confined to development.

## Data at rest

An engine an application hosts can encrypt its storage: given `EngineOptions.EncryptionKey`, or `EncryptionKeyProvider` for keys with stable identifiers and rotation, it encrypts and authenticates its pages, WAL, backups and plugin snapshots with AES-GCM, and it refuses a page or a WAL record that was altered, moved or replaced. The application supplies the keys and keeps them; the engine never derives a key from a password. An existing unencrypted database is migrated into new encrypted storage, not converted in place.

The standalone server and its container image do not configure storage encryption. Encrypt a server's data with the operating system, the filesystem or the cloud volume, covering database files, WAL, backups, archives, temporary files and crash dumps. Keep the keys outside the data volume and test recovery with the real key lifecycle.

Page checksums detect corruption in every database. Without storage encryption they do not conceal what a page holds.

## Network, plugins, backup, and audit

The data listener binds loopback by default. Use firewall rules to admit only expected clients when exposing another address. The diagnostics HTTP listener is loopback-only and omits sensitive labels.

Install plugins only from trusted sources. Every plugin must declare exact API version `1` and implement the complete current contract. Version validation is not a sandbox or malware check.

Backups and WAL archives contain user data. Apply the same confidentiality, integrity, retention, and access controls as the live database. Backup, archive, authentication, and audit records are read only in their version 1 shape ([compatibility](compatibility.md)).

## Reporting a vulnerability

Report a suspected vulnerability privately to [security@mnemic.io](mailto:security@mnemic.io), never in a public forum. Include the affected version and artifact digest, the impact, its prerequisites, a minimal reproduction, and any mitigation you suggest. Remove credentials, private keys and unrelated production data; if a demonstration needs a database, build the smallest synthetic one that reproduces the issue.

Mnemic acknowledges the report, validates its scope and severity, and coordinates a fix and a release with you, crediting you if you want it. Please keep the issue private until a fixed release and upgrade guidance are available. Security fixes are released for the latest 1.x release.
