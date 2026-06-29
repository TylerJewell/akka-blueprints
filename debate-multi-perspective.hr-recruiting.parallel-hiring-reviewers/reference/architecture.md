# Architecture — parallel-hiring-reviewers

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ReviewEndpoint` is the entry point. It writes a `ProfileReceived` event to `CandidateQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ReviewWorkflow` per profile submission. The workflow sanitizes the résumé text, then calls four agents — `PanelCoordinator` to plan, then `HrReviewer`, `ManagerReviewer`, and `TeamReviewer` in parallel, then `PanelCoordinator` again to aggregate. Each transition emits an event on `CandidateEntity`. `EvaluationView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `CandidateSimulator` drips sample profiles for demonstration, and `FairnessSampler` ticks every 10 minutes to score one decided evaluation for cross-perspective proportionality, calling `FairnessJudge`.

The special-category sanitizer is not a component box of its own — it is the deterministic `SpecialCategorySanitizer` helper invoked inside `ReviewWorkflow.sanitizeStep`. It makes no Akka call; it transforms the raw résumé text into a redacted text and a count before the text is ever persisted.

## Interaction sequence

The sequence diagram traces the happy path (J1). Two notes matter. First, the sanitize step runs before any agent is called and before any text is persisted — the raw résumé text lives only in the workflow's transient start command. Second, the `par` block: the three reviewers run **concurrently**, not in a chain. The workflow awaits all three via a CompletionStage zip with a 60-second per-step timeout, which is the heart of the debate-multi-perspective pattern.

## State machine

The evaluation has five states. `INTAKE` is the initial state when `EvaluationCreated` fires. `EVALUATING` is reached once the profile is sanitized and the reviewers are dispatched. The evaluation then lands in one of three terminal states: `DECIDED` (all perspectives returned and the guardrail passed), `DEGRADED` (a reviewer timed out; aggregation ran on partial input), or `BLOCKED` (the output guardrail rejected the aggregated recommendation). A `FairnessScored` event may land on a `DECIDED` evaluation without changing its status.

## Entity model

`CandidateEntity` is the system's source of truth; every transition writes one of nine event types. `CandidateQueue` is the audit log of profile submissions. `EvaluationView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for each reviewer call and for the aggregate call. The default 5-second workflow step timeout is far too short for an LLM call (Lesson 4).
- Degraded path: a reviewer timeout transitions to aggregation-from-partial rather than failing the whole workflow; `failureReason` names the missing reviewer.
- Idempotency: `(candidateName, roleApplied)` over a 10 s window deduplicates `POST /api/evaluations`.
- View indexing: `getAllEvaluations` has no `WHERE status` clause — the `EvaluationStatus` enum cannot be auto-indexed (Lesson 2); callers filter client-side.
- Fairness sampler: one evaluation per 10-minute tick; the oldest `DECIDED` evaluation without a `fairnessScore` wins.
- Sanitizer ordering: redaction happens before planning and before the raw text could be persisted, which is what realises control S1.
