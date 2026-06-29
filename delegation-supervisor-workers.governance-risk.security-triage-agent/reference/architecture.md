# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

An incident signal enters through `IncidentEndpoint` (`POST /api/incidents`) or is dripped by `IncidentSimulator` every 90 seconds. Either path writes an `IncidentSubmitted` event onto `IncidentQueue`. `IncidentConsumer` subscribes to those events and starts one `TriageWorkflow` per submission, keyed by `incidentId`.

The workflow is the supervisor. It asks `SecurityCoordinator` to decompose the signal into a scan plan, fans the work out to `VulnerabilityScanner` and `ThreatContextAgent` in parallel, then asks `SecurityCoordinator` to produce a `TriageReport`. The workflow then parks at `AWAITING_APPROVAL`, writing an `ApprovalRequested` event to `ApprovalEntity`. A security officer calls `POST /api/incidents/{id}/approve` or `.../reject`; `ApprovalEntity` sends a resume signal that unparks the workflow for its final step.

Every transition is written as a command to `IncidentEntity`, whose events project into `IncidentView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a mitigated incident to score.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out, join, triage, park for approval, resume on officer approval, persist as MITIGATED. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results before triaging. The approval gate is the human-in-the-loop core — the workflow will not advance to MITIGATED or REJECTED without a named officer's decision.

## State machine

`Incident` moves `RECEIVED → TRIAGING → AWAITING_APPROVAL`, then to one of three terminals: `MITIGATED` (officer approved), `REJECTED` (officer denied), or `DEGRADED` (a worker timed out during the triage phase). `MITIGATED` accepts one further `TriageEvalScored` self-transition when `EvalSampler` records a quality score.

## Entity model

`IncidentQueue` seeds one `Incident` per signal. An incident owns at most one `VulnerabilityBundle`, one `ThreatContextBundle`, and one `TriageReport`. It also owns at most one `ApprovalDecision`, which records the officer identity and outcome. The view row mirrors the incident with `Optional<T>` on every field that is null before its transition (Lesson 6).
