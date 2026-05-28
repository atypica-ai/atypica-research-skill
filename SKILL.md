---
name: atypica-research
description: Access atypica.ai's AI-powered business research capabilities through MCP protocol. Use when users need to conduct consumer research, market analysis, product testing, or generate insights about business questions. Supports creating research sessions, sending messages with AI execution, handling user interactions (question prompts and plan confirmations), retrieving conversation history, checking research status, accessing generated reports/podcasts, and searching AI personas for interviews.
---

# atypica Research

Access atypica.ai's multi-agent research framework for understanding consumer emotions, market perceptions, and decision preferences.

## Prerequisites

**IMPORTANT**: This skill provides two ways to access atypica.ai research capabilities:

### Option 1: MCP Server (Recommended for AI assistants)

If tools starting with `atypica_` are already available, the MCP server is configured. Otherwise, guide the user to configure it.

**Configuration parameters**:
- **Endpoint**: `https://atypica.ai/mcp/study`
- **API Key**: From https://atypica.ai/account/api-keys (format: `atypica_xxx`)
- **Authentication**: HTTP header `Authorization: Bearer <api_key>`

**Example: Claude Desktop** - Edit config file at:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

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

Restart Claude Desktop to load. For other MCP clients, configuration syntax may differ.

### Option 2: Direct Bash Script (Works anywhere)

If MCP server is not available or for simpler use cases, use the bundled bash script:

```bash
scripts/mcp-call.sh <tool_name> <json_args> [options]
```

**Setup**:
```bash
export ATYPICA_TOKEN="atypica_xxx"
```

**Examples**:
```bash
# Create research session
scripts/mcp-call.sh atypica_study_create '{"content":"Research coffee preferences"}'

# Get messages with tail parameter (3-5 parts, increase if more context needed)
scripts/mcp-call.sh atypica_study_get_messages '{"userChatToken":"abc123","tail":5}'
```

**Options**:
- `-t, --token` - API token (overrides ATYPICA_TOKEN)
- `-o, --output` - Output format: text|json|structured|auto
- `-f, --file` - Write output to file instead of stdout
- `-v, --verbose` - Enable verbose output
- `-h, --help` - Show help message

See [scripts/mcp-call.sh](scripts/mcp-call.sh) for full documentation.

## Quick Start

Once the MCP server is installed:

```typescript
// 1. Create research session
const result = await callTool("atypica_study_create", {
  content: "Research young people's coffee preferences"
});
const userChatToken = result.structuredContent.token;

// 2. Send message (starts the AI run; completion usually takes 10-120s in background)
await callTool("atypica_study_send_message", {
  userChatToken,
  message: {
    role: "user",
    lastPart: { type: "text", text: "Start research" }
  }
});

// 3. Poll for research progress
let result;
let pollInterval = 30000; // 30 seconds before plan confirmation
let tailSize = 3; // Start with 3-5 parts, increase if needed
do {
  await wait(pollInterval);
  result = await callTool("atypica_study_get_messages", {
    userChatToken,
    tail: tailSize  // Get last few parts for efficiency
  });

  // After plan confirmation, use longer interval
  const hasPlanConfirmed = result.structuredContent.messages.some(m =>
    m.parts.some(p => p.type === "tool-makeStudyPlan" && p.state === "output-available")
  );
  if (hasPlanConfirmed) {
    pollInterval = 300000; // 5 minutes after plan confirmation
  }

  // If needed more context, increase tail size (up to 10-15)
  // or remove tail parameter to get all messages
} while (result.structuredContent.isRunning);

// 4. Handle pending interactions or get final report
const { messages } = result.structuredContent;
const reportTool = messages
  .flatMap(m => m.parts)
  .find(p => p.type === "tool-generateReport" && p.output);

if (reportTool?.output?.reportToken) {
  const report = await callTool("atypica_study_get_report", {
    token: reportTool.output.reportToken
  });
  // Access report details
  console.log(report.structuredContent.title);
  console.log(report.structuredContent.shareUrl);  // Public share URL
  console.log(report.structuredContent.content);   // HTML content
} else {
  // If stopped without report, you can continue:
  await callTool("atypica_study_send_message", {
    userChatToken,
    message: {
      role: "user",
      lastPart: { type: "text", text: "Please continue the research" }
    }
  });
  // Then poll again...
}
```

