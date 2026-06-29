# Architecture — lambda-error-diagnoser

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`LogPoller` is the heartbeat — a TimedAction that ticks every 20 s and writes simulated `ErrorDetected` events into `LogQueue` (event-sourced for audit). A `LogNormalizer` Consumer subscribes to that queue, extracts structured error context from the raw log payload, and emits `ErrorNormalised` on the per-incident `IncidentEntity`. That normalised event starts an `IncidentWorkflow` instance, which orchestrates: diagnose → eval → finalise to RESOLVED.

There is no separate eval scheduler. The eval fires inside the same workflow execution that produced the diagnosis, in the step immediately after. Every incident that reaches RESOLVED carries a score.

## Interaction sequence

The sequence traces the full automated path (J1). Note that the human appears only at step 12 (optional Dismiss) — the system reaches RESOLVED without any human intervention. The `Note over E,U` block marks where the incident becomes visible on the board; by that point diagnosis and score are already attached.

The key architectural decision: placing `EvalJudge` as `evalStep` inside `IncidentWorkflow` rather than as a separate TimedAction means there is zero latency between diagnosis and scoring. Operators cannot see an unscored diagnosis.

## State machine

Seven states. The resolved branch is the primary path: DETECTED → NORMALISED → DIAGNOSED → EVALUATED → RESOLVED. From RESOLVED, operators can dismiss or leave the incident alone. DISMISSED → REOPENED supports the case where a false-positive dismissal is corrected.

There is no AWAITING_APPROVAL state — this system is fully automated. The governance is the on-incident eval score that travels with every diagnosis, not a human gate before resolution.

## Entity model

`IncidentEntity` is the source of truth for a single incident's lifecycle; it emits seven distinct event types. `LogQueue` is the upstream audit log — only `LogNormalizer` subscribes to it. `IncidentView` projects `IncidentEntity` events into rows the UI reads via SSE and REST.

## On-incident eval positioning

For every incident that resolves, the sequence is:
1. `LogNormalizer` extracts structured error context — the model never sees raw stack traces with potentially sensitive paths mixed in.
2. `DiagnosisAgent` classifies root cause and recommends a fix.
3. `EvalJudge` scores the diagnosis before `IncidentResolved` is emitted.
4. `IncidentEntity` stores the result; `IncidentView` projects it; the UI surfaces it.

Each step is independent. If `DiagnosisAgent` times out, the incident resolves with a stub diagnosis and score=1, surfacing the timeout to operators rather than silently discarding the event.
