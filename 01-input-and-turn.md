# 01 — Input and turn loop

Owner: `packages/core/agent-loop/README.md`; lifecycle diagram `docs/agent-lifecycle.md`; extension events `docs/subsystems/core.md`.

## Inbox

`ReactLoopInbox` in `packages/core/agent-loop/src/inbox.ts`. One inbox per agent, backed by the durable `inbox` projection (`inboxProjectionDefinition`, `inbox.ts:27-65`). Every mutation records an `agent/inbox/spliced` event and publishes live `agent/inbox/{inserted,discarded}` notices. `claim(target, turn)` at `inbox.ts:109-114` drains `next-step`, then one queued `next-turn` message when `target === 'next-turn'`, emitting `agent/inbox/claimed` per message. `send(message, target, wakeup)` in `packages/core/agent-loop/src/agent.ts:154-161` splices, then optionally wakes the driver. Three entry verbs (`agent.ts:163-173`): `followup` goes to `next-turn` with wakeup, `steer` to `next-step` with wakeup, `inject` to `next-step` without wakeup. Injected context therefore waits for a waking message; one inbox feeds the driver (`docs/architecture.md`).

## Turn loop

`turn()` in `packages/core/agent-loop/src/agent.ts:298-398`. Opens with `session.append('turn/start', { turn })` (`:307`) before claiming first input. Each iteration proposes one step via `preStep(target, { turn, step })` (`:318`), commits `step/start` (`:331`), runs `step(decision)` (`:340`), commits `step/end` in a `finally` (`:358`). After a step that ends the turn and an empty `next-step` queue, the loop offers `agent/turn-stopping` (serial, no `next()`) so plugins can steer before close (`:361-364`). Closes with `session.append('turn/end', { turn, reason })` (`:387`).

## Turn endings

`reject` from `agent/pre-step` closes with `blocked` and no step (`:319-321`). An empty first claim (or an enter rewritten empty) closes with `completed` and no model call (`:326-328`). `max-tokens` is sticky: once any step hits the ceiling, later `completed` steps must not downgrade it (`:338-343`). Cancellation maps the abort cause to `aborted` (`:369-373`); other failures flatten to `error` with `LlmError` facts verbatim or `{ message: errorChain(error), code: 'UNKNOWN' }` (`:375-382`). `TurnEndReason` union lives in `packages/core/session/src/types.ts:201-232`, including `interrupted` and `forked` markers the loop never emits live.

## Verify

Owned by `packages/core/agent-loop/src/inbox.ts` (`claim`), `agent.ts` (`turn/start|turn/end|turn-stopping`), and `packages/core/session/src/types.ts` (`TurnEndReasonMap`). If the harness has moved since this was written, re-check those files for the same symbols.