## Core Workflow

1. **Create** research session with initial query
2. **Send** messages to drive research forward (AI executes synchronously)
3. **Poll** for pending interactions that require user input
4. **Handle** interactions by submitting tool results
5. **Monitor** progress and retrieve artifacts (reports/podcasts)

## Available Tools

### Session Management

**atypica_study_create** - Create research session
- Input:
  ```typescript
  {
    content: string;          // Initial user message to start the study
    panelId?: number;         // Optional panel ID to use as persona source (from atypica_panel_search)
    attachments?: Array<{     // Optional file attachments (upload first via atypica_get_upload_credentials)
      objectUrl: string;      // S3 object URL from upload credentials
      name: string;           // File name
      mimeType: string;       // MIME type
      size: number;           // File size in bytes
    }>;
  }
  ```
- Returns:
  ```typescript
  {
    token: string;          // Study session token for subsequent operations
    studyId: number;        // Internal study ID
    status: "created";      // Always "created" on success
  }
  ```

**atypica_study_send_message** - Send message and start/continue AI execution
- Two input types:
  - User text: `{ userChatToken, message: { role: "user", lastPart: { type: "text", text } }, attachments?: [...] }`
  - Tool result: See "User Interactions" section
- Attachments: Same format as `atypica_study_create` (upload first via `atypica_get_upload_credentials`)
- Returns:
  ```typescript
  {
    messageId: string;           // Message identifier
    role: "user" | "assistant";  // Message role
    status: "running" | "saved_no_ai" | "ai_failed";
    attachmentCount?: number;    // Number of attachments (if any)
    error?: string;              // Error message (if status is "ai_failed")
    reason?: string;             // Reason (if status is "saved_no_ai")
  }
  ```
- Notes:
  - `running` means the message was saved and the study agent started/resumed in background
  - Poll `atypica_study_get_messages` until `isRunning` becomes `false`

**atypica_study_get_messages** - Retrieve conversation history and execution status
- Input: `{ userChatToken: string, tail?: number }`
- Returns:
  ```typescript
  {
    isRunning: boolean;  // true = AI executing, false = can interact
    messages: Array<{
      messageId: string;
      role: "user" | "assistant";
      parts: Array<MessagePart>;  // Text, tool calls, tool results
      createdAt: string;           // ISO timestamp
    }>;
  }
  ```
- **Critical**:
  - `isRunning: true` → AI is executing, wait and poll again later
  - `isRunning: false` → Can interact, check for pending tool calls in `parts`
  - `tail` (optional): Limit to last N parts across all messages (3-5 recommended)

**atypica_study_list** - List historical research sessions
- Input:
  ```typescript
  {
    page?: number;      // Default: 1
    pageSize?: number;  // Default: 20, max: 100
  }
  ```
- Returns:
  ```typescript
  {
    data: Array<{
      studyId: number;
      token: string;           // Study session token
      title: string;           // Auto-generated title
      topic: string;           // Research topic
      hasReport: boolean;      // Has generated report
      hasPodcast: boolean;     // Has generated podcast
      replayUrl: string;       // URL to replay the study: https://atypica.ai/study/{token}/share?replay=1
      createdAt: string;       // ISO timestamp
      updatedAt: string;       // ISO timestamp
    }>;
    pagination: {
      page: number;
      pageSize: number;
      totalCount: number;
      totalPages: number;
    };
  }
  ```

### Artifacts

**atypica_study_get_report** - Get research report
- Input: `{ token: string }`
- Returns:
  ```typescript
  {
    token: string;          // Report token
    instruction: string;    // Generation instruction
    title: string;          // Report title
    description: string;    // Report description
    content: string;        // HTML content (one-page format)
    coverUrl?: string;      // Signed CDN URL for cover image
    shareUrl: string;       // Public share URL: https://atypica.ai/artifacts/report/{token}/share
    generatedAt: string;    // ISO timestamp when generated
    createdAt: string;      // ISO timestamp when created
    updatedAt: string;      // ISO timestamp when last updated
  }
  ```

