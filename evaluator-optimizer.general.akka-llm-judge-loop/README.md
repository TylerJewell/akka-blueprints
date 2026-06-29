# Akka Sample: LLM-as-a-Judge Evaluator-Optimizer

A generator agent produces an answer to a user-supplied question; a judge agent scores the answer against a rubric and returns feedback; the loop continues until the judge accepts the answer or the workflow reaches its attempt ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound question stream and the scoring loop are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.akka-llm-judge-loop  ~/my-projects/llm-judge-loop
cd ~/my-projects/llm-judge-loop
```

(Optional) Edit `SPEC.md` to change the generator's persona, the judge's rubric, the scoring threshold, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GeneratorAgent** — AutonomousAgent that produces an answer to a question, incorporating prior judge feedback on revision calls.
- **JudgeAgent** — AutonomousAgent that scores an answer against a rubric, returning either `ACCEPT` or `REVISE` with a typed `JudgeFeedback` payload.
- **EvaluationWorkflow** — Workflow that runs the generate → judge → revise loop up to a configurable attempt ceiling, transitions the evaluation to `ACCEPTED` on judge approval or to `REJECTED_FINAL` when the ceiling is hit.
- **EvaluationEntity** — EventSourcedEntity that holds the evaluation lifecycle, every attempt's answer, every judgment, and the final outcome.
- **QuestionQueue** — EventSourcedEntity that logs each submitted question for replay and audit.
- **EvaluationsView** — read-side projection that the UI lists and streams via SSE.
- **QuestionConsumer** — Consumer that starts a workflow per inbound submission.
- **QuestionSimulator** — TimedAction that drips a sample question every 60 s so the App UI is never empty.
- **JudgeSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **EvaluationEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the questions the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Evaluation` record fields (e.g., raise `maxAttempts`, lower the score acceptance threshold).
- `prompts/generator.md` — narrow the generator's domain expertise (e.g., legal research, code review, scientific fact-checking).
- `prompts/judge.md` — change the rubric (e.g., add a citation-required dimension).
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory on the judge's feedback).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a question → evaluation progresses `GENERATING` → `JUDGING` → `ACCEPTED` within the retry ceiling.
2. Force-fail rubric → evaluation hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every attempt and every judgment for audit.
3. The output guardrail blocks an answer that fails a structural check before the judge runs, returning a deterministic feedback note.
4. Each completed cycle emits a `JudgmentRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
