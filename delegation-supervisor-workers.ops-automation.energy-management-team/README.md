# Akka Sample: Energy Efficiency Management Agent

A building energy supervisor delegates optimization work to specialist agents — one per subsystem (HVAC, lighting, equipment) — running in parallel, then consolidates their recommendations into one actionable efficiency report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the building telemetry stream and subsystem control tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.ops-automation.energy-management-team  ~/my-projects/energy-management-team
cd ~/my-projects/energy-management-team
```

(Optional) Edit `SPEC.md` to change the building zones simulated, the model provider, or the output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **EnergyCoordinator** — AutonomousAgent that decomposes a building optimization cycle into subsystem tasks and consolidates subsystem reports into one efficiency report.
- **HvacSpecialist** — AutonomousAgent that analyzes HVAC telemetry and recommends setpoint adjustments.
- **LightingSpecialist** — AutonomousAgent that analyzes occupancy and daylight data and recommends lighting schedules.
- **EquipmentSpecialist** — AutonomousAgent that identifies equipment running outside efficient operating bands.
- **OptimizationWorkflow** — Workflow that fans work out to three specialists in parallel, then asks the Coordinator to consolidate, with a before-tool-call guardrail gating any control action.
- **OptimizationCycleEntity** — EventSourcedEntity holding the full optimization-cycle lifecycle.
- **CycleView** — projection the UI streams via SSE.
- **CycleEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change which building zones the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `OptimizationCycle` record fields (e.g., add `estimatedSavingsKwh`).
- `prompts/hvac-specialist.md` — narrow the agent to a specific HVAC system type.
- `eval-matrix.yaml` — extend the `before-tool-call` guardrail to cover additional control action categories.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Trigger an optimization cycle → cycle enters `DISPATCHED`, then `CONSOLIDATING`, then `COMPLETE`.
2. One specialist fails-fast (timeout) → cycle enters `PARTIAL` with whichever subsystem reports came back; the consolidated report notes the missing subsystem.
3. Guardrail blocks a control action that would exceed safety limits → the action is rejected and the cycle records the blocked action.
4. Wait after a successful cycle; the cycle row shows an efficiency eval score.

## License

Apache 2.0.
