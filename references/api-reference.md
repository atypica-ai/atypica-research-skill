# atypica Research API Reference

Complete reference for all 10 MCP tools provided by the atypica.ai research framework.

## Setup

The MCP server is available at `https://atypica.ai/mcp/study`. Authentication uses a Bearer token in the Authorization header. API keys follow the format `atypica_xxx`, obtained from the atypica.ai account settings page.

---

## atypica_study_create

Create a new research session.

Input:

```
{
  content: string,            // Initial user message to start the study
  panelId?: number,           // Panel ID as persona source (from atypica_panel_search)
  attachments?: [{            // Upload first via atypica_get_upload_credentials
    objectUrl: string,        // S3 object URL from upload credentials
    name: string,             // File name
    mimeType: string,         // MIME type
    size: number              // Bytes
  }]
}
```

Output:

```
{
  token: string,              // userChatToken for all subsequent operations
  studyId: number,
  status: "created"
}
```

Notes:

- After creation, send a follow-up message via atypica_study_send_message to trigger the AI agent.
- If attachments are provided, upload each file first using atypica_get_upload_credentials.

---

## atypica_study_send_message

Send a user message or submit a tool-interaction result. Message is persisted, then study agent starts/resumes in background.

### Type 1: User Text Message

Input:

```
{
  userChatToken: string,
  message: {
    role: "user",             // Must be "user"
    lastPart: {
      type: "text",           // Must be "text"
      text: string            // The user's message content
    },
    metadata?: {
      shouldCorrectUserMessage?: boolean  // Treat as correction of previous message
    }
  },
  attachments?: [{
    objectUrl: string,
    name: string,
    mimeType: string,
    size: number
  }]
}
```

### Type 2: Tool Result Submission

Used to respond to a pending tool call (requestInteraction or makeStudyPlan) found in messages.

Input:

```
{
  userChatToken: string,
  message: {
    id: string,               // REQUIRED: original message ID containing the pending tool call
    role: "assistant",        // REQUIRED: must be "assistant"
    lastPart: {
      type: "tool-requestInteraction" | "tool-makeStudyPlan",
      toolCallId: string,     // The toolCallId from the pending tool part
      state: "output-available",
      input: { ... },         // Copy the original tool call's input verbatim
      output: { ... }         // Your response (see shapes below)
    }
  }
}
```

### Interaction Output Shapes

For requestInteraction, single-question mode (pending input has a `question` field):

```
{ answer: string[], plainText: string }
// answer: selected option values. Always string[] even for single-choice (maxSelect: 1).
// plainText: human-readable summary, e.g. "User selected: 23-28"
// To skip all options: { answer: [], plainText: "None of the above" }
```

For requestInteraction, multi-question mode (pending input has a `questions` array):

```
{ answers: [{ label: string, answer: string[] }], plainText: string }
// One entry per question. label matches the question's label field exactly.
// plainText: summary, e.g. "Gender: Female\nAge: 25-34"
```

For makeStudyPlan:

```
{ confirmed: boolean, plainText: string }
// confirmed: true to approve, false to reject and restart planning
// plainText: e.g. "User confirmed research plan"
```

### Output (both types)

```
{
  messageId: string,
  role: "user" | "assistant",
  status: "running" | "saved_no_ai" | "ai_failed",
  attachmentCount?: number,   // Present when attachments included
  reason?: string,            // Present when status is "saved_no_ai" (e.g. "quota_exceeded")
  error?: string              // Present when status is "ai_failed"
}
```

Notes:

- "running": message saved, AI execution started in background.
- "saved_no_ai": message saved but quota exhausted, no AI execution.
- "ai_failed": AI startup failed. Message is persisted. Retry by sending another message.
- If client times out, use atypica_study_get_messages to check whether AI is still running.
- The answer array values must match the option label strings exactly.
- For single-choice (maxSelect: 1), still send answer as a one-element array.

---

## atypica_study_get_messages

Retrieve conversation history and execution status.

Input:

