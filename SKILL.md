---
name: atypica-research
description: Run user research with atypica.ai through its MCP server. Use when the user wants to understand consumers, test a product or concept, or analyze a market. AI-simulated consumers are interviewed and discuss the question, and a report comes out. Covers starting a study, answering the agent's questions and plan, telling the user what the study is doing while it runs, reading interviews and discussions, and fetching the report or podcast.
---

# atypica Research

A study: AI-simulated consumers are interviewed and discuss the user's question, and a report comes out with traceable sources. Results are AI simulations; say so when you present them.

After the user confirms the plan, a study usually runs half an hour to an hour. During that time you are the user's only view of it: keep them in the loop the way a researcher would, without making them wait in silence or read logs.

Read [references/api-reference.md](references/api-reference.md) for every tool's input and output.

## Connection

- **Endpoint**: `https://atypica.ai/mcp/study`
- **API key**: from https://atypica.ai/account/api-keys, sent as `Authorization: Bearer atypica_xxx`

If tools named `atypica_study_*` are available, it is connected. Otherwise add it to the client, for example Claude Desktop:

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

Without an MCP client, every tool is one JSON-RPC call over HTTP. `scripts/mcp-call.sh` wraps it:

```bash
export ATYPICA_TOKEN="atypica_xxx"
scripts/mcp-call.sh atypica_study_create '{"content":"Why do young people buy sugar-free coffee?"}'
scripts/mcp-call.sh atypica_study_get_messages '{"userChatToken":"<token>","tail":5}' -o structured
```

The research is the same on every client; the account's credits pay for it.

## The loop

One question is one study.

1. `atypica_study_create({ content })` starts it and returns its `token`, the `userChatToken` every other study tool takes. The research agent starts at once.
2. Poll `atypica_study_get_messages({ userChatToken, tail: 5 })` and act on `next`:

| `next.kind`      | Do                                                                                                  |
| ---------------- | --------------------------------------------------------------------------------------------------- |
| `running`        | Tell the user what changed in `steps` (below); poll again in 1–2 minutes                            |
| `tool_result`    | Ask the user (see "When the agent asks"), then `atypica_study_submit_tool_result` with their answer |
| `artifact_ready` | Fetch the report or podcast with `next.token` and deliver it                                        |
| `text_message`   | Relay what the agent said; send the user's reply with `atypica_study_send_text_message`             |
| `stalled`        | Send `"Please continue the research"` as a text message; `next.error` says why the last run stopped |

Write results carry a `status`: `running` (the agent started), `saved_no_ai` (out of credits: tell the user), `saved_while_running` (saved, but another write started the agent first: do not write it again; once `next` leaves `running`, send a short message to continue), `ai_failed` (write again). A write while the agent runs is rejected; wait for `next` to leave `running`.

## Telling the user what is happening

`steps` lists what the study has done, one entry per step (`social_observation`, `personas`, `interviews`, `discussion`, `audience_call`, `desk_research`, `report`, `podcast`), each with a `state` (`running`, `done`, `failed`, `interrupted`) and a short summary when done. Interviews, a discussion and a social observation also list their `records`, one per interview, each with its own `state`. Report changes, not the list:

- **When the study starts**, say what it will do in one sentence and give `replayUrl`: the user can open it without signing in to watch the study. It shows the study as of opening; reloading brings it up to date.
- **When a step starts or finishes**, say so in a sentence or two from its summary: "Five AI consumers are ready: …", "Three of five interviews are done; the first ones point to price as the main hesitation."
- **When a long step starts** (`social_observation`, `interviews`, `discussion`), mention `replayUrl` again: that is where the user can follow them as they happen.
- **When nothing changed**, one short line or nothing. Never invent progress, counts or findings that `steps` does not show.

Personas: when a `personas` step is done, show the names and tags from its `details` as a short list labeled as AI-simulated consumers. For one in full, `atypica_persona_get({ personaId })`.

Going deeper: when the user asks what was actually said, read the record with `atypica_study_get_step({ userChatToken, stepToken })` and quote from it. One record at a time; pass `after` with the previous `total` to read only what is new in a running step. Do not read steps the user has not asked about.

## When the agent asks

The agent stops twice for the user: to clarify the question (`tool-requestInteraction`) and to confirm its plan (`tool-makeStudyPlan`). The answer is always the user's: show the question, wait, submit what the user chose. Never answer, pick an option or confirm a plan for the user, and never take silence as a yes.

| `next.toolType`           | Show                                                                | Submit as `output`                                                                                                     |
| ------------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `tool-requestInteraction` | Each question with its options, as choices the user can pick        | `{ "answer": ["<option>"] }`; several questions `{ "answers": [{ "label", "answer": [...] }] }`; `{ "skipped": true }` |
| `tool-makeStudyPlan`      | The plan in a few lines: what is studied, with whom, what comes out | `{ "confirmed": true }`, or `{ "confirmed": false }` and send the user's changes as a text message                     |

`atypica_study_submit_tool_result({ userChatToken, toolCallId: next.toolCallId, output })`. Option text is copied exactly; `answer` is always an array. An answer in the wrong shape is rejected and not saved.

## Delivering

- Report: `atypica_study_get_report({ token: next.token })`. Give the title, a short summary in your own words and `shareUrl`.
- Podcast, when the study made one: `atypica_study_get_podcast({ token: next.token })`.
- Earlier studies: `atypica_study_list`; each `token` is that study's `userChatToken`.

## Who is simulated

By default the agent builds the consumers from the question. To use a group the user already has, find it with `atypica_panel_search` and pass `panelId` to `atypica_study_create`. `atypica_persona_search` needs `locales` (`["zh-CN"]`, `["en-US"]` or both).

## Files

`atypica_get_upload_credentials({ fileName, mimeType })`, then HTTP PUT the file to `putUrl` with the same `Content-Type` within 5 minutes, then pass `{ objectUrl, name, mimeType, size }` in `attachments` of `atypica_study_create` or `atypica_study_send_text_message`. Images: jpeg, png, gif, webp, bmp, svg. Documents: pdf, doc, docx, xls, xlsx, ppt, pptx, txt, md, json, csv. A study holds up to 5 images and 10 documents in all, 3 MB each, 50 MB in total.
