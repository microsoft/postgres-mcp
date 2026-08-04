# @microsoft/postgres-mcp

> **Public Preview:** Features and interfaces may change before general availability.

**PostgreSQL for AI assistants — over the Model Context Protocol (MCP).**

Give your AI agent safe, structured access to PostgreSQL: query and modify
databases, explore schema, diagnose performance, manage connections, and
bulk‑load CSVs — from GitHub Copilot, Claude Code, Cursor, VS Code, or any
MCP‑compatible client.

- **npm:** https://www.npmjs.com/package/@microsoft/postgres-mcp
- **Changelog:** [CHANGELOG.md](./CHANGELOG.md)
- **No install required** — run it straight from `npx`.
- **Layered safety** — `pgsql_query` is read‑only; profiles can explicitly block
  write tools; path approval constrains local CSV reads.

---

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
assistant discovers and connects to your saved profile at runtime.

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

**VS Code** (`.vscode/mcp.json`):

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

> Running headless or in CI, without a keyring? The server can also take a
> database connection string from the environment — see
> [Connecting without a profile](./USAGE.md#connecting-without-a-profile) in the
> guide.

### 3. Ask your assistant

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
- **On Windows:** the [Microsoft Visual C++ Redistributable for Visual Studio
  2015-2022 (x64)](https://aka.ms/vs/17/release/vc_redist.x64.exe). A fresh
  Windows installation does not include it, and the server cannot start
  without it.
- A reachable **PostgreSQL** database (self‑hosted, Azure Database for PostgreSQL,
  or any wire‑compatible server).

## License

Licensed under the [MIT License](https://github.com/microsoft/postgres-mcp/blob/main/LICENSE) © Microsoft.
