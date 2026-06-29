# Akka Sample: Evaluator-Optimizer Workflow

A writer agent produces a candidate article headline; a reviewer agent critiques it against editorial standards; the two pass feedback back and forth until the reviewer accepts the headline or the loop hits its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound request stream and the outbound editorial surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.content-editorial.evaluator-optimizer-pattern  ~/my-projects/evaluator-optimizer-workflow
cd ~/my-projects/evaluator-optimizer-workflow
```

(Optional) Edit `SPEC.md` to change the editorial brief format, the acceptance rubric, the per-headline word ceiling, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WriterAgent** — AutonomousAgent that drafts a headline from an article brief, accepting prior reviewer feedback when revising.
- **ReviewerAgent** — AutonomousAgent that scores a headline against editorial criteria, returns either `APPROVE` or `REVISE` with a typed `ReviewNotes` payload.
- **HeadlineWorkflow** — Workflow that runs the draft → review → revise loop up to a configurable retry ceiling, transitions the headline to `APPROVED` on reviewer approval or to `REJECTED_FINAL` when the ceiling is hit.
- **HeadlineEntity** — EventSourcedEntity that holds the headline lifecycle, every attempt's draft, every review, and the final outcome.
- **SubmissionQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **HeadlinesView** — read-side projection that the UI lists and streams via SSE.
- **SubmissionConsumer** — Consumer that starts a workflow per inbound submission.
- **BriefSimulator** — TimedAction that drips a sample article brief every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **EditorialEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Headline` record fields (e.g., raise `maxAttempts`, lower the per-headline word ceiling).
- `prompts/writer.md` — narrow the editorial style (e.g., tabloid, academic journal, long-form newsletter).
- `prompts/reviewer.md` — change the rubric (e.g., add an SEO-keyword constraint).
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory for the terminal response check).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a brief → headline progresses `DRAFTING` → `REVIEWING` → `APPROVED` within the retry ceiling.
2. Force-fail rubric → headline hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every attempt and every review for audit.
3. The output guardrail blocks a draft that violates the word-count ceiling, so the reviewer never sees an over-length headline.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
