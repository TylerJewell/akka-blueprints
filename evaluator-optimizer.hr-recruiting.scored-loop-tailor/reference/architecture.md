# Architecture — scored-loop-tailor

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`RecruitingEndpoint` is the entry point. It writes a `JobPostingSubmitted` event to `SubmissionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `TailoringWorkflow` per submission. The workflow alternates two agents — `CvTailorAgent` produces a CV draft, `CvReviewerAgent` scores it — with a deterministic section-presence guardrail check between them. Each step emits an event on `ApplicationEntity`. `ApplicationsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `SubmissionSimulator` drips a sample job posting every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records a `ReviewEvalRecorded` event for any reviewed attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first CV draft is sent back for revision and the second draft is approved. The Tailor → guardrail → Reviewer alternation is explicit. Each attempt's events are written to `ApplicationEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The application moves between two transient states (`TAILORING`, `REVIEWING`) and two terminal states (`APPROVED`, `REJECTED_FINAL`). The self-loop on `TAILORING` represents the guardrail-blocked path: a draft missing mandatory sections sends the workflow back to `tailorStep` with structured feedback. The `REVIEWING → TAILORING` transition is the `REVISE` path — taken when the reviewer returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every draft and every review on the entity.

## Entity model

`ApplicationEntity` is the system's source of truth; every transition writes one of seven event types. `SubmissionQueue` is the audit log of job posting submissions. `ApplicationsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `tailorStep` and `reviewStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `RecruitingEndpoint.submit` deduplicates on `(jobTitle, candidateProfile)` over a 10 s window. `EvalSampler` deduplicates on `(applicationId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Application.attempts` with its own draft, guardrail verdict, and (once produced) review. The `attemptNumber` is monotonic across the whole loop, including guardrail-blocked attempts that never reached the reviewer.
