# Architecture — ragas-eval-harness

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`EvalEndpoint` is the entry point. It writes a `QuestionSubmitted` event to `QuestionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `EvalWorkflow` per submission. The workflow alternates two agents — `RetrievalAgent` produces a grounded answer, `RagasEvaluatorAgent` scores it against three RAGAS metrics — with a deterministic grounding confidence check between them. Each step emits an event on `EvalRunEntity`. `EvalRunsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `QuestionSimulator` drips a sample question every 60 seconds so the UI is not empty when first loaded; `ScoreSampler` ticks every 30 seconds and records a `ScoreRecorded` event for any scored attempt that has not yet been sampled. The CI gate query path runs through `EvalEndpoint → EvalRunsView` and is entirely read-only.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first answer scores below the RAGAS thresholds on faithfulness and context precision, triggering a revision, and the second answer passes all three metrics. The Retrieval agent → grounding check → RAGAS Evaluator alternation is explicit. Each attempt's events are written to `EvalRunEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The evaluation run moves between two transient states (`ANSWERING`, `EVALUATING`) and two terminal states (`PASSED`, `FAILED_FINAL`). The self-loop on `ANSWERING` represents the grounding-blocked path: an answer whose confidence is below the floor sends the workflow back to `answerStep` with structured feedback without ever reaching the RAGAS Evaluator. The `EVALUATING → ANSWERING` transition is the `RETRY` path — taken when the evaluator returns `RETRY` and the attempt count is still below the ceiling. `FAILED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every attempt, every score set, and the best-scoring answer on the entity.

## Entity model

`EvalRunEntity` is the system's source of truth; every transition writes one of seven event types. `QuestionQueue` is the audit log of submissions. `EvalRunsView` is the only read-side projection — the UI and the CI gate query never touch the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `answerStep` and `scoreStep`; the grounding step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 3). The ceiling is checked BEFORE scheduling the next `answerStep`; the workflow never recurses past it.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any unrecoverable agent failure ends in `FAILED_FINAL`, not in a hung workflow.
- Idempotency: `EvalEndpoint.submit` deduplicates on `(questionText, submittedBy)` over a 10 s window. `ScoreSampler` deduplicates on `(runId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- CI gate read path: `GET /api/ci/gate-status` is a synchronous read over `EvalRunsView.getAllRuns`. It takes the most recent 100 completed runs by `createdAt` and returns `{ gatePassed, passRate, windowSize }` in one query; no entity state is mutated.
- Per-attempt accounting: every attempt is appended to `EvalRun.attempts` with its own answer record, grounding verdict, and (once produced) RAGAS score set. The `attemptNumber` is monotonic across the whole loop, including grounding-blocked attempts that never reached the evaluator.
