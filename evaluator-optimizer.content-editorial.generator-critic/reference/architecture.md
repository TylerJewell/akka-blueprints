# Architecture — generator-critic

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`EssayEndpoint` is the entry point. It writes a `TopicSubmitted` event to `SubmissionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ReflectionWorkflow` per submission. The workflow alternates two agents — `GeneratorAgent` produces a draft, `ReflectorAgent` scores it — until the reflector accepts or the round ceiling is hit. On acceptance, a deterministic policy guardrail vets the draft before it is released. Each step emits an event on `EssayEntity`. `EssaysView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `TopicSimulator` drips a sample topic every 60 seconds so the UI is never empty on first load; `EvalSampler` ticks every 30 seconds and records a `ReflectionRecorded` event for any reflected round that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence: the first draft receives a `REVISE` verdict, the generator revises, the second draft receives `ACCEPT`, the guardrail passes, and the essay is released. The generate → reflect → guardrail path is made explicit. Each round's events are written to `EssayEntity` before the next step begins, so the UI's per-round timeline reconstructs the loop exactly.

## State machine

The essay moves through two inner states (`DRAFTING`, `REFLECTING`) before reaching `ACCEPTED`, then transitions to `RELEASED` after the guardrail passes. `REJECTED_FINAL` is reached either when the round ceiling is hit with the last verdict being `REVISE`, or when an irrecoverable agent failure triggers the `defaultStepRecovery` failover. The self-loop between `ACCEPTED` and `DRAFTING` represents the policy-revise path: a guardrail failure sends the essay back to the Generator for a targeted policy edit without re-entering the reflection loop. Both terminal states (`RELEASED`, `REJECTED_FINAL`) preserve every round's draft and every reflection on the entity.

## Entity model

`EssayEntity` is the system's source of truth; every transition writes one of eight event types. `SubmissionQueue` is the audit log of submissions. `EssaysView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `draftStep` and `reflectStep`; the guardrail step is in-process and effectively instant.
- Round ceiling: the workflow reads `maxRounds` (default 4) at start and checks the count before scheduling the next `draftStep`. The loop never exceeds the ceiling.
- Policy-revise loop: bounded by `maxRetries(2)` on `policyReviseStep`. A second consecutive guardrail failure fails over to `rejectStep`, so the policy-revise path cannot loop indefinitely.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` on all agent-calling steps — any unrecoverable failure ends in `REJECTED_FINAL`, not a hung workflow.
- Idempotency: `EssayEndpoint.submit` deduplicates on `(topic, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(essayId, roundNumber)` so a tick that fires twice for the same round is a no-op.
- Per-round accounting: every round is appended to `Essay.rounds` with its own draft, guardrail verdict, and (once produced) reflection. The `roundNumber` is monotonic and includes policy-revise rounds in the guardrail pass but the round counter only increments on the reflect-revise loop.
