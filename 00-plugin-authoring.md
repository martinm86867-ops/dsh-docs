# 00 — Plugin authoring

This is the page to start from. Everything you need to write a correct, elite plugin is inline here; the deep runtime mechanics are in `01`–`10`, the *why* behind the rules is in `11-intelligence.md`, and the harness's own guides (`docs/cordis-tutorial/`, `docs/cookbook/`) are verification pointers for when you are inside the checkout.

A plugin in this harness is a **Cordis plugin**: a named function `apply(ctx, config)` mounted from `cordis.yml`. Every capability — tools, LLM adapters, the agent loop itself — is a plugin wired into one shared `ctx`. There is no special-casing: learn the one model and everything else is the same shape.

## The five disciplines (read once, apply always)

### 1. Registrations are effects

Every registration — a tool, a section, a listener — goes through `ctx.effect()` / `ctx.on()`, and its return is a **disposer**. When the plugin unloads (or a dependency it `inject`-ed disappears), Cordis runs the disposers in reverse. A registration written outside an effect is the most common bug that leaks across HMR or teardown.

```ts
ctx.effect(() => ctx.tools.register({ /* … */ }))
ctx.on('agent/created', (agent) => { /* … */ })
```

### 2. A capability is a three-role seam — declare, provide, consume

Never one role. A consumer names the service, so config can swap providers without touching consumers.

```ts
// Declare (type only) + provide (runtime), in the owning package:
declare module '@deepseek-ai/cordis' {
  interface Context { greeter: GreeterService }
}
export class GreeterService extends Service {
  constructor(ctx: Context) { super(ctx, 'greeter') }
}
ctx.plugin(GreeterService)

// Consume, in any other package:
export const inject = ['greeter']
// inside apply: ctx.greeter is guaranteed ready
```

`inject` is a hard requirement — the consumer stays PENDING until the service exists, and unloads if it disappears. For an optional capability, skip `inject` and probe at the use site (`ctx.get('name')` → `undefined`).

### 3. Events: the dispatch mode is the contract

| Mode | Call | Semantics |
|---|---|---|
| `emit` | `ctx.emit(name, ...args)` | Fire-and-forget broadcast; returns ignored |
| `serial` | `await ctx.serial(...)` | In order; first non-nullish return wins and stops the chain |
| `waterfall` | `ctx.waterfall(name, ...args, next)` | Around-middleware; transform `next()`'s result or short-circuit |

**A waterfall listener that only observes must call `next()`**; returning without it is a deliberate veto that silently swallows the default for everyone downstream. Typed events use declaration merging (`interface Events { … }`) and carry `@mode` + `@param` JSDoc so the mode is self-documenting.

### 4. Config is declared, validated, fails loud

Deployment-varying choices are a validated `Config`, never a hardcoded constant. Schemas use **Schemastery** (`@deepseek-ai/schemastery`), not zod — the exported `Config` is both the TypeScript type and the runtime validator:

```ts
import Schema from '@deepseek-ai/schemastery'
export interface Config { greeting: string }
export const Config: Schema<Config> = Schema.object({
  greeting: Schema.string().default('Hello'),
})
export function apply(ctx: Context, config: Config) { /* … */ }
```

Bad config fails the load with a precise error — the plugin never starts half-configured. Values are detached snapshots; for a live re-readable field use `Schema.string().volatile()` and read through `.get()`.

### 5. Secrets are config, not code

Secrets come from `cordis.yml` via `!!js process.env.MY_KEY` (never `!js`), with Schemastery env fallbacks. Never read ad-hoc key files in code.

## The four plugin shapes

### Tool plugin

Registers a model-callable tool on `ctx.tools`. `defineTool` is the typed helper; raw JSON-Schema `ToolDefinition`s (how MCP tools arrive) are also accepted by `ctx.tools.register()`.

```ts
import { readFile } from 'node:fs/promises'
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'my-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'read_file',
    description: 'Read a file from disk.',          // what the model sees
    parameters: {
      path: { type: 'string', required: true, description: 'Absolute path' },
      limit: { type: 'number' },                     // optional by default
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args, exec) {
      // args is TYPED from the schema: { path: string; limit?: number }
      // exec carries immutable identity + token; exec.signal is the operational field
      return readFile(args.path, { encoding: 'utf8', signal: exec.signal })
    },
  }))
}
```

