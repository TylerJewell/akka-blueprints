# Akka Sample: Self-Critiquing Writer Loop

A writer agent drafts a content passage on a given brief; a separate critic agent evaluates it against a quality rubric; the two iterate until the critic accepts the draft or the loop exhausts its iteration budget. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the inbound request stream and the outbound writing surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.content-editorial.self-critiquing-writer-loop  ~/my-projects/self-critiquing-writer-loop
cd ~/my-projects/self-critiquing-writer-loop
```

(Optional) Edit `SPEC.md` to change the briefs the simulator drips, the acceptance rubric, the iteration budget, or the quality threshold.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WriterAgent** — AutonomousAgent that drafts a content passage on a brief, incorporating prior critique notes on revision calls.
- **CriticAgent** — AutonomousAgent that scores a draft against a quality rubric, returning either `ACCEPT` with a rationale or `REVISE` with a typed `CritiqueNotes` payload.
- **RefinementWorkflow** — Workflow that runs the draft → critique → revise loop up to a configurable iteration budget, transitions the article to `ACCEPTED` on critic approval or to `BUDGET_EXHAUSTED` when the ceiling is hit.
- **ArticleEntity** — EventSourcedEntity that holds the article lifecycle, every attempt's draft, every critique, and the final outcome.
- **RequestQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **ArticlesView** — read-side projection that the UI lists and streams via SSE.
- **DraftRequestConsumer** — Consumer that starts a workflow per inbound submission.
- **RequestSimulator** — TimedAction that drips a sample brief every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **WritingEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Article` record fields (e.g., raise `maxIterations`, add a minimum word count).
- `prompts/writer.md` — narrow the register (e.g., journalistic, academic, marketing copy).
- `prompts/critic.md` — change the rubric (e.g., add a readability score constraint).
- `eval-matrix.yaml` — tighten enforcement on the budget-cap halt (currently system-level).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a brief → article progresses `DRAFTING` → `EVALUATING` → `ACCEPTED` within the iteration budget; the App UI shows every attempt's draft and critique.
2. Force-fail rubric → article hits `BUDGET_EXHAUSTED` after the configured number of iterations; the entity preserves every attempt and every critique for audit, with the best-scoring draft surfaced.
3. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.
4. The iteration budget fires the halt event on exhaustion; the workflow ends cleanly with a structured reason and best-of draft preserved.

## License

Apache 2.0.
