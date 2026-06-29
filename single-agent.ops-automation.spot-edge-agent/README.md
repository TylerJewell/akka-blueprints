# Akka Sample: Spot Agent (Edge)

An edge agent embedded with a Boston Dynamics Spot robot via an MCP server for robot control. An operator sends a high-level mission (inspect zone, navigate to waypoint, capture telemetry), and one AI agent translates the intent into a sequence of Spot SDK motion commands. Every motion command passes through a `before-tool-call` guardrail before it reaches the robot, a human-approval gate blocks non-trivial maneuvers, and an automatic safety halt cuts power to the drive system whenever sensor telemetry crosses a safety threshold.

Demonstrates the **single-agent** coordination pattern in the ops-automation domain. Three governance mechanisms sit between the operator's intent and the physical actuators: a motion-command guardrail, an application-layer human-in-the-loop approval gate, and an automatic safety halt wired to live sensor feeds.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The Spot MCP server connection is simulated in-process; no physical robot or Spot SDK installation is required to run the blueprint locally.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.spot-edge-agent  ~/my-projects/spot-edge-agent
cd ~/my-projects/spot-edge-agent
```

(Optional) Edit `SPEC.md` to point at a different Spot deployment: adjust the seeded waypoint map in `src/main/resources/sample-events/waypoints.jsonl` or replace the simulated telemetry thresholds with values from your robot's config.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SpotNavigatorAgent** — an AutonomousAgent that receives a mission brief and a waypoint map as a task attachment, then calls Spot SDK tools in sequence to execute the mission.
- **MissionWorkflow** — orchestrates approval-wait → execution → telemetry-harvest per submitted mission.
- **MissionEntity** — an EventSourcedEntity holding the per-mission lifecycle from SUBMITTED through COMPLETED or HALTED.
- **TelemetryMonitor** — a Consumer that subscribes to `MissionStarted` events and streams robot sensor data; emits `SafetyThresholdBreached` if a threshold is crossed.
- **MissionView + MissionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded waypoint map for your facility's map (the JSONL file under `src/main/resources/sample-events/waypoints.jsonl` after generation).
- `SPEC.md §5` — extend `MissionBrief` with site-specific metadata (e.g., `buildingZone`, `accessLevel`, `requiredSensors`).
- `prompts/spot-navigator.md` — narrow the agent's allowed motion vocabulary (a confined-space deployer would restrict stair-climbing; a public-space deployer would require obstacle-hold before any gait change).
- `eval-matrix.yaml` — tighten the `CommandGuardrail` by adding site-specific forbidden zones as static polygons in the check.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. An operator submits a patrol mission → the approval gate presents a summary → the operator approves → the agent executes waypoint traversal → telemetry is recorded → mission completes.
2. The agent attempts a stair-climb command on a flat-floor mission; the `before-tool-call` guardrail rejects it; the agent reroutes via the flat corridor; the mission completes.
3. A battery voltage drop to 10 % during execution triggers the automatic safety halt; the mission transitions to HALTED; the operator sees the halt reason and sensor snapshot in the UI.
4. A mission requiring a controlled tipping maneuver lands in the PENDING_APPROVAL queue; the operator rejects it; the agent receives the rejection and aborts gracefully.

## License

Apache 2.0.
