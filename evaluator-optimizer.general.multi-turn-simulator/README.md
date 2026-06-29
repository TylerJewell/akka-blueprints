# Akka Sample: Multi-Turn Actor Simulator

A simulator agent plays the role of a user across a multi-turn dialog; an evaluator agent scores the target agent's responses turn by turn and issues a final fairness/drift verdict when the session closes. Demonstrates the **evaluator-optimizer** coordination pattern used to run automated behavioral testing against a conversational agent.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the target agent and the simulated actor are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.multi-turn-simulator  ~/my-projects/multi-turn-simulator
cd ~/my-projects/multi-turn-simulator
```

(Optional) Edit `SPEC.md` to change the dialog scenario set, the turn limit, the evaluation rubric, or the drift-detection window.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ActorAgent** — AutonomousAgent that plays a user persona across multi-turn dialog, sending one turn at a time to the target agent and advancing the conversation according to a scripted scenario.
- **EvaluatorAgent** — AutonomousAgent that scores each target-agent response against a rubric (coherence, policy adherence, persona consistency, bias) and produces a per-turn `TurnVerdict`. At session close it aggregates turn verdicts into a `SessionVerdict` with a drift-fairness flag.
- **SimulationWorkflow** — Workflow that drives the turn loop: ActorAgent sends a turn → TargetAgentProxy forwards it → EvaluatorAgent scores the response → repeat until the scenario ends or the turn ceiling is reached.
- **SessionEntity** — EventSourcedEntity that holds the full dialog history, every turn verdict, and the final session verdict.
- **ScenarioQueue** — EventSourcedEntity that logs each scenario submission for replay and audit.
- **SessionsView** — read-side projection the UI lists and streams via SSE.
- **ScenarioConsumer** — Consumer that starts a SimulationWorkflow per submitted scenario.
- **ScenarioSimulator** — TimedAction that drips a canned scenario every 90 s so the App UI is never empty.
- **DriftSampler** — TimedAction that records a `DriftEvalRecorded` event each cycle, enabling downstream quality measurement (control E1).
- **SimulationEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the scenarios the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Session` record fields (e.g., raise `maxTurns`, change the drift threshold).
- `prompts/actor.md` — switch the user persona (e.g., a non-native speaker, a power user, an adversarial probe).
- `prompts/evaluator.md` — change the rubric (e.g., add a tone dimension, change the scoring scale).
- `eval-matrix.yaml` — tighten the drift-fairness watch (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a scenario → session progresses `RUNNING` → `COMPLETED` with every turn scored.
2. Force-fail rubric → session reaches `FLAGGED` after the configured turn limit; entity preserves every turn and verdict for audit.
3. The output guardrail blocks a target-agent response that violates the policy check, so the evaluator scores it as a policy violation without accepting the response.
4. Each completed cycle emits a `DriftEvalRecorded` event that surfaces in the App UI's per-turn timeline.

## License

Apache 2.0.
