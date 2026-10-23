# Changelog

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
