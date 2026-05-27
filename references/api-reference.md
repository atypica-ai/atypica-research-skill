# atypica Research API Reference

Complete reference for all 9 MCP tools provided by atypica.ai research framework.

## Table of Contents

- [Setup](#setup)
- [Tool Schemas](#tool-schemas)
  - [atypica_study_create](#atypica_study_create)
  - [atypica_study_send_message](#atypica_study_send_message)
  - [atypica_study_get_messages](#atypica_study_get_messages)
  - [atypica_study_list](#atypica_study_list)
  - [atypica_study_get_report](#atypica_study_get_report)
  - [atypica_study_get_podcast](#atypica_study_get_podcast)
  - [atypica_panel_search](#atypica_panel_search)
  - [atypica_persona_search](#atypica_persona_search)
  - [atypica_persona_get](#atypica_persona_get)
- [Complete Workflow Examples](#complete-workflow-examples)
- [Error Handling](#error-handling)
- [Security & Limitations](#security--limitations)

## Setup

**MCP Server URL**: `https://atypica.ai/mcp/study`

**Authentication**: `Authorization: Bearer <api_key>`

Get API key from atypica.ai account settings (format: `atypica_xxx`)

## Tool Schemas

### atypica_study_create

Create a new research session.

**Input Schema**:
```typescript
{
  content: string  // Initial user message to start the study
}
```

**Output Schema**:
```typescript
{
  content: [{ type: "text", text: string }],
  structuredContent: {
    token: string,      // userChatToken for subsequent operations
    studyId: number,
    status: "created"
  }
}
```

**Example**:
```json
// Input
{ "content": "Research young people's coffee preferences" }

// Output
{
  "content": [{ "type": "text", "text": "Study created successfully. Token: abc123" }],
  "structuredContent": {
    "token": "abc123",
    "studyId": 123,
    "status": "created"
  }
}
```

---

### atypica_study_send_message

Send or continue a study turn. The message is persisted first, then the study agent starts or resumes in background. Use `atypica_study_get_messages` to monitor progress.

**IMPORTANT**: Two distinct input types based on use case.

**Interaction tool result formats** (for submitting pending `requestInteraction` or `makeStudyPlan` calls):

| Pending tool | Mode | `output` shape |
|---|---|---|
| `tool-requestInteraction` | single-question (`input.question` present) | `{ answer: string[], plainText: string }` |
| `tool-requestInteraction` | multi-question (`input.questions` present) | `{ answers: Array<{label: string, answer: string[]}>, plainText: string }` |
| `tool-makeStudyPlan` | — | `{ confirmed: boolean, plainText: string }` |

See [Complete Workflow Examples](#complete-workflow-examples) for full code.

#### Type 1: User Text Message

**Input Schema**:
```typescript
{
  userChatToken: string,
  message: {
    role: "user",
    lastPart: {
      type: "text",
      text: string
    },
    metadata?: {
      shouldCorrectUserMessage?: boolean
    }
  },
  attachments?: Array<{
    objectUrl: string,    // S3 object URL without signature
    name: string,
    mimeType: string,
    size: number
  }>
}
```

**Example**:
```json
{
  "userChatToken": "abc123",
  "message": {
    "role": "user",
    "lastPart": {
      "type": "text",
      "text": "Start the research"
    }
  }
}
```

#### Type 2: Tool Result Submission

**Input Schema**:
```typescript
{
  userChatToken: string,
  message: {
    id: string,              // REQUIRED: original message ID
    role: "assistant",       // REQUIRED: must be "assistant"
    lastPart: {
      type: "tool-requestInteraction" | "tool-makeStudyPlan",
      toolCallId: string,    // REQUIRED: original tool call ID
      state: "output-available",
      input: object,         // Copy original AI call parameters
      output: object         // User's response (see interaction formats)
    }
  }
}
```

**Output Schema** (same for both types):
```typescript
{
  content: [{ type: "text", text: string }],
  structuredContent: {
    messageId: string,
    role: "user" | "assistant",
    status: "running" | "saved_no_ai" | "ai_failed",
    attachmentCount?: number,
    reason?: string,        // Present if status is "saved_no_ai"
    error?: string          // Present if status is "ai_failed"
  }
}
```

**Status Values**:
- `"running"` - Message saved and study execution started or resumed in background
- `"saved_no_ai"` - Message saved, quota exceeded, no AI execution
- `"ai_failed"` - AI startup failed before the background run could continue

---

### atypica_study_get_messages

Retrieve conversation history from study session.

**Input Schema**:
```typescript
{
  userChatToken: string,
  tail?: number  // Optional: Max parts to return (from tail, most recent parts across all messages)
}
```

**Output Schema**:
```typescript
{
  content: [{ type: "text", text: string }],
  structuredContent: {
    messages: Array<{
      messageId: string,
      role: "user" | "assistant",
      parts: Array<
        | { type: "text", text: string }
        | { type: "reasoning", text: string }
        | {
            type: "tool-requestInteraction" | "tool-makeStudyPlan" | string,
            state: "input-available" | "output-available" | "output-error",
            toolCallId: string,
            input: object,
            output?: object,
            errorText?: string
          }
      >,
      createdAt: string  // ISO 8601
    }>
  }
}
```

**CRITICAL**: Scan `parts` array for tool calls with `state === "input-available"` - these require user interaction.

**Note on `tail` parameter**: When specified, returns only the last N parts across all messages. For example, if `tail: 5`, you might get 2 messages where the first has partial parts and the second has all parts, totaling 5 parts. This is useful for getting recent context without fetching the entire conversation.

**Example**:
```json
{
  "userChatToken": "abc123",
  "tail": 5  // Get last 3-5 parts (increase if more context needed)
}
```

---

### atypica_study_list

List user's historical study sessions.

**Input Schema**:
```typescript
{
  page?: number,      // Default 1
  pageSize?: number   // Default 20
}
```

**Output Schema**:
```typescript
{
  content: [{ type: "text", text: string }],
  structuredContent: {
    data: Array<{
      studyId: number,
      token: string,
      title: string,
      topic: string,
      hasReport: boolean,
      hasPodcast: boolean,
      replayUrl: string,  // Full URL to replay the study: https://atypica.ai/study/{token}/share?replay=1
      createdAt: string,  // ISO 8601
      updatedAt: string
    }>,
    pagination: {
      page: number,
      pageSize: number,
      totalCount: number,
      totalPages: number
    }
  }
}
```

---

### atypica_study_get_report

Retrieve generated research report.

**Input Schema**:
```typescript
{
  token: string  // Report token from study status artifacts
}
```

**Output Schema**:
```typescript
{
  content: [{ type: "text", text: string }],
  structuredContent: {
    token: string,
    instruction: string,
    title: string,
    description: string,
    content: string,        // HTML format
    coverUrl?: string,      // Signed CDN URL (1 hour expiry)
    shareUrl: string,       // Full shareable URL: https://atypica.ai/artifacts/report/{token}/share
    generatedAt: string,    // ISO 8601
    createdAt: string,
    updatedAt: string
  }
}
```

---

### atypica_study_get_podcast

Retrieve generated podcast content.

**Input Schema**:
```typescript
{
  token: string  // Podcast token from study status artifacts
}
```

**Output Schema**:
```typescript
{
  content: [{ type: "text", text: string }],
  structuredContent: {
    token: string,
    instruction: string,
    script: string,
    audioUrl?: string,      // Signed CDN URL (1 hour expiry)
    coverUrl?: string,      // Signed CDN URL (1 hour expiry)
    metadata: {
      title: string,
      duration: number,     // Seconds
      size: number,         // Bytes
      mimeType: string,     // e.g., "audio/mpeg"
      showNotes: string
    },
    shareUrl: string,       // Full shareable URL: https://atypica.ai/artifacts/podcast/{token}/share
    generatedAt: string,    // ISO 8601
    createdAt: string,
    updatedAt: string
  }
}
```

---

### atypica_panel_search

Search the user's persona panels. Each panel is a curated group of AI personas that can be used as the research subject pool for a study.

**Input Schema**:
```typescript
{
  query?: string,      // Optional: filter panels by title (case-insensitive). Without a query, returns all panels.
  page?: number,       // Default 1
  pageSize?: number    // Default 20, max 50
}
```

**Output Schema**:
```typescript
{
  content: [{ type: "text", text: string }],
  structuredContent: {
    data: Array<{
      panelId: number,
      title: string,
      personaCount: number,
      createdAt: string,  // ISO 8601
      updatedAt: string
    }>,
    pagination: {
      page: number,
      pageSize: number,
      totalCount: number,
      totalPages: number
    }
  }
}
```

**Example**:
```json
// List all panels
{}

// Search by title
{ "query": "coffee" }
```

---

### atypica_persona_search

Search AI personas using semantic embedding similarity.

**Input Schema**:
```typescript
{
  query?: string,   // Optional: semantic search query
  privateOnly?: boolean,  // Optional: true = only your own private personas
  limit?: number    // Default 10, max 50
}
```

**Search Logic**:
- With `query`: Uses indexed text search over visible personas
- Without `query`: Returns the latest visible personas
- `privateOnly: true`: Restricts results to the caller's own private personas

**Output Schema**:
```typescript
{
  content: [{ type: "text", text: string }],
  structuredContent: {
    data: Array<{
      personaId: number,
      token: string,
      name: string,
      source: string,
      tier: number,
      tags: string[],
      createdAt: string  // ISO 8601
    }>
  }
}
```

**Example**:
```json
// Semantic search
{
  "query": "young tech enthusiasts",  // Matches "programmers", "geeks", etc.
  "privateOnly": true,
  "limit": 10
}
```

---

### atypica_persona_get

Get detailed persona information including full prompt.

**Input Schema**:
```typescript
{
  personaId: number
}
```

**Output Schema**:
```typescript
{
  content: [{ type: "text", text: string }],
  structuredContent: {
    personaId: number,
    token: string,
    name: string,
    source: string,
    prompt: string,       // Full persona description
    tier: number,
    tags: string[],
    locale: string,
    createdAt: string,    // ISO 8601
    updatedAt: string
  }
}
```

---

## Complete Workflow Examples

### Example 1: Basic Research Flow

```typescript
// 1. Create study
const createResult = await mcp.callTool("atypica_study_create", {
  content: "Research young people's coffee preferences"
});
const userChatToken = createResult.structuredContent.token;

// 2. Send initial message (AI enters Plan Mode)
await mcp.callTool("atypica_study_send_message", {
  userChatToken,
  message: {
    role: "user",
    lastPart: { type: "text", text: "Start research" }
  }
});

// 3. Poll for interactions and completion
while (true) {
  const result = await mcp.callTool("atypica_study_get_messages", {
    userChatToken
  });

  // Check if AI is still executing
  if (result.structuredContent.isRunning) {
    await sleep(5000);  // Wait 5 seconds
    continue;
  }

  // Check for pending interactions
  const lastMsg = result.structuredContent.messages[result.structuredContent.messages.length - 1];
  const pendingTool = lastMsg?.parts?.find(
    p => p.type.startsWith("tool-") && p.state === "input-available"
  );

  if (pendingTool) {
    // Handle interaction (see Example 2 below)
    await handleInteraction(userChatToken, lastMsg.messageId, pendingTool);
    continue;
  }

  // 4. Check if report is ready
  const reportTool = result.structuredContent.messages
    .flatMap(m => m.parts)
    .find(p => p.type === "tool-generateReport" && p.state === "output-available");

  if (reportTool?.output?.reportToken) {
    // 5. Get report
    const report = await mcp.callTool("atypica_study_get_report", {
      token: reportTool.output.reportToken
    });
    console.log(report.structuredContent.content);  // HTML report
    break;
  }

  break;  // No more work to do
}
```

### Example 2: Handling requestInteraction

Two modes — check for `input.questions` to distinguish them.

```typescript
async function handleRequestInteraction(userChatToken, messageId, toolPart) {
  const input = toolPart.input;

  if (input.questions) {
    // ── Multi-question mode ──
    // Render each question as a separate tab; collect one answer array per question
    const answers = [];
    for (const q of input.questions) {
      console.log(`[${q.label}] ${q.question}`);
      q.options.forEach((opt, i) => console.log(`  ${i + 1}. ${opt}`));

      const selected = q.maxSelect === 1
        ? [await getUserSingleChoice(q.options)]
        : await getUserMultiChoice(q.options, q.maxSelect);

      answers.push({ label: q.label, answer: selected });
    }

    const plainText = answers
      .map(a => `${a.label}: ${a.answer.join(", ")}`)
      .join("\n");

    await mcp.callTool("atypica_study_send_message", {
      userChatToken,
      message: {
        id: messageId,
        role: "assistant",
        lastPart: {
          type: "tool-requestInteraction",
          toolCallId: toolPart.toolCallId,
          state: "output-available",
          input: toolPart.input,
          output: { answers, plainText }
        }
      }
    });

  } else {
    // ── Single-question mode ──
    console.log(input.question);
    input.options.forEach((opt, i) => console.log(`${i + 1}. ${opt}`));

    const selected = input.maxSelect === 1
      ? [await getUserSingleChoice(input.options)]
      : await getUserMultiChoice(input.options, input.maxSelect);

    // If user skips: send answer: [], plainText: "None of the above"
    await mcp.callTool("atypica_study_send_message", {
      userChatToken,
      message: {
        id: messageId,
        role: "assistant",
        lastPart: {
          type: "tool-requestInteraction",
          toolCallId: toolPart.toolCallId,
          state: "output-available",
          input: toolPart.input,
          output: {
            answer: selected,          // string[] always, even for single-choice
            plainText: `User selected: ${selected.join(", ")}`
          }
        }
      }
    });
  }
}
```

### Example 3: Handling makeStudyPlan

```typescript
async function handleMakeStudyPlan(userChatToken, messageId, toolPart) {
  // Display plan to user
  console.log("=== Research Plan ===");
  console.log(toolPart.input.planContent);  // Markdown format
  console.log("\n1. Confirm");
  console.log("2. Cancel");

  const confirmed = await getUserConfirmation();  // Returns boolean

  // Submit decision
  await mcp.callTool("atypica_study_send_message", {
    userChatToken,
    message: {
      id: messageId,
      role: "assistant",
      lastPart: {
        type: "tool-makeStudyPlan",
        toolCallId: toolPart.toolCallId,
        state: "output-available",
        input: toolPart.input,
        output: {
          confirmed,  // boolean: true for confirm, false for cancel
          plainText: `User ${confirmed ? "confirmed" : "cancelled"} research plan`
        }
      }
    }
  });
}
```

### Example 4: Semantic Persona Search

```typescript
// Find personas matching semantic query
const personas = await mcp.callTool("atypica_persona_search", {
  query: "young coffee enthusiasts",  // Semantic similarity
  limit: 10
});

// Get full details for first persona
const personaDetails = await mcp.callTool("atypica_persona_get", {
  personaId: personas.structuredContent.data[0].personaId
});

console.log(personaDetails.structuredContent.prompt);  // Full persona description
```

---

## Error Handling

### JSON-RPC Error Codes

```typescript
{
  jsonrpc: "2.0",
  error: {
    code: number,
    message: string
  },
  id: null
}
```

**Error Code Reference**:
- `-32001` - Unauthorized (invalid API key)
- `-32602` - Invalid params (check input schema)
- `-32603` - Internal error (server issue, check logs)
- `-32000` - Business error (e.g., study not found, unauthorized access)

### Common Error Scenarios

**Unauthorized**:
```json
{ "error": { "code": -32001, "message": "Invalid API key" } }
```
- Verify API key format: `atypica_xxx`
- Check key not expired in account settings
- Ensure Authorization header set correctly

**Study Not Found**:
```json
{ "error": { "code": -32000, "message": "Study not found" } }
```
- Verify `userChatToken` is correct
- Study may have been deleted
- Check using correct user account

**Quota Exceeded**:
```json
// Not an error - check sendMessage response status
{
  "structuredContent": {
    "status": "saved_no_ai",
    "reason": "quota_exceeded"
  }
}
```
- Message saved but AI didn't execute
- User needs to upgrade plan

**AI Execution Failed**:
```json
{
  "structuredContent": {
    "status": "ai_failed",
    "error": "Error message here"
  }
}
```
- Message already saved in database
- Can retry by sending another message

### Timeout Handling

If your MCP client times out or disconnects while calling `sendMessage`:

```typescript
try {
  await mcp.callTool("atypica_study_send_message", {
    userChatToken,
    message: { ... }
  });
} catch (timeoutError) {
  // Check if AI is still working
  const result = await mcp.callTool("atypica_study_get_messages", {
    userChatToken
  });

  if (result.structuredContent.isRunning) {
    console.log("AI still working, continue polling...");
    // Keep polling getMessages until isRunning becomes false
  }
}
```

---

## Security & Limitations

### Authentication & Authorization

- **API Key Scope**: User-scoped only (not team-scoped)
- **Ownership Verification**: All operations verify resource ownership
- **Token Isolation**: Tokens are globally unique but require ownership check
- **CDN Signatures**: File URLs are signed and expire after 1 hour

### Current Limitations

**Phase 5 Features (Not Yet Available)**:
- File attachments in `create` and `sendMessage`
- Referencing historical studies
- Custom team prompts
- Real-time MCP notifications

**MCP-Specific Limitations**:
- Team API keys not supported (returns 403 error)
- Maximum message history: 100 messages per request
- Persona search can return public personas plus the caller's own private personas

### Rate Limits & Quotas

- API calls follow standard rate limits
- Token consumption tracked per user
- `sendMessage` returns after the run is accepted; execution continues in background and often finishes within 10-120s
- Concurrent study sessions: No hard limit

### Data Privacy

- User data isolated per account
- Studies and personas not shared between users
- AI responses saved to user's account only
- CDN URLs time-limited to prevent unauthorized access

---

## Performance Optimization

### Best Practices

1. **Parallel Operations**: Run independent calls concurrently
   ```typescript
   const [report, podcast] = await Promise.all([
     mcp.callTool("atypica_study_get_report", { token: reportToken }),
     mcp.callTool("atypica_study_get_podcast", { token: podcastToken })
   ]);
   ```

2. **Use Tail Parameter for Recent Context**: Use `tail` to limit parts returned
   ```typescript
   // Get last 5 parts for quick context
   const recentMessages = await mcp.callTool("atypica_study_get_messages", {
     userChatToken,
     tail: 5  // Start with 3-5, increase if more context needed
   });
   ```

3. **Poll with getMessages**: Check `isRunning` to know when AI completes
   ```typescript
   async function waitForCompletion(userChatToken) {
     while (true) {
       const result = await mcp.callTool("atypica_study_get_messages", {
         userChatToken
       });

       if (!result.structuredContent.isRunning) {
         return result;  // AI finished
       }

       await sleep(5000);  // Poll every 5 seconds
     }
   }
   ```

4. **Cache Personas**: Store persona search results locally
   ```typescript
   const cachedPersonas = {};
   async function getPersona(personaId) {
     if (!cachedPersonas[personaId]) {
       cachedPersonas[personaId] = await mcp.callTool("atypica_persona_get", { personaId });
     }
     return cachedPersonas[personaId];
   }
   ```
