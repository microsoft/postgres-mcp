# Postgres MCP Server

[![Platforms: Claude | Copilot | Codex | Open Code](https://img.shields.io/badge/Platforms-Claude_|_Copilot_|_Codex_|_Open_Code-purple.svg)](#2-add-it-to-your-mcp-client)
[![Postgres: local | on-prem | azure | aws | gcp](https://img.shields.io/badge/Postgres-local_|_on--prem_|_azure_|_aws_|_gcp-0f766e.svg)](#requirements)
[![Release status: Preview](https://img.shields.io/badge/Release_status-Preview-blue.svg)](./CHANGELOG.md)

**Connect your coding agent to PostgreSQL.** `postgres-mcp` lets you generate
queries and run analytics, design database schemas, diagnose performance of the
server and queries, securely manage connections, and import data. It is
compatible with any MCP client — GitHub Copilot, Claude Code, Codex, Open Code,
Cursor, VS Code, and more.

- **npm:** https://www.npmjs.com/package/@microsoft/postgres-mcp
- **No install required** — run it straight from `npx`.

## Quick start

### 1. Save a connection profile

Create a named profile once — the password is stored in your OS keyring, never in
a config file:

```sh
# add a profile from a libpq URI or key=value string
npx -y @microsoft/postgres-mcp connection add local \
  "postgresql://postgres@localhost:5432/postgres"

# store its password securely (hidden prompt)
npx -y @microsoft/postgres-mcp connection set-password local

# check it
npx -y @microsoft/postgres-mcp connection list
```

> Profiles created by `connection add` permit write tools unless
> `access_mode: ro` is set. For autonomous agents that should not write, use an
> explicit read-only profile **and** a read-only database role — see
> [Read-only profiles](./USAGE.md#read-only-profiles-for-autonomous-agents).

### 2. Add it to your MCP client

The server speaks MCP over stdio. Point your client at
`npx @microsoft/postgres-mcp run` — no credentials go in the client config; the
coding agent discovers and connects to your saved profile at runtime.

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

> Running headless or in CI, without a keyring? The server can also take a
> database connection string from the environment — see
> [Connecting without a profile](./USAGE.md#connecting-without-a-profile) in the
> guide.

### 3. Ask your coding agent

> "List the tables in my PostgreSQL database."
> "Show me the 10 most recent orders from my PostgreSQL database."
> "What's slowing down my PostgreSQL server right now?"

The agent calls the server's tools (`pgsql_connect`, `pgsql_query`,
`pgsql_db_context`, …) for you.

---

## What it can do

| Area | Tools |
|------|-------|
| **Connections** | list / add / remove profiles, connect, disconnect, list databases |
| **Query** | read‑only SQL (`pgsql_query`), DDL/DML (`pgsql_modify`) |
| **Schema** | fetch `CREATE` scripts for tables, indexes, functions, sequences… |
| **Data** | describe a CSV, bulk‑load a CSV via `COPY` |
| **Diagnostics** | probe server capabilities, collect performance metric groups |

See the [usage guide](./USAGE.md) for tools, configuration, authentication
(including Microsoft Entra ID), TLS, and the security model.

---

## Security in one minute

- **`pgsql_query` is read‑only.** Omitted profile `access_mode` permits write
  tools; set `access_mode: ro` and use a read-only database role unless writes
  are intentionally delegated.
- **Microsoft Entra ID (AAD)** is selected automatically for Azure profiles
  without a stored keyring password. Store a profile password to use password
  authentication instead.
- **Local file reads** (`pgsql_bulk_load_csv`, `pgsql_describe_csv`) are limited
  to the MCP server's startup working directory by default and paths allowed
  with `allow-access-to-path <path>`. A file allows only itself; a directory
  allows recursive access.
- **Secrets are scrubbed** from error messages before they leave the process.

Full details in the [usage guide](./USAGE.md#security--consent).

---

## Requirements

- **Node.js 22+** (to run via `npx`).
- Linux x64/arm64, macOS x64/arm64, or Windows x64. Windows ARM uses x64
  emulation.
- A reachable **PostgreSQL** database — local, Docker, on‑premises, Azure
  Database for PostgreSQL, Amazon RDS/Aurora, Google Cloud SQL/AlloyDB, or any
  wire‑compatible server.

## Changelog

See [CHANGELOG.md](./CHANGELOG.md) for release notes.

## License

Licensed under the [MIT License](https://github.com/microsoft/postgres-mcp/blob/main/LICENSE) © Microsoft.
