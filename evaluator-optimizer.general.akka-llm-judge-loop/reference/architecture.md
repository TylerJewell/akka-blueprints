# Architecture — llm-judge-loop

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`EvaluationEndpoint` is the entry point. It writes a `QuestionSubmitted` event to `QuestionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `EvaluationWorkflow` per submission. The workflow alternates two agents — `GeneratorAgent` produces an answer, `JudgeAgent` scores it — with a deterministic structural guardrail check between them. Each step emits an event on `EvaluationEntity`. `EvaluationsView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `QuestionSimulator` drips a sample question every 60 seconds so the UI is not empty when first loaded; `JudgeSampler` ticks every 30 seconds and records a `JudgmentRecorded` event for any judged attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first answer is sent back for revision and the second answer is accepted. The Generator → guardrail → Judge alternation is explicit. Each attempt's events are written to `EvaluationEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The evaluation moves between two transient states (`GENERATING`, `JUDGING`) and two terminal states (`ACCEPTED`, `REJECTED_FINAL`). The self-loop on `GENERATING` represents the guardrail-blocked path: an answer that fails the structural check sends the workflow back to `generateStep` with structured feedback. The `JUDGING → GENERATING` transition is the `REVISE` path — taken when the judge returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached when the ceiling is exhausted; both terminal states preserve every attempt and every judgment on the entity.

## Entity model

`EvaluationEntity` is the system's source of truth; every transition writes one of seven event types. `QuestionQueue` is the audit log of submissions. `EvaluationsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `generateStep` and `judgeStep`; the guardrail step is in-process and runs without an LLM call.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be layered on by converting `maxAttempts` into a duration limit; that is out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `EvaluationEndpoint.submit` deduplicates on `(questionText, submittedBy)` over a 10 s window. `JudgeSampler` deduplicates on `(evaluationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Evaluation.attempts` with its own answer, guardrail verdict, and (once produced) judgment. The `attemptNumber` is monotonic across the whole loop, including guardrail-blocked attempts that never reached the judge.
