# Architecture — job-posting-eeo

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`JobPostingEndpoint` is the entry point. It writes a `RequestSubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `JobPostingWorkflow` per submission. The workflow calls three agents — `PostingLead` to decompose, then `CultureAnalyst` and `RoleAnalyst` in parallel, then `PostingLead` again to draft. Each transition emits an event on `JobPostingEntity`. `JobPostingView` projects those events into the read model the UI streams via SSE. `RequestSimulator` drips sample requests so the App UI is never empty.

## Interaction sequence

The sequence diagram traces the happy path (J1). Note the `par` block — `CultureAnalyst` and `RoleAnalyst` run **concurrently**, not in a chain. The workflow awaits both via a CompletionStage zip with a 60-second per-step timeout. After the draft is produced, two governance steps follow inline: the sanitizer strips protected-class terms, then the documentation gate verifies the EEO statement.

## State machine

The posting has seven states. `PLANNING` is initial, `ANALYZING` is reached when the workflow dispatches the parallel work, `DRAFTED` and `SANITIZED` are intermediate, and the posting lands in one of three terminal states: `CLEARED` (happy path), `BLOCKED` (the EEO guardrail flagged discriminatory language), or `DEGRADED` (one worker timed out; the draft was synthesised from partial input).

## Entity model

`JobPostingEntity` is the system's source of truth; every transition writes one of eight event types. `RequestQueue` is the audit log of submissions. `JobPostingView` is the only read-side projection — the UI never queries the entity directly.

## Governance wiring

Three controls sit on the path from draft to publication:

- **G1 — EEO guardrail** runs on `PostingLead`'s `before-agent-response` hook inside `draftStep`. Blocking; failure routes to `BLOCKED`.
- **S1 — protected-class sanitizer** runs in `sanitizeStep`, removing matched terms and recording them on the `SanitizedPosting`.
- **C1 — documentation gate** runs in `complianceStep` (and as a CI build-gate), verifying the EEO statement is present before `CLEARED`.

## Concurrency & timeouts

- Per-step timeout: 60 s for worker calls and the draft call.
- Idempotency: `(company, roleTitle, requestedBy)` over a 10 s window deduplicates `POST /api/postings`.
- Forward-only lifecycle: terminal states are absorbing; no compensation logic is required.