```
{
  userChatToken: string,
  tail?: number               // Limits to last N parts across all messages. Recommended: 3-5.
}
```

Output:

```
{
  isRunning: boolean,         // true = AI executing, poll again. false = can interact.
  messages: [{
    messageId: string,
    role: "user" | "assistant",
    parts: [
      { type: "text", text: string }
      | { type: "reasoning", text: string }
      | {
          type: string,       // e.g. "tool-requestInteraction", "tool-makeStudyPlan", "tool-generateReport"
          state: "input-available" | "output-available" | "output-error",
          toolCallId: string,
          input: { ... },
          output?: { ... },   // Present when state is "output-available"
          errorText?: string  // Present when state is "output-error"
        }
    ],
    createdAt: string         // ISO 8601
  }]
}
```

Notes:

- Scan parts for state === "input-available" — these require user interaction (submit via send_message Type 2).
- With tail, you get the last N parts (newest). Useful for quick state checks without full history.
- Without tail, returns all messages. Can be large for long studies.
- Polling strategy: before plan confirmation every 30s, after plan confirmation every 5 min.

---

## atypica_study_list

List historical research sessions.

Input:

```
{
  page?: number,              // Default: 1
  pageSize?: number           // Default: 20, max: 100
}
```

Output:

```
{
  data: [{
    studyId: number,
    token: string,            // userChatToken to resume/query this study
    title: string,            // Auto-generated title
    topic: string,            // Research topic
    hasReport: boolean,
    hasPodcast: boolean,
    replayUrl: string,        // https://atypica.ai/study/{token}/share?replay=1
    createdAt: string,        // ISO 8601
    updatedAt: string
  }],
  pagination: { page: number, pageSize: number, totalCount: number, totalPages: number }
}
```

Notes:

- Ordered by most recently updated first.
- Use the token to call get_messages or send_message on existing studies.

---

## atypica_study_get_report

Retrieve a generated research report.

Input:

```
{
  token: string               // Report token from generateReport tool output in messages
}
```

Output:

```
{
  token: string,
  instruction: string,        // The generation instruction used
  title: string,
  description: string,
  content: string,            // Full report in HTML format
  coverUrl?: string,          // Signed CDN URL, expires in 1 hour
  shareUrl: string,           // Public: https://atypica.ai/artifacts/report/{token}/share
  generatedAt: string,        // ISO 8601
  createdAt: string,
  updatedAt: string
}
```

Notes:

- The report token is found in message parts where type is "tool-generateReport" and state is "output-available", at output.reportToken.
- coverUrl is signed and expires after 1 hour. Re-fetch if expired.
- content is self-contained HTML.

---

## atypica_study_get_podcast

Retrieve a generated podcast.

Input:

```
{
  token: string               // Podcast token from generatePodcast tool output in messages
}
```

Output:

```
{
  token: string,
  instruction: string,        // The generation instruction used
  script: string,             // Full podcast transcript
  audioUrl?: string,          // Signed CDN URL for audio, expires in 1 hour
  coverUrl?: string,          // Signed CDN URL for cover image, expires in 1 hour
  metadata: {
    title: string,
    duration: number,         // Seconds
    size: number,             // Bytes
    mimeType: string,         // e.g. "audio/mpeg"
    showNotes: string
  },
  shareUrl: string,           // Public: https://atypica.ai/artifacts/podcast/{token}/share
  generatedAt: string,        // ISO 8601
  createdAt: string,
  updatedAt: string
}
```

Notes:

- The podcast token is found in message parts where type is "tool-generatePodcast" and state is "output-available", at output.podcastToken.
- audioUrl and coverUrl are signed and expire after 1 hour.

---

## atypica_get_upload_credentials

Get a presigned URL to upload a file. After uploading via HTTP PUT, use the objectUrl as attachment reference.

Input:

```
{
  fileName: string,           // With extension, e.g. "survey.pdf"
  mimeType: string            // e.g. "image/png", "application/pdf", "text/csv"
}
```

