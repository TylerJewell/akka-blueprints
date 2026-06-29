# Architecture — guided-intake-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ConversationEndpoint` is the entry point. A session submission writes a `SessionSubmitted` event to `SessionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ConversationWorkflow` per submission. The workflow drives the planner-executor loop: it asks `IntakeAgent` to plan the question sequence, then on each turn asks it to propose the next question, sanitizes the user reply via `SanitizerAgent`, records the turn, and asks `IntakeAgent` to evaluate. Every transition emits an event on `ConversationEntity`; `ConversationView` projects those events into the read model the UI streams via SSE.

User replies are delivered out-of-band: the workflow suspends in `awaitReplyStep` while the conversation is live; `POST /api/conversations/{id}/reply` resumes it. This means the workflow does not hold a thread between turns.

Two TimedActions sit alongside: `SessionSimulator` drips sample sessions for demo purposes; `StaleSessionMonitor` ticks every 60 s to mark long-running `ELICITING` conversations as `ABANDONED`.

`GoalCatalogEntity` is a single-instance entity keyed by `"catalog"`. Operators manage goal definitions via `POST /api/goals`; the workflow reads the catalog in `planStep`. `SystemControlEntity` is a single-instance entity keyed by `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every question.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is the planner-executor cycle — each iteration runs check-halt → propose → guardrail → ask → await-reply → sanitize → record → evaluate → decide. `awaitReplyStep` is a workflow suspension: the workflow parks until the user submits a reply via `POST /api/conversations/{id}/reply`, which calls `ConversationWorkflow.resume`. This keeps the workflow's thread count at zero between turns.

## State machine

`ConversationEntity` has six states. `PLANNING` is the initial state. `ELICITING` is the loop state — most events fire here without changing the status. A conversation lands in one of four terminal states: `COMPLETED` (goal met, summary produced), `ESCALATED` (agent exhausted the replan or turn budget without meeting the goal), `HALTED` (operator pressed Halt), or `ABANDONED` (no reply for 10 minutes).

## Entity model

`ConversationEntity` is the system's source of truth; every transition writes one of ten event types. `GoalCatalogEntity` holds the operator-managed set of intake goals. `SystemControlEntity` carries the operator halt flag. `SessionQueue` is the audit log of submissions. `ConversationView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `proposeStep` 45 s, `awaitReplyStep` 300 s (user has up to 5 min per turn), `sanitizeStep` 30 s, `evaluateStep` 45 s, `summaryStep` 60 s.
- Replan budget: 2 consecutive `Replan` outputs; the third becomes `Escalate`.
- Turn budget: each goal specifies `maxTurns`; reaching it triggers `escalateStep`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every loop iteration; no caching. A halt arriving during `awaitReplyStep` is picked up at the next `checkHaltStep` after the reply lands.
- Idempotency: `(goalId, submittedBy)` over a 10 s window deduplicates `POST /api/conversations`.
- Stale detection: `StaleSessionMonitor` every 60 s; conversations `ELICITING` for > 10 minutes are marked `ABANDONED`.
- PII scrubber determinism: `PiiScrubber.scrub` is pure; raw replies are never stored in any event.
