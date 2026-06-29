# Akka Sample: Sales Roleplay Coach

A buyer-simulator agent plays the role of a prospect with a configured deal context; a coach agent scores the rep's pitch against a structured rubric; the two exchange turns until the coach accepts the pitch or the session hits its attempt ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **a valid CRM credential if you enable live deal-context pull** (the same key-sourcing options as the LLM key apply). The blueprint runs without a CRM credential by using the built-in scenario library; CRM integration is opt-in.

## Generate the system

```sh
cp -r ./evaluator-optimizer.sales-marketing.sales-roleplay-coach  ~/my-projects/sales-roleplay-coach
cd ~/my-projects/sales-roleplay-coach
```

(Optional) Edit `SPEC.md` to change the buyer personas, the scoring rubric, the per-session attempt ceiling, or the scenario library.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BuyerSimulatorAgent** — AutonomousAgent that plays an AI-controlled buyer persona, responding to the rep's pitch with objections, questions, and signals calibrated to a deal context.
- **CoachAgent** — AutonomousAgent that scores each rep turn against the rubric, returns either `ACCEPT` (pitch ready for the real call) or `REVISE` with typed `CoachingNotes` identifying what to improve.
- **SessionWorkflow** — Workflow that runs the pitch → buyer-response → coach-score loop up to a configurable attempt ceiling, transitions the session to `PASSED` on coach approval or to `EXHAUSTED` when the ceiling is hit.
- **SessionEntity** — EventSourcedEntity that holds the session lifecycle, every rep turn, every buyer response, every coach verdict, and the final outcome.
- **ScenarioQueue** — EventSourcedEntity that logs each practice request for replay and audit.
- **SessionsView** — read-side projection that the UI lists and streams via SSE.
- **ScenarioRequestConsumer** — Consumer that starts a workflow per inbound request.
- **ScenarioSimulator** — TimedAction that drips a sample scenario every 90 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-turn eval event each cycle (control E1).
- **CoachEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the buyer personas the simulator uses, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Session` record fields (e.g., raise `maxTurns`, add a deal-stage filter).
- `prompts/buyer-simulator.md` — narrow the buyer to a specific industry, title, or objection style.
- `prompts/coach.md` — change the rubric (e.g., add a pricing-objection dimension).
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a practice request → session progresses `PITCHING` → `EVALUATING` → `PASSED` within the attempt ceiling.
2. Force-fail rubric → session hits `EXHAUSTED` after the configured number of turns; the entity preserves every turn and every coach verdict for review.
3. The output guardrail blocks a rep turn that contains prohibited content (competitor brand claims, pricing fabrications) before the buyer simulator sees it.
4. Each completed turn emits an `EvalRecorded` event that surfaces in the App UI's per-turn timeline.

## License

Apache 2.0.
