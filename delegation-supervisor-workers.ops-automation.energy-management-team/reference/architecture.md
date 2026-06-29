# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

An optimization cycle enters through `CycleEndpoint` (`POST /api/cycles`) or is dripped by `TelemetrySimulator` every 90 seconds. Either path writes a `TelemetryReceived` event onto `TelemetryQueue`. `TelemetryConsumer` subscribes to those events and starts one `OptimizationWorkflow` per event, keyed by `cycleId`.

The workflow is the supervisor. It asks `EnergyCoordinator` to plan the subsystem queries, then fans the work out to `HvacSpecialist`, `LightingSpecialist`, and `EquipmentSpecialist` in parallel. Before any specialist's control-action tool executes, a before-tool-call guardrail checks the action against configured safety thresholds. After all three (or available) reports return, the workflow asks `EnergyCoordinator` to consolidate. Every lifecycle transition is written as a command to `OptimizationCycleEntity`, whose events project into `CycleView`. The endpoint reads and streams the view; `EfficiencyEvalSampler` reads it to pick a completed cycle to score.

## Interaction sequence

The sequence diagram traces the primary journey: plan, parallel three-way fan-out (with guardrail interceptions), join, consolidate, persist. The `par` block is the delegation-supervisor-workers core — all three specialists run concurrently and the workflow joins their results. The note on the guardrail indicates it runs inside each specialist's tool-call boundary, invisible to the workflow step.

## State machine

`OptimizationCycle` moves `QUEUED → DISPATCHED → CONSOLIDATING`, then to one of three terminals: `COMPLETE` (all reports consolidated), `PARTIAL` (at least one specialist timed out, consolidation proceeded from partial input), or `BLOCKED` (the consolidation step itself failed a guardrail check). `COMPLETE` accepts one further `EvalScored` self-transition when `EfficiencyEvalSampler` records a quality score.

## Entity model

`TelemetryQueue` seeds one `OptimizationCycle` per telemetry event. A cycle owns at most one `HvacReport`, one `LightingReport`, one `EquipmentReport`, and one `ConsolidatedReport`. The `CycleView` row mirrors the cycle entity with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6). Each specialist report carries its own `currentLoadKw` field so the Coordinator can compute an aggregate load picture during consolidation.
