# Akka Sample: SEO Article Evaluation

A rubric agent audits a blog post against a 21-point helpful-content checklist; an optimizer agent proposes targeted revisions; the two exchange until the rubric passes or the loop hits its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the article feed and the revision surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.content-editorial.seo-rubric-critic  ~/my-projects/seo-rubric-critic
cd ~/my-projects/seo-rubric-critic
```

(Optional) Edit `SPEC.md` to change the checklist weights, the acceptance threshold, the maximum revision rounds, or the article length cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RubricAgent** — AutonomousAgent that scores a blog post against the 21-point helpful-content checklist and returns a typed `RubricScore` with a pass/fail verdict and per-dimension breakdowns.
- **OptimizerAgent** — AutonomousAgent that rewrites or patches specific sections of the article in response to `RubricFeedback`, returning a revised `ArticleDraft`.
- **AuditWorkflow** — Workflow that runs the score → revise loop up to a configurable ceiling, transitions the article to `APPROVED` on rubric pass or to `FAILED_FINAL` when the ceiling is hit.
- **ArticleEntity** — EventSourcedEntity that holds the article lifecycle, every draft, every rubric score, and the final outcome.
- **SubmissionQueue** — EventSourcedEntity that logs each article submission for replay and audit.
- **ArticlesView** — read-side projection the UI lists and streams via SSE.
- **SubmissionConsumer** — Consumer that starts a workflow per inbound article.
- **ArticleSimulator** — TimedAction that drips a sample article brief every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-round eval event each cycle (control E1).
- **AuditEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample articles the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Article` record fields (e.g., raise `maxRounds`, change the acceptance threshold).
- `prompts/rubric-agent.md` — swap in a different checklist (e.g., E-E-A-T criteria, brand-voice rubric).
- `prompts/optimizer-agent.md` — change the revision strategy (e.g., restrict to headline-only edits).
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an article → it progresses `SCORING` → `REVISING` → `APPROVED` within the retry ceiling.
2. Submit an article with an unfixable defect → it reaches `FAILED_FINAL` after the configured number of rounds; the entity preserves every draft and every rubric score for audit.
3. The output guardrail blocks an article draft that exceeds the word-count ceiling, so the rubric agent never sees over-length material.
4. Each completed round emits an `EvalRecorded` event that surfaces in the App UI's per-round timeline.

## License

Apache 2.0.