**atypica_study_get_podcast** - Get podcast content
- Input: `{ token: string }`
- Returns:
  ```typescript
  {
    token: string;          // Podcast token
    instruction: string;    // Generation instruction
    script: string;         // Full podcast script/transcript
    audioUrl: string;       // Signed CDN URL for audio file
    coverUrl?: string;      // Signed CDN URL for cover image
    metadata: {             // Podcast metadata
      title: string;
      duration?: number;    // Duration in seconds
      coverObjectUrl?: string;
      // ... other metadata fields
    };
    shareUrl: string;       // Public share URL: https://atypica.ai/artifacts/podcast/{token}/share
    generatedAt: string;    // ISO timestamp when generated
    createdAt: string;      // ISO timestamp when created
    updatedAt: string;      // ISO timestamp when last updated
  }
  ```

### Panels

**atypica_panel_search** - Search persona panels
- Input:
  ```typescript
  {
    query?: string;      // Optional: filter by title (case-insensitive). Without a query, returns all panels.
    page?: number;       // Default: 1
    pageSize?: number;   // Default: 20, max: 50
  }
  ```
- Each panel is a curated group of AI personas used as the research subject pool
- Without `query`, returns all panels
- Returns:
  ```typescript
  {
    data: Array<{
      panelId: number;
      title: string;
      personaCount: number;
      createdAt: string;   // ISO timestamp
      updatedAt: string;   // ISO timestamp
    }>;
    pagination: {
      page: number;
      pageSize: number;
      totalCount: number;
      totalPages: number;
    };
  }
  ```

### Personas

**atypica_persona_search** - Semantic search for AI personas
- Input:
  ```typescript
  {
    query?: string;        // Text query for name/source matching
    privateOnly?: boolean; // true = only your own private personas
    limit?: number;    // Max results (default: 10, max: 50)
  }
  ```
- With `query`, uses indexed text search
- Without `query`, returns the latest personas visible to you (public + your private, unless `privateOnly` is true)
- Returns:
  ```typescript
  {
    data: Array<{
      personaId: number;
      token: string;       // Persona token
      name: string;        // Persona name
      source: string;      // Persona source/origin
      tier: number;        // Access tier (0-3)
      tags: string[];      // Associated tags
      createdAt: string;   // ISO timestamp
    }>;
  }
  ```

**atypica_persona_get** - Get persona details
- Input: `{ personaId: number }`
- Returns:
  ```typescript
  {
    personaId: number;
    token: string;       // Persona token
    name: string;        // Persona name
    source: string;      // Persona source/origin
    prompt: string;      // Full persona prompt (system prompt for AI)
    tier: number;        // Access tier (0-3)
    tags: string[];      // Associated tags
    locale: string;      // Persona language locale
    createdAt: string;   // ISO timestamp
    updatedAt: string;   // ISO timestamp
  }
  ```

### File Upload

**atypica_get_upload_credentials** - Get presigned URL to upload a file
- Input:
  ```typescript
  {
    fileName: string;   // File name with extension (e.g. "design.png", "report.pdf")
    mimeType: string;   // MIME type (e.g. "image/png", "application/pdf", "text/csv")
  }
  ```
- Supported file types: images (jpeg, png, gif, webp, bmp, svg) and documents (pdf, json, csv)
- Returns:
  ```typescript
  {
    putUrl: string;     // Presigned PUT URL (5 min expiry) - upload file here
    objectUrl: string;  // S3 object URL - pass this to create/sendMessage attachments
    fileName: string;   // Original file name
    mimeType: string;   // MIME type
  }
  ```

**Upload workflow**:
```typescript
// 1. Get upload credentials
const creds = await callTool("atypica_get_upload_credentials", {
  fileName: "survey-data.pdf",
  mimeType: "application/pdf"
});
const { putUrl, objectUrl } = creds.structuredContent;

// 2. Upload file via HTTP PUT
// curl -X PUT -H "Content-Type: application/pdf" --data-binary @survey-data.pdf "<putUrl>"

// 3. Use objectUrl in study creation or message
await callTool("atypica_study_create", {
  content: "Analyze this survey data",
  attachments: [{
    objectUrl,
    name: "survey-data.pdf",
    mimeType: "application/pdf",
    size: 102400  // file size in bytes
  }]
});
```

