# 11 — Intelligence: invariants and troubleshooting

This page is the layer that isn't in any one source file: the invariants, mental models, and symptom→cause knowledge you would otherwise re-derive by reading the code. It is what makes the rest of the playbook *usable* for developing and, especially, troubleshooting, without opening the harness source.

Every "law" below is enforced by the source, not advisory. Where a law points at a file, that path is verification, not required reading.

## The invariants (the laws)

These hold across the whole system. When behavior surprises you, it is almost always one of these asserting itself.

### Law 1 — The session log is the only source of truth; everything else is derived

`Session` is an append-only, contiguous, lossless-JSON event log (`SessionEventMap`). The model-visible conversation is **not stored** — it is *derived* from the log by folding surface operations (`foldSurface` → `deriveMessages`). There is no "conversation object" or "prompt string" persisted anywhere.

- **Implication for troubleshooting:** if the model saw something, an event produced it. If you can't find the event, it never reached the model. Trace events, not state.

### Law 2 — Model-visible ⟺ logged

Anything that reaches a model request must be reconstructable from the log. A new model-visible input requires a new session event. This is why injections are `user/message` events, why the system prompt is a `system/message` surface node, and why `EpochHeader.system` is forbidden (`system?: never`, verified by `validateSessionEventData` rejecting `header.system`).

- **Implication:** to add model-visible content, you add an event — you never mutate a stored prompt. `00-plugin-authoring.md` → injection paths are `01`/`10`.

### Law 3 — The system prompt is derived history, never a header field

The rendered prompt travels as `system/message` surface node 0. There is no `header.system`; a loop-built request leaves `GenerateOptions.system` undefined. On resume, fork, or compaction, the prompt is **re-derived** from whatever surface survives.

- **Implication:** "why is my prompt different after resume/compaction?" is answered by "what surface nodes survived," not by any stored prompt. See `02` (assembly) and `05` (surface).

### Law 4 — Scope shadows by name; nearest wins

Scoped registrations shadow globals by the same name. Preset `persona` rows shadow the global `deployment:persona-prefix/suffix`, so **global `system-prompt` config is dead under every web preset**. This single rule explains the most common override bug.

- **Implication:** before assuming your config didn't load, check whether a nearer scope shadowed it (`09-override-strategy.md`).

### Law 5 — Every registration is an effect; a waterfall listener that doesn't call `next()` is a veto

Registrations return disposers and unwind in reverse on unload/dependency-loss (this is *why* the harness is HMR-safe). A waterfall listener that returns without `next()` deliberately short-circuits the chain — including a "logging" listener that forgot to delegate.

- **Implication:** the two silent-killers are (a) registering outside an effect, and (b) a waterfall observer that drops `next()`.

### Law 6 — The log is append-only; change is expressed as surface *replacement*, not mutation

Nothing edits a committed event. Compaction and prompt rewrites *replace a surface range* (append a new node that shadows old seqs) while the raw log keeps the shadowed events. Human transcripts read **append-origin** events, not the model surface, precisely because the surface shadows replaced ranges.

- **Implication:** "my message disappeared from the transcript but the model changed" means a replacement shadowed it; the raw log still has it.

### Law 7 — Denials live in the executor, never in the prompt

Prompt prose ("do not use X") is a suggestion a direct caller can bypass. Enforcement lives in `tools/execute` / `ctx.tools.guard()` / `tools/pre-execute`. The `tools:ptc-only` collapse section reuses the *same predicate* the executor denies by, so the prompt can never state a rule the registry doesn't enforce.

## The mental models (how to reason about behavior)

These are the shapes to hold in your head; they turn "read 10 files" into "one model."

### Time: turn → step → request-attempt → tool-call

A turn owns steps; a step owns one model call plus the tool executions it requested, and retries are *attempts within a step* (`assistant/attempt`, not new steps). Boundaries (`turn/start|end`, `step/start|end`) are log-only. `max-tokens` is **sticky**: once any step hits the ceiling, a later `completed` step must not downgrade the turn. `agent/turn-stopping` is a serial (no-`next()`) hook that runs after the turn ends but before it closes.

