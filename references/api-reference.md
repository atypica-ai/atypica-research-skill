# atypica Research API Reference

Every tool of the MCP server at `https://atypica.ai/mcp/study`, with its input and output. The same tools are on the unified server `https://atypica.ai/mcp`, together with human interviews and desk research.

## Conventions

**Transport and sign-in.** Streamable HTTP, JSON-RPC 2.0, at `https://atypica.ai/mcp/study`. Send `Authorization: Bearer atypica_…`, an API key from https://atypica.ai/account/api-keys. It acts as the account the user signed in as (a personal account or a team member) and spends that account's credits. A missing or invalid key is HTTP 401.

**Results.** Every tool returns `content` (one text block, for clients that read only text) and, on success, `structuredContent` (the JSON below). Both are made from the tool's declared `outputSchema`, which `tools/list` returns; no field outside it is sent. A failure is a normal result with `isError: true` and one sentence in the text, not a JSON-RPC error:

- one the client can act on (not found, still running, a wrong answer shape) says what to do;
- an unexpected one says only that something went wrong on atypica's side; the cause stays in the server log.

Input that does not match the schema is the same kind of failure: `isError: true`, and the text starts with `MCP error -32602: Input validation error` and names the field.

**Ownership.** A token or id that belongs to another account gives the same "not found" as one that does not exist.

**Links.** Public pages (study replay, report and podcast share, interview invite) are absolute URLs on the site the request came to: `https://atypica.ai` in production. They open without sign-in.

**Notation.** Shapes are TypeScript-like. `?` marks an optional field; dates are ISO 8601 strings.

**Annotations.** Read tools: `readOnlyHint: true`. Write tools: `readOnlyHint: false, destructiveHint: false`.

## The loop

A study: AI-simulated consumers are interviewed and discuss the user's question; the agent writes a report. One question is one study, named by its `userChatToken`. The research agent runs in the background; after the user confirms the plan it usually runs half an hour to an hour.

```
atypica_study_create ──► agent runs
  └─► atypica_study_get_messages (poll), then by next.kind:
        running         poll again; tell the user what changed in steps
        tool_result     atypica_study_submit_tool_result with the user's answer
        artifact_ready  atypica_study_get_report / atypica_study_get_podcast
        text_message    atypica_study_send_text_message with the user's reply
        stalled         atypica_study_send_text_message ("Please continue the research")
```

## Tools

### atypica_study_create

Starts a study with the user's question as its first message. The research agent starts at once.

Input:

```ts
{
  content: string                // the user's research question; at least 1 character
  panelId?: integer              // use this panel's personas (atypica_panel_search) instead of building new ones
  attachments?: Attachment[]
}

type Attachment = {
  objectUrl: string              // from atypica_get_upload_credentials, after the PUT
  name: string
  mimeType: string
  size: number                   // bytes
}
```

