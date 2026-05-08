# guide_mcp_servers_pt2

> Stream E — MCP Server configuration, scoped tools, and approval workflows (servers 6–9) + universal rules.  
> Continued from: `guide_mcp_servers_pt1.md`

---

## Table of Contents

1. [FastAPI-docs MCP Server](#1-fastapi-docs-mcp-server)
2. [Context7 MCP Server](#2-context7-mcp-server)
3. [Playwright MCP Server](#3-playwright-mcp-server)
4. [InfluxDB MCP Server](#4-influxdb-mcp-server)
5. [Universal MCP Security Rules](#5-universal-mcp-security-rules)

---

## 1. FastAPI-docs MCP Server

**Repo**: `tadata-org/fastapi_mcp` | **Transport**: Streamable HTTP (embedded in FastAPI app)

**Function**: Mounts the FastAPI OpenAPI schema as an MCP-accessible resource; the AI agent can read API specs and issue validated requests.

**Available tools**: All FastAPI routes exposed as MCP tools (derived from OpenAPI spec).  
**Minimum scoped toolset**: Read-only inspection of the spec + GET endpoints.

**Configuration** (within FastAPI app):
```python
from fastapi_mcp import FastApiMCP

mcp = FastApiMCP(app, name="aibuilder-api", description="AI Builder API Schema")
mcp.mount()  # Mounts at /mcp by default
```

**External config**:
```json
{
  "fastapi-docs": {
    "type": "http",
    "url": "${env:API_INTERNAL_URL}/mcp"
  }
}
```

**User-approval required**:
- Any POST/PUT/DELETE tool call derived from the OpenAPI spec
- No destructive operations expected in schema introspection mode

---

## 2. Context7 MCP Server

**Docs**: `context7.com` | **Transport**: Streamable HTTP (cloud-hosted)

**Function**: Fetches real-time, up-to-date documentation for any library and injects into LLM context.

**Available tools**:
- `resolve-library-id` — maps library name to Context7 ID
- `get-library-docs` — fetches paginated docs for a specific library version

**Minimum scoped toolset**: both tools (core functionality)  
**No destructive operations** — read-only documentation fetching.

**Configuration**:
```json
{
  "context7": {
    "type": "http",
    "url": "https://mcp.context7.com/mcp",
    "headers": {
      "Authorization": "Bearer ${env:CONTEXT7_API_KEY}"
    }
  }
}
```

**Use cases** for this skill:
- Fetch LangGraph latest API docs before writing any LangGraph code.
- Fetch FastAPI, Pydantic v2, Qdrant, mem0 docs when referencing APIs.
- Validate that recommended patterns match the currently installed version.

---

## 3. Playwright MCP Server

**Repo**: `microsoft/playwright-mcp` | **Transport**: `stdio` / HTTP (`--port` flag)

**Core tools**:
- `browser_navigate` — navigate to URL
- `browser_click` — click element by selector
- `browser_fill_form` — fill form fields
- `browser_screenshot` — visual screenshot
- `browser_snapshot` — accessibility tree snapshot
- `browser_evaluate` — execute JavaScript on page

**Minimum scoped toolset for E2E testing**: `navigate`, `click`, `fill_form`, `screenshot`, `snapshot`  
**Disable**: `browser_evaluate`, `browser_run_code_unsafe` (arbitrary JS execution)

**Configuration**:
```json
{
  "playwright": {
    "type": "stdio",
    "command": "npx",
    "args": ["@playwright/mcp", "--headless", "--no-sandbox"]
  }
}
```

**User-approval required**:
- Any action against a production URL (form submissions, file uploads, state mutations)
- `browser_evaluate` — executes arbitrary JavaScript (RCE risk)

**Usage pattern in E2E testing**:
1. Agent navigates to staging URL (auto-approved).
2. Agent fills form and clicks submit on staging (auto-approved).
3. Any action on production requires `human_approval_required=True`.

---

## 4. InfluxDB MCP Server

**Docs**: `docs.influxdata.com/influxdb3/enterprise/admin/mcp-server` | **Transport**: Streamable HTTP

**Available tools**:
- `query` — execute SQL queries against InfluxDB 3
- `list_databases` — list databases/buckets
- `describe_table` — show table schema
- `create_token` — create API token
- `delete_token` — delete API token
- `write` — write Line Protocol data

**Minimum scoped toolset**: `query` (SELECT only), `list_databases`, `describe_table`  
**Disable by default**: `write`, `create_token`, `delete_token`

**Configuration**:
```json
{
  "influxdb": {
    "type": "http",
    "url": "${env:INFLUXDB_MCP_URL}",
    "headers": {
      "Authorization": "Bearer ${env:INFLUX_TOKEN}"
    }
  }
}
```

**User-approval required**:
- `write` — injects data into time-series DB
- `create_token` — creates new access credential
- `delete_token` — revokes access (irreversible until recreated)
- Any DROP TABLE equivalent

**Security**: Use a read-only InfluxDB token for the MCP server in development/research tasks; reserve write token for IoT ingest pipeline only.

---

## 5. Universal MCP Security Rules

These rules apply to **all 9 MCP servers** without exception:

### 5.1 Minimum Scope Principle

- Each MCP server exposes **only the minimum required tools** for its declared purpose.
- Disable all tools not explicitly listed in the "Minimum scoped toolset" for that server.
- Audit enabled tools on each SKILL activation; log any unexpected tool exposure.

### 5.2 User-Approval Gate

- **Any tool marked "User-approval required"** must set `human_approval_required = True` in LangGraph state.
- LangGraph `interrupt_before` must be configured on the `CONFIRM` and `IMPLEMENT` nodes.
- Approval messages must describe the exact operation, affected resource, and expected effect.

### 5.3 Credential Management

- All MCP server credentials stored as **environment variables** only.
- Never embed tokens in `mcp_servers` config JSON as literal values.
- Use `${env:VAR_NAME}` syntax for all secrets in MCP config blocks.
- Rotate tokens after any suspected compromise; n8n ntfy alert on failed auth.

### 5.4 Audit Logging

- Every MCP tool call must be logged with: timestamp, tool name, parameters (redacted sensitive fields), result status.
- Privileged tool calls logged at WARN level with OTel span attached.
- LangFuse used to trace all MCP calls that involve LLM decisions.

### 5.5 Multi-Instance Configuration

| Server | Multi-instance pattern |
|---|---|
| GitHub | Separate PAT per environment; `GITHUB_TOKEN_DEV`, `GITHUB_TOKEN_PROD` |
| PostgreSQL | Separate `DATABASE_URL` per environment |
| Qdrant | Separate `QDRANT_URL` per environment |
| n8n | Separate `N8N_MCP_URL` per environment |
| InfluxDB | Separate `INFLUX_TOKEN` per environment (read-only for dev) |
| LangFuse | Single instance acceptable (all envs share observability) |
| Playwright | Staging vs production target URLs controlled by env var |
| GitHub FastAPI-docs | `API_INTERNAL_URL` varies by environment |
| Context7 | Single cloud instance; no environment isolation needed |
