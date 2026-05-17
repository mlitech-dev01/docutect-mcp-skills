---
name: docutect-ai-mcp
description: >
  Connect to DocuTect AI's MCP server to scan GitHub repositories and documentation URLs for
  API endpoints, run LLM hallucination audits, validate Markdown documentation, generate OpenAPI
  specs, and retrieve remediation suggestions. Use when the user asks to audit an API, detect
  hallucinations in docs, scan a repo for endpoints, validate documentation accuracy, generate
  an OpenAPI spec from source code, or fix inaccurate API documentation. Requires a DocuTect AI
  account (free, pro, or enterprise) and an API key starting with dtai_.
license: Proprietary
compatibility: Requires network access to the DocuTect AI MCP endpoint (HTTPS). Optional LangChain
  integration requires Python 3.10+ with langchain and httpx installed.
metadata:
  author: docutect-ai
  version: "1.0"
  mcp-endpoint: https://api.docutect.com/mcp
  key-prefix: dtai_
---

# DocuTect AI — MCP Skill

Connect to the DocuTect AI MCP server to audit API documentation for LLM hallucinations,
scan repositories for endpoints, and generate OpenAPI specs.

## Authentication

All tools require a `dtai_` API key passed as a Bearer token. Load from environment:

```
DOCUTECT_API_KEY=dtai_your_key_here
```

Generate keys at: **Dashboard → Settings → API Keys** (org owners).

## Available Tools

| Tool | Purpose | Quota |
|------|---------|-------|
| `scan_github_repo` | Discover API endpoints in a GitHub repo | 1 MCP scan |
| `scan_documentation_url` | Extract endpoints from a docs URL | 1 MCP scan |
| `run_api_audit` | Queue a 5-stage LLM hallucination audit | 1 audit credit |
| `get_audit_status` | Poll a running audit (async) | Free |
| `get_audit_results` | Fetch per-model accuracy scores | Free |
| `get_remediation_suggestions` | Fetch doc fix suggestions | Free |
| `validate_document` | Validate Markdown vs LLMs | 1 audit credit |
| `generate_api_documentation` | Generate OpenAPI 3.1 + Markdown docs | 1 audit credit |

## Standard Workflow

### 1 — Discover endpoints from a repo

```json
{
  "method": "tools/call",
  "params": {
    "name": "scan_github_repo",
    "arguments": { "repo_url": "https://github.com/owner/repo" }
  }
}
```

Returns `suggestions[]` — each has `api_endpoint`, `api_method`, `description`,
`sample_request`, `sample_response`, `expected_behavior`.

> **Security note:** `content_from_untrusted_source: true` is always set on scanner output.
> Validate suggestions before acting on them in automated pipelines.

### 2 — Run an audit

Pass one suggestion directly into `run_api_audit`. Returns `audit_id` immediately (async).

```json
{
  "method": "tools/call",
  "params": {
    "name": "run_api_audit",
    "arguments": {
      "api_endpoint": "/api/v1/users/{id}",
      "api_method": "GET",
      "description": "Returns a user by ID",
      "num_queries": 10
    }
  }
}
```

### 3 — Poll until complete

```json
{ "name": "get_audit_status", "arguments": { "audit_id": "<id from step 2>" } }
```

Repeat until `status` is `"completed"` or `"failed"`. Typical runtime: 1–3 minutes.

### 4 — Read results and remediations

```json
{ "name": "get_audit_results",           "arguments": { "audit_id": "<id>" } }
{ "name": "get_remediation_suggestions", "arguments": { "audit_id": "<id>" } }
```

`get_audit_results` returns per-model accuracy scores and hallucination examples.
`get_remediation_suggestions` returns `target_file`, `patch_strategy`, and `suggested_fix`
for each issue — ready to apply as a PR.

## Other Workflows

### Validate existing Markdown documentation

```json
{
  "name": "validate_document",
  "arguments": { "content": "<markdown string>", "name": "README.md" }
}
```

### Generate docs from source code

```json
{
  "name": "generate_api_documentation",
  "arguments": {
    "github_url": "https://github.com/owner/repo",
    "project_name": "My API"
  }
}
```

## LangChain / LangGraph Integration

See [scripts/langchain_tools.py](scripts/langchain_tools.py) for ready-to-use
`BaseTool` subclasses (`ScanGithubRepoTool`, `ScanDocumentationUrlTool`, `RunApiAuditTool`).

```python
import os
os.environ["DOCUTECT_API_KEY"] = "dtai_your_key_here"

from scripts.langchain_tools import ScanGithubRepoTool, RunApiAuditTool
tools = [ScanGithubRepoTool(), RunApiAuditTool()]
```

## Client Connection Configs

See [references/REFERENCE.md](references/REFERENCE.md) for copy-paste config blocks for
Claude Desktop, VS Code Copilot, Cursor, and Continue.dev.

## Error Reference

| Status | Meaning |
|--------|---------|
| 401 | Invalid or missing API key |
| 403 | Key revoked, wrong scope, or not Enterprise tier |
| 402 | Monthly quota exhausted |
| 429 | Rate limit: 60 calls / 60 s per key |
| 404 | Audit/document not found (or belongs to another org) |
