# Akka Sample: Brand-Aligned Presentations

A builder agent assembles a slide-by-slide presentation draft on a given topic; a brand reviewer agent scores each slide set against brand guidelines (tone, colour palette references, logo-usage rules, messaging hierarchy); the two pass feedback back and forth until the reviewer approves or the loop hits its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound request stream and the outbound presentation surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.sales-marketing.brand-presentation-builder  ~/my-projects/brand-presentation-builder
cd ~/my-projects/brand-presentation-builder
```

(Optional) Edit `SPEC.md` to change the brand ruleset, the acceptance rubric, the per-slide word ceiling, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BuilderAgent** — AutonomousAgent that drafts a slide deck on a brief, observing word-count limits per slide and applying prior reviewer feedback on revision passes.
- **BrandReviewerAgent** — AutonomousAgent that scores a slide set against brand guidelines, returns either `APPROVE` or `REVISE` with a typed `BrandFeedback` payload.
- **PresentationWorkflow** — Workflow that runs the build → brand-check → revise loop up to a configurable retry ceiling, transitions the presentation to `APPROVED` on reviewer approval or to `REJECTED_FINAL` when the ceiling is hit.
- **PresentationEntity** — EventSourcedEntity that holds the presentation lifecycle, every attempt's slide set, every round of brand feedback, and the final outcome.
- **RequestQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **PresentationsView** — read-side projection that the UI lists and streams via SSE.
- **PresentationRequestConsumer** — Consumer that starts a workflow per inbound submission.
- **RequestSimulator** — TimedAction that drips a sample brief every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **PresentationEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Presentation` record fields (e.g., raise `maxAttempts`, lower the per-slide word ceiling, add extra slide types).
- `prompts/builder.md` — narrow the slide structure to a different format (e.g., only three slides, executive summary format).
- `prompts/brand-reviewer.md` — change the rubric (e.g., add a forbidden-phrase list, tighten colour-palette checks).
- `eval-matrix.yaml` — tighten enforcement on the brand-compliance guardrail (currently non-blocking advisory on over-length slides).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a brief → presentation progresses `BUILDING` → `REVIEWING` → `APPROVED` within the retry ceiling.
2. Force-fail rubric → presentation hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every attempt and every round of feedback for audit.
3. The brand guardrail blocks a slide set with an over-ceiling word count on any slide, so the reviewer never sees non-compliant material.
4. Each completed cycle emits a `BrandEvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
