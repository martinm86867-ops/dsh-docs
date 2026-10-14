# 09 — Override strategy

Goal: replace the default prompt architecture's behavior with a deliberate one, at the cheapest layer that holds. Layers are ordered by cost; prefer the lowest that satisfies the goal. Full mechanism table: `07-hack-points.md`; default structure: `08-default-agents.md`; the *why* behind the traps (scope shadowing, `complete` semantics) is in `11-intelligence.md`.

## Override traps (read first)

1. Global `system-prompt` config does not reach web preset sessions: the preset `persona` row shadows `deployment:persona-prefix/suffix` by name (`packages/preset/persona/src/index.ts:62-73`). Edit the preset (profile patch `config.plugins` by id), not the global row.
2. `suffix: ''` is an active shadow, not an inherit (`persona/src/index.ts:69-73`). A global suffix is silently dead under every shipped preset.
3. Only `headless/sdk/acp` honor the global row (preset-free). One override never covers all profiles; patch each profile or own a preset.
4. `{{variables}}` are strict and only two exist: `provider, model` (`packages/core/agent-loop/src/index.ts:389-390`). Any other `{{name}}` fails the whole assembly at runtime. The working directory is no longer a prompt variable — it is a runtime-context contribution (`working-directory:current`).
5. `complete: true` wipes guidance prose but keeps wire schemas — the model sees tools it was never told how to use. Prefer restriction over wipe (trap: the `minimal` preset demonstrates the wipe). Note `complete` lives on the persona row, not on the global `system-prompt` registry config (that field was removed).
6. The `complete` restore runs after the `system-prompt/assemble` waterfall; waterfall listeners cannot defeat it, and two effective `complete` sections fail assembly (`packages/core/system-prompt/src/index.ts:649-683`).
7. Empty section text drops at render; shadowing with `''` removes. There is no placeholder trick.
8. Tool restriction removes that tool's guidance prose automatically (tools self-hide). Restricting tools is prompt reduction for free.

## Layer 1 — Config flags (no code)

On the `system-prompt` row (global, for preset-free profiles): `personaPrefix`, `personaSuffix`, `includeHarnessIdentity`, `includeRuntimeContext`, `toolOrder` (must contain `<unlisted-tools>`). On the preset `persona` row (web): `prefix`, `suffix`, `complete`, `includeRuntimeContext`. `surfaceContext: false` on the web-app row drops `harness:source` and `app:web-surface`. Lower `agent-instructions.maxBytes` (default 65536) before touching any prose — it is the largest prompt input.

## Layer 2 — Preset composition (recommended for behavior change)

Copy `standard.patch.yml` into a profile patch as a new `preset-<name>` row: own persona prefix/suffix, trimmed tool list (restriction also trims prose and schemas), explicit `toolOrder`, tightened `maxBytes`. The shipped header comment blesses this path: profile-patch `config.plugins` by id overrides the shipped row. Selectable per session, keeps upstream guidance improvements, survives rebase better than editing shipped files.

## Layer 3 — Waterfall plugin (programmatic control)

Listen `system-prompt/assemble`, call `next()`, transform the returned assembly (drop/reorder/rewrite sections, inject conditional guidance). Scope-filter to the target agents. Never fight `complete`; preserve `startsRequestSeries`-style declarations when wrapping sibling waterfalls (`agent/pre-step`). For input shaping rather than prompt shaping, prefer `agent/pre-step` (`enter`/`reject`).

## Layer 4 — Complete rewrite (last resort)

`complete: true` persona with a hand-authored full prompt. Total voice control; total maintenance burden; every upstream guidance fix must be re-ported manually. Justify it only when a single-voice prompt provably beats the composed stack on recorded snapshots.

## Composition principles (derived from the code)

- The persona prefix (order 0) carries identity and task directive; the suffix (order 10200) is the last word before nothing — response format and initiation rules belong there. Never put load-bearing rules in middle sections.
- One voice: align harness identity, prefix, and suffix. Do not restate the same first-token rule in multiple sections.
- Prose ceiling: sections stay 1-5 lines; anything budget-sensitive becomes a budgeted `user/message` injection in the `agent-instructions` pattern (byte budget, broadest-first eviction, truncation markers, semantic `source.form`).
- Enforce in the executor, suggest in the prompt: denials live in `tools/execute`, never as "do not use X" prose a direct caller can bypass.
- Pin model-visible text in snapshot tests (`pnpm run test:snapshot -t <name>`); update snapshots in the same change as the wording.

## Recipes

- Helpful-assistant web preset: Layer 2; prefix `You are a helpful software engineer assistant.` without `complete` (keeps tool guidance), `suffix: ''`, `includeRuntimeContext: true`.
- Strict output-enforcement preset: Layer 2; keep the harness identity, put the response-initiation rules into `suffix`, restrict tools that break the discipline.
- Token-lean preset: Layer 2; restrict to a shell/fs/search tool subset, `toolOrder` with the survivors first, `maxBytes: 8192`, `includeRuntimeContext: false`, `surfaceContext: false`.
- Conditional guidance: Layer 3; waterfall listener injecting sections only when the scope or assembled tools call for them.

## Verify

Run these from inside the harness checkout only — they inspect a live profile and the snapshot suite:

```sh
dsh --profile web --dump-config | rg -n "persona|system-prompt|tool-presentation"
pnpm run test:snapshot -t <name>
```
