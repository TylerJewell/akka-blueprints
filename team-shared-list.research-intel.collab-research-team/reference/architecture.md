# Architecture — collab-research-team

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ResearchEndpoint` is the entry point. A submitted question is logged as a `QuestionSubmitted` event on `IntakeQueue` (event-sourced for audit). `QuestionRequestConsumer` subscribes to that queue, creates a `QuestionEntity`, and starts a `PlanningWorkflow`. The workflow runs the `ResearchCoordinator` agent to decompose the question into sub-topics, then writes one `ResearchTaskEntity` per sub-topic onto the board (each `OPEN`). Every task transition projects into `TaskBoardView` — the shared list.

The researcher side runs in parallel and independently. `Bootstrap` starts one `ResearcherWorkflow` per researcher id (`researcher-1`, `researcher-2`, `researcher-3`). Each researcher loop polls `TaskBoardView`, claims an eligible task on its `ResearchTaskEntity`, runs the `ResearcherAgent`, and marks the task done. An after-llm-response guardrail (G1) inspects each `FindingsReport` before it reaches the entity; a report with fewer than two source URLs is blocked and the task is recorded `BLOCKED`. When a researcher needs scope clarification from a peer, it posts to that peer's `ResearcherMailbox`. `OperatorControl` (a key-value entity) holds the operator halt flag that both the poll loop and the guardrail read.

Once all sub-topics for a question are `DONE`, `QuestionCompletionMonitor` starts a `SynthesisWorkflow` for that question. The workflow runs `SynthesisAgent`, which merges all findings into a `ResearchReport`. Each conclusion is evaluated by `CitationEvaluator` (control E1); any un-grounded conclusion moves the question to `NEEDS_REVIEW` for operator review.

Three TimedActions sit alongside: `QuestionSimulator` drips sample questions; `QuestionCompletionMonitor` watches for all-done questions; `StuckTaskMonitor` releases stranded claims.

## Interaction sequence

The sequence diagram traces the happy path (J1): submit → plan → sub-topic tasks on the board → a researcher claims, produces findings, the task completes, all tasks done triggers synthesis, the citation eval passes, and the question moves to `COMPLETED`. The `Note over ResearcherAgent` block marks where the after-llm-response guardrail (G1) vets every findings report. The synthesis sequence shows the per-conclusion eval (E1) firing inside `SynthesisWorkflow.evalStep`.

## State machine

`ResearchTaskEntity` is the heart of the pattern. `OPEN` is the initial state. `claim` is atomic — only an `OPEN` task can be claimed, and the entity's single-writer guarantee means exactly one researcher wins a contested task. From `CLAIMED` the researcher `start`s into `IN_PROGRESS`, the `ResearcherAgent` produces a `FindingsReport`, the after-llm-response guardrail vets it, and on pass the workflow calls `submitFindings` (-> `FINDINGS_READY`) then `complete` (-> `DONE`). A guardrail refusal or a coordination request moves the task to `BLOCKED` directly. `StuckTaskMonitor` returns a claimed-but-idle task to `OPEN` for liveness.

## Entity model

`ResearchTaskEntity` is the source of truth for each sub-topic; every transition writes one of seven event types, and `TaskBoardView` is the only read-side projection. `QuestionEntity` tracks the question lifecycle and owns the set of task ids, as well as the final `ResearchReport` once synthesis completes. `IntakeQueue` is the audit log of submissions. `ResearcherMailbox` (one per researcher) holds cross-researcher coordination messages.

## Concurrency & timeouts

- Atomic claim through `ResearchTaskEntity`'s single writer is the coordination primitive — no lock, no external queue.
- Per-step timeout: 90 s on the agent-calling steps (`PlanningWorkflow.planStep`, `ResearcherWorkflow.researchStep`, `SynthesisWorkflow.synthesizeStep`).
- Idle researchers are paused workflows with a 5 s resume timer, not busy loops.
- Synthesis is triggered by `QuestionCompletionMonitor` polling every 30 s; the `questionId` is the `SynthesisWorkflow` id, making repeated starts idempotent.
- `StuckTaskMonitor` releases a task claimed-but-idle for more than three minutes so a failed researcher does not strand a sub-topic.
- `CitationEvaluator` is deterministic, so a given `ReportConclusion` always yields the same eval result; the same evaluator runs in tests.
- Halt reads `OperatorControl` at the top of `pollStep` and inside the G1 guardrail so a halt both stops new claims and blocks any in-flight researcher response.