Output:

```
{
  putUrl: string,             // Presigned PUT URL. Expires in 5 minutes.
  objectUrl: string,          // Permanent S3 URL to reference in attachments.
  fileName: string,
  mimeType: string
}
```

Notes:

- Supported images: jpeg, png, gif, webp, bmp, svg.
- Supported documents: pdf, json, csv.
- Upload: HTTP PUT to putUrl with Content-Type header matching mimeType, file as body.
- putUrl is single-use and expires after 5 minutes.
- Limits per message: max 5 images, max 3 documents, max 3MB per file, max 50MB total.

---

## atypica_panel_search

Search persona panels. Each panel is a curated group of AI personas used as research subject pool.

Input:

```
{
  query?: string,             // Filter by title, case-insensitive. Without query, returns all.
  page?: number,              // Default: 1
  pageSize?: number           // Default: 20, max: 50
}
```

Output:

```
{
  data: [{
    panelId: number,
    title: string,
    personaCount: number,
    createdAt: string,        // ISO 8601
    updatedAt: string
  }],
  pagination: { page: number, pageSize: number, totalCount: number, totalPages: number }
}
```

Notes:

- Pass panelId to atypica_study_create to use that panel as the persona source.
- Without query, all accessible panels are returned.

---

## atypica_persona_search

Search AI personas using text matching.

Input:

```
{
  query?: string,             // Text search over name/source fields
  privateOnly?: boolean,      // true = only caller's own private personas. Default: false.
  limit?: number              // Default: 10, max: 50
}
```

Output:

```
{
  data: [{
    personaId: number,
    token: string,
    name: string,
    source: string,           // Origin/background description
    tier: number,             // Access tier 0-3
    tags: string[],
    createdAt: string         // ISO 8601
  }]
}
```

Notes:

- With query: uses indexed text search.
- Without query: returns the latest visible personas (public + caller's private).
- privateOnly: true restricts to only the caller's own personas.

---

## atypica_persona_get

Get full details of a persona including its complete prompt.

Input:

```
{
  personaId: number
}
```

Output:

```
{
  personaId: number,
  token: string,
  name: string,
  source: string,
  prompt: string,             // Full persona system prompt text
  tier: number,               // 0-3
  tags: string[],
  locale: string,             // e.g. "en-US", "zh-CN"
  createdAt: string,          // ISO 8601
  updatedAt: string
}
```

Notes:

- The prompt field contains the full text used as the AI persona's system prompt.
- Persona details don't change often — safe to cache locally.

---

## Error Handling

All errors follow JSON-RPC 2.0 format: `{ error: { code: number, message: string } }`

- **-32001 Unauthorized**: Invalid, expired, or missing API key. Verify format (`atypica_xxx`) and Bearer scheme.
- **-32602 Invalid params**: Field type mismatch or constraint violation.
- **-32603 Internal error**: Server-side failure. Safe to retry after brief delay.
- **-32000 Business error**: Study not found, unauthorized access.
- **Quota exceeded**: Not a JSON-RPC error. sendMessage returns status "saved_no_ai" with reason "quota_exceeded".
- **AI failed**: sendMessage returns status "ai_failed" with error message. Message is persisted, retry by sending another.
- **Timeout**: If sendMessage times out, poll getMessages — AI may still be running.
- **Retry guidance**: All read operations are idempotent and safe to retry.

---

## Security and Limitations

- API keys are user-scoped only. Team keys return 403.
- All operations verify resource ownership.
- CDN URLs (cover, audio) are signed and expire after 1 hour.
- Upload presigned URLs expire after 5 minutes, single-use.
- Max 5 images, 3 documents per message. Max 3MB per file. Max 50MB total per message.
- Max 100 messages returned per get_messages request.
- Persona search returns public personas plus caller's own private personas.
- Not available via MCP: referencing historical studies in new ones, custom team prompts, real-time push notifications.
- Data isolation: per-user account. Studies and personas not shared between users. AI responses saved to user's account only.
