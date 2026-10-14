# 07 — Hack points

Rule of thumb: new behavior attaches to a documented extension point; changing the loop itself is a last resort. Each row gives the API (actionable) plus the owning source (reference — the `file:line` drifts, re-check it when inside the checkout). This table is the prompt-and-tool slice of the authoring surface; the full feature → mechanism map and the four plugin shapes are inlined in `00-plugin-authoring.md`.

| Goal | API | Anchor |
|---|---|---|
| Add prompt prose | `ctx.systemPrompt.section({ name, order, text })` | `packages/core/system-prompt/src/index.ts:474-483`; order via `getSectionOrder` (`:490-492`) |
| Override a web session's persona | profile-patch `config.plugins` entry for the preset's `persona` row (`prefix/suffix/complete/includeRuntimeContext`) — global `system-prompt` config never reaches preset sessions | `packages/bundle/web-app/presets/standard.patch.yml:11-14`; row mechanics `packages/preset/persona/src/index.ts:62-74`; traps in `09-override-strategy.md` |
| New preset composition | copy `standard.patch.yml` to a profile patch as `preset-<name>`: own persona prefix/suffix, trimmed tools (restriction also trims prose and schemas), `toolOrder`, tightened `agent-instructions.maxBytes` | `packages/bundle/web-app/package.json:44-50` patch order; `agent-instructions.maxBytes` default 65536 (`standard.patch.yml:17-18`) |
| Drop web-only sections | `surfaceContext: false` on the web-app row removes `harness:source` and `app:web-surface` | `packages/bundle/web-app/src/index.ts:248-259` |
| Add dynamic context snapshot | `ctx.systemPrompt.context({ name, order, text })` | `packages/core/system-prompt/src/index.ts:509-518` |
| Per-agent prompt override | register through `agent.ctx`; same name shadows global | `system-prompt/src/index.ts:404-411` |
| One-shot full-prompt takeover | `complete: true` section (sole survivor, waterfall still runs) | `system-prompt/src/index.ts:69-76,649-683` |
| Prompt variable | `ctx.systemPrompt.variable(name, provider)` | `system-prompt/src/index.ts:557-566` |
| Rewrite waterfall assembly | `system-prompt/assemble` listener, return authoritative value | `system-prompt/src/index.ts:18-31,673-683` |
| Suppress runtime context | `ctx.systemPrompt.suppressRuntimeContext()` | `system-prompt/src/index.ts:526-532` |
| Add model-facing capability | register on `ctx.tools`; schema joins automatically | `packages/core/tools/src/index.ts:855`; cookbook `docs/cookbook/adding-a-tool.md` |
| PTC-only presentation | `modeFor(scope)` + `tools:ptc-only` / `tools:sdk` sections | `packages/core/tools/src/index.ts:877-915,923-934` |
| Rewrite/redirect user input | `agent/pre-step` waterfall: `enter{messages}` or `reject` | `packages/core/agent-loop/src/agent.ts:274-284` |
| Swap model route | `agent/request` waterfall returning `LlmCallConfig` | `packages/core/agent-loop/src/agent.ts:593-596` |
| Serve/short-circuit a call | `llm/stream` waterfall or `prepareCall` result | `packages/llm/llm/src/index.ts:75,933` |
| Retry policy on failure | `agent/request-error` waterfall returning `{ kind: 'retry' }` | `packages/core/agent-loop/src/agent.ts:511-526` |
| Stop a turn | `agent/turn-stopping` serial listener with inbox state | `packages/core/agent-loop/src/agent.ts:361-364` |
| Add model provider | register adapter on `ctx.llm` | cookbook `docs/cookbook/adding-an-llm-adapter.md` |
| Enforce tool denial | in the executor (`tools/execute`), not schema omission or prompt filtering | `packages/AGENTS.md` enforcement rule; scheduler `packages/core/agent-loop/src/tool-calls.ts:122-247` |
| Host-only tool card | `presentCall/presentResult` + `meta` | `packages/core/tools/src/presentation.ts` |
| Web card for a tool or event | `uiConversation.events.register(definition)` + `views.register` + `slots.inject` | `docs/subsystems/conversation.md`; `packages/client/ui-tool/README.md:36-41` |
| New durable state | extend `SessionEventMap`; model-visible input requires an event | `packages/core/session/src/types.ts:281`; invariant `docs/architecture.md` |
| New message projection | `SessionMessageProjection` for an owned event type | `packages/core/session/src/surface.ts:34-47` |
| Inspect live composition | `dsh --profile web --dump-config`; patch any row | `docs/architecture.md` |

Patch mechanics: layers apply as bundles in profile order, then profile `cordis.patch.yml`, then home-level, then `--patch` overlays; a patch targets a row by id and replaces its whole config. `sdk-minimal` is the exception with its own complete tree. Only `dsh` profiles launch supported Node apps.
