# Akka Sample: RAGAS Evaluation for Agents

A retrieval agent answers questions drawn from a document corpus; a RAGAS evaluation agent scores each answer against faithfulness, answer relevance, and context precision; the pair iterate until the answer meets the quality threshold or the harness hits its attempt ceiling. Demonstrates the **evaluator-optimizer** coordination pattern applied to continuous RAG quality assurance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the question corpus and the retrieval context are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.ragas-eval-harness  ~/my-projects/ragas-eval-harness
cd ~/my-projects/ragas-eval-harness
```

(Optional) Edit `SPEC.md` to change the quality thresholds, the metric weights, the maximum retry ceiling, or the sample question set.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RetrievalAgent** — AutonomousAgent that answers a question by retrieving context chunks from the in-memory corpus and composing a grounded response; accepts prior RAGAS feedback when revising.
- **RagasEvaluatorAgent** — AutonomousAgent that scores a (question, answer, context) triple against three RAGAS metrics — faithfulness, answer relevance, and context precision — and returns either `PASS` or `RETRY` with per-metric scores and a structured feedback record.
- **EvalWorkflow** — Workflow that runs the answer → evaluate → revise loop up to a configurable attempt ceiling, transitions the evaluation run to `PASSED` on evaluator approval or `FAILED_FINAL` when the ceiling is hit.
- **EvalRunEntity** — EventSourcedEntity holding the evaluation run lifecycle, every attempt's answer, every RAGAS score set, and the final outcome.
- **QuestionQueue** — EventSourcedEntity that logs each submitted question for replay and audit.
- **EvalRunsView** — read-side projection that the UI lists and streams via SSE.
- **QuestionConsumer** — Consumer that starts a workflow per inbound question.
- **QuestionSimulator** — TimedAction that drips a sample question every 60 s so the App UI is never empty.
- **ScoreSampler** — TimedAction that records a per-attempt eval event each cycle (control E1).
- **EvalEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the questions the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `EvalRun` record fields (e.g., raise `maxAttempts`, tighten the RAGAS score threshold).
- `prompts/retrieval-agent.md` — narrow the retrieval strategy or the corpus scope.
- `prompts/ragas-evaluator-agent.md` — change the metric weights or the acceptance threshold.
- `eval-matrix.yaml` — switch the score-regression ci-gate from build-gate to blocking.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a question → evaluation run progresses `ANSWERING` → `EVALUATING` → `PASSED` within the retry ceiling.
2. Force-fail thresholds → run hits `FAILED_FINAL` after the configured number of attempts; the entity preserves every attempt and every score set for audit.
3. The score-floor guardrail blocks an answer whose faithfulness score is below the minimum, so the evaluator never scores an ungrounded answer.
4. Each completed cycle emits a `ScoreRecorded` event that surfaces in the App UI's per-attempt timeline.
5. The CI gate blocks a deployment when the rolling pass rate over the last 100 runs falls below the configured threshold.

## License

Apache 2.0.