The `execute()` contract, condensed (full detail in the harness `docs/cookbook/adding-a-tool.md`):

- **Args are validated for you.** `defineTool` validates model `arguments` against `parameters` before `execute` runs; inside, `args` matches `InferArgs`. Hand-check constraints the DSL can't express (non-empty, positive, cross-field).
- **Return one canonical JSON value**, validated by `output.schema`; `output.render(args, value)` turns it into model-facing content. Throwing or returning an invalid value means `isError`.
- **Honor `exec.signal`** — cancel in-flight work when it fires.
- **`exec.agent.inject(...)`** appends durable context the NEXT request sees; it is not a wake-up (an idle agent stays idle).
- **Long-running work**: register through `ctx.jobs.start({ kind, label, owner: exec.agent, run })`; a successful background branch returns a typed handle like `{ kind: 'background', jobId }` — never make PTC parse prose to recover the id.
- **UI cards** are a separate concern: `presentCall`/`presentResult` return `card`-tagged intents (`generic|terminal|diff|search|read|web`). They run on live streaming AND replay, so they must be **pure** (no I/O, no session state, no clock/random). Web does not consume them — it derives cards from raw events + `result.meta` (`06-display.md`).
- **Policy stays out of the tool.** Use `tools/pre-execute` (allow/deny/ask), `ctx.tools.guard()` (monotonic final deny), `tools/execute` (wrap dispatch: deadline/retry/metrics), `tools/post-execute` (transform result / attach context), `tools/result` (observe immutable outcome). See `04-tools.md`.

### Hook plugin (permission gate example)

Intercepts an extension point and returns a typed decision. `tools/pre-execute` is the reorderable policy layer; sandbox, permission, and plan-mode plugins all use it.

```ts
import type { Context } from '@deepseek-ai/cordis'
import type { PreToolDecision, ToolExecution } from '@deepseek-ai/dsh-tools'

declare function isAllowed(exec: ToolExecution): Promise<boolean>

export const name = 'permission-gate'

export function apply(ctx: Context) {
  ctx.on('tools/pre-execute', async (exec, next): Promise<PreToolDecision> => {
    if (!(await isAllowed(exec))) {
      return { kind: 'deny', reason: 'Denied by policy.' }
    }
    return next()
  })
}
```

### UI plugin

Combines durable `session/event` records (assistant settlements, turn/step boundaries, tool activity) with transient `agent/assistant-stream` frames for live tokens, and drives input back in via `agent.followup()` / `agent.steer()`. A Web Client business row instead registers a `ConversationNodeDefinition` + keyed Chat renderer (`06-display.md`).

```ts
import type { Context } from '@deepseek-ai/cordis'
import { createUserMessage } from '@deepseek-ai/dsh-llm'

export const name = 'my-ui'
export const inject = ['agents']

export function apply(ctx: Context) {
  ctx.on('agent/assistant-stream', ({ frame }) => {
    if (frame.type === 'chunk' && frame.chunk.type === 'text-delta') {
      render(frame.chunk.text)
    }
  })
  onUserInput(text => ctx.agents.get(sessionId)?.followup(createUserMessage({
    content: [{ type: 'text', text }],
    source: { kind: 'user' },
  })))
}
```

### Protocol driver (external wire peer)

Adapts a wire peer to `ctx.agents`. Maps protocol requests to `followup()`/`cancel()`; publishes whole-agent status separately; tears down via `AgentHandle.dispose()`. `packages/acp/acp` is the worked example.

### LLM adapter (model provider)

```ts
class MyAdapter extends LlmAdapter {
  async * stream(options: GenerateOptions): AsyncIterable<StreamChunk> { … }
}
export const name = 'llm-myprovider'
export const inject = ['llm']
export function apply(ctx: Context, config: Config) {
  ctx.llm.registerAdapter(['my-provider'], new MyAdapter(…))
}
```

Protocol obligations (condensed; full detail in the harness `docs/cookbook/adding-an-llm-adapter.md`): emit `usage` before `finish`, nothing after `finish`; tool `arguments` are raw JSON strings end-to-end (`argumentsDelta` fragments, re-stringify at `block-end`); allocate block `index`es in first-seen order; two sanctioned error paths — throw `LlmError` (transport/protocol) or end the stream with `finish {kind:'error'|'aborted'}` (provider in-band); honor `options.signal`; throw `LlmError(…,'UNSUPPORTED_OPTION')` for options you can't honor; emit `finish.replayState` for required native follow-up metadata.

