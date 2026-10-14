# 04 — Tools: schemas into request, calls into log

Owners: `packages/core/tools/README.md`, `docs/tool-execution-pipeline.md`, `docs/cookbook/adding-a-tool.md`.

## Schemas join the request, not the prose

`ctx.systemPrompt.tools(context => this.wireSchemas(context.scope))` in `packages/core/tools/src/index.ts:855`. `wireSchemas` (`:1017-1035`) resolves visible schemas per scope and presentation mode: `native` shows tool schemas, `ptc` shows only the `run_code` transport (`knownNames: [RUN_CODE_NAME]`), `both` shows both. Assembly clones, orders (`orderTools`, `packages/core/system-prompt/src/index.ts:222-236`, `<unlisted-tools>` rest marker), and sends them as wire `header.tools` (`packages/core/agent-loop/src/agent.ts:628-629`); adapters map `GenerateOptions.tools` (`packages/llm/llm/src/types.ts:529-530`) to provider fields. `ToolSchema{name, description, parameters, deferLoading?}` is declared in `packages/llm/llm/src/types.ts:473-484` because it belongs to the request, not the tool registry. The `TOOL_*` prompt sections (order `1000-2900`) are guidance prose only and unaffected by restriction; each tool returns `''` when hidden. PTC mode adds the `tools:ptc-only` collapse rule (`tools/src/index.ts:877-885`) and the generated `tools:sdk` section (`:897-915`, `interpolate: false`). Restriction therefore doubles as prompt reduction — the token-lean recipe in `09-override-strategy.md` builds on this.

## Execution scheduler

`executeToolCalls` in `packages/core/agent-loop/src/tool-calls.ts:60-102`. Model arguments parse with invalid JSON preserved as text and empty input mapped to `{}` (`:105-111`). Calls group by `ctx.tools.executionMode(exec).kind`: exclusive calls form barriers, parallel calls run in a bounded rolling pool capped by `maxParallelToolCalls`, reclassified before each start (`:89-99`, `:199-214`). Ordered `prepare`, overlapping `dispatch`, model-ordered `commitReady` through `TOOL_RUNTIME_SCHEDULER.finalize/finish` (`:147-161`). Abort drains started calls, records synthetic `TOOL_ABORTED_BEFORE_DISPATCH` results for skipped ones, and returns with the signal still aborted (`:238-243`, `:250-260`).

## Commits

`appendToolCall` (`:263-266`) appends `tool/call{turn, step, callId, name, arguments: raw JSON string}` (log-only) and returns its seq. `appendToolResult` (`:269-290`) builds the message with `createToolResultMessage` (`packages/llm/llm/src/message.ts:299-307`) and appends `tool/result{turn, step, message, error?, meta?}` with `{ surfaceOp: 'append', sourceEventSeqs: [callSeq] }`. `error` is allowed only with `message.isError: true`; `meta` is tool-private, JSON-validated, opaque to core, and persists the Host card for replay. `additionalContexts` feed the `next-step` inbox; `concludesTurn` ends the turn, otherwise the loop continues (`tool-calls.ts:157-159`, `agent.ts:549-555`).

## Verify

Owned by `packages/core/tools/src/index.ts` (`wireSchemas`, `collapseSection`, `sdkSection`), `packages/core/agent-loop/src/tool-calls.ts` (`executeToolCalls`, `appendToolCall`, `appendToolResult`), and `packages/llm/llm/src/types.ts` (`ToolSchema`). Re-check those files if the harness has moved.
