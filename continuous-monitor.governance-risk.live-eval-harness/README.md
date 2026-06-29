# Akka Sample: Live Evals

A continuous evaluation harness scores every agent output in production against rubrics, accumulates per-decision scores into aggregate metrics, and raises drift alarms when quality falls below a threshold. Demonstrates the **continuous-monitor** coordination pattern wired with two governance mechanisms (on-decision eval and periodic drift-fairness watch).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None**. The harness runs fully in-memory against a simulated decision stream.

## Generate the system

```sh
cp -r ./continuous-monitor.governance-risk.live-eval-harness  ~/my-projects/live-eval-harness
cd ~/my-projects/live-eval-harness
```

(Optional) Edit `SPEC.md §3` to point `DecisionPoller` at a real agent output stream, or keep the in-memory simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DecisionPoller** — TimedAction firing every 10 s that drips simulated agent decisions into `DecisionQueue`.
- **DecisionSanitizer** — Consumer that strips user-identifiable fields before any rubric judge sees them.
- **RubricEvalAgent** — Agent (typed) that scores a single decision against an operator-supplied rubric and returns `EvalResult`.
- **EvalOrchestrationWorkflow** — Workflow per decision: sanitize → eval → store result → check threshold.
- **DecisionEntity** — EventSourcedEntity holding the per-decision lifecycle (received → sanitized → evaluated → alarmed / ok).
- **DriftWatchAgent** — AutonomousAgent called by `DriftSampler` to detect fairness and quality drift across a rolling window.
- **DriftSampler** — TimedAction firing every 15 minutes; aggregates recent eval scores and calls `DriftWatchAgent`.
- **EvalView + EvalEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated decision stream for a real source (e.g., a Kafka topic carrying your agent's output events).
- `SPEC.md §5` — extend `DecisionRecord` with domain-specific fields (`agentId`, `tenantId`, `modelVersion`).
- `prompts/rubric-eval.md` — replace the default rubric with your own scoring dimensions.
- `eval-matrix.yaml` — add `regulation_anchors` for any regulatory framework your sector requires.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated decision arrives → sanitized → scored by the rubric judge → stored with a score.
2. A score below the alarm threshold triggers an `EvalAlarm` event visible in the UI.
3. `DriftSampler` fires and the drift agent either issues `DriftOk` or `DriftAlarm` on the rolling window.
4. All recent scores are visible on the App UI tab with their rubric breakdown.

## License

Apache 2.0.
