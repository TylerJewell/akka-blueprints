# Akka Sample: HVAC Data Analytics Agent

A smart-building operations agent that accepts natural-language analytical questions about HVAC telemetry — temperatures, pressures, airflow rates, energy consumption — and returns structured answers with supporting data points, trend assessments, and recommended actions.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a periodic performance evaluator that scores every answer for analytical accuracy and tracks drift over time, giving operators a running signal about which answers to trust.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Telemetry data lives in-process; the agent's tool calls are served by local components.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.hvac-analytics  ~/my-projects/hvac-analytics
cd ~/my-projects/hvac-analytics
```

(Optional) Edit `SPEC.md` to adjust the seeded telemetry zones, change the set of supported question types, or extend the `AnalyticsAnswer` record with site-specific fields (e.g., `buildingId`, `floorZone`, `tenantRef`).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **HvacAnalyticsAgent** — an AutonomousAgent that receives a natural-language question plus a telemetry snapshot as a task attachment and returns a typed `AnalyticsAnswer`.
- **QueryWorkflow** — orchestrates snapshot-assembly → analysis → eval per submitted question.
- **QueryEntity** — an EventSourcedEntity holding the per-question lifecycle.
- **TelemetryStore** — a Consumer that subscribes to `QueryInitiated` events, assembles the relevant telemetry snapshot from the in-process store, and emits `SnapshotReady` back to the entity.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded telemetry zones (currently three fictional building zones) for your own zone identifiers from `src/main/resources/sample-events/telemetry-zones.jsonl`.
- `SPEC.md §5` — extend `AnalyticsAnswer` with site-specific fields such as `buildingId`, `floorZone`, or `regulatoryThreshold`.
- `prompts/hvac-analytics-agent.md` — narrow the agent's analytical scope (e.g., restrict to energy-consumption questions only, or add domain-specific thresholds for a particular ASHRAE standard).
- `eval-matrix.yaml` — configure the periodic evaluation window and accuracy thresholds appropriate for your deployment context.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an analytical question → a telemetry snapshot is assembled → the agent answers → the answer appears in the UI with supporting data points.
2. The agent returns an answer that references zones not present in the snapshot — the answer is accepted (the guardrail is structural, not semantic), and the eval scorer flags the low evidence score.
3. A question submitted over multiple sessions accumulates eval scores; the UI's performance trend chart shows score drift.
4. The telemetry snapshot assembled for the agent never exposes raw equipment identifiers in the agent's task instructions — only the sanitized zone labels.

## License

Apache 2.0.
