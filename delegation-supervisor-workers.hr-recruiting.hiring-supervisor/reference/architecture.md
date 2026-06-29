# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A candidate application enters through `HiringEndpoint` (`POST /api/hiring`) or is dripped by `CandidatePipelineSimulator` every 90 seconds. Either path writes an `ApplicationSubmitted` event onto `ApplicationQueue`. `ApplicationConsumer` subscribes to those events and starts one `HiringWorkflow` per submission, keyed by `applicationId`.

The workflow is the supervisor. Its first act is a sanitization step that strips protected-attribute fields from the raw `ApplicationRequest`, producing a `SanitizedApplication` that is the only payload any agent ever sees. The workflow then asks `HiringSupervisor` to plan the evaluation — a before-invocation guardrail fires at this point — fans the work out to `ScreeningAgent` and `SchedulingAgent` in parallel, then asks `HiringSupervisor` to consolidate. Every transition is written as a command to `ApplicationEntity`, whose events project into `ApplicationView`. `HiringEndpoint` reads and streams the view; `EvalSampler` reads it to pick a recommended application to score.

## Interaction sequence

The sequence diagram traces the primary journey: sanitize, plan (with guardrail), parallel fan-out, join, consolidate, persist. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results. The guardrail check in `planStep` decides whether the evaluation proceeds or enters `POLICY_HOLD` before any sub-agent call is made.

## State machine

`CandidateApplication` moves `RECEIVED → UNDER_REVIEW`, then to one of four terminals: `RECOMMENDED` (supervisor decision + guardrail passed), `REJECTED` (supervisor decision + guardrail passed), `DEGRADED` (a worker timed out), or `POLICY_HOLD` (guardrail failed before a delegate call). `RECOMMENDED` accepts one further `EvalScored` self-transition when `EvalSampler` records a score.

## Entity model

`ApplicationQueue` seeds one `CandidateApplication` per submission. An application owns at most one `ScreeningReport`, one `SchedulingProposal`, and one `HiringRecommendation`. The view row mirrors the application with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6). Protected-attribute fields are never stored; they are removed in `sanitizeStep` before any entity command is issued.
