# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Trigger a cycle and watch parallel specialist analysis

**Preconditions:** service running on `http://localhost:9371/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a building ID and zone, then click "Trigger Cycle".
2. Observe the new cycle row via SSE.

**Expected:** the cycle progresses `QUEUED → DISPATCHED → CONSOLIDATING → COMPLETE` within ~90 s. The expanded row shows an `HvacReport` (2–4 setpoint recommendations), a `LightingReport` (2–4 schedule entries), an `EquipmentReport` (2–4 equipment actions), and a consolidated summary with an `estimatedSavingsKwh` figure. The three specialist reports arrive close together because the specialists ran in parallel.

## J2 — Specialist timeout produces a partial cycle

**Preconditions:** `HvacSpecialist` step timeout set to 1 s (test override).

**Steps:**
1. Trigger a cycle.
2. Watch the cycle row.

**Expected:** the `hvacStep` times out; the workflow routes to `partialStep`. The Coordinator consolidates from the lighting and equipment reports only. The cycle enters `PARTIAL`; the consolidated summary notes that the HVAC subsystem report is missing. No infinite retry; `hvacReport` is null in the row.

## J3 — Guardrail blocks an unsafe control action

**Preconditions:** `HvacSpecialist` mock fixture returns a cooling setpoint recommendation that exceeds the configured safety delta (e.g., more than 5°C change in one step).

**Steps:**
1. Trigger a cycle that routes to the fixture.
2. Watch the HVAC report in the expanded cycle row.

**Expected:** the before-tool-call guardrail rejects the unsafe setpoint change. The `SetpointRecommendation` for that action has `blockedByGuardrail = true` in the report; it is excluded from the consolidated execution list. The cycle still reaches `COMPLETE` with the remaining approved recommendations. The UI highlights the blocked action in the HVAC report expand view.

## J4 — Eval score appears beside a completed cycle

**Preconditions:** at least one `COMPLETE` cycle without an `evalScore`.

**Steps:**
1. Wait for `EfficiencyEvalSampler` to run (every 10 minutes), or trigger it manually.
2. Refresh the cycle row.

**Expected:** the cycle gains an `evalScore` (1–5) and an `evalRationale`. The App UI row shows the score alongside the status pill. Delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the telemetry simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TelemetrySimulator` drips a telemetry event from `building-telemetry.jsonl` every 90 s; each becomes a cycle that flows through the full pipeline. The App UI is non-empty on first load, showing cycles across several zones.
