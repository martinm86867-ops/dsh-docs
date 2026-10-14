# 10 — Injection plane: everything that becomes `user/message`

Three delivery paths put non-human content in front of the model, all landing as durable `user/message` events (verbatim projection, `05-output-structures.md`). Nothing on this plane can veto or rewrite calls — it only adds context.

## Path 1 — `pre-step` listeners (claim-time)

Listeners append to the entered batch for the step being claimed. All must delegate via `next()` and preserve downstream `messages`/`startsRequestSeries` unless intentionally replacing.

- Workspace instructions: `agent-instructions` folds the baseline after claimed messages on the first qualifying pre-step, then reconciles touches via projection (`02-prompt-assembly.md`).
- Skills: `tool-skill` scans only `source.kind === 'user'` messages, then injects `skill-catalog` and per-invocation `skill-invocation` (`form: 'instructions'`) messages (`packages/skill/tool-skill/src/index.ts:173-215`). External text is never scanned — only genuine user messages trigger loading.
- Goals: `goal-round-driver` enters round continuations as `source: { kind: 'goal', goalId, revision, round }` (`packages/goal/goal-round-driver/src/index.ts:178`), skipping round-0 seeding already in history (`:129`).
- Guard reset: `repeat-tool-reminder` deletes the agent's repeat chain when a genuine user message appears — repetition across an interjection is not a loop.

## Path 2 — `additionalContexts` (tool-result time)

`executeToolCalls` accepts each committed result's `additionalContexts` into the `next-step` inbox for the step boundary (`packages/core/agent-loop/src/tool-calls.ts:157-159`; driver in `agent.ts:553`). The producer is the tool pipeline, not the loop.

- Repeat guard: `repeat-tool-reminder` counts consecutive identical calls (canonical JSON args) in `tools/post-execute` — counting here deliberately includes denied calls, since hammering a denial is exactly the loop worth breaking — and folds a reminder onto `additionalContexts` of whatever decision comes back, including `block` (`packages/guard/repeat-tool-reminder/src/index.ts:190-231`). Thresholds default `[3, 5, 8]` with gentle-then-detailed escalation (`:70-86`); `form: 'notice'` summaries bound at `CONTEXT_SUMMARY_MAX_CHARS = 120` (`packages/llm/llm/src/message.ts:122`). Observe-and-enrich, never veto.
- Any tool may return `additionalContexts` the same way; the loop stages all of them identically.

## Path 3 — `agent.inject()` (out-of-band)

`inject(input)` sends to `next-step` without wakeup (`packages/core/agent-loop/src/agent.ts:171-173`): file-change notices, cron/schedule notifications, subagent relays wait for the next waking message. Contrast `steer` (wakeup) and `followup` (next turn + wakeup) in `01-input-and-turn.md`.

## Label discipline

Every injector stamps a semantic `source.kind` plus a `ContextForm` (`packages/llm/llm/src/message.ts:55-67`): `repeat-tool-reminder` uses `form: 'notice'` with a bounded `summary` (`repeat-tool-reminder/src/index.ts:212`; bound at `CONTEXT_SUMMARY_MAX_CHARS = 120`, `message.ts:122-133`), skills use `form: 'instructions'`, runtime context uses `form: 'snapshot'` with sections, the working directory uses a `snapshot` contribution too. An unlabeled context would render as a user prompt in derived history — the label is load-bearing. Display layers switch on these labels: Chat shows a notice row without expanding it, snapshots supersede, instructions render as guidance.

## Ordering note

Within one step, order is: claimed human messages first, then pre-step injections in listener order (agent-instructions baseline, skills, goals — registration order decides), with `additionalContexts` from the previous step's tools already in the queue ahead of them. Wrapping pre-step listeners prepend or append deliberately: the reminder guard prepends (`prependContext`, `repeat-tool-reminder/src/index.ts:154-156`) so the nudge reads before other context.

## Verify

Owned by `packages/core/agent-loop/src/tool-calls.ts` and `packages/guard/repeat-tool-reminder/src/index.ts` (`additionalContexts`), `packages/skill/tool-skill/src/index.ts` and `packages/goal/goal-round-driver/src/index.ts` (`agent/pre-step`), and `packages/llm/llm/src/message.ts` (`CONTEXT_SUMMARY_MAX_CHARS`). Re-check those files if the harness has moved.
