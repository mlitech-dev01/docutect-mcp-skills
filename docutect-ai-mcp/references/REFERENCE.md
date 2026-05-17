# DocuTect AI — Agent Skills & MCP Integration

Connect Claude Desktop, VS Code Copilot, Cursor, or any LangChain agent to
DocuTect AI's API audit pipeline via the **Model Context Protocol (MCP)**.

---

## What you can do

| Tool | Description | Quota Cost |
|------|-------------|------------|
| `scan_github_repo` | Discover API endpoints in a GitHub repository | 1 MCP scan |
| `scan_documentation_url` | Extract endpoints from a docs URL | 1 MCP scan |
| `run_api_audit` | Queue a 5-stage LLM hallucination audit | 1 audit credit |
| `get_audit_status` | Poll a running audit (async) | Free |
| `get_audit_results` | Fetch per-model accuracy scores | Free |
| `get_remediation_suggestions` | Fetch documentation fix suggestions | Free |
| `validate_document` | Validate a Markdown document vs LLMs | 1 audit credit |
| `generate_api_documentation` | Generate OpenAPI 3.1 + Markdown docs | 1 audit credit |

> **Tier requirement:** All tools require an **Enterprise** subscription. Generate API keys at `https://app.docutect.com/settings/api-keys`.

---

## Quick Start

### 1. Generate an API key

In the DocuTect AI dashboard → Settings → API Keys → **Create key**.

Copy the `dtai_...` key — it is shown once only.

---

## Connection Guides

### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`
(macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

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

Restart Claude Desktop. You should see the DocuTect AI tools listed.

---

### VS Code Copilot (GitHub Copilot Chat — Agent Mode)

In VS Code settings (`settings.json`):

```json
{
  "mcp": {
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
}
```

Open Copilot Chat → select **Agent** mode → the tools are now available.

---

### Cursor

In Cursor → Preferences → MCP Servers:

```json
{
  "mcpServers": {
    "docutect-ai": {
      "url": "https://api.docutect.com/mcp",
      "headers": {
        "Authorization": "Bearer dtai_your_key_here"
      }
    }
  }
}
```

---

### Continue.dev

In `.continue/config.json`:

```json
{
  "tools": [
    {
      "type": "mcp",
      "name": "docutect-ai",
      "transport": {
        "type": "http",
        "url": "https://api.docutect.com/mcp"
      },
      "headers": {
        "Authorization": "Bearer dtai_your_key_here"
      }
    }
  ]
}
```

---

## LangChain / LangGraph Integration

Install dependencies:

```bash
pip install langchain langchain-openai httpx
```

Set environment variables:

```bash
export DOCUTECT_API_KEY=dtai_your_key_here
export OPENAI_API_KEY=sk-...
```

### Basic agent example

```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate
from scripts.agent_skills.langchain_tools import (
    ScanGithubRepoTool,
    ScanDocumentationUrlTool,
    RunApiAuditTool,
)

tools = [ScanGithubRepoTool(), ScanDocumentationUrlTool(), RunApiAuditTool()]
llm = ChatOpenAI(model="gpt-4o", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an API quality agent. Use DocuTect AI tools to scan repos and audit endpoints."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = executor.invoke({
    "input": "Scan https://github.com/myorg/myapi and audit the most important endpoint."
})
print(result["output"])
```

### LangGraph workflow example

```python
import asyncio
from langgraph.graph import StateGraph, END
from typing import TypedDict, Optional
from scripts.agent_skills.langchain_tools import ScanGithubRepoTool, RunApiAuditTool
import json

class State(TypedDict):
    repo_url: str
    suggestions: Optional[list]
    audit_id: Optional[str]

scan_tool = ScanGithubRepoTool()
audit_tool = RunApiAuditTool()

async def scan_node(state: State) -> State:
    raw = await scan_tool.arun({"repo_url": state["repo_url"]})
    data = json.loads(raw)
    state["suggestions"] = data.get("suggestions", [])[:3]  # Top 3
    return state

async def audit_node(state: State) -> State:
    top = state["suggestions"][0] if state["suggestions"] else None
    if not top:
        return state
    raw = await audit_tool.arun({
        "api_endpoint": top["api_endpoint"],
        "api_method": top["api_method"],
        "description": top["description"],
    })
    data = json.loads(raw)
    state["audit_id"] = data.get("audit_id")
    return state

graph = StateGraph(State)
graph.add_node("scan", scan_node)
graph.add_node("audit", audit_node)
graph.set_entry_point("scan")
graph.add_edge("scan", "audit")
graph.add_edge("audit", END)

app = graph.compile()

asyncio.run(app.comnvoke({"repo_url": "https://github.com/myorg/myapi"}))
```

---

## Security Notes

- **API keys** are loaded exclusively from the `DOCUTECT_API_KEY` environment variable.
  They are never passed as constructor arguments or logged.
- **GitHub tokens** passed to `scan_github_repo` are forwarded to the server over TLS
  and are never stored or logged. They are masked in any exception messages.
- All tool output from repository/URL scans carries `content_from_untrusted_source: true`.
  Validate suggestions before acting on them in automated pipelines.
- DocuTect AI enforces **HTTPS only** on the MCP endpoint in production.
  Connections over plain HTTP are rejected.

---

## Error Reference

| HTTP Status | Meaning |
|-------------|---------|
| `401 Unauthorized` | Invalid or missing `DOCUTECT_API_KEY` |
| `403 Forbidden` | Key revoked, scope mismatch, or tier not Enterprise |
| `402 Payment Required` | MCP scan or audit quota exhausted for the month |
| `429 Too Many Requests` | Rate limit: 60 calls / 60 seconds per key |
| `404 Not Found` | Audit/document ID not found (or belongs to another org) |
| `500 Internal Server Error` | Server error — retry with exponential back-off |

---

## Example prompts for Claude / Copilot

```
Scan https://github.com/stripe/stripe-python and audit the top 3 most important endpoints.

Run a DocuTect AI audit on POST /api/v1/payments with these sample request/response payloads:
  request: {"amount": 1000, "currency": "usd"}
  response: {"id": "pi_xxx", "status": "succeeded"}

Find all hallucinations in the audit I just ran and give me the remediation suggestions.

Generate OpenAPI 3.1 documentation for https://github.com/myorg/myapi.
```
