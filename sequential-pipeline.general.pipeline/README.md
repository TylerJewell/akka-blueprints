# Akka Sample: Pipeline Briefing

A single `ReportAgent` walks a topic through three task phases — **COLLECT → ANALYZE → REPORT** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a topic and receives a structured `Briefing`.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that blocks tools when their phase's preconditions are not yet met, and an `on-decision-eval` evaluator that scores every emitted briefing for completeness and source attribution.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every collect / analyze / report tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.general.pipeline  ~/my-projects/pipeline
cd ~/my-projects/pipeline
```

(Optional) Edit `SPEC.md` to point at a different topic seed list, a different model provider, or a richer set of analysis tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReportAgent** — one AutonomousAgent declaring three Task constants (`COLLECT_SIGNALS`, `ANALYZE_SIGNALS`, `WRITE_REPORT`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **BriefingPipelineWorkflow** — runs `collectStep → analyzeStep → reportStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `BriefingEntity` before the next step starts.
- **BriefingEntity** — an EventSourcedEntity holding the per-briefing lifecycle (`SignalsCollected`, `AnalysisProduced`, `ReportWritten`, `EvaluationScored`).
- **CollectTools / AnalyzeTools / ReportTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase.
- **PhaseGuardrail** — the runtime check that backs the dependency contract. A tool call referencing a phase whose precondition has not yet been recorded on the entity is rejected before the tool runs.
- **CompletenessScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `ReportWritten` and emits a 1–5 score.
- **BriefingView + BriefingEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded topic set under `src/main/resources/sample-events/topics.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/report-agent.md` — narrow the agent's role (e.g., constrain it to security advisories, to product-release notes, to vendor-risk briefs) by tightening the system prompt and renaming the typed records (`Briefing`, `Section`, `Theme`).
- `SPEC.md §5` — extend the typed outputs (`SignalSet`, `Analysis`, `Briefing`) with industry-specific fields. The phase-gating guardrail does not need editing — it checks recorded-phase preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real source-attribution evaluator (replace the deterministic stub with a vector-similarity check against a corpus of real signals) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a topic → `COLLECT` runs → `ANALYZE` runs → `REPORT` runs → a typed `Briefing` lands in the UI within ~60 s. Every transition is visible in real time.
2. A tool from a later phase is called out of order (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries in-phase → the pipeline completes correctly.
3. Every `Briefing` emitted has an on-decision eval score visible on the same UI card; reports whose sections cite no source receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the COLLECT task does not see the report instructions, and the REPORT task does not see raw signal fetches — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
