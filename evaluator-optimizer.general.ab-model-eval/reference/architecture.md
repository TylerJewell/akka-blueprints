# Architecture — ab-model-eval

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`EvalEndpoint` is the entry point. It writes a `TaskSubmitted` event to `TaskQueueEntity` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `EvalWorkflow` per submission. The workflow fans out to two agents — `CandidateAAgent` and `CandidateBAgent` — in parallel, collects both responses, and calls `JudgeAgent` to score them and declare a winner. Each step emits an event on `TrialEntity`. `TrialsView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside the main flow: `TaskSimulator` drips a sample task prompt every 60 seconds so the UI shows data when first opened; `AccuracySampler` ticks every 30 seconds and writes an `EvalRecorded` event for any completed trial that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style trial where both candidates respond within the timeout, the judge scores them, and candidate A wins. The parallel fan-out step is explicit — both agent calls run at the same time, and the workflow waits for both before calling the judge. Each event is written to `TrialEntity` before the next step begins, so the UI's per-trial detail reconstructs the full sequence without additional fetches.

## State machine

The trial has one transient state (`RUNNING`) and two terminal states (`JUDGED`, `TIMED_OUT`). The self-loop on `JUDGED` represents the `RecertificationFlagged` transition: the status does not change when the recertification flag is set, but the flag is observable on the entity and propagates to the view's computed `recertificationRequired` field. `TIMED_OUT` is reached when either candidate step exhausts its retry budget and the `defaultStepRecovery` failover fires.

## Entity model

`TrialEntity` is the source of truth. Every transition writes one of seven event types. `TaskQueueEntity` is the audit log of submitted task prompts. `TrialsView` is the only read-side projection — the UI never queries either entity directly. Win-rate aggregates (`winRateA`, `winRateB`) are computed on the view side so the workflow's `recertStep` can read them without performing an in-workflow aggregation.

## Concurrency & timeouts

- **Fan-out parallelism:** `candidateAStep` and `candidateBStep` run in parallel branches; each carries `stepTimeout(Duration.ofSeconds(60))`. The judge step also carries `stepTimeout(Duration.ofSeconds(60))`. The default 5-second timeout never applies to agent-calling steps (Lesson 4).
- **Default step recovery:** `defaultStepRecovery(maxRetries(1).failoverTo(timeoutStep))` — any unrecoverable agent failure ends in `TIMED_OUT`, not in a hung workflow.
- **Idempotency:** `EvalEndpoint.submit` deduplicates on `(text, submittedBy)` over a 10-second window. `AccuracySampler` deduplicates on `trialId`.
- **Recertification step:** `recertStep` is pure-function; it reads `TrialsView` computed fields in memory and runs in the same JVM thread as the workflow, adding no observable latency to the trial.
- **Promotion gate semantics:** the gate does not prevent running new trials. It prevents `POST /api/trials/promote` from succeeding. An operator who disagrees with the win-rate calculation can call `POST /api/trials/recertify` to clear the flag and re-enable promotion, with the action logged on `TrialEntity` as a `RecertificationFlagged(false)` event.
