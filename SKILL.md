---
name: atypica-research
description: Access atypica.ai's AI-powered business research capabilities through MCP protocol. Use when users need to conduct consumer research, market analysis, product testing, or generate insights about business questions. Supports creating research sessions, sending messages with AI execution, handling user interactions (question prompts and plan confirmations), retrieving conversation history, checking research status, accessing generated reports/podcasts, and searching AI personas for interviews.
---

# atypica Research

Access atypica.ai's multi-agent research framework for understanding consumer emotions, market perceptions, and decision preferences.

## Calling Methods

Three ways to call the Research API, depending on your environment:

### Method 1: MCP Client (Recommended for AI assistants)

If tools starting with `atypica_` are already available, the MCP server is configured. Otherwise, configure it:

- **Endpoint**: `https://atypica.ai/mcp/study`
- **API Key**: From https://atypica.ai/account/api-keys (format: `atypica_xxx`)
- **Authentication**: HTTP header `Authorization: Bearer <api_key>`

**Claude Desktop** config (`~/Library/Application Support/Claude/claude_desktop_config.json`):

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

### Method 2: curl + jq (Best for scripting without MCP client)

When you don't have an MCP client, curl + jq is the most flexible combination. All MCP tools are plain JSON-RPC over HTTP:

```bash
export ATYPICA_TOKEN="atypica_xxx"
export ATYPICA_ENDPOINT="https://atypica.ai/mcp/study"

# Generic pattern: call any tool
call_tool() {
  local tool_name="$1"
  local args="$2"
  curl -s -X POST "$ATYPICA_ENDPOINT" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $ATYPICA_TOKEN" \
    -d "{\"jsonrpc\":\"2.0\",\"method\":\"tools/call\",\"params\":{\"name\":\"$tool_name\",\"arguments\":$args},\"id\":1}"
}

# Create research session
call_tool atypica_study_create '{"content":"Research young people coffee preferences"}' | jq '.result.structuredContent'

# Get messages (tail=5 for last 5 parts)
call_tool atypica_study_get_messages '{"userChatToken":"abc123","tail":5}' \
  | jq '.result.structuredContent | {isRunning, messageCount: (.messages | length)}'

# Check for pending interactions
call_tool atypica_study_get_messages '{"userChatToken":"abc123","tail":3}' \
  | jq '.result.structuredContent.messages[-1].parts[] | select(.state == "input-available") | {type, toolCallId}'

# Get report
call_tool atypica_study_get_report '{"token":"report_xxx"}' | jq '.result.structuredContent | {title, shareUrl}'

# Search panels
call_tool atypica_panel_search '{"query":"coffee"}' | jq '.result.structuredContent.data[] | {panelId, title, personaCount}'
```

**Tips**:

- Use `jq '.result.structuredContent'` to extract the structured response
- Use `jq '.result.content[0].text'` to get the human-readable summary
- Pipe into `jq -r` for raw strings (no quotes)
- Use `tail` parameter in `get_messages` (3-5 parts) instead of fetching everything

### Method 3: mcp-call.sh (Bundled convenience script)

A standalone script that wraps curl+jq with nicer output, error handling, and options:

```bash
export ATYPICA_TOKEN="atypica_xxx"

# Create research session
scripts/mcp-call.sh atypica_study_create '{"content":"Research coffee preferences"}'

# Get messages with tail
scripts/mcp-call.sh atypica_study_get_messages '{"userChatToken":"abc123","tail":5}'

# Output as raw JSON
scripts/mcp-call.sh atypica_study_get_messages '{"userChatToken":"abc123"}' -o json

# Write to file
scripts/mcp-call.sh atypica_study_get_report '{"token":"report_xxx"}' -f report.json

# Verbose mode
scripts/mcp-call.sh atypica_study_create '{"content":"test"}' -v
```

Options: `-t <token>`, `-o text|json|structured|auto`, `-f <file>`, `-v`, `-h`

See [scripts/mcp-call.sh](scripts/mcp-call.sh) for full documentation.

## Core Workflow

1. **Create** research session with initial query
2. **Send** messages to drive research forward (AI executes asynchronously in background)
3. **Poll** `get_messages` — check `isRunning` and look for pending interactions
4. **Handle** interactions by submitting tool results (plan confirmation, user choices)
5. **Retrieve** artifacts (reports, podcasts) when research completes

```
study_create → send_message → [AI runs in background]
                                    ↓
                              get_messages (poll)
                                    ↓
                    isRunning=true? → wait, poll again
                    isRunning=false? → check for:
                      - pending interaction → submit tool result → send_message
                      - generateReport output → get_report (done!)
                      - no report, no interaction → send "continue" message
```

## Available Tools

### Session Management

| Tool                         | Description                                                               |
| ---------------------------- | ------------------------------------------------------------------------- |
| `atypica_study_create`       | Create research session with initial query and optional panel/attachments |
| `atypica_study_send_message` | Send user text or tool result submission, starts/resumes AI execution     |
| `atypica_study_get_messages` | Get conversation history + execution status (`isRunning`)                 |
| `atypica_study_list`         | List historical research sessions with pagination                         |

### Artifacts

| Tool                        | Description                                    |
| --------------------------- | ---------------------------------------------- |
| `atypica_study_get_report`  | Get research report (HTML content + share URL) |
| `atypica_study_get_podcast` | Get podcast (audio URL + script + share URL)   |

