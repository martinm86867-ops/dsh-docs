# 03 — Request, freeze, stream, settle

Owners: `packages/core/agent-loop/README.md`, `docs/subsystems/llm-streaming.md`, `docs/subsystems/core.md`.

## `preStep()`

`packages/core/agent-loop/src/agent.ts:267-288`. Assembles the prompt with `assembleContextFor(this, signal)` — defined in `packages/core/agent/src/dispatch.ts` — then projects runtime context and runs the `agent/pre-step` waterfall with default `enter{claimed + [context?]}` (`:271-284`). Cancellation during assembly or waterfall commits nothing. Decision `reject` spends no step; `enter{messages, startsRequestSeries?, assembly}` is authoritative and wrapping listeners must preserve the `assembly` the loop already resolved.

## `step()` admission

`agent.ts:400-561`. Renders the prompt (`renderPrompt`, `:407`), resolves the route via `prepareRequest` (`:410`) before any commit, refreshes runtime context (`refreshContext`, `:411-413`), reconciles the system surface with `SystemPromptProjection.project(rendered, { inHistory, startsSeries })` (`:416-421`, logic in `runtime-context.ts:88-103`: first prompt reserves node 0 even when empty; capable continuing series appends; otherwise consolidate to head and empty later nodes), appends one `system/message` per commit (`:422-424`) and the entered `user/message`s on the first attempt only (`:426-437`), then builds the frozen request (`:442`).

## `prepareRequest()` and `buildRequest()`

`prepareRequest` (`agent.ts:564-613`): first request seeds from `AgentOptions`; later ones strip adapter-materialized fields via `requestProposal` (`:64-70`); the seed is `deepFreeze(structuredClone(...))`; the `agent/request` waterfall proposes the route (frozen seed, no message mutation); missing `provider/model` throws (`:598-600`); `llm.prepareCall` binds the adapter, with a `NO_ADAPTER` fallback so middleware can serve unregistered routes (`:601-610`).

`buildRequest` (`:616-704`): appends `request/header{initial|resume|change|series}` only when the header changed or a new series starts (`:635-651`); tool deltas become `developer/message{tool-addition/tool-removal + headerSeq}` (`:652-665`); route changes become `request/context` (`:671-683`); then `deepFreeze(header)`, `deriveMessages()` plus one-shot per-identity `deepFreeze` tracked in `frozenMessages:WeakSet` (`:687-693`), and `markAgentLoopRequest(Object.freeze({...config, messages, toolHistory, tools?, sessionId, signal}))` (`:695-702`). Loop requests leave `system` undefined (`packages/llm/llm/src/types.ts:528`); the prompt travels as `system/message` history.

## Streaming and settling

`preparedCall?.stream(request) ?? llm.stream(request)` (`agent.ts:453`) flows through the `llm/stream` waterfall (frozen loop requests are read-only). `AssistantStreamAttempt` in `packages/core/agent-loop/src/assistant-stream.ts:18-140` publishes transient `agent/assistant-stream{start|chunk|end}` frames while accumulating the durable compact stream; `settle(eventType, append)` commits the durable event first, then emits the terminal frame (`:78-97`). `BlockAssembler` (imported from `dsh-llm`) folds `StreamChunk`s (`block-start|text-delta|reasoning-delta|tool-call-delta|block-end|usage|finish`, `packages/llm/llm/src/types.ts:452-464`) into `ContentBlock`s; `block-end` is authoritative and stragglers are ignored.

Settlement (`agent.ts:504-560`): `finish{error|aborted}` commits `assistant/attempt` and offers the `agent/request-error` waterfall — `retry` re-runs `prepareRequest` on the same assembly without repeating `pre-step` or users (`:511-526`); otherwise `throw LlmError` ends the turn in `error`. Mid-stream cancel with delivered blocks commits `assistant/message{interrupted: true}`; with none, `assistant/attempt` (`:462-503`). Success commits `assistant/message{message, stream, usage?}` built by `createAssistantMessage` (`packages/llm/llm/src/message.ts:260-271`); `max-tokens` returns sticky (`:547`); zero tool calls returns `completed`, else `executeToolCalls` (`:549-555`).

## Verify

Owned by `packages/core/agent-loop/src/agent.ts` (`prepareRequest`, `buildRequest`, `frozenMessages`, `request-error`), `assistant-stream.ts` (`AssistantStreamAttempt`), and `packages/core/agent/src/dispatch.ts` (`assembleContextFor`). Re-check those files if the harness has moved.
