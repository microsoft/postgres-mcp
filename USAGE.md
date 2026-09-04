# postgres-mcp — Usage Guide

A guide to configuring and using **`@microsoft/postgres-mcp`**, the
PostgreSQL Model Context Protocol (MCP) server.

- [Overview](#overview)
- [Running the server](#running-the-server)
- [Configuring your MCP client](#configuring-your-mcp-client)
- [Connecting to a database](#connecting-to-a-database)
- [Authentication](#authentication)
- [Security model](#security-model)
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

**Codex CLI** — `~/.codex/config.toml` (TOML, uses the `mcp_servers` table):

```toml
[mcp_servers.postgres]
command = "npx"
args = ["-y", "@microsoft/postgres-mcp", "run"]
```

**Open Code** — `opencode.json` (uses the `mcp` key; `command` is a single array):

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "postgres": {
      "type": "local",
      "command": ["npx", "-y", "@microsoft/postgres-mcp", "run"]
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

Your assistant can manage profiles with `postgres_mcp_add_connection`,
`postgres_mcp_list_connection_profiles`, `postgres_mcp_remove_connection`, and
`postgres_mcp_connect`. Passwords are set separately with the CLI so they are stored in
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

- Add the profile with `postgres_mcp_add_connection` and set `accessMode` to `ro`.
  This blocks write tools for that connection.
- Give the database user read-only permissions at the database/server level.

### Connecting without a profile

For headless / CI environments (no OS keyring), set `POSTGRES_MCP_CONNECTION_STRING`
to a libpq URI or key=value string. It creates an implicit profile named by
`POSTGRES_MCP_PROFILE_NAME` (default `default`). Use a name that is different from
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

Non‑interactive/CI: set `POSTGRES_MCP_PASSWORD` and run `set-password` without a
prompt. Avoid the `--password` flag (it appears in `ps` and shell history — the
server prints a warning if you use it).

### Microsoft Entra ID (AAD) — Azure Database for PostgreSQL

If a saved profile's host looks like an Azure PostgreSQL host
(`*.postgres.database.azure.com`) and no keyring password is stored, the server
authenticates with an Entra ID token. Credential sources are tried in order:

1. **Workload identity** — `AZURE_TENANT_ID`, `AZURE_CLIENT_ID` and
   `AZURE_FEDERATED_TOKEN_FILE` (AKS and other federated setups).
2. **Service principal** — `AZURE_TENANT_ID`, `AZURE_CLIENT_ID` and
   `AZURE_CLIENT_SECRET`.
3. **Managed identity** — only when `POSTGRES_MCP_MANAGED_IDENTITY=1`; see
   below.
4. **Developer tools** — `az login`, then `azd auth login`.

`AZURE_CLIENT_SECRET` is what tells a service principal apart from a managed
identity: with a secret, `AZURE_CLIENT_ID` names a service principal; without
one it names a user‑assigned managed identity. A secret set without a tenant is
reported as an error rather than silently skipped.

> **Azure CLI 2.54.0 or newer** is required for `az login` to work as a
> credential source.

To use password authentication instead, store a keyring password with
`connection set-password <name>`; a stored password takes precedence over
Entra ID.

#### Managed identity is opt-in

```sh
# system-assigned
POSTGRES_MCP_MANAGED_IDENTITY=1 npx -y @microsoft/postgres-mcp run

# user-assigned — AZURE_CLIENT_ID names the identity
POSTGRES_MCP_MANAGED_IDENTITY=1 AZURE_CLIENT_ID=<client-id> \
  npx -y @microsoft/postgres-mcp run
```

#### Selecting a tenant, or signing in as a group

Two optional profile settings, both ignored for non‑Entra connections:

| Purpose | Tool argument | CLI flag |
|---------|---------------|----------|
| Tenant to mint the token in — a tenant GUID or a verified domain. Use it to reach a server in a tenant you are a guest of. | `tenantId` | `--tenant-id` |
| PostgreSQL role to sign in as, when it differs from the identity in the token. | `loginAsUser` | `--login-as-user` |

```sh
npx -y @microsoft/postgres-mcp connection add prod \
  "host=myserver.postgres.database.azure.com user=app dbname=appdb sslmode=require" \
  --tenant-id contoso.onmicrosoft.com \
  --login-as-user "Contoso DB Admins"
```

`loginAsUser` is what makes **group login** work: on an Entra‑enabled server the
security group is the PostgreSQL role, but a token only ever carries the
*member's* identity — so the role has to be named explicitly. Use the group's
**display name** (`Contoso DB Admins`), not its email address.

Pinning a tenant turns off the managed‑identity source, which can only issue
tokens in its own home tenant.

### Connection string

The `POSTGRES_MCP_CONNECTION_STRING` profile uses the credentials embedded in the
string and does not use the keyring or Entra ID authentication.

---

## Security model

### The server is a gateway, not a policy engine

`postgres-mcp` translates MCP tool calls into PostgreSQL operations and runs
them with the identity and permissions of the role in the connection profile you
selected. It does **not** authorize MCP requests or tool invocations, and it
cannot tell whether a tool call reflects your intent or something the model
inferred, hallucinated, or was manipulated into producing. **Anything your
database role is allowed to do, an agent talking to this server can do.**

Treat the server as plumbing rather than as a security control for
model-generated requests. The boundaries that actually hold are your
**PostgreSQL role privileges**, your **MCP client's approval and governance
settings**, and the **model and content** you let drive the agent.

### Shared responsibility

| Concern | Handled by `postgres-mcp` | Your responsibility |
|---------|---------------------------|---------------------|
| Whether a tool call *should* run | Nothing — requests are not authorized | Client approval prompts and organizational policy |
| What the SQL may read or change | `postgres_mcp_query` is read‑only and single‑statement; write tools honor `access_mode: ro` | PostgreSQL role privileges — the boundary that is actually enforced by the database |
| Which local files may be read | Canonical‑path allowlist for the CSV tools | Keeping that allowlist minimal |
| Credentials | OS keyring / Entra ID; secrets scrubbed from error text | Choosing least‑privilege database users and rotating credentials |
| Trustworthiness of the model and its context | Nothing | Model selection, client selection, vetting skills and other MCP servers |
| Attribution and audit | Pseudonymous telemetry of tool names and outcomes only | Database‑side audit logging, and a dedicated role for agent traffic |

### Threats to plan for

- **Prompt injection.** An agent reads untrusted content — query results, table
  and column comments, file contents, web pages, issue text, output from other
  MCP servers — and any of it can carry instructions. A successful injection
  produces tool calls you never asked for, and this server executes them under
  your role.
- **Approval fatigue.** Most clients prompt before running a tool, but blanket
  "always allow" and auto‑approve modes remove the last human check, and
  repeated prompts train people to click through them.
- **Untrusted agent extensions.** Skills, prompt packs, and additional MCP
  servers all feed the same context window; a malicious one can steer the agent
  toward your database.
- **Ungoverned models.** An unreviewed or self‑hosted model may be poorly
  aligned or deliberately tampered with.

The realistic consequences are unauthorized data access and exfiltration (rows
pulled into the model's context can leave your environment), unauthorized
modification of data or schema, destructive administrative actions, and loss of
attribution — PostgreSQL sees your role, not "the agent", so logs alone cannot
tell you which actions you initiated.

### Hardening checklist

1. **Connect as a least‑privilege role.** Create a dedicated database role for
   agent traffic and grant it only the schemas and tables it needs. Do not point
   the agent at a superuser or table‑owner role for exploratory work.
2. **Default to read‑only.** Set `access_mode: ro` on the profile *and* give the
   database role read‑only permissions — see
   [Read-only profiles](#read-only-profiles-for-autonomous-agents). The profile
   flag is a gate inside this server; the role is enforced by PostgreSQL. Use
   both.
3. **Prefer non‑production data.** Point agents at a development or anonymized
   copy whenever the task does not require live data.
4. **Choose a client with organizational governance.** Some clients let
   administrators centrally allow or deny MCP servers, so `postgres-mcp` runs
   only where your organization sanctions it. GitHub Copilot's
   [enterprise managed settings](https://docs.github.com/en/copilot/reference/enterprise-administrators/enterprise-managed-settings)
   do this with `allowedMcpServers` / `deniedMcpServers` in Copilot CLI, VS Code,
   the Copilot app, and JetBrains IDEs; VS Code also exposes the equivalent as
   [device policies](https://code.visualstudio.com/docs/enterprise/ai-settings#_configure-mcp-server-access).
5. **Keep a human in the loop for writes.** Avoid blanket auto‑approval for
   `postgres_mcp_modify` and `postgres_mcp_bulk_load_csv`, and review the SQL in the request,
   not just the tool name. Where the client supports it, administrators can
   disable allow‑all approval modes centrally — in Copilot clients that is
   `permissions.disableBypassPermissionsMode`.
6. **Vet everything else in the agent's context.** Install skills, prompts, and
   other MCP servers only from reputable sources, and confirm what they do
   first.
7. **Use models from trusted, governed sources**, and assume the model and
   client may still be exposed to prompt injection or manipulated content.
8. **Keep the CSV allowlist tight.** Approve specific directories, and set
   `POSTGRES_MCP_DISABLE_CWD_ACCESS=1` when the startup working directory should
   not be readable.
9. **Audit on the database side.** Enable `log_statement` or `pgaudit`, and use
   a distinct role for agent connections so agent activity is separable from
   yours.
10. **Bound the blast radius.** Set statement and idle timeouts, cap connections
    per role, and keep restorable backups.

### Local file reads — `allow-access-to-path`

`postgres_mcp_bulk_load_csv` streams a file's contents into a table you can then read
back, and `postgres_mcp_describe_csv` reveals a file's column names and structure — so
an unrestricted path is effectively an arbitrary local‑file read. The server only
reads CSVs whose **canonical path** (symlinks and `..` resolved) is an approved
file or is under an approved directory:

```sh
npx -y @microsoft/postgres-mcp allow-access-to-path /data/imports
```

- The **MCP server's startup working directory is approved by default** (unless
  it is your home directory or the filesystem root). This is the directory from
  which the MCP client starts the server, and is not necessarily the agent's
  workspace. Disable that default with `POSTGRES_MCP_DISABLE_CWD_ACCESS=1`.
- Approvals persist in `~/.postgres-mcp/approved_paths.yaml`. Headless/CI:
  `POSTGRES_MCP_ALLOWED_PATHS` (platform path list).

### Other protections

- **Read‑only queries.** `postgres_mcp_query` rejects multi‑statement input and is
  read‑only; writes must go through `postgres_mcp_modify`, which honors a read‑only
  connection flag.
- **Secret redaction.** Connection URIs, passwords, bearer tokens, JWTs, and
  Azure connection strings are stripped from tool **error** text before it
  leaves the process (paths and hostnames are kept so errors stay actionable).
- **Frame cap.** A single inbound JSON‑RPC message is capped (default 32 MiB,
  see `POSTGRES_MCP_MAX_FRAME_BYTES`) to bound memory against a hostile client.

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
`POSTGRES_MCP_EXTRA_CA_CERTS` — so a private/corporate CA works with `verify-full`
without rebuilding. For **AAD** connections the OS store is deliberately
excluded (only bundled roots + `POSTGRES_MCP_EXTRA_CA_CERTS`), so a machine‑trusted
MITM CA cannot impersonate the Azure host and capture your token.

`POSTGRES_MCP_EXTRA_CA_CERTS` is a platform path list: `:`‑separated on
macOS/Linux, `;`‑separated on Windows.

---

## Tools reference

| Tool | Purpose |
|------|---------|
| `postgres_mcp_list_connection_profiles` | List saved profiles (+ the env profile, if any). |
| `postgres_mcp_add_connection` | Add a profile. |
| `postgres_mcp_remove_connection` | Remove a profile; keyring cleanup is best-effort. |
| `postgres_mcp_connect` | Open a pooled connection (by profile id). |
| `postgres_mcp_disconnect` | Close a connection. |
| `postgres_mcp_list_databases` | List databases on the server. |
| `postgres_mcp_db_context` | Fetch `CREATE` scripts (tables, indexes, functions, sequences, …). |
| `postgres_mcp_query` | Run a **read‑only** SQL query. |
| `postgres_mcp_modify` | Run DDL / DML (`CREATE` / `ALTER` / `INSERT` / `UPDATE` / …). |
| `postgres_mcp_describe_csv` | Describe a CSV file's structure (YAML). |
| `postgres_mcp_bulk_load_csv` | Bulk‑load a CSV into a table via `COPY`. |
| `postgres_mcp_get_server_capabilities` | Probe available diagnostic capabilities. |
| `postgres_mcp_get_metrics_group` | Collect a group of performance/diagnostic metrics. |

> **Note:** `postgres_mcp_add_connection` never accepts a password — set passwords out
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
| `connection add <name> "<connection-string>"` | Add a profile from a URI or key=value string. Names allow `a‑z A‑Z 0‑9 _ -`. Optional `--access-mode MODE`, and for Entra ID `--tenant-id TENANT` / `--login-as-user NAME`. |
| `connection set-password <name> [--password X]` | Store a password in the OS keyring (hidden prompt by default). |
| `connection remove <name> [-f]` | Remove a profile; keyring cleanup is best-effort. |
| `allow-access-to-path <path>` | Allow CSV access to an existing file or directory. |

`connection` and `allow-access-to-path` commands accept `--log-level` too
(default `warning`).

---

## Environment variables

All server configuration uses the `POSTGRES_MCP_` prefix.

| Variable | Purpose | Default |
|----------|---------|---------|
| `POSTGRES_MCP_CONNECTION_STRING` | libpq URI or key=value string for an environment profile. | *(unset)* |
| `POSTGRES_MCP_PROFILE_NAME` | Name of the environment profile. | `default` |
| `POSTGRES_MCP_PASSWORD` | Password for `connection set-password` in non‑interactive/CI (skips the hidden prompt). | *(unset)* |
| `POSTGRES_MCP_QUERY_TIMEOUT_MS` | Query execution timeout in milliseconds. Unset uses the default; zero, negative, or invalid values disable it. | `540000` (9 min) |
| `POSTGRES_MCP_ALLOWED_PATHS` | Platform path list of files or directories approved for CSV access (`:` on Unix, `;` on Windows). | *(unset)* |
| `POSTGRES_MCP_DISABLE_CWD_ACCESS` | Set to a non‑empty value to remove the MCP server's startup working directory from the default CSV-access paths. | *(unset)* |
| `POSTGRES_MCP_EXTRA_CA_CERTS` | Platform path list (`:` unix / `;` windows) of PEM files with extra CA certificates to trust on verified TLS. | *(unset)* |
| `POSTGRES_MCP_MAX_FRAME_BYTES` | Max size (bytes) of one inbound JSON‑RPC frame. Invalid/non‑positive keeps the default. | `33554432` (32 MiB) |
| `POSTGRES_MCP_DISABLE_KEYRING` | Makes `connection set-password` fail fast with a "use a connection string instead" message on hosts without an OS keyring (headless/CI), rather than attempting a store. Only affects that CLI command — the server connects fine without a keyring. | *(unset)* |
| `POSTGRES_MCP_LOG` | Increase log verbosity for troubleshooting (e.g. `debug`). Overrides `--log-level`. | `info` (run) |
| `POSTGRES_MCP_MANAGED_IDENTITY` | Set to `1` (or `true`/`yes`/`on`) to allow a managed identity as an Entra ID credential source. Off by default — see [Managed identity is opt-in](#managed-identity-is-opt-in). | *(unset)* |

**External (Entra ID):** the credential chain reads the standard Azure SDK
variables — `AZURE_TENANT_ID`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`,
`AZURE_FEDERATED_TOKEN_FILE` — and falls back to `az login` / `azd auth login`
when they are absent. Managed identity is tried only when
`POSTGRES_MCP_MANAGED_IDENTITY=1`.

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

- **"Profile with ID '' not found"** — the `profileId` passed to `postgres_mcp_connect`
  was empty; list profiles first (`postgres_mcp_list_connection_profiles`) and use the
  returned id.
- **"Refusing to read '…' … outside the approved bulk-load paths"** — approve
  the file or its parent directory with `allow-access-to-path <path>` (or set
  `POSTGRES_MCP_ALLOWED_PATHS`).
- **"OS keyring backend unavailable"** (headless/Docker) — provide credentials
  via `POSTGRES_MCP_CONNECTION_STRING`; no keyring is needed to connect.
- **`verify-full` fails against a corporate CA** — point
  `POSTGRES_MCP_EXTRA_CA_CERTS` at the CA's PEM file, or add it to your OS trust
  store (non‑AAD connections trust the OS store automatically).
- **See more logs** — set `POSTGRES_MCP_LOG=debug` (logs go to stderr).