### Panels & Personas

| Tool                     | Description                                           |
| ------------------------ | ----------------------------------------------------- |
| `atypica_panel_search`   | Search persona panels (curated groups of AI personas) |
| `atypica_persona_search` | Semantic search for individual AI personas            |
| `atypica_persona_get`    | Get persona details including full prompt             |

### File Upload

| Tool                             | Description                                            |
| -------------------------------- | ------------------------------------------------------ |
| `atypica_get_upload_credentials` | Get presigned URL for file upload (images, PDFs, CSVs) |

For complete input/output schemas, see [references/api-reference.md](references/api-reference.md).

## Understanding Research State from Messages

**All research state is in the messages** — no separate status API. After `get_messages`:

### 1. Check execution status

```bash
call_tool atypica_study_get_messages '{"userChatToken":"abc","tail":5}' \
  | jq '.result.structuredContent.isRunning'
```

- `true` → AI working in background, wait and poll again
- `false` → Can interact, check for pending tool calls or completion

**Polling strategy**:

- Before plan confirmation: every 30 seconds
- After plan confirmation: every 5 minutes (research takes time)

### 2. Detect pending interactions

```bash
# Find tool calls that need input
call_tool atypica_study_get_messages '{"userChatToken":"abc","tail":3}' \
  | jq '.result.structuredContent.messages[-1].parts[] | select(.state == "input-available") | {type, toolCallId}'
```

Two interaction types require user input:

- `tool-requestInteraction` — User choice/input (single or multi-question)
- `tool-makeStudyPlan` — Plan confirmation (approve or reject)

### 3. Check research completion

```bash
# Look for report/podcast output
call_tool atypica_study_get_messages '{"userChatToken":"abc"}' \
  | jq '.result.structuredContent.messages[].parts[] | select(.type == "tool-generateReport" and .state == "output-available") | .output.reportToken'
```

### 4. Continue stopped research

If `isRunning: false` but no report was generated:

```bash
call_tool atypica_study_send_message '{
  "userChatToken":"abc",
  "message":{"role":"user","lastPart":{"type":"text","text":"Please continue the research"}}
}'
```

## User Interactions

Two tools require user input. Detect them by `state === "input-available"` in message parts.

### requestInteraction — User Choice/Input

**Two modes**:

- **Single-question**: `input.question` + `input.options` present
- **Multi-question**: `input.questions` array present (2+ related questions)

**Single-question submit**:

```json
{
  "userChatToken": "...",
  "message": {
    "id": "msg_2",
    "role": "assistant",
    "lastPart": {
      "type": "tool-requestInteraction",
      "toolCallId": "call_abc",
      "state": "output-available",
      "input": { "question": "...", "options": [...], "maxSelect": 1 },
      "output": {
        "answer": ["23-28"],
        "plainText": "User selected: 23-28"
      }
    }
  }
}
```

`answer` is always `string[]`. `maxSelect: 1` = single-choice, `2+` = multi with limit, omitted = unlimited.

**Multi-question submit**:

```json
{
  "output": {
    "answers": [
      { "label": "Gender", "answer": ["Female"] },
      { "label": "Age", "answer": ["25-34"] }
    ],
    "plainText": "Gender: Female\nAge: 25-34"
  }
}
```

### makeStudyPlan — Plan Confirmation

```json
{
  "output": {
    "confirmed": true,
    "plainText": "User confirmed research plan"
  }
}
```

Set `confirmed: false` to reject and restart planning.

## Study Types

- `userResearch` — User research with personas, interviews, discussions, reports
- `fastInsight` — Quick insights with deep research and podcast generation
- `productRnD` — Product research & development with audience and social trends
- `panelOnly` — Build a persona panel without running a full study

Sessions start in **Planning** phase automatically — AI clarifies intent via `requestInteraction` before creating a plan with `makeStudyPlan`.

## Best Practices

1. **Async execution**: `sendMessage` starts the run and returns immediately. Poll `getMessages` for results.
2. **Use `tail` parameter**: `tail: 3-5` in `get_messages` for efficiency. Increase if you need more context.
3. **Handle timeouts**: If `sendMessage` times out, check `getMessages` — the AI may still be running.
4. **Error recovery**: Check `status` field from `sendMessage`: `"running"` | `"saved_no_ai"` | `"ai_failed"`.
5. **Upload first**: Files must be uploaded via `get_upload_credentials` before attaching. Max 5 images, 3 docs, 3MB each, 50MB total.
6. **Persona panels**: Use `panel_search` to find curated groups, pass `panelId` to `study_create`.
7. **Parallelize independent reads**: `get_report` and `get_podcast` can run concurrently. Same for fetching multiple persona details.
8. **Cache persona data locally**: Persona prompts don't change often — avoid re-fetching the same personaId repeatedly.

## Performance Expectations

- Plan Mode: 5-10s
- Fast Insight: 20-40s
- Full Study/Product R&D: 30-120s
- Message/persona retrieval: < 2s

## Reference Documentation

**IMPORTANT**: Read [references/api-reference.md](references/api-reference.md) in full before making any API calls. Do not read it partially or look up individual tools on demand — the notes and constraints for each tool inform how other tools behave. Reading the complete reference upfront prevents subtle mistakes that come from missing cross-tool dependencies.
