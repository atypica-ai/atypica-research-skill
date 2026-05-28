# atypica Research Skill

AI-powered business research capabilities through MCP protocol. Enables multi-agent research workflows for consumer insights, product R&D, market analysis, and automated report generation.

## Features

- 🤖 Multi-agent research (Plan Mode, Study, Fast Insight, Product R&D)
- 🗣️ Interview simulation with AI personas
- 📊 Automated report and podcast generation
- 🔍 Semantic persona search
- 💬 Interactive research workflows

## Prerequisites

### Required

- **atypica.ai account** - Sign up at https://atypica.ai
- **API Key** - Get from https://atypica.ai/account/api-keys (format: `atypica_xxx`)
- **MCP-compatible AI assistant** OR **bash/curl** for direct API access

### Usage & Billing

- Research operations consume atypica.ai token credits
- Token usage tracked per user account
- Check usage at https://atypica.ai/account/tokens

## Installation

### Option 1: MCP Integration

Requires MCP-compatible AI assistant (e.g., Claude Desktop, Cline).

**Configuration**:
```json
{
  "mcpServers": {
    "atypica-research": {
      "transport": "http",
      "url": "https://atypica.ai/mcp/study",
      "headers": {
        "Authorization": "Bearer atypica_xxx"
      }
    }
  }
}
```

See [SKILL.md](SKILL.md) for detailed setup instructions.

### Option 2: Bash Script (No MCP required)

```bash
export ATYPICA_TOKEN="atypica_xxx"
scripts/mcp-call.sh atypica_study_create '{"content":"Your research query"}'
```

## Documentation

- **[SKILL.md](SKILL.md)** - Detailed usage guide for AI assistants
- **[API Reference](references/api-reference.md)** - Complete API documentation
- **[scripts/mcp-call.sh](scripts/mcp-call.sh)** - Bash fallback tool

## Available Tools

**Research**: `atypica_study_create`, `atypica_study_send_message`, `atypica_study_get_messages`, `atypica_study_list`

**Artifacts**: `atypica_study_get_report`, `atypica_study_get_podcast`

**Panels & Personas**: `atypica_panel_search`, `atypica_persona_search`, `atypica_persona_get`

**File Upload**: `atypica_get_upload_credentials`

## Quick Links

- Website: https://atypica.ai
- Documentation: https://atypica.ai/docs/mcp
- Issues: https://github.com/bmrlab/atypica-research-skill/issues

---

**Auto-synced from [atypica-llm-app](https://github.com/bmrlab/atypica-llm-app)**
