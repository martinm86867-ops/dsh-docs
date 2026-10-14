# DeepSeek Harness — Agent Development Playbook

A self-contained, hackathon-style playbook for developing tools, plugins, and integrations against the DeepSeek Harness. This folder is the single actionable source of truth: every page carries the copy-paste recipe first, and cites the harness source (`packages/…`, `docs/…`) only as verification pointers for when you are inside the checkout.

Drop this folder anywhere you develop against the harness — a scratch repo, a plugin package, a fork. Nothing here requires the harness tree to be readable; the recipes and contracts stand on their own.

## How to use this playbook

Find your task, read one page, write code.

| I want to… | Read |
|---|---|
| Understand the plugin model and build my first plugin | `00-plugin-authoring.md` |
| Add a model-callable tool | `00-plugin-authoring.md` → "Tool plugin", then `04-tools.md` |
| Add a model provider / LLM adapter | `00-plugin-authoring.md` → "LLM adapter" |
| Add prompt prose, a persona, or override the system prompt | `02-prompt-assembly.md`, `09-override-strategy.md` |
| Change how input reaches the model / add injected context | `01-input-and-turn.md`, `10-injections.md` |
| Route/retry model calls, intercept streaming | `03-request-and-stream.md` |
| Persist new state or project to the model surface | `05-output-structures.md` |
| Render output (Host card / Web / CLI / SDK / ACP) | `06-display.md` |
| Find the right extension point for any feature | `00-plugin-authoring.md` → "Feature → mechanism", `07-hack-points.md` |
| Know what the shipped presets already do | `08-default-agents.md` |
| **Reason about *why* it behaves this way, or debug a failure** | `11-intelligence.md` |

## The runtime in one picture

```text
input (followup/steer/inject/webhook/SDK/ACP)
 -> inbox (01) -> turn/start -> claim -> assemble prompt + context (02)
 -> agent/pre-step -> step/start -> agent/request + prepareRequest (03)
 -> commit system/message + user/message -> request/header + request/context
 -> deriveMessages() + freeze -> llm/stream -> agent/assistant-stream (transient)
 -> assistant/message | assistant/attempt (durable)
 -> tool/call -> tools/pre-execute -> tools/execute -> tools/post-execute -> tool/result (04)
 -> step/end -> agent/turn-stopping -> turn/end
 -> session/event -> Host presenter | Web ConversationNode | headless/SDK/ACP projection (06)
```

Composition precedes assembly: profile → bundles → (web only) agent preset → scoped layers → `assemble()`. Only `web` ships presets (`standard` default); `headless/sdk/acp/sdk-minimal` read the global `system-prompt` row directly. Preset `persona` rows shadow the global `deployment:persona-prefix/suffix` by name, so global persona config never reaches web sessions (`08-default-agents.md`, `09-override-strategy.md`).

## Pages

- `00-plugin-authoring.md` — **start here to build anything.** The plugin model, the five disciplines, the feature → mechanism map, and the four plugin shapes (tool, hook, UI, protocol driver) with copy-paste skeletons.
- `01-input-and-turn.md` — inbox targets, claim, turn/step loop, turn endings.
- `02-prompt-assembly.md` — section registry, ordering, rendering, runtime context, workspace injections.
- `03-request-and-stream.md` — request resolution, freezing, streaming, settling, retries.
- `04-tools.md` — schema wiring, execution scheduler, call/result commits.
- `05-output-structures.md` — session events, message types, surface projection.
- `06-display.md` — Host vs Web rendering, headless/SDK/ACP, titles, compaction, transcripts.
- `07-hack-points.md` — extension-point table: goal to exact API and patch sketch.
- `08-default-agents.md` — shipped presets, assembled default prompt, structural findings.
- `09-override-strategy.md` — override traps, layers, composition principles, recipes.
- `10-injections.md` — the injection plane: pre-step appends, tool-result contexts, out-of-band inject, label discipline.
- `11-intelligence.md` — the derived layer: the invariants (laws), mental models, and a symptom → cause → fix troubleshooting playbook. Read this before you debug.

## Conventions used throughout

- **Recipe first, reference second.** A page leads with the actionable content; a `file:line` or `docs/…` path is a verification pointer, never a required read.
- **`ctx.<service>`** names a capability on the shared plugin context (e.g. `ctx.tools`, `ctx.systemPrompt`, `ctx.llm`).
- **"Fails loud"** means a misconfiguration throws at load time rather than silently degrading.
- **"Model-visible"** means the bytes actually reach the model request; "log-only" means it is recorded but never sent.