**Limits**: Max 5 images, max 3 documents, max 3MB per file, max 50MB total per message.

## Understanding Research State from Messages

**All research state is in the messages** - you don't need a separate status API. After calling `getMessages`, follow this pattern:

### 1. Check if AI is executing

```typescript
const { isRunning, messages } = result.structuredContent;

if (isRunning) {
  // AI is working in background, cannot interact now
  // Polling strategy:
  // - Before plan confirmation: 30 seconds
  // - After plan confirmation: 5 minutes
  return "Research is running, please wait...";
}
```

### 2. Check for pending interactions

Scan the last assistant message for tool calls needing user input:

```typescript
const lastMsg = messages[messages.length - 1];
if (lastMsg.role === "assistant") {
  for (const part of lastMsg.parts) {
    if (part.type.startsWith("tool-") && part.state === "input-available") {
      // Handle this pending tool call (see User Interactions section)
    }
  }
}
```

### 3. Understand research progress

Look at recent tool calls to see what's happening:

| Tool Call | Meaning |
|-----------|---------|
| `makeStudyPlan` | In Plan Mode, AI is clarifying intent |
| `interviewChat`, `discussionChat` | Conducting interviews/discussions |
| `webSearch`, `webFetch` | Gathering information |
| `reasoningThinking` | Deep analysis in progress |
| `generateReport` | Generating final report |
| `generatePodcast` | Generating podcast |

### 4. Check if research is complete

Look for `generateReport` or `generatePodcast` tool call with output:

```typescript
const reportTool = messages
  .flatMap(m => m.parts)
  .find(p => p.type === "tool-generateReport" && p.state === "output-available");

if (reportTool?.output?.reportToken) {
  // Research complete! Get the report
  const report = await callTool("atypica_study_get_report", {
    token: reportTool.output.reportToken
  });
  console.log(report.structuredContent.title);
  console.log(report.structuredContent.shareUrl);  // https://atypica.ai/artifacts/report/{token}/share
}

// Similarly for podcasts
const podcastTool = messages
  .flatMap(m => m.parts)
  .find(p => p.type === "tool-generatePodcast" && p.state === "output-available");

if (podcastTool?.output?.podcastToken) {
  const podcast = await callTool("atypica_study_get_podcast", {
    token: podcastTool.output.podcastToken
  });
  console.log(podcast.structuredContent.audioUrl);
  console.log(podcast.structuredContent.shareUrl);  // https://atypica.ai/artifacts/podcast/{token}/share
}
```

### 5. Continue a stopped research

**When to use**: If research stops (`isRunning: false`) but no report/podcast was generated, you can continue it.

**How to continue**:
```typescript
// Check if research stopped without completing
const { isRunning, messages } = result.structuredContent;
const hasReport = messages.some(m =>
  m.parts.some(p =>
    (p.type === "tool-generateReport" || p.type === "tool-generatePodcast") &&
    p.state === "output-available"
  )
);

if (!isRunning && !hasReport) {
  // Research stopped but not complete - you can continue it
  await callTool("atypica_study_send_message", {
    userChatToken,
    message: {
      role: "user",
      lastPart: {
        type: "text",
        text: "Please continue the research"  // or "[CONTINUE ASSISTANT STEPS]"
      }
    }
  });

  // Then poll again to check progress
}
```

**Important**:
- The AI will resume from where it stopped, not restart from scratch
- It may retry the last interrupted tool or continue with the next step
- After sending continue message, poll `getMessages` again to track progress

## User Interactions

Two tools require user interaction. Check `getMessages` response for pending calls with `state === "input-available"`.

### requestInteraction - User Choice/Input

Two modes: **single-question** (one question with options) and **multi-question** (2+ related questions with tab navigation). Detect which mode by checking whether `input.questions` array is present.

#### Mode 1: Single Question

**Detect**:
```json
{
  "type": "tool-requestInteraction",
  "state": "input-available",
  "toolCallId": "call_abc",
  "input": {
    "question": "Which age group to focus on?",
    "options": ["18-22", "23-28", "29-35"],
    "maxSelect": 1
  }
}
```