## Feature → mechanism map

"How do I build X" → the exact extension point. (Full table + worked snippets live in the harness `docs/cookbook/extension-cookbook.md`; this is the portable subset.)

| Feature | Mechanism |
|---|---|
| Hook system (user + project) | listeners on `agent/created`, `agent/pre-step`, `agent/request`, `tools/pre-execute`, `tools/post-execute`, `agent/turn-stopping` |
| Goals (`/goal`) | `ctx.goals` (durable) + `goal-round-driver` (same-session rounds) + command/tool producers |
| Loop (`/loop`) | on `turn/end`, `followup()` the next iteration |
| Dynamic workflow | `ctx.workflowEngine` + PTC engine + the `workflow` tool |
| Queued + steering messages | `Agent.followup()` / `Agent.steer()` |
| Compaction (auto + manual) | `ctx.compaction` seam + `compaction-basic`; pressure on serial `agent/pre-step`, recovery on `agent/request-error` |
| System-prompt configurability | `ctx.systemPrompt.section()` with ordering + scope shadowing |
| AGENTS.md / file-change notices | a section provider (root) / `agent.inject()` from a watcher (subdir) |
| Built-in tools | `ctx.tools.register()`; schemas flow into assembly automatically |
| Progressive disclosure / tool search | replace a scoped `ctx.tools.restrict()` registration |
| Tool deadline / retry / metrics | wrap dispatch with `tools/execute` (replace `exec.signal`, delegate) |
| Final result metrics / audit | observe with `tools/result` (or `tools/post-execute` to transform) |
| Monotonic terminal-turn policy | `ToolExecution.concludeTurn()` from the terminal tool |
| Sandbox | `ctx.sandbox` backend + `tools/pre-execute` capability denial |
| Permission / ask-user | return `ask` from `tools/pre-execute`; answer through `ctx.approval` |
| Plan mode | `plan-mode` — `plan:policy` section, `/plan`, `exit_plan_mode` |
| Sub-agent delegation | `ctx.subagents` provider registry + `tool-subagent` |
| MCP | one plugin per server: discover tools → `ctx.tools.register()` |
| Skills | section + tool registration; `inject()` content on invocation |
| Scheduled tasks (cron) | register scheduling tools; deliver via `followup(…, {source:{kind:'schedule'}})` |
| UI | `agent/assistant-stream` (live) + `session/event` (durable); input → `followup()` |
| Web Client node | `ConversationNodeDefinition` + `conversation.chat.node` renderer |
| Telemetry / replayable trace | `session/event` → JSONL; replay = `sessions.create(id, { seed })` |
| Model adapters | `LlmAdapter` subclass via `registerAdapter` |
| Hot-reload | every registration is a `ctx.effect` → HMR just works |

## Two questions to answer before code

1. **Which roles do I play?** Declaring a new service (`declare module` + `Service` + `ctx.plugin`), providing an existing one (`inject` + implement), or consuming it (`inject`/`ctx.get`)?
2. **Which extension point, at which dispatch mode?** A decision (allow/deny, choose a route) is a `waterfall`; a monotonic terminal policy is a guard; an observation that must not interfere is an `emit` listener that never short-circuits. Getting the mode wrong is how a "safe" listener breaks the loop.

## Verification pointers (inside the harness checkout)

These are authoritative in the harness repo; read them for depth and to re-check any contract that has drifted — but none is required to use this playbook standalone.

- `docs/cordis-tutorial/` — seven runnable chapters (function → effects → services → events → config → HMR → a real tool).
- `docs/cordis-api/` — the exhaustive `Context`/`Events`/`Service`/`Registry`/`Fiber`/`Inherited` API.
- `docs/user/develop/basic/` — first Harness plugin, tool, config, publish.
- `docs/cookbook/extension-cookbook.md` — full feature → mechanism table + the four plugin shapes.
- `docs/capability-seams.md` — the complete `ctx.<service>` catalog.
- `docs/cookbook/adding-a-tool.md`, `adding-an-llm-adapter.md`, `adding-a-package.md` — ship-this checklists.
- `docs/cordis-primer.md` — condensed concept reference.