Output: the study plus a [write result](#write-result).

```ts
{
  token: string                  // the study's userChatToken
  studyId: number
  messageId?: string
  status: "running" | "saved_no_ai" | "saved_while_running" | "ai_failed"
  reason?: "quota_exceeded"      // with saved_no_ai
}
```

### Write result

What `atypica_study_create`, `atypica_study_send_text_message` and `atypica_study_submit_tool_result` return after writing. The study page on atypica.ai writes the same way.

| Case                                             | `isError` | `structuredContent`                                                                                                                                                                |
| ------------------------------------------------ | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Saved, the agent is running                      | no        | `{ messageId, status: "running", attachmentCount? }`                                                                                                                               |
| Saved, the account has no credits                | no        | `{ messageId, status: "saved_no_ai", reason: "quota_exceeded", attachmentCount? }`                                                                                                 |
| Saved, the agent did not start                   | no        | `{ messageId, status: "ai_failed", attachmentCount? }`; write again to retry                                                                                                       |
| Saved, but another write started the agent first | no        | `{ messageId, status: "saved_while_running", attachmentCount? }`; do not write it again: that run does not read it; once `next` leaves `running`, send a short message to continue |
| The agent was already running                    | yes       | none; nothing was saved                                                                                                                                                            |
| A tool result in the wrong shape                 | yes       | none; nothing was saved; the text names the problem                                                                                                                                |
| No tool call with that id is waiting             | yes       | none; nothing was saved                                                                                                                                                            |
| Too many or unsupported attachments              | yes       | none; nothing was saved                                                                                                                                                            |
| The study does not exist or is not the caller's  | yes       | none                                                                                                                                                                               |

`attachmentCount` is present on `atypica_study_send_text_message`.

### atypica_study_send_text_message

Writes a text message from the user into a study and starts the agent on it: a follow-up, a correction, the reply to something the agent said, "please continue".

Input:

```ts
{
  userChatToken: string
  text: string                   // in the user's words
  attachments?: Attachment[]
}
```

Output: a [write result](#write-result).

Attachments count over the whole study: at most 5 images and 10 documents, 3 MB per file, 50 MB in total. Images: jpeg, png, gif, webp, bmp, svg. Documents: pdf, doc, docx, xls, xlsx, ppt, pptx, txt, md, json, csv.

### atypica_study_submit_tool_result

Answers a tool call the agent left for the user, reported by `atypica_study_get_messages` as `next.kind: "tool_result"`. The server finds the waiting call in the study's last message and keeps its input; the client sends only the id and the answer. The answer is validated against that tool's output schema before it is saved.

Input:

```ts
{
  userChatToken: string
  toolCallId: string             // next.toolCallId
  output: unknown                // the user's answer, shaped by next.toolType:
}

// tool-requestInteraction, one question (next.questions has one):
{ answer: string[] }             // option texts copied exactly; an array even for one choice
// tool-requestInteraction, several questions (next.questions has more than one):
{ answers: { label: string, answer: string[] }[] }
// tool-requestInteraction, the user declined:
{ skipped: true }
// tool-makeStudyPlan:
{ confirmed: boolean }           // false: the agent plans again; send the user's changes as a text message
```

Output: a [write result](#write-result).

### atypica_study_get_messages

Reads a study: what to do next, what it has done so far, and its conversation.

Input:

```ts
{
  userChatToken: string
  tail?: integer                 // keep only the last N conversation entries; does not change next or steps
}
```

Output:

```ts
{
  isRunning: boolean
  next: Next
  steps: Step[]
  replayUrl: string              // https://atypica.ai/study/<token>/share?replay=1
  conversation: Entry[]
}

// What the user and the research agent said to each other, in order (src/app/(study)/tools/conversation.ts).
// The research the agent ran is in steps; its reasoning and other tool calls are not returned.
type Entry =
  | { type: "user"; text: string; createdAt?: string }
  | { type: "agent"; text: string; createdAt?: string }
  | {
      type: "question"           // the agent asked the user to choose (tool-requestInteraction)
      questions: { label?: string; question: string; options: string[] }[]
      answer?: { label?: string; answer: string[] }[] | "skipped"   // absent while unanswered
      createdAt?: string
    }
  | {
      type: "plan"               // the agent asked the user to confirm a plan (tool-makeStudyPlan)
      plan: string               // markdown
      confirmed?: boolean        // absent while unanswered
      createdAt?: string
    }
```

`replayUrl` opens without sign-in and shows the study as it was when opened; reloading brings it up to date.

The text content holds `next`, `replayUrl`, the steps and the last 3000 characters of the conversation.

#### next

What the client does now, decided by the server. One kind at a time, checked in this order:

```ts
type Next =
  | { kind: "running" } // poll again in 1–2 minutes
  | {
      kind: "tool_result"; // the agent asked the user to choose: show the questions, then atypica_study_submit_tool_result
      toolCallId: string;
      toolType: "tool-requestInteraction";
      questions: { label?: string; question: string; options: string[]; maxSelect?: number }[]; // maxSelect: how many the user may choose; absent: any number
    }
  | {
      kind: "tool_result"; // the agent asked the user to confirm a plan: show it, then atypica_study_submit_tool_result
      toolCallId: string;
      toolType: "tool-makeStudyPlan";
      plan: string; // markdown
    }
  | {
      kind: "artifact_ready"; // fetch with atypica_study_get_report / atypica_study_get_podcast
      artifactType: "report" | "podcast";
      token: string;
    }
  | { kind: "stalled" } // the last run failed or stopped; send a text message to start again
  | { kind: "text_message" }; // the agent spoke and stopped; relay it, send the user's reply as text
```

When the last run failed, `stalled` comes before `text_message`: a failed run often ends in half a sentence. Why it failed is not returned; it stays in the server log.

#### steps

What the study has done, one entry per step, in order.

```ts
type Step = {
  kind:
    | "social_observation"
    | "personas"
    | "interviews"
    | "discussion"
    | "audience_call"
    | "desk_research"
    | "report"
    | "podcast";
  state: State;
  brief?: string; // what the agent asked the step to do, at most 200 characters
  summary?: string; // what came out, at most 500 characters; once done
  details?: object; // facts, by kind (below)
  records?: {
    // parts with their own record, each read by atypica_study_get_step
    stepToken: string;
    label?: string; // e.g. the persona interviewed
    state: State;
  }[];
};

type State = "running" | "done" | "failed" | "interrupted";
```

| `kind`               | `brief` | `summary`                | `details`                                         | `records`            |
| -------------------- | ------- | ------------------------ | ------------------------------------------------- | -------------------- |
| `social_observation` | yes     | `scoutSocialTrends` only | `platforms?`, `searches?` (searches per platform) | one: the observation |
| `personas`           | no      | no                       | `personas: { personaId, name, tags }[]`           | none                 |
| `interviews`         | yes     | yes                      | none                                              | one per persona      |
| `discussion`         | yes     | yes                      | none                                              | one: the discussion  |
| `audience_call`      | yes     | yes                      | none                                              | none                 |
| `desk_research`      | yes     | yes                      | `sources?` (count)                                | none                 |
| `report`             | no      | no                       | `token?`                                          | none                 |
| `podcast`            | no      | no                       | `token?`                                          | none                 |

States:

- A step is `running` only when the study is running and the step is in the study's last message: a run writes only that message. A step left without output by an earlier run is `interrupted`.
- A step whose tool returned its own failure is `failed`, not `done`: a discussion with fewer than two panel personas, personas built from a social observation that was not found, an interview group where no interview ran. Its records are `failed` too.
- An interview record follows that interview's own result: `done` when it is in the tool output's list of interviews that ran, `failed` when it failed, was skipped (already interviewed) or was not in the panel. While the group is unfinished, an interview is already `done` once it has saved its conclusion and its last messages: a record that says `done` gets no more entries.

### atypica_study_get_step

Reads what happened inside one record of a step: what the interviewer and one persona said, what each persona said in a discussion, what a social observation searched and found. Works while the step runs.

Input:

```ts
{
  userChatToken: string
  stepToken: string              // from steps[].records[] of this study
  after?: integer                // skip the first this-many entries; pass the previous total
}
```

Output:

```ts
{
  kind: string                   // the step's kind
  state: State                   // the record's state
  label?: string
  total: number                  // entries so far
  entries: Entry[]               // entries after `after`
  summary?: string               // the interview's conclusion or the discussion's summary, once written
}

type Entry =
  | { type: "say", speaker: string, text: string }
  | { type: "action", action: string, query?: string, result: string }  // a tool the observer or interviewer called: query is what it searched for; result cut to 600 characters
```

Speakers: `interviewer` and the persona's name in an interview; the moderator, the question's author and each persona's name in a discussion; `research brief` and `observer` in a social observation.

A `stepToken` that is not in this study's own steps gives `isError: true` ("Step not found in this study"), whether or not it exists elsewhere.

### atypica_study_list

The caller's studies, most recently updated first.

Input:

```ts
{
  page?: integer                 // default 1
  pageSize?: integer             // default 20, at most 100
}
```

Output:

```ts
{
  data: {
    studyId: number;
    token: string; // the study's userChatToken
    title: string;
    topic: string;
    hasReport: boolean;
    hasPodcast: boolean;
    replayUrl: string;
    createdAt: string;
    updatedAt: string;
  }
  [];
  pagination: {
    (page, pageSize, totalCount, totalPages);
  }
}
```

### atypica_study_get_report

Input:

```ts
{
  token: string;
} // the report token: next.token or a report step's details.token; not the userChatToken
```

Output:

```ts
{
  token: string
  title?: string
  description?: string
  content: string                // the report as one self-contained HTML page
  coverUrl?: string              // proxy CDN link, does not expire
  shareUrl: string               // https://atypica.ai/artifacts/report/<token>/share, no sign-in
  generatedAt: string
  createdAt: string
  updatedAt: string
}
```

### atypica_study_get_podcast

Input:

```ts
{
  token: string;
} // the podcast token: next.token or a podcast step's details.token
```

Output:

```ts
{
  token: string
  script: string
  audioUrl: string | null        // proxy CDN link, does not expire; null until the audio exists
  coverUrl?: string              // proxy CDN link, does not expire
  title?: string
  duration?: number              // seconds
  showNotes?: string
  shareUrl: string               // https://atypica.ai/artifacts/podcast/<token>/share
  generatedAt?: string
  createdAt: string
  updatedAt: string
}
```

### atypica_panel_search

The caller's persona panels. A panel is a saved group of AI-simulated consumers; pass its id to `atypica_study_create`.

Input:

```ts
{
  query?: string                 // matches the title, case-insensitive; without it, all panels
  page?: integer                 // default 1
  pageSize?: integer             // default 20, at most 50
}
```

Output:

```ts
{
  data: { panelId: number, title: string, personaCount: number, createdAt: string, updatedAt: string }[]
  pagination: { page, pageSize, totalCount, totalPages }
}
```

### atypica_persona_search

AI-simulated consumers visible to the caller: public ones and the caller's own.

Input:

```ts
{
  locales: ("zh-CN" | "en-US")[] // required, at least one: the personas' language
  query?: string                 // name or source; without it, the newest
  privateOnly?: boolean          // only the caller's own
  limit?: integer                // default 10, at most 50
}
```

Output:

```ts
{
  data: { personaId: number, token: string, name: string, source: string, tags: string[], createdAt: string }[]
}
```

### atypica_persona_get

Input:

```ts
{
  personaId: number;
}
```

Output:

```ts
{
  personaId: number
  token: string
  name: string
  source: string
  prompt: string                 // the persona's profile: who it is and how it speaks
  tags: string[]
  locale: string | null
  createdAt: string
  updatedAt: string
}
```

### atypica_get_upload_credentials

A URL to upload one file to. PUT the file to `putUrl` with the same `Content-Type` within 5 minutes, then pass `objectUrl` as a study attachment.

Input:

```ts
{
  fileName: string; // with extension
  mimeType: string; // images: jpeg, png, gif, webp, bmp, svg+xml; documents: pdf, doc, docx, xls, xlsx, ppt, pptx, txt, md, json, csv
}
```

Output:

```ts
{
  putUrl: string; // presigned PUT, valid 5 minutes
  objectUrl: string; // the file's address, without signature
  fileName: string;
  mimeType: string;
}
```

Another type gives `isError: true`.
