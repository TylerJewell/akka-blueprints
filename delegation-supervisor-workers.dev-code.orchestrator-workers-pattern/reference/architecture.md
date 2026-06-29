# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A task enters through `EditEndpoint` (`POST /api/tasks`) or is dripped by `TaskSimulator` every 60 seconds. Either path writes a `TaskSubmitted` event onto `EditJobQueue`. `EditJobConsumer` subscribes to those events and starts one `EditWorkflow` per submission, keyed by `taskId`.

The workflow is the supervisor. It asks `Orchestrator` to decompose the task into per-file instructions, fans the work out to `FileEditor` workers in parallel (one per target file), then collects `EditReviewer` verdicts for each edited file, and finally asks `Orchestrator` to synthesise the changeset. Before each `FileEditor` tool call, the before-tool-call guardrail verifies the target path. Every lifecycle transition is written as a command to `EditTaskEntity`, whose events project into `EditTaskView`. `EditEndpoint` reads and streams the view; `QualitySampler` reads it to pick a task to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel file-edit fan-out with guardrail gating, parallel review collection, synthesis, and persist. The `par` block is the delegation-supervisor-workers core — all `FileEditor` workers run concurrently and the workflow joins their outputs before proceeding to review and synthesis.

## State machine

`EditTask` moves `PLANNING → IN_PROGRESS` after the decomposition plan is ready, then to one of three terminals: `COMPLETED` (all edits synthesised and guardrail passed), `DEGRADED` (a worker timed out; synthesised from whatever files returned), or `BLOCKED` (before-tool-call guardrail rejected a write). `COMPLETED` accepts one further `QualityScored` self-transition when `QualitySampler` records a score.

## Entity model

`EditJobQueue` seeds one `EditTask` per submission. A task owns at most one `DecompositionPlan`, a list of `EditedFile` outputs (one per target file), a list of `ReviewVerdict` outputs, and one `Changeset`. The view row mirrors the task with `Optional<T>` on every field that is null before its transition (Lesson 6).
