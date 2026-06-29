# Akka Sample: Multi-Step Query Engine

A single `QueryAgent` walks a research question through a sequence of refining steps — **DECOMPOSE → RETRIEVE → SYNTHESIZE** — each step building on the typed output of the previous one. The user submits a question and receives a structured `QueryAnswer` with cited evidence passages and a stopping-criterion evaluation score.

Demonstrates the **sequential-pipeline** coordination pattern wired with one governance mechanism: an `on-decision-eval` evaluator that scores every emitted answer for evidence coverage and stopping-criterion satisfaction.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every decompose, retrieve, and synthesize tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.research-intel.multi-step-query  ~/my-projects/multi-step-query
cd ~/my-projects/multi-step-query
```

(Optional) Edit `SPEC.md` to point at a different seed question set, a different model provider, or a richer retrieval corpus.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **QueryAgent** — one AutonomousAgent declaring three Task constants (`DECOMPOSE_QUESTION`, `RETRIEVE_EVIDENCE`, `SYNTHESIZE_ANSWER`); the workflow runs them in order, feeding each typed output forward as the next step's instruction context.
- **QueryPipelineWorkflow** — runs `decomposeStep → retrieveStep → synthesizeStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `QueryEntity` before the next step starts.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle (`QuestionDecomposed`, `EvidenceRetrieved`, `AnswerSynthesized`, `AnswerEvaluated`).
- **DecomposeTools / RetrieveTools / SynthesizeTools** — three function-tool classes registered on the agent, one per phase. Each class's tools are only meaningful in their own phase; `StoppingEvaluator` enforces the per-step criterion after every synthesize task.
- **StoppingEvaluator** — deterministic, rule-based on-decision evaluator that runs immediately after `AnswerSynthesized` and emits a 1–5 coverage score.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded question set under `src/main/resources/sample-events/questions.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/query-agent.md` — narrow the agent's role (e.g., constrain it to legal-document retrieval, to academic literature synthesis, to competitive-intelligence queries) by tightening the system prompt and renaming the typed records (`SubQuestion`, `EvidenceSet`, `QueryAnswer`).
- `SPEC.md §5` — extend the typed outputs (`SubQuestion`, `Passage`, `QueryAnswer`) with domain-specific fields. The per-step stopping evaluation does not need editing — it checks recorded step results, not field shapes.
- `eval-matrix.yaml` — wire a real evidence-quality evaluator (replace the deterministic stub with a relevance-scoring function against a live corpus) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a question → `DECOMPOSE` runs → `RETRIEVE` runs → `SYNTHESIZE` runs → a typed `QueryAnswer` lands in the UI within ~60 s. Every transition is visible in real time.
2. Every `QueryAnswer` emitted has an on-decision eval score visible on the same UI card; answers whose passages contain no direct evidence for a sub-question receive a score ≤ 2 and are flagged.
3. Each task receives only its own typed inputs; the DECOMPOSE task does not see synthesis instructions, and the SYNTHESIZE task does not see raw sub-question text — the workflow's task-chaining is the only path information travels between phases.
4. A question with no matching passages in the offline corpus returns an honestly empty answer without crashing the pipeline.

## License

Apache 2.0.
