# Akka Sample: Evals with Memory

An answer agent responds to user questions by retrieving relevant context from a long-term memory store; a scorer agent evaluates each response against a quality rubric; the system records every verdict as an eval event and runs a periodic drift check across sessions. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance over a stateful memory subsystem.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the memory store and the evaluation pipeline are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.memory-eval-loop  ~/my-projects/memory-eval-loop
cd ~/my-projects/memory-eval-loop
```

(Optional) Edit `SPEC.md` to change the retrieval window, the scoring rubric, the drift-check interval, or the PII sanitization rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AnswerAgent** — AutonomousAgent that retrieves relevant memory entries for a question and produces a cited answer.
- **ScorerAgent** — AutonomousAgent that scores an answer against a quality rubric (relevance, grounding, completeness, coherence) and returns either `PASS` or `IMPROVE` with structured feedback.
- **AnswerWorkflow** — Workflow that runs the retrieve → answer → score loop up to a configurable retry ceiling, transitions the session turn to `ACCEPTED` on scorer approval or to `REJECTED_FINAL` when the ceiling is hit.
- **SessionEntity** — EventSourcedEntity that holds the conversation turn lifecycle, every answer attempt, every score, and the final outcome.
- **MemoryStore** — EventSourcedEntity that accumulates sanitized memory entries for a user across sessions; the retrieve step reads from its view.
- **MemoryView** — read-side projection that `AnswerWorkflow` queries for retrieval candidates and that the UI streams via SSE.
- **SessionView** — read-side projection that the UI lists and streams via SSE.
- **QuestionConsumer** — Consumer that starts a workflow per inbound question.
- **QuestionSimulator** — TimedAction that drips a sample question every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-answer eval event each cycle (control E1).
- **DriftWatcher** — TimedAction that runs a periodic drift check across all sessions (control E2).
- **SessionEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the questions the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Session` record fields (e.g., raise `maxAttempts`, change the memory retrieval window).
- `prompts/answer-agent.md` — narrow the answer domain or change the citation format.
- `prompts/scorer-agent.md` — change the rubric (e.g., add a factual-accuracy dimension).
- `eval-matrix.yaml` — tighten enforcement on the PII sanitizer (currently system-level) or the drift watcher (currently non-blocking).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a question → session turn progresses `ANSWERING` → `SCORING` → `ACCEPTED` within the retry ceiling; the App UI shows every attempt's answer, its memory citations, and the scorer's verdict.
2. Force-fail rubric → session turn hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every attempt and every score for audit.
3. The PII sanitizer strips a detected PII token before the entry is written to `MemoryStore`; the raw value never reaches the persistent store.
4. The drift watcher fires on schedule and records a `DriftCheckRecorded` event; the App UI surfaces the drift report.

## License

Apache 2.0.
