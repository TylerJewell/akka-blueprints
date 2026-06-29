# Architecture — peerreview

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ReviewEndpoint` is the entry point. It writes a `SubmissionReceived` event to `SubmissionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ReviewWorkflow` per submission. The workflow redacts the document, then calls four agents — `ReviewModerator` to plan, then `TechnicalReviewer`, `StyleReviewer`, and `ComplianceReviewer` in parallel, then `ReviewModerator` again to reconcile. Each transition emits an event on `ReviewEntity`. `ReviewView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `SubmissionSimulator` drips sample documents for demonstration, and `EvalSampler` ticks every 5 minutes to grade one synthesised review for cross-axis consistency, calling `EvalJudge`.

The PII sanitizer is not a component box of its own — it is the deterministic `PiiSanitizer` helper invoked inside `ReviewWorkflow.sanitizeStep`. It makes no Akka call; it transforms the raw body into a redacted body and a count before the body is ever persisted.

## Interaction sequence

The sequence diagram traces the happy path (J1). Two notes matter. First, the sanitize step runs before any agent is called and before any text is persisted — the raw body lives only in the workflow's transient start command. Second, the `par` block: the three reviewers run **concurrently**, not in a chain. The workflow awaits all three via a CompletionStage zip with a 60-second per-step timeout, which is the heart of the debate-multi-perspective pattern.

## State machine

The review has five states. `INTAKE` is the initial state when `ReviewCreated` fires. `REVIEWING` is reached once the document is sanitized and the reviewers are dispatched. The review then lands in one of three terminal states: `SYNTHESISED` (all axes returned and the guardrail passed), `DEGRADED` (a reviewer timed out; reconciliation ran on partial input), or `BLOCKED` (the output guardrail rejected the reconciled verdict). A `ConsistencyScored` event may land on a `SYNTHESISED` review without changing its status.

## Entity model

`ReviewEntity` is the system's source of truth; every transition writes one of nine event types. `SubmissionQueue` is the audit log of submissions. `ReviewView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for each reviewer call and for the synthesis call. The default 5-second workflow step timeout is far too short for an LLM call (Lesson 4).
- Degraded path: a reviewer timeout transitions to synthesis-from-partial rather than failing the whole workflow; `failureReason` names the missing reviewer.
- Idempotency: `(title, submittedBy)` over a 10 s window deduplicates `POST /api/reviews`.
- View indexing: `getAllReviews` has no `WHERE status` clause — the `ReviewStatus` enum cannot be auto-indexed (Lesson 2); callers filter client-side.
- Eval sampler: one review per 5-minute tick; the oldest `SYNTHESISED` review without a `consistencyScore` wins.
- Sanitizer ordering: redaction happens before planning and before the raw body could be persisted, which is what realises control S1.