### Two tracks: prompt prose vs runtime context vs wire schemas

- **Prompt prose** (`sections`) renders into the `system/message` node.
- **Runtime context** (`contexts`) is a separate `user/message` snapshot headed "This snapshot supersedes earlier…" — it diffs and replaces, never merges.
- **Wire schemas** (`tools`) travel as `header.tools`, independent of prose. `complete: true` keeps schemas but drops guidance prose — the model sees tools it was never told how to use.

### Two values per tool: canonical vs rendered

A tool returns one **canonical JSON value** (validated by `output.schema`); `output.render` projects it to model-facing content. PTC mode sees the canonical value; native mode sees the rendered content. UI cards are a third projection (`presentCall`/`presentResult`), pure and replay-safe. The three never leak into each other.

### Injection is context, not control

The injection plane (`10`) adds `user/message` context on three paths (pre-step listeners, `additionalContexts`, `agent.inject`). **None can veto or rewrite a call, and `inject()` does not wake an idle agent** — it waits for the next waking message. Labels (`source.kind` + `form`) are load-bearing: an unlabeled context would render as an ordinary user prompt.

### The seam: declare / provide / consume

A capability is three roles, never one (`00`). A consumer names the service (`inject`), so config swaps providers without touching consumers. This is why the whole app is a hot-swappable composition rather than a static dependency graph.

## Troubleshooting playbook (symptom → cause → fix)

| Symptom | Cause (law/model) | Fix |
|---|---|---|
| Prompt override didn't take effect | Global `system-prompt` config is shadowed by the preset `persona` row (Law 4) | Patch the preset's `persona` row, or own a preset (`09`) |
| Model sees tools with no guidance prose | `complete: true` keeps wire schemas, drops prose | Prefer tool restriction over `complete` (`09` trap 5) |
| Injected context never shows up | `agent.inject()` has no wakeup; idle agent stays idle (injection model) | Use `followup`/`steer`, or ensure a waking message |
| A listener silently broke downstream behavior | Waterfall listener forgot `next()` (Law 5) | Call `next()` unless deliberately vetoing |
| Prompt differs after resume/compaction | Prompt is re-derived from surviving surface nodes (Law 3) | Inspect surface nodes / replacement history, not a stored prompt |
| A failed/retried call "polluted" the transcript | Failures commit to `assistant/attempt`, not the surface — this is correct | If it *is* surface, it's `assistant/message` with `interrupted` |
| Turn ended `completed` after an earlier max-tokens | `max-tokens` is sticky (time model) | Correct behavior; check the step that hit the ceiling |
| `{{cwd}}` / third variable fails assembly | Only `provider` and `model` variables exist; cwd is a runtime context | Reference `provider`/`model` only; use context for cwd (`02`) |
| Tool's model text looks wrong / UI leaks into model | Canonical value vs rendered content vs card are separate | Keep UI formatting out of `output.render` and the canonical value |
| Registration doesn't unload / leaks on HMR | Registered outside an effect (Law 5) | Wrap in `ctx.effect()` |
| "Do not use X" prompt ignored | Denials must be enforced in the executor, not prose (Law 7) | Move enforcement to `tools/execute` / `guard()` |
| Message gone from transcript but model changed | A surface replacement shadowed it (Law 6) | Read append-origin events; the raw log retains it |

## Where the laws point (verification, inside the harness)

- Law 1–3: `packages/core/session/src/{types,surface,index}.ts` (`SessionEventMap`, `foldSurface`, `deriveMessages`).
- Law 4: `packages/preset/persona/src/index.ts`, `packages/core/system-prompt/src/index.ts`.
- Law 5–7: `packages/core/agent-loop/src/agent.ts`, `packages/core/tools/src/index.ts`.
- Time model: `packages/core/agent-loop/src/agent.ts` (`turn`, `step`, `prepareRequest`, `buildRequest`).
