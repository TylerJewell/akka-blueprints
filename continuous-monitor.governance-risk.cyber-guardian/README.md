# Akka Sample: Cyber Guardian Agent

A continuous background worker monitors infrastructure telemetry, classifies incoming threat signals, triggers automatic safety halts for critical-severity incidents, and emits structured evaluation events for every detected threat. Demonstrates the **continuous-monitor** coordination pattern wired with two governance mechanisms (on-incident eval event, automatic safety halt).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None**. All threat feeds are simulated in-process.

## Generate the system

```sh
cp -r ./continuous-monitor.governance-risk.cyber-guardian  ~/my-projects/cyber-guardian
cd ~/my-projects/cyber-guardian
```

(Optional) Edit `SPEC.md §3` to point `TelemetryPoller` at a real SIEM endpoint or event bus instead of the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TelemetryPoller** — TimedAction firing every 10 s that drips simulated threat signals into `TelemetryQueue`.
- **ThreatClassifierAgent** — Agent (typed) that scores an event and returns a `ThreatAssessment` with severity, category, and reasoning.
- **RemediationAdvisorAgent** — AutonomousAgent that proposes a remediation playbook for HIGH and CRITICAL threats.
- **SafetyHaltAgent** — AutonomousAgent that issues and logs an automatic isolation directive for CRITICAL threats.
- **IncidentWorkflow** — per-event Workflow orchestrating: classify → assess severity → (CRITICAL) halt → remediation playbook → emit eval event.
- **IncidentEntity** — EventSourcedEntity tracking each threat signal's full lifecycle (received → classified → halted/remediated → closed).
- **IncidentView + ThreatEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **EvalEventEmitter** — TimedAction that fires an `IncidentEvalEvent` for every closed incident, feeding a downstream eval pipeline.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the in-memory feed for a real SIEM or Kafka topic.
- `SPEC.md §5` — extend the `ThreatSignal` record with asset identifiers, CVE IDs, or MITRE ATT&CK tactic codes.
- `prompts/threat-classifier.md` — narrow the severity rubric to your organisation's taxonomy.
- `eval-matrix.yaml` — wire a real isolation API under the halt mechanism's implementation.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated CRITICAL signal arrives → classified → automatic halt issued → eval event emitted.
2. A HIGH signal arrives → classified → remediation playbook produced → incident closed.
3. A LOW signal arrives → classified → no halt triggered → eval event emitted on close.
4. The operator views the live threat feed and drills into a specific incident's full lifecycle.

## License

Apache 2.0.
