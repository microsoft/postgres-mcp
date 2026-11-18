# Changelog

## v0.3.1

- Added Linux D-Bus socket fallback (`/run/user/<euid>/bus`) for OS keyring
  access when `DBUS_SESSION_BUS_ADDRESS` and `XDG_RUNTIME_DIR` are unset,
  allowing Codex to run without extra `env_vars` (#13).
- Fixed `azure_query_store` diagnostics parameter-count mismatches by removing
  drill-down-only queries from the group and defaulting the native collection
  window to a 1-hour lookback (#17).
- Fixed `postgres_mcp_db_context` to respect `schemaName` when `objectType` is
  `all` (#18).
- Fixed long-lived Microsoft Entra ID connection pools by refreshing tokens
  dynamically, validating and recovering stale pools on `connect`, and
  surfacing underlying PostgreSQL and Entra errors (#19).
- Fixed CSV query result formatting to use RFC 4180 quoting and escaping for
  commas, quotes, and newlines (#20).
- Fixed `postgres_mcp_modify` to preserve and format `RETURNING` result rows,
  including duplicate column labels (#21).
- Fixed `postgres_mcp_db_context` to preserve `GENERATED ... AS IDENTITY`
  options, including custom or renamed sequence names, in `CREATE TABLE`
  scripts (#22).

## v0.3.0

- Bug fixes.

## v0.2.0

- Renamed MCP tools from `pgsql_*` to `postgres_mcp_*` and environment variables
  from `PGSQL_MCP_*` to `POSTGRES_MCP_*`.
- Changed MCP tool arguments to camelCase; stored connection profile keys are
  unchanged.
- Added optional `tenantId` and `loginAsUser` profile settings for Entra ID,
  so a connection can authenticate in a chosen tenant and sign in as a
  Microsoft Entra security group.
- Managed identity is now opt-in via `POSTGRES_MCP_MANAGED_IDENTITY`.

## v0.1.0

- Initial public preview of the PostgreSQL MCP server.
- Added PostgreSQL query, modification, schema, CSV, diagnostics, and connection
  management tools.
- Added saved connection profiles with OS keyring and Microsoft Entra ID
  authentication support.