`maxSelect` controls selection behavior: `1` = single choice (mutually exclusive), `2+` = multi-choice with limit, `undefined`/omitted = unlimited multi-choice.

**Submit** via `sendMessage`:
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

`answer` is always `string[]` (array), even for single-choice (`maxSelect: 1`). If the user skips all options, send `answer: [], plainText: "None of the above"`.

#### Mode 2: Multi-Question

Used when the AI collects multiple related dimensions at once (e.g., demographic traits: gender, age, region). Detected by the presence of `input.questions` array — the top-level `question`/`options`/`maxSelect` fields are absent.

**Detect**:
```json
{
  "type": "tool-requestInteraction",
  "state": "input-available",
  "toolCallId": "call_abc",
  "input": {
    "questions": [
      {
        "label": "Gender",
        "question": "What is your gender?",
        "options": ["Male", "Female", "Non-binary", "Prefer not to say"],
        "maxSelect": 1
      },
      {
        "label": "Age",
        "question": "Which age group do you belong to?",
        "options": ["18-24", "25-34", "35-44", "45+"],
        "maxSelect": 1
      },
      {
        "label": "Location",
        "question": "Which region are you in?",
        "options": ["Tier-1 cities", "Tier-2/3 cities", "Rural areas"],
        "maxSelect": 1
      }
    ]
  }
}
```

Each item in `questions` has a `label` (short tab name, 1-2 words), `question`, `options`, and optional `maxSelect`.

**Submit** via `sendMessage`:
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
      "input": { "questions": [...] },
      "output": {
        "answers": [
          { "label": "Gender", "answer": ["Female"] },
          { "label": "Age", "answer": ["25-34"] },
          { "label": "Location", "answer": ["Tier-1 cities"] }
        ],
        "plainText": "Gender: Female\nAge: 25-34\nLocation: Tier-1 cities"
      }
    }
  }
}
```

Use `answers` (array of `{label, answer: string[]}`) instead of `answer`. The `plainText` should summarize each label/answer pair on a new line.

### makeStudyPlan - Plan Confirmation

**Detect**:
```json
{
  "type": "tool-makeStudyPlan",
  "state": "input-available",
  "toolCallId": "call_xyz",
  "input": {
    "planContent": "# Research Plan\n\n## Goals\n...\n## Methods\n..."
  }
}
```

**Submit** via `sendMessage`:
```json
{
  "userChatToken": "...",
  "message": {
    "id": "msg_2",
    "role": "assistant",
    "lastPart": {
      "type": "tool-makeStudyPlan",
      "toolCallId": "call_xyz",
      "state": "output-available",
      "input": { /* copy from above */ },
      "output": {
        "confirmed": true,  // boolean: true to confirm, false to cancel
        "plainText": "User confirmed research plan"
      }
    }
  }
}
```

## Best Practices

1. **Async execution**: `sendMessage` starts or resumes the run, then returns while the AI continues in background
2. **Poll for interactions**: After each `sendMessage`, check `getMessages` for pending tool calls
3. **Handle timeouts**: If `sendMessage` times out, use `getMessages` to check progress
4. **Error handling**: Check `status` field: `"running" | "saved_no_ai" | "ai_failed"`
5. **Persona search**: Use natural language queries in `persona_search`, or `privateOnly: true` to limit results to your own personas

## Study Types

- `userResearch` - User research with personas, interviews, discussions, and reports
- `fastInsight` - Quick insights with deep research and podcast generation
- `productRnD` - Product research & development with audience calls and social trends
- `panelOnly` - Build a persona panel without running a full study

New study sessions start in **Planning** phase automatically, where AI clarifies the research intent via `requestInteraction` before creating a plan with `makeStudyPlan`.

## Performance Expectations

- Plan Mode: 5-10 seconds
- Fast Insight: 20-40 seconds
- Full Study/Product R&D: 30-120 seconds
- Message/persona retrieval: < 2 seconds

## Reference Documentation

See [references/api-reference.md](references/api-reference.md) for complete API documentation including:
- Detailed input/output schemas for all tools
- Error codes and troubleshooting
- Complete workflow examples
- Security and limitations
