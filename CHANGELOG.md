# Changelog

## v0.1.0 (Not yet published)

- Initial public preview of the PostgreSQL MCP server.
- Added PostgreSQL query, modification, schema, CSV, diagnostics, and connection
  management tools.
- Added saved connection profiles with OS keyring and Microsoft Entra ID
  authentication support.
- Added optional `tenantId` and `loginAsUser` profile settings for Entra ID,
  so a connection can authenticate in a chosen tenant and sign in as a
  Microsoft Entra security group.
- Managed identity is now opt-in via `POSTGRES_MCP_MANAGED_IDENTITY`.
