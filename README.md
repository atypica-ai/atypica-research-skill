# atypica Research Skill

User research with atypica.ai through MCP: AI-simulated consumers are interviewed and discuss your question, and a report comes out with traceable sources.

## Prerequisites

- An atypica.ai account: https://atypica.ai
- An API key: https://atypica.ai/account/api-keys (format `atypica_xxx`)
- An MCP client, or bash with curl and jq

Research uses the account's credits; see https://atypica.ai/account/tokens.

## Installation

### MCP client

```json
{
  "mcpServers": {
    "atypica-research": {
      "transport": "http",
      "url": "https://atypica.ai/mcp/study",
      "headers": { "Authorization": "Bearer atypica_xxx" }
    }
  }
}
```

### Bash

```bash
export ATYPICA_TOKEN="atypica_xxx"
scripts/mcp-call.sh atypica_study_create '{"content":"Your research question"}'
```

## Documentation

- [SKILL.md](SKILL.md): how an AI assistant runs a study with the user
- [references/api-reference.md](references/api-reference.md): every tool's input and output
- [scripts/mcp-call.sh](scripts/mcp-call.sh): call a tool without an MCP client

## Tools

- **Studies**: `atypica_study_create`, `atypica_study_send_text_message`, `atypica_study_submit_tool_result`, `atypica_study_get_messages`, `atypica_study_get_step`, `atypica_study_list`
- **Artifacts**: `atypica_study_get_report`, `atypica_study_get_podcast`
- **Panels and personas**: `atypica_panel_search`, `atypica_persona_search`, `atypica_persona_get`
- **Files**: `atypica_get_upload_credentials`

## Links

- Website: https://atypica.ai
- Issues: https://github.com/atypica-ai/atypica-research-skill/issues

---

**Auto-synced from [atypica-llm-app](https://github.com/bmrlab/atypica-llm-app)**
