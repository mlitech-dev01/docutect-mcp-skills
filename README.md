# docutect-mcp-skills

Agent Skills for [DocuTect AI](https://docutect.com) — connect any MCP-compatible AI client
to the DocuTect AI audit and documentation scanning pipeline.

## What's included

| Skill | Description |
|-------|-------------|
| [`docutect-ai-mcp`](docutect-ai-mcp/SKILL.md) | Scan repos, audit API docs for hallucinations, validate Markdown, generate OpenAPI specs |

## Quick start

### 1 — Get an API key

Sign in at [docutect.com](https://docutect.com), go to **Dashboard → Settings → API Keys**,
and create a key. Keys start with `dtai_`.

### 2 — Add to your MCP client

**Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "docutect-ai": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://api.docutect.com/mcp"],
      "env": {
        "DOCUTECT_API_KEY": "dtai_your_key_here"
      }
    }
  }
}
```

**VS Code Copilot** (`.vscode/mcp.json`):

```json
{
  "servers": {
    "docutect-ai": {
      "type": "http",
      "url": "https://api.docutect.com/mcp",
      "headers": {
        "Authorization": "Bearer dtai_your_key_here"
      }
    }
  }
}
```

**cURL test**:

```bash
curl -s https://api.docutect.com/mcp/health
curl -s https://api.docutect.com/mcp/tools \
  -H "Authorization: Bearer dtai_your_key_here"
```

## Skill format

Skills follow the [agentskills.io specification](https://agentskills.io/specification).
Each skill is a directory containing a `SKILL.md` with YAML frontmatter and optional
`scripts/` and `references/` subdirectories.

## Available tools

| Tool | Quota |
|------|-------|
| `scan_github_repo` | 1 scan / call |
| `scan_documentation_url` | 1 scan / call |
| `run_api_audit` | 1 audit credit |
| `get_audit_status` | Free |
| `get_audit_results` | Free |
| `get_remediation_suggestions` | Free |
| `validate_document` | 1 audit credit |
| `generate_api_documentation` | 1 audit credit |

Monthly limits by plan: **Free** 3 scans / 3 audits · **Pro** 100 scans / 50 audits · **Enterprise** unlimited.

## LangChain integration

See [`docutect-ai-mcp/scripts/langchain_tools.py`](docutect-ai-mcp/scripts/langchain_tools.py)
for ready-to-use `BaseTool` wrappers for LangChain / LangGraph agents.

## License

Proprietary — © DocuTect AI. See individual skill `SKILL.md` for license details.
