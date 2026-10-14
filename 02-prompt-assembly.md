# 02 — Prompt assembly

Owner: `packages/core/system-prompt/README.md`; type reference `docs/subsystems/system-prompt.md`.

## `assemble()`

`SystemPrompt.assemble()` in `packages/core/system-prompt/src/index.ts:610-684`. Layers resolve through `ScopedLayers.chainLayers(scope)` (`:612`) so scoped registrations shadow globals. Variables merge global-first, nearest scope wins (`:616-625`). Sections and contexts merge by name before sorting (`:627-628`); tool providers concatenate global + scoped (`:630-633`), each result's `parameters` detached by `structuredClone` (`:641`) while `knownNames` accumulates the pre-restriction universe for order validation (`:646`). Sections sort by `comparePromptSections` — ascending `order`, code-unit name tiebreak (`:244-246`). At most one `complete: true` section survives; two throw (`:649-652`); each section's `text` resolves against the assembly context (`:654-663`). Contexts sort by `order` unless suppressed (`:666-669`). Tools canonicalize via `orderTools` (`:670`, `:222-236`). The `system-prompt/assemble` waterfall runs scope-filtered and its return is authoritative (`:673-676`), except an effective `complete` section is restored as the sole prompt afterwards (`:677-683`).

## Section order

`SECTION_ORDERS` in `packages/core/system-prompt/src/index.ts:131-165`: `-1000 harness:identity`, `0 deployment:persona-prefix`, `500 plan:policy`, `600 team:policy`, `800 ptc-only`, `900 file-reference`, `1000-2900 TOOL_*` (bash, pwsh, read, write, edit, glob, grep, jobs, pty, web-search, web-fetch, lsp, session-query, goal, workflow, ralph, subagent, report), `3000 computer-use`, `3100 mcp`, `5000 tools:sdk`, `9000 deliverables`, `9900 structured-output`, `10000 harness:source`, `10100 web-surface`, `10200 deployment:persona-suffix`. The constructor registers the harness-owned sections (`:441-464`): `harness:identity` (the one-liner `"You are an AI agent powered by DeepSeek Harness."`, `:445-451`), `deployment:persona-prefix` from `Config.personaPrefix` (`:452-457`), and `deployment:persona-suffix` from `Config.personaSuffix` (`:458-462`). A scoped section with the same name (e.g. `deployment:persona-prefix` in `packages/preset/persona/src/index.ts:62-74`) shadows the global one instead of duplicating it.

## Rendering

`renderPrompt` (`:286-291`) interpolates each section unless `interpolate: false`, drops empties, joins with blank lines. `interpolate` (`:336-373`) is strict: malformed, unknown, or valueless `{{name}}` throws; a lone `{{` without later `}}` is literal prose; substituted values are not rescanned. Variable names match `^[a-z][a-z0-9_]*$` (`:192`); registration at `variable()` (`:557-566`).

## Runtime context

Separate track, never part of `renderPrompt`. `CONTEXT_ORDERS` (`:170-175`): `100 working-directory`, `110 sandbox-policy`, `115 approval-policy`, `120 subagent-delegation`. `renderContextSections` (`:326-333`) plus `joinContextSections` (`:310-314`) produce one snapshot headed `Current runtime context. This snapshot supersedes earlier runtime-context snapshots.` `RuntimeContextProjection.project()` in `packages/core/agent-loop/src/runtime-context.ts:152-163` diffs against the retained snapshot and returns `undefined` when nothing changed; empty context commits the `CLEARED` marker (`:20`). Suppression via `suppressRuntimeContext()` (`system-prompt/src/index.ts:526-532`) or `Config.includeRuntimeContext: false`; `required: true` contexts survive suppression.

## Persona shadowing and override traps

Scoped same-name registration shadows the global persona (`packages/preset/persona/src/index.ts:62-74`); `complete: true` restores the prefix as the sole section; empty text drops at render. The traps that bite overriders — global config dead under presets, `suffix: ''` as active shadow, three-variable limit, wipe-keeps-schemas, waterfall-cannot-defeat-complete — are catalogued with recipes in `09-override-strategy.md`. Shipped preset contents and the structural findings are in `08-default-agents.md`.

## Workspace injections

`packages/context/*` plugins contribute `user/message` events through `agent/pre-step`, not sections. Example: `packages/context/agent-instructions/src/render.ts` bakes the full `<system-reminder>` frame into message content (`buildInstructionText`, `:304-316`), escapes literal closers (`escapeInstructionFrameBody`, `:87-89`), and budgets bytes broadest-first with binary-search truncation (`truncateToFit`, `:322-346`). Sources carry a semantic `form` (`instructions|catalog|snapshot|notice|relay|recall`, `packages/llm/llm/src/message.ts:55-67`) so consumers switch on meaning, never prose.

## Verify

Owned by `packages/core/system-prompt/src/index.ts` (`SECTION_ORDERS`, `CONTEXT_ORDERS`, `renderPrompt`, `renderContextSections`), `packages/core/agent-loop/src/runtime-context.ts` (`RuntimeContextProjection`), and `packages/context/agent-instructions/src/render.ts` (`SYSTEM_REMINDER_OPEN`). Re-check those files if the harness has moved.
