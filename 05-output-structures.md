# 05 — Output structures

Owners: `packages/core/session/README.md` (log), `docs/subsystems/session.md` (projection), `packages/llm/llm/README.md` (vocabulary).

## Session events

`SessionEventMap` in `packages/core/session/src/types.ts:281-428`. Boundaries (`turn/start|end`, `step/start|end`), model-visible surface (`system|developer|user|assistant/message`, `tool/result`), attempts and calls (`assistant/attempt`, `tool/call`), request envelope (`request/header{header, reason: initial|resume|change|series, startsSeries?}`, `request/context{provider, model, contextWindow?, systemPromptUpdate?}`), lineage (`session/end-seed`). `assistant/message{turn, step, message, stream: AssistantStreamRecord[], usage?, interrupted?}` carries the exact compact timed stream that produced it; `assistant/attempt{turn, step, stream}` preserves failed, retried, cancelled, and stream-error attempts without adding history. `EpochHeader{config, adapterDefaults?, tools?}` (`:240-251`) never carries `system` — the prompt is derived history. `SESSION_FORMAT_VERSION = 4` (`:89`); only structural changes bump it, ordinary event additions ride the per-event `ignorable` guard.

## Messages and blocks

`packages/llm/llm/src/message.ts:139-196`: `MessageBase{id, content, source}`, then `SystemMessage{role: system, source: {kind: system-prompt}}`, `DeveloperMessage{role: developer}`, `UserMessage{role: user, source: any kind}`, `AssistantMessage{role: assistant, source: {kind: model, provider, model, replayState?}}`, `ToolResultMessage{role: tool, toolCallId, isError?}`. `MessageRoleMap` (`:187-193`) is closed: every model-visible role needs a durable event and an adapter projection. `ContentBlockMap` (`packages/llm/llm/src/types.ts:137-145`): `text | reasoning | image | file | tool-call | tool-addition | tool-removal`; switch on `type`, fall through unknowns (merge-extensible). `MessageSourceMap` (`message.ts:110-115`) is likewise merge-extensible: switch on `kind`, fall through unknowns.

## Surface projection

Eligible types in `packages/core/session/src/surface.ts:50-56`: `system|developer|user|assistant/message, tool/result`. `deriveEventMessage` (`:120-152`): `user` projects verbatim (framing is producer-owned); empty `system/developer/assistant` content projects to `null` (retains surface position, sends no wire message — so an empty head records "no system prompt" and a content-less `max-tokens` message hosts only usage); `tool/result` projects its message; everything else projects to `null`. `foldSurface` (`:594-609`) tracks `nodes/replacements/projectedMessages`; `Session.deriveMessages()` returns a frozen copy, pure tail-append cost. `validateSessionEventData` (`surface.ts:166-220`) rejects `header.system`, empty `tools`, and `error` without `isError: true`. `isAppendSurfaceEvent` vs `isReplacementSurfaceEvent` (`:89-106`): the model surface shadows replaced ranges, so human transcripts read append-origin events, not the model surface.

## Verify

Owned by `packages/core/session/src/types.ts` (`assistant/message`, `assistant/attempt`, `tool/result`, `request/header`), `surface.ts` (`deriveEventMessage`, `isAppendSurfaceEvent`, `isReplacementSurfaceEvent`), and `packages/llm/llm/src/types.ts` + `message.ts` (`ContentBlockMap`, `MessageSourceMap`, `GenerateOptions`). Re-check those files if the harness has moved.
