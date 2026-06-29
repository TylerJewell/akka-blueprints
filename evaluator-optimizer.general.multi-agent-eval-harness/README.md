# Akka Sample: Multi-Agent Evaluation Harness

An orchestrator agent drives a set of specialist agents through a scenario suite; a judge agent scores the aggregate result against a quality rubric; the harness blocks CI promotion when scores fall below threshold and continuously records per-scenario eval events for quality trending.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the scenario suite, the agent-under-test stubs, and the judgment surface are all modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.multi-agent-eval-harness  ~/my-projects/multi-agent-eval-harness
cd ~/my-projects/multi-agent-eval-harness
```

(Optional) Edit `SPEC.md` to change the scenario set, the judgment rubric, the pass threshold, or the retry ceiling for flaky scenarios.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OrchestratorAgent** — AutonomousAgent that dispatches each scenario to the appropriate specialist agents and collects their raw responses.
- **JudgeAgent** — AutonomousAgent that scores aggregate scenario results against a rubric, returning either `PASS` or `FAIL` with structured feedback.
- **EvalRunWorkflow** — Workflow that drives scenario loading → agent dispatch → judgment → threshold check, and records the terminal outcome.
- **EvalRunEntity** — EventSourcedEntity that holds the eval run lifecycle, every scenario result, every judgment, and the final pass/fail outcome.
- **ScenarioRegistry** — EventSourcedEntity that stores the scenario definitions used by a run.
- **EvalRunsView** — read-side projection that the UI lists and streams via SSE.
- **ScenarioResultConsumer** — Consumer that starts a workflow per submitted eval run.
- **ScenarioLoader** — TimedAction that drips a canned eval run every 120 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records per-scenario eval events each cycle (control E1).
- **EvalEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change which scenario JSONL file the loader uses, or remove the loader entirely.
- `SPEC.md §5` — adjust the `EvalRun` record fields (e.g., raise `passThreshold`, add scenario categories).
- `prompts/orchestrator.md` — narrow how the orchestrator selects and delegates scenarios.
- `prompts/judge.md` — change the rubric dimensions or the numeric scoring range.
- `eval-matrix.yaml` — tighten the CI gate threshold or promote the eval-periodic control from non-blocking to blocking.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a scenario suite → eval run progresses `RUNNING` → `JUDGING` → `PASSED` within the configured pass threshold.
2. Submit a suite where agents return poor results → eval run lands in `FAILED` with every scenario result preserved and a structured failure reason.
3. The CI gate control fires when the aggregate score falls below the threshold; a failing run emits a `CIGateBlocked` event that surfaces in the App UI.
4. Each completed scenario emits a `ScenarioEvalRecorded` event visible in the per-run timeline.

## License

Apache 2.0.
