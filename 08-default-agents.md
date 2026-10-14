# 08 — Default agents: prompt structure

Shipped presets live only in `web` (`packages/bundle/web-app/presets/*.patch.yml`, order in `packages/bundle/web-app/package.json:44-50`): `standard`, `ptc`, `minimal`, `cordis`. Registry default is `standard`. `headless`, `sdk`, `acp`, and `sdk-minimal` are preset-free and read the global `system-prompt` row directly; `sdk-minimal` owns a standalone tree (`README.md` scope note; details in `07-hack-points.md`).

## The assembled default (standard)

Render order is `renderPrompt` over sorted sections (`packages/core/system-prompt/src/index.ts:286-291`):

1. `harness:identity` (-1000): the one-liner `"You are an AI agent powered by DeepSeek Harness."`, registered by the registry constructor when `includeHarnessIdentity` (default true) (`system-prompt/src/index.ts:445-451`).
2. `deployment:persona-prefix` (0): standard prefix (`standard.patch.yml:13-14`) — `You are a coding agent powered by the {{model}} model.` No `suffix` is set in the preset, so `deployment:persona-suffix` (10200) renders empty.
3. `plan:policy` (500): empty unless plan mode is active; the plan-mode section text lives in the preset's `plan-mode` row (`standard.patch.yml:54-65`), not in `packages/plan/plan-mode/src/index.ts`.
4. `team:policy` (600): only in team-member scopes (`packages/experimental/tool-agent-team/src/index.ts:169-170`).
5. `TOOL_*` guidance (1000-2900): one short section per tool, several self-hiding when their tool is restricted — e.g. bash, read, web-search guidance.
6. `tools:sdk` (5000): empty in `native` mode; generated SDK in `ptc`/`both` (`packages/core/tools/src/index.ts:897-915`).
7. `harness:source` (10000) + `app:web-surface` (10100): web-only, gated by `surfaceContext` (default true, `packages/bundle/web-app/src/index.ts:248-259`).
8. `deployment:persona-suffix` (10200): `''` in standard (empty section drops at render).

Separately, the runtime-context snapshot (`working-directory` 100, `sandbox` 110, `approval` 115, `subagent-delegation` 120) commits as a `user/message` headed "This snapshot supersedes earlier…", and `agent-instructions` injects up to 64 KB of `<system-reminder>` workspace guidance (`standard.patch.yml:15-18`). Tool schemas travel on the wire (`header.tools`), not as prose.

## Preset personas

- `standard` and `cordis`: `You are a coding agent powered by the {{model}} model.` — cordis deliberately reuses the standard persona; its operating detail lives in tool descriptions and the skill catalog (`cordis.patch.yml:11-17`).
- `ptc`: same coding-agent prefix, presented through the PTC `run_code` transport.
- `minimal`: `You are a helpful software engineer assistant.` with `complete: true` and `includeRuntimeContext: false` — a demonstration of the full-prompt wipe (`minimal.patch.yml:11-16`).
- `sdk-minimal`: `personaPrefix: … 'You are a helpful software engineer assistant.'` with `includeHarnessIdentity: false` and `includeRuntimeContext: false` (`packages/bundle/sdk-minimal/cordis.patch.yml:93-98`).

## Structural findings

- The harness identity is a single attribution line, not a behavioral directive: every agent (subagents included) carries `"You are an AI agent powered by DeepSeek Harness."`, and a deployment can drop it via `includeHarnessIdentity: false` or remove the section in the `system-prompt/assemble` waterfall.
- The working directory is a runtime-context contribution (`working-directory:current`, `interpolate: false`), not a `{{cwd}}` prompt variable. Only two prompt variables exist today: `provider` and `model` (`packages/core/agent-loop/src/index.ts:389-390`).
- Persona mechanics (`packages/preset/persona/src/index.ts:62-74`): scoped same-name registration shadows the global persona; `suffix` omitted means `''` (inherits nothing); `complete: true` restores the prefix as the sole section (wire schemas survive, guidance prose does not); `includeRuntimeContext: false` suppresses snapshots. Scope-only by design — mounted globally it collides and fails loud.
- The prompt bill is dominated by `agent-instructions` (64 KB budget), tool schemas, runtime snapshots, and compaction checkpoints; the fixed sections are only a few KB. Prompt-trimming that ignores the injections trims the wrong thing.
- Special agents reuse the base: child agents join the parent preset, add a delegation *context* (not section), optionally shadow the prefix, restrict tools, pin approval `never` (`packages/subagent/subagent/src/child-agent.ts`); structured-output children append a terminal tool plus an order-9900 mandate (`packages/subagent/subagent/src/structured.ts`); plan mode contributes one conditional section; compaction and title calls are one-shot auxiliaries, not agents.

## Verify

Owned by `packages/bundle/web-app/presets/*.patch.yml` (`prefix:`, `suffix:`, `complete:`, `includeRuntimeContext`), `packages/preset/persona/src/index.ts` (`PERSONA_PREFIX_SECTION`, `complete`), and `packages/core/system-prompt/src/index.ts` (`harness:identity`, `personaPrefix`, `personaSuffix`). Re-check those files if the harness has moved.
