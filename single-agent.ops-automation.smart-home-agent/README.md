# Akka Sample: Connected House Agent

A single smart-home agent accepts natural-language commands from a resident and translates them into typed device-control actions — turn lights on, set the thermostat, lock the front door. Before any physically consequential action fires, a before-tool-call guardrail checks authorization; before the action is dispatched, the resident confirms via an in-loop HITL step.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a before-tool-call guardrail that blocks unauthorized or dangerous device commands, and a HITL confirmation step that gates lock/unlock and thermostat override actions on explicit resident approval.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — device state is in-process and the agent's tool calls are simulated against a seeded device registry.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.smart-home-agent  ~/my-projects/smart-home-agent
cd ~/my-projects/smart-home-agent
```

(Optional) Edit `SPEC.md` to change the seeded device catalogue (e.g., add a garage door, a sprinkler zone, or a security camera) or adjust the list of commands that require HITL confirmation.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **HomeControlAgent** — an AutonomousAgent that accepts a natural-language command from a resident, resolves which devices are involved, and emits one or more typed `DeviceAction` results.
- **CommandWorkflow** — orchestrates guard-check → confirmation (HITL for sensitive commands) → dispatch per submitted command.
- **CommandEntity** — an EventSourcedEntity holding the per-command lifecycle.
- **DeviceRegistry** — an EventSourcedEntity that tracks the current state of every device (on/off, temperature set-point, lock status).
- **CommandView** — read model projecting command events for the UI.
- **CommandEndpoint + AppEndpoint** — REST/SSE API and embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the seeded device catalogue and the set of commands that trigger HITL.
- `SPEC.md §5` — extend `DeviceAction` with additional actuator types (garage, irrigation, camera).
- `prompts/home-control-agent.md` — restrict the agent to a specific home profile or locale.
- `eval-matrix.yaml` — wire the guardrail to a real authorization service (e.g., a resident-role lookup) by naming it in the implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A resident issues a light-on command → it dispatches immediately without confirmation.
2. A resident issues a lock command → the HITL step fires; confirming causes the lock action to dispatch; rejecting cancels it.
3. The guardrail blocks a command that references a device not in the seeded registry.
4. A thermostat set-point outside the allowed range is blocked at the guardrail; the resident sees a structured refusal, not a runtime exception.

## License

Apache 2.0.
