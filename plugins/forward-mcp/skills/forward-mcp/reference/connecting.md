# Connecting an agent to Forward's official MCP server

Unofficial guidance, not a Forward document. The endpoint and authentication below come from Forward's MCP documentation and from a live read-only check against one internal build; the documentation page was not publicly available when this was written.

- **Endpoint:** `<forward-host>/api/mcp`, streamable HTTP, on SaaS and self-hosted Forward. The host is the one you sign in to; no path other than `/api/mcp`.
- **Authentication:** a Forward API token (an access key and a secret) sent as HTTP Basic credentials: `Authorization: Basic base64(accessKey:secret)`. Mint one in Forward under Account, API tokens. The token belongs to a real user and carries that user's organization and per-network permissions. There is no OAuth: clients that only offer OAuth connectors cannot connect; clients that take a static header can.
- **Session:** `initialize` returns an `Mcp-Session-Id` header; send it back on every later request. A request without it answers HTTP 400, an unknown id 404.
- **Enabled per organization:** an organization property ("Enable MCP server") gates the endpoint and is on by default. When it is off, or the license is expired, every call answers HTTP 403 `MCP is not enabled for this organization`. On self-hosted Forward an administrator can change it in the organization preferences (experimental features); on SaaS ask Forward support.
- **Tools:** 14 on the build checked, all read-only; `nqe`, `generate_nqe_query` and `search_docs` need Forward AI enabled for the organization. Read `tools/list`, which is the source of truth.
- **Smoke test:** an MCP `initialize` followed by `tools/list` with the same session id should list the tools; a 401 or 403 says which part is wrong.
