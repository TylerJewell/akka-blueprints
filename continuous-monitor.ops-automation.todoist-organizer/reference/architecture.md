# Architecture — todoist-organizer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`TodoistPoller` is the heartbeat — a TimedAction that ticks every 30 s and writes `TaskFetched` events into `TaskQueue` (event-sourced for audit). A `TaskFetchConsumer` Consumer subscribes to that queue, registers each task with `TodoistTaskEntity`, and starts a per-task `OrganizerWorkflow`. That workflow orchestrates: classify → guardrail check → update or skip.

`EvalRunner` runs alongside as a second TimedAction, ticking every 60 minutes and scoring sampled updated tasks.

## Interaction sequence

The sequence traces the happy path (J1). Note the guardrail check occurs entirely within the workflow — there is no round-trip to an external system at that step. The workflow reads `ClassificationResult.confidence` and the project allow-list from application config, makes the allow/block decision synchronously, and emits `GuardrailChecked` before deciding whether to proceed to `updateStep`.

The `Note over E,U: Task in UPDATED state` block marks the terminal state for a successful pass. The SKIPPED terminal (J2) follows the same path up to `guardrailStep` but branches to `TaskSkipped` when `allowed == false`.

## State machine

Six states. The interesting branches:

- After CLASSIFIED, the guardrail verdict drives the branch: `allowed=true` proceeds to UPDATED; `allowed=false` goes to GUARDRAIL_BLOCKED terminal; explicit skip (e.g., duplicate taskId) goes to SKIPPED terminal.
- UPDATED is the only non-terminal state that can receive a second event: `EvalScored` attaches asynchronously later without changing the status label.
- FAILED captures workflow errors (classifier timeout, Todoist API error in the update step).

## Entity model

`TodoistTaskEntity` is the source of truth; it emits seven distinct event types covering the full lifecycle including eval. `TaskQueue` is the upstream audit log — only `TaskFetchConsumer` subscribes to it.

## Defence-in-depth governance flow

For any Todoist write that goes out, the task passed through:

1. **TaskClassifierAgent** — typed classifier defaults to low confidence under ambiguity.
2. **Guardrail (in-workflow)** — confidence threshold + project allow-list check blocks the write before it reaches the Todoist tool.
3. **Eval-periodic sampler** — continuous quality monitoring flags drift in classification correctness over time.

Each step is independent. The guardrail catches individual bad classifications immediately; the eval sampler catches systematic drift that no single task would reveal.
