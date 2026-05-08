# guide_mcp_servers_pt1

> Stream E — MCP Server configuration, scoped tools, and approval workflows (servers 1–5).  
> Continued in: `guide_mcp_servers_pt2.md`

---

## Table of Contents

1. [GitHub MCP Server](#1-github-mcp-server)
2. [PostgreSQL MCP Server](#2-postgresql-mcp-server)
3. [Qdrant MCP Server](#3-qdrant-mcp-server)
4. [n8n MCP Server](#4-n8n-mcp-server)
5. [LangFuse MCP Server](#5-langfuse-mcp-server)

---

## 1. GitHub MCP Server

**Repo**: `github/github-mcp-server` | **Transport**: `stdio` (local) / SSE (remote)

**Minimum scoped toolset for this skill**: `repos`, `issues`, `pull_requests`  
**Disable**: `actions`, `code_security`, `gists`, `dependabot`

**Configuration** (`mcp_servers` block):
```json
{
  "github": {
    "type": "stdio",
    "command": "github-mcp-server",
    "args": ["stdio", "--toolsets=repos,issues,pull_requests"],
    "env": {
      "GITHUB_PERSONAL_ACCESS_TOKEN": "${env:GITHUB_TOKEN}"
    }
  }
}
```

**User-approval required** (irreversible or externally visible):
- `push_files` — modifies repository content
- `delete_file` — permanent file deletion
- `create_pull_request` — creates external PR
- `merge_pull_request` — merges PR (irreversible)
- `issue_write` — posts visible comments/issues
- `pull_request_review_write` — posts visible review

**Read-only mode** (safe default for research tasks): add `--read-only` flag.

**Security**: Scope the PAT to the minimum required permissions (`repo`, `read:org`). Never use a PAT with `admin` scope.

---

## 2. PostgreSQL MCP Server

**Repo**: community (multiple implementations) | **Transport**: `stdio`

**Minimum scoped toolset**: `execute_query` (SELECT only), `get_schema`, `describe_table`, `list_tables`  
**Disable by default**: `execute_delete`, `execute_alter`, `execute_migration`

**Configuration**:
```json
{
  "postgres": {
    "type": "stdio",
    "command": "python",
    "args": ["-m", "mcp_server_postgres"],
    "env": {
      "DATABASE_URL": "${env:POSTGRES_URL}",
      "READ_ONLY_MODE": "true"
    }
  }
}
```

**User-approval required**:
- `execute_delete` — data loss, irreversible without backup
- `execute_alter` — schema changes, potentially irreversible
- `execute_migration` — large DDL; test on staging first
- Any bulk `execute_update` without a WHERE clause validator

**Multi-instance**: Use separate `DATABASE_URL` env vars for `_dev`, `_staging`, `_prod`.  
**Security**: Never allow raw user-supplied SQL; tool should use parameterised templates only.

---

## 3. Qdrant MCP Server

**Repo**: `qdrant/mcp-server-qdrant` | **Transport**: `stdio` / SSE / Streamable HTTP

**Available tools**: `qdrant-store` (upsert), `qdrant-find` (search)  
**Minimum scoped toolset**: both enabled (core functionality)

**Configuration**:
```json
{
  "qdrant": {
    "type": "stdio",
    "command": "uvx",
    "args": ["mcp-server-qdrant"],
    "env": {
      "QDRANT_URL": "${env:QDRANT_URL}",
      "QDRANT_API_KEY": "${env:QDRANT_API_KEY}",
      "COLLECTION_NAME": "ai_rules",
      "QDRANT_READ_ONLY": "false",
      "QDRANT_SEARCH_LIMIT": "20"
    }
  }
}
```

**User-approval required**:
- `qdrant-store` with `overwrite=true` on existing document IDs
- Any collection-level deletion (done via Qdrant admin API, not MCP)

**Gotcha**: `EMBEDDING_MODEL` defaults to `sentence-transformers/all-MiniLM-L6-v2` (384 dims); override to match production embedding dimension (1536 for `text-embedding-3-small`).

---

## 4. n8n MCP Server

**Docs**: `docs.n8n.io/advanced-ai/mcp` | **Transport**: Streamable HTTP (self-hosted)

**Available tools** (n8n exposes custom tools based on workflows tagged `McpTool`):
- Trigger named webhooks / workflows
- Retrieve workflow execution status
- List active workflows

**Minimum scoped toolset**: trigger specific named workflows (deny wildcard execution)

**Configuration**:
```json
{
  "n8n": {
    "type": "http",
    "url": "${env:N8N_MCP_URL}",
    "headers": {
      "Authorization": "Bearer ${env:N8N_MCP_TOKEN}"
    }
  }
}
```

**User-approval required**:
- Deleting a workflow
- Disabling a workflow
- Any workflow execution that modifies external production state

**Security**: n8n MCP URL must be on internal network only; never expose to public internet.

---

## 5. LangFuse MCP Server

**Docs**: `langfuse.com/docs/api-and-data-platform/features/mcp-server` | **Transport**: Streamable HTTP

**Available tools**: Read traces, scores, sessions; list datasets; create score annotations

**Minimum scoped toolset**: read-only (traces, scores, datasets)  
**Disable**: delete operations on traces/datasets

**Configuration**:
```json
{
  "langfuse": {
    "type": "http",
    "url": "${env:LANGFUSE_HOST}/api/mcp",
    "headers": {
      "Authorization": "Basic ${env:LANGFUSE_BASIC_AUTH}"
    }
  }
}
```

**User-approval required**:
- `delete_trace` — permanent deletion of observability data
- `delete_dataset` — removes evaluation benchmark data

**Stable version**: LangFuse `2.x` | MCP server available in `2.x` cloud and self-hosted.
