# Akka Sample: Non-AI Agents (Score Aggregator)

A candidate application moves through four phases — **SCREEN → SCORE → AGGREGATE → NOTIFY** — driven by a mix of an LLM agent (`ScreeningAgent`) and two deterministic Java components (`ScoreAggregator`, `StatusNotifier`) wired as agents alongside it in a unified workflow. Each phase has its own typed input, typed output, and a pre-deploy test gate that validates the deterministic components before they can be promoted.

Demonstrates the **sequential-pipeline** coordination pattern with a single governance mechanism: a **CI test gate** that verifies `ScoreAggregator` and `StatusNotifier` against unit tests before the service starts. Because these two components are pure Java with no LLM calls, their behaviour is reproducible and their gate is reliable — a failed test stops the pipeline before any candidate data enters the system.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every screening, scoring, aggregation, and notification tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.hr-recruiting.mixed-deterministic-agents  ~/my-projects/score-aggregator
cd ~/my-projects/score-aggregator
```

(Optional) Edit `SPEC.md` to adjust the seeded candidate set, change the scoring weights, or narrow `ScreeningAgent`'s role to a specific job family.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ScreeningAgent** — one AutonomousAgent declaring two Task constants (`SCREEN_RESUME`, `GENERATE_RECOMMENDATION`); runs during the SCREEN and RECOMMEND phases. The LLM handles judgment calls: does the resume match the role? What recommendation should the hiring manager see?
- **ScoreAggregator** — a deterministic Java agent (no LLM). Receives `ScreenResult` and applies a weighted rubric across five dimensions (skills match, experience years, role seniority, location preference, screening flag). Returns a `CandidateScore`. Unit-testable; the CI gate validates it before deploy.
- **StatusNotifier** — a deterministic Java agent (no LLM). Reads the final `AggregatedScore` and mints a `StatusUpdate` record that a downstream HR system would consume. No model calls; no external I/O in this blueprint — the notification is recorded on `ApplicationEntity`.
- **ApplicationPipelineWorkflow** — four steps: `screenStep → scoreStep → aggregateStep → notifyStep`. Each step calls the relevant component, reads the typed result, writes the corresponding event onto `ApplicationEntity`, and advances.
- **ApplicationEntity** — an EventSourcedEntity holding the per-application lifecycle (`ApplicationCreated`, `ScreeningStarted`, `ScreeningCompleted`, `ScoringStarted`, `ScoringCompleted`, `AggregationCompleted`, `NotificationRecorded`, `ApplicationFailed`).
- **ScoreTools / RecommendTools** — two function-tool classes registered on `ScreeningAgent`, one per LLM phase. Tools are gated to their phase by the `PhaseGuard` check inside the workflow step.
- **QualityGate** — deterministic, rule-based on-decision evaluator that runs after `AggregationCompleted`. Checks that the aggregated score is within the expected range, that all five dimension scores are present, and that the recommendation text is non-empty. Emits a `GateResult` (pass/fail + reason) recorded on the entity.
- **ApplicationView + ApplicationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded candidate set in `src/main/resources/sample-events/applications.jsonl` to fit your demo role.
- `SPEC.md §4` and `prompts/screening-agent.md` — narrow the agent's role (e.g., constrain it to engineering roles, to executive search, to entry-level screening) by tightening the system prompt and adjusting the scoring rubric weights.
- `SPEC.md §5` — extend the typed outputs (`ScreenResult`, `CandidateScore`, `AggregatedScore`) with role-specific fields. The scoring logic in `ScoreAggregator` does not need an LLM and can be edited directly.
- `eval-matrix.yaml` — wire a stricter test gate (e.g., add integration tests that verify `ScoreAggregator` against known candidate fixtures) by editing the `H1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a candidate → `SCREEN` runs (LLM) → `SCORE` runs (deterministic) → `AGGREGATE` runs (deterministic) → `NOTIFY` runs (deterministic) → a typed `StatusUpdate` lands in the UI within ~60 s. Every transition is visible in real time.
2. `ScoreAggregator` and `StatusNotifier` have passing unit tests; the CI gate is the evidence that the deterministic path is correct before any live candidate data enters the system.
3. Every `AggregatedScore` emitted has a `QualityGate` result visible on the same UI card; scores outside the valid range or missing dimension scores cause a gate failure flagged in the UI.
4. The LLM's `ScreeningAgent` is the only component that makes a model call. Every other phase result is deterministic and reproducible.

## License

Apache 2.0.
