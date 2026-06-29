# Akka Sample: Chatbot Simulation Evaluation

A simulated-user agent drives a multi-turn conversation against the assistant chatbot; an evaluator agent scores each completed dialogue end-to-end and returns a pass/fail verdict with structured feedback. Demonstrates the **evaluator-optimizer** coordination pattern for automated CX-support quality gates.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the simulated-user persona library and the assistant chatbot are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.cx-support.chatbot-sim-eval  ~/my-projects/chatbot-sim-eval
cd ~/my-projects/chatbot-sim-eval
```

(Optional) Edit `SPEC.md` to change the persona library, the evaluation rubric, the turn ceiling, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SimulatedUserAgent** — AutonomousAgent that plays a CX customer persona through a configurable maximum number of dialogue turns, responding to each assistant reply as a real user would.
- **ChatbotAgent** — AutonomousAgent that plays the support assistant, answering each simulated-user turn using the configured knowledge base context.
- **EvaluatorAgent** — AutonomousAgent that scores a completed dialogue transcript against a fixed rubric, returning either `PASS` or `FAIL` with a typed `EvalFinding` payload.
- **SimulationWorkflow** — Workflow that runs the turn loop between SimulatedUserAgent and ChatbotAgent, then hands the transcript to EvaluatorAgent when the conversation concludes or the turn ceiling is hit.
- **SimulationEntity** — EventSourcedEntity that holds the simulation lifecycle: every turn, the transcript, and the evaluator's final verdict.
- **ScenarioQueue** — EventSourcedEntity that logs each submitted scenario for replay and audit.
- **SimulationsView** — read-side projection that the UI lists and streams via SSE.
- **ScenarioConsumer** — Consumer that starts a workflow per inbound scenario submission.
- **ScenarioSimulator** — TimedAction that drips a canned scenario every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-simulation eval event each cycle (control E1).
- **SimulationEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the scenario briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Simulation` record fields (e.g., raise `maxTurns`, add persona attributes).
- `prompts/simulated-user.md` — change the persona or add new persona archetypes.
- `prompts/chatbot.md` — adjust the knowledge base context or the reply style.
- `prompts/evaluator.md` — change the rubric (e.g., add a compliance dimension).
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory for turns that exceed the safe-content policy).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a scenario → simulation progresses `RUNNING` → `EVALUATING` → `PASSED` within the turn ceiling; the App UI shows every turn's exchange.
2. Force-fail rubric → simulation hits `FAILED_EVALUATION` after the dialogue concludes; the entity preserves every turn and the evaluator's findings for audit.
3. The output guardrail flags a chatbot reply that violates the safe-content policy; the turn is still logged, the evaluator sees the violation in the transcript.
4. Each completed dialogue emits an `EvalRecorded` event that surfaces in the App UI's per-turn timeline.

## License

Apache 2.0.
