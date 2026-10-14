# 06 — Display: from log to screen

Owners: `docs/subsystems/conversation.md`, `docs/subsystems/web-client.md`, `docs/cookbook/adding-a-tool.md`.

## Two rendering philosophies

Host presenters stay pure and local: `defineTool({ output: { render, schema }, presentCall, presentResult, output.presentationMeta })` (`packages/core/tools/src/presentation.ts`, `packages/core/tools/README.md`). Intents are `card`-tagged — call views `generic|terminal|diff`, result views `generic|terminal|diff|search|read|web` (`presentation.ts:46-140`) — and the payload persists verbatim on `tool/result.meta` (`packages/core/tools/src/index.ts`, see `appendToolResult` `:288`) for replay. Web explicitly does not consume them (`packages/client/ui-tool/README.md:12,54`): the Session page carries raw `tool/call + tool/result + meta` (`packages/api/session-controller/README.md:30`) and the client derives card props from raw args, content, failure, `meta`, and session `cwd`.

## Web conversation assembly

Path (`docs/subsystems/web-client.md`): Host log, remote follow/page (packed), client `SessionEventLikeEntry{type: event|transient}`, `ui-conversation` assembler, per-target snapshot (`chat|trajectory`), Slot view, React. One `ConversationNodeAssembler` per `SessionBinding` routes by stable `(kind, id)`; `match(event)` extracts identity only, `start` folds the earliest loaded start, `update` replays deterministically by `seq`, `publication` is `immediate|animation-frame|none`, `buildLocationData` shares Turn/Step data, `buildViewNode` emits renderer-ready payloads (`docs/subsystems/conversation.md`). Business packages register a `ConversationNodeDefinition` plus view plus keyed Slot renderer, e.g. `packages/client/ui-chat/src/client/conversation-nodes/*.ts` (`assistant.ts`, `message.ts`, `inbox.ts`, `compaction.ts`, …), trajectory definitions in `packages/client/ui-trajectory/src/client/`, plus deliverables, plan, workflow-run, and schedule nodes. Tool cards dispatch twice: `conversation.chat.node:key=tool-call` then `tool.call.toolview:key=<wireToolName>` (`packages/client/ui-tool/README.md:12,36-41`). Live assistant deltas arrive as client-only `assistant/live-chunk` transients; durable settlements replay from embedded streams (`docs/subsystems/conversation.md`).

## Headless, SDK, ACP

Headless one-shot (`packages/bundle/headless/README.md`, `src/index.ts|startup.ts|json-stream.ts`): stderr `dsh: reasoning:` live deltas, stdout final assistant text, exit `0` on `completed` else `1`. `--json` projects NDJSON (`session,status,text,thinking,tool_call,tool_result,final,error`) at commit points only — text comes from committed `assistant/message`, never live attempts — capped at 8 KiB per string and 32 KiB per line with `final` exempt (`json-stream.ts:16,19`). SDK JSON-RPC (`packages/sdk/protocol/README.md`, `server/README.md`, `client/README.md`): `initialize, session/prompt -> messageId, shutdown` plus `session.event` (every event, unfiltered); `run()` collects to `idle` with `finalResponse` as the last `assistant/message`. ACP (`packages/acp/acp/README.md`): standard ACP v1 verbs plus `session/update`; omits presentation data, plans, titles, todos, terminals, replay, and fork/delete (`README.md:12,32,80,175`).

## Titles, compaction, transcripts

Titles (`packages/session/session-title/README.md`, `docs/subsystems/session-title.md`): `session/title` is log-only, never in `deriveMessages()` (`README.md:121`); newest wins, explicit rename pins (`README.md:32`); served as a `sessionProjection` unit to list rows. Compaction (`packages/compaction/compaction/README.md`): log bracket `compaction/start|summary|end` plus one `user/message` replacement checkpoint (`README.md:95-97`); Chat shows a dim marker row with expandable summary (`packages/client/ui-chat/src/client/chat/CompactionItem.tsx`) without replacing shadowed rows; Trajectory puts standalone requests in `Between turns`. Transcripts are views, not objects: Chat renders `chat`-target nodes in Turn order; raw form ships as a `session-log-export` ZIP (`session.vN.jsonl` + `media/|files/`); query/export (not render) lives behind `ctx.sessionQuery` (`packages/session-query/session-query/README.md`).

## Verify

Owned by `packages/core/tools/src/presentation.ts` (`presentCall`, `presentResult`), `packages/client/ui-tool/README.md` (`conversation.chat.node`, `tool.call.toolview`), and `packages/bundle/headless/src/json-stream.ts` (`MAX_STRING_BYTES`, `MAX_EVENT_BYTES`). Re-check those files if the harness has moved.
