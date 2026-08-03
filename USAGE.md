# postgres-mcp — Usage Guide

A guide to configuring and using **`@microsoft/postgres-mcp`**, the
PostgreSQL Model Context Protocol (MCP) server.

- [Overview](#overview)
- [Running the server](#running-the-server)
- [Configuring your MCP client](#configuring-your-mcp-client)
- [Connecting to a database](#connecting-to-a-database)
- [Authentication](#authentication)
- [Security & consent](#security--consent)
- [TLS / SSL](#tls--ssl)
- [Tools reference](#tools-reference)
- [Common CLI commands](#common-cli-commands)
- [Environment variables](#environment-variables)
- [Files & locations](#files--locations)
- [Telemetry & privacy](#telemetry--privacy)
- [Troubleshooting](#troubleshooting)

---

## Overview

`postgres-mcp` is an MCP **server** with connection-management utilities. It
exposes PostgreSQL operations as tools an AI assistant can call. It speaks
**JSON‑RPC 2.0 over stdio** (MCP protocol version `2024-11-05`) and is launched
by your MCP client.

It ships as a self‑contained binary wrapped in an npm package, so the usual way
to run it is `npx @microsoft/postgres-mcp …` — no global install and no
toolchain required.

---

## Running the server

```sh
# start the MCP server (stdio) — this is what your MCP client runs for you
npx -y @microsoft/postgres-mcp run

# the same binary also exposes a CLI for managing connections
npx -y @microsoft/postgres-mcp connection list
```

Logs go to **stderr** only — `stdout` is reserved for MCP protocol frames.

---

## Configuring your MCP client

The server is started by your client via `command` + `args`.

**GitHub Copilot CLI, Claude Code, Cursor & Claude Desktop** — clients that use the `mcpServers` format (`~/.copilot/mcp-config.json`, `mcp.json`, `.cursor/mcp.json`, `claude_desktop_config.json`, …):

```jsonc
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@microsoft/postgres-mcp", "run"]
    }
  }
}
```

**VS Code** — `.vscode/mcp.json` (uses the `servers` key):

```jsonc
{
  "servers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@microsoft/postgres-mcp", "run"]
    }
  }
}
```

---

## Connecting to a database

The recommended way is a **saved profile** — credentials live in your OS keyring
and nothing sensitive goes in the client config.

### Saved connection profiles (recommended)

Create named profiles stored in `~/.postgres-mcp/connections.yaml`, with passwords
in your **OS keyring** (never written to disk in plaintext):

Your assistant can manage profiles with `pgsql_add_connection`,
`pgsql_list_connection_profiles`, `pgsql_remove_connection`, and
`pgsql_connect`. Passwords are set separately with the CLI so they are stored in
the keyring:

```sh
# add from a URI or key=value string
npx -y @microsoft/postgres-mcp connection add prod \
  "host=db.example.com user=app dbname=app sslmode=verify-full"

# store the password (hidden prompt by default)
npx -y @microsoft/postgres-mcp connection set-password prod

# list / remove
npx -y @microsoft/postgres-mcp connection list
npx -y @microsoft/postgres-mcp connection remove prod
```

### Read-only profiles for autonomous agents

Use two independent controls:

- Add the profile with `pgsql_add_connection` and set `access_mode` to `ro`.
  This blocks write tools for that connection.
- Give the database user read-only permissions at the database/server level.

### Connecting without a profile

For headless / CI environments (no OS keyring), set `PGSQL_MCP_CONNECTION_STRING`
to a libpq URI or key=value string. It creates an implicit profile named by
`PGSQL_MCP_PROFILE_NAME` (default `default`). Use a name that is different from
all saved profiles.

```
postgresql://user:password@host:5432/dbname?sslmode=require
host=host port=5432 user=user password=secret dbname=dbname sslmode=require
```

Because it carries credentials in an environment variable (visible to other
processes in the same environment), prefer a saved profile on interactive
machines and reserve this for containers / CI where a keyring isn't available.

---

## Authentication

### Password (OS keyring)

`connection set-password <name>` stores the password in the OS keyring
(Keychain on macOS, Secret Service on Linux, Credential Manager on Windows) under
the service name `postgres-mcp`. The password is read at connect time and never
persisted to the YAML config.

Non‑interactive/CI: set `PGSQL_MCP_PASSWORD` and run `set-password` without a
prompt. Avoid the `--password` flag (it appears in `ps` and shell history — the
server prints a warning if you use it).

### Microsoft Entra ID (AAD) — Azure Database for PostgreSQL

If a saved profile's host looks like an Azure PostgreSQL host
(`*.postgres.database.azure.com`) and no keyring password is stored, the server
authenticates with an Entra ID token via **`DefaultAzureCredential`**. That
credential chain automatically picks up, in order: environment credentials
(`AZURE_CLIENT_ID` / `AZURE_TENANT_ID` / `AZURE_CLIENT_SECRET`), managed
identity, and the Azure CLI (`az login`).

To use password authentication instead, store a keyring password with
`connection set-password <name>`; a stored password takes precedence over
Entra ID.

### Connection string

The `PGSQL_MCP_CONNECTION_STRING` profile uses the credentials embedded in the
string and does not use the keyring or Entra ID authentication.

---

## Security & consent

The server supports a constrained posture for autonomous agents. Unless writes
are intentionally delegated, configure an explicit `access_mode: ro` profile
and a read-only database role as described in
[Read-only profiles](#read-only-profiles-for-autonomous-agents). One additional
fail‑closed path-approval gate protects local-file operations from prompt
injection.

### Local file reads — `allow-access-to-path`

`pgsql_bulk_load_csv` streams a file's contents into a table you can then read
back, and `pgsql_describe_csv` reveals a file's column names and structure — so
an unrestricted path is effectively an arbitrary local‑file read. The server only
reads CSVs whose **canonical path** (symlinks and `..` resolved) is an approved
file or is under an approved directory:

```sh
npx -y @microsoft/postgres-mcp allow-access-to-path /data/imports
```

- The **MCP server's startup working directory is approved by default** (unless
  it is your home directory or the filesystem root). This is the directory from
  which the MCP client starts the server, and is not necessarily the agent's
  workspace. Disable that default with `PGSQL_MCP_DISABLE_CWD_ACCESS=1`.
- Approvals persist in `~/.postgres-mcp/approved_paths.yaml`. Headless/CI:
  `PGSQL_MCP_ALLOWED_PATHS` (platform path list).

### Other protections

- **Read‑only queries.** `pgsql_query` rejects multi‑statement input and is
  read‑only; writes must go through `pgsql_modify`, which honors a read‑only
  connection flag.
- **Secret redaction.** Connection URIs, passwords, bearer tokens, JWTs, and
  Azure connection strings are stripped from tool **error** text before it
  leaves the process (paths and hostnames are kept so errors stay actionable).
- **Frame cap.** A single inbound JSON‑RPC message is capped (default 32 MiB,
  see `PGSQL_MCP_MAX_FRAME_BYTES`) to bound memory against a hostile client.

> **Scope note:** this gate protects the MCP tool surface. An agent host that
> also gives the model a raw shell can bypass it — grant shell access
> accordingly.

---

## TLS / SSL

Set `sslmode` in the connection string or profile:

| `sslmode` | Behavior |
|-----------|----------|
| `disable` | No TLS (cleartext). The server warns for non‑loopback hosts. |
| `allow` / `prefer` | Opportunistic TLS without certificate verification. |
| `require` | Encrypted, but the certificate is **not verified** (libpq parity). The server warns for non‑loopback hosts. |
| `verify-ca` / `verify-full` | Encrypted; the CA and hostname are verified. |

**Entra ID (AAD) connections always verify** the certificate.

**Trusted CAs.** On a verifying connection the trust store is the bundled Mozilla
roots, **plus** your host OS trust store, **plus** any PEM files listed in
`PGSQL_MCP_EXTRA_CA_CERTS` — so a private/corporate CA works with `verify-full`
without rebuilding. For **AAD** connections the OS store is deliberately
excluded (only bundled roots + `PGSQL_MCP_EXTRA_CA_CERTS`), so a machine‑trusted
MITM CA cannot impersonate the Azure host and capture your token.

`PGSQL_MCP_EXTRA_CA_CERTS` is a platform path list: `:`‑separated on
macOS/Linux, `;`‑separated on Windows.

---

## Tools reference

| Tool | Purpose |
|------|---------|
| `pgsql_list_connection_profiles` | List saved profiles (+ the env profile, if any). |
| `pgsql_add_connection` | Add a profile. |
| `pgsql_remove_connection` | Remove a profile; keyring cleanup is best-effort. |
| `pgsql_connect` | Open a pooled connection (by profile id). |
| `pgsql_disconnect` | Close a connection. |
| `pgsql_list_databases` | List databases on the server. |
| `pgsql_db_context` | Fetch `CREATE` scripts (tables, indexes, functions, sequences, …). |
| `pgsql_query` | Run a **read‑only** SQL query. |
| `pgsql_modify` | Run DDL / DML (`CREATE` / `ALTER` / `INSERT` / `UPDATE` / …). |
| `pgsql_describe_csv` | Describe a CSV file's structure (YAML). |
| `pgsql_bulk_load_csv` | Bulk‑load a CSV into a table via `COPY`. |
| `pgsql_get_server_capabilities` | Probe available diagnostic capabilities. |
| `pgsql_get_metrics_group` | Collect a group of performance/diagnostic metrics. |

> **Note:** `pgsql_add_connection` never accepts a password — set passwords out
> of band with the `connection set-password` CLI so they land in the OS keyring.

---

## Common CLI commands

```
npx -y @microsoft/postgres-mcp <command>
```

| Command | Description |
|---------|-------------|
| `run [--log-level L] [--no-telemetry]` | Start the MCP server over stdio. |
| `connection list` | List profiles, their hosts, and whether a password is set. |
| `connection add <name> "<connection-string>"` | Add a profile from a URI or key=value string. Names allow `a‑z A‑Z 0‑9 _ -`. |
| `connection set-password <name> [--password X]` | Store a password in the OS keyring (hidden prompt by default). |
| `connection remove <name> [-f]` | Remove a profile; keyring cleanup is best-effort. |
| `allow-access-to-path <path>` | Allow CSV access to an existing file or directory. |

`connection` and `allow-access-to-path` commands accept `--log-level` too
(default `warning`).

---

## Environment variables

All server configuration uses the `PGSQL_MCP_` prefix.

| Variable | Purpose | Default |
|----------|---------|---------|
| `PGSQL_MCP_CONNECTION_STRING` | libpq URI or key=value string for an environment profile. | *(unset)* |
| `PGSQL_MCP_PROFILE_NAME` | Name of the environment profile. | `default` |
| `PGSQL_MCP_PASSWORD` | Password for `connection set-password` in non‑interactive/CI (skips the hidden prompt). | *(unset)* |
| `PGSQL_MCP_QUERY_TIMEOUT_MS` | Query execution timeout in milliseconds. Unset uses the default; zero, negative, or invalid values disable it. | `540000` (9 min) |
| `PGSQL_MCP_ALLOWED_PATHS` | Platform path list of files or directories approved for CSV access (`:` on Unix, `;` on Windows). | *(unset)* |
| `PGSQL_MCP_DISABLE_CWD_ACCESS` | Set to a non‑empty value to remove the MCP server's startup working directory from the default CSV-access paths. | *(unset)* |
| `PGSQL_MCP_EXTRA_CA_CERTS` | Platform path list (`:` unix / `;` windows) of PEM files with extra CA certificates to trust on verified TLS. | *(unset)* |
| `PGSQL_MCP_MAX_FRAME_BYTES` | Max size (bytes) of one inbound JSON‑RPC frame. Invalid/non‑positive keeps the default. | `33554432` (32 MiB) |
| `PGSQL_MCP_DISABLE_KEYRING` | Makes `connection set-password` fail fast with a "use a connection string instead" message on hosts without an OS keyring (headless/CI), rather than attempting a store. Only affects that CLI command — the server connects fine without a keyring. | *(unset)* |
| `PGSQL_MCP_LOG` | Increase log verbosity for troubleshooting (e.g. `debug`). Overrides `--log-level`. | `info` (run) |

**External (Entra ID):** `DefaultAzureCredential` reads the standard Azure SDK
variables — `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_CLIENT_SECRET`, etc. —
and falls back to managed identity / `az login` when they are absent.

---

## Files & locations

Configuration files live under `~/.postgres-mcp/` (mode `0700` on Unix):

| File | Contents |
|------|----------|
| `connections.yaml` | Saved connection profiles (no passwords). |
| `approved_paths.yaml` | Files and directories approved for CSV reads. |
| `machine-id` | Fallback stable telemetry identifier when no OS machine ID is available. |

---

## Telemetry & privacy

The server emits pseudonymous usage telemetry (machine, platform, and session
identifiers; tool names, durations, and outcomes — no query text, data, or
credentials). Disable it with `run --no-telemetry`.

---

## Troubleshooting

- **"Profile with ID '' not found"** — the `profileId` passed to `pgsql_connect`
  was empty; list profiles first (`pgsql_list_connection_profiles`) and use the
  returned id.
- **"Refusing to read '…' … outside the approved bulk-load paths"** — approve
  the file or its parent directory with `allow-access-to-path <path>` (or set
  `PGSQL_MCP_ALLOWED_PATHS`).
- **"OS keyring backend unavailable"** (headless/Docker) — provide credentials
  via `PGSQL_MCP_CONNECTION_STRING`; no keyring is needed to connect.
- **`verify-full` fails against a corporate CA** — point
  `PGSQL_MCP_EXTRA_CA_CERTS` at the CA's PEM file, or add it to your OS trust
  store (non‑AAD connections trust the OS store automatically).
- **See more logs** — set `PGSQL_MCP_LOG=debug` (logs go to stderr).
