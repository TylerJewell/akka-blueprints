# Akka Sample: Scrum Master Assistant

A single Scrum Master agent runs daily standups, captures sprint planning data, and writes ticket updates. The agent reads current sprint state, asks each team member the three standup questions, and posts a structured standup summary back to the ticket tracker. All ticket writes are gated by a `before-tool-call` guardrail that prevents the agent from modifying tickets it has not been explicitly authorized to touch.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that validates every outbound write against the sprint's authorized ticket scope before the call leaves the agent.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The sprint data lives in-process and the ticket tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.scrum-master-bot  ~/my-projects/scrum-master-bot
cd ~/my-projects/scrum-master-bot
```

(Optional) Edit `SPEC.md` to point at a different team profile — for example, swap the seeded sprint fixtures for a platform-engineering team using different ticket prefixes and standup cadences.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ScrumMasterAgent** — an AutonomousAgent that conducts standups, collects blockers, and writes standup summaries to tickets via a tool call.
- **StandupWorkflow** — orchestrates collect-sprint → run-standup → post-summary per standup session.
- **StandupEntity** — an EventSourcedEntity holding the per-session lifecycle.
- **SprintConsumer** — a Consumer that subscribes to `SprintActivated` events and seeds the session state.
- **StandupView + StandupEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded sprint team profiles for your own (the JSONL file under `src/main/resources/sample-events/sprint-fixtures.jsonl` after generation).
- `SPEC.md §5` — extend `StandupSummary` with team-specific fields (e.g., `velocityDelta`, `sprintGoal`, `riskFlags`).
- `prompts/scrum-master.md` — narrow the agent's role (a platform team would constrain it to infrastructure tickets; a product team would extend it with capacity-planning heuristics).
- `eval-matrix.yaml` — swap the simulated ticket tool for a real integration by naming the HTTP client under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user activates a sprint and triggers a standup → the agent conducts the session → a summary appears in the UI and is posted to each team member's tickets.
2. The agent attempts to write to a ticket outside the sprint scope → the `before-tool-call` guardrail blocks the call → the agent adjusts → only authorized writes land.
3. A standup session where all members report blockers produces a summary flagged `BLOCKED`, visible in the live list.
4. The sprint's ticket list updates mid-session → the agent sees the refreshed scope on subsequent writes.

## License

Apache 2.0.
