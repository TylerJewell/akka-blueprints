# Akka Sample: Brand Aligner

A copywriter agent generates marketing material variants; a brand-compliance reviewer agent scores each variant against brand standards (tone, vocabulary, visual-reference policy, and claim accuracy); the two iterate until the reviewer accepts a variant or the loop reaches its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the inbound brief stream and the outbound review surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.sales-marketing.brand-aligner  ~/my-projects/brand-aligner
cd ~/my-projects/brand-aligner
```

(Optional) Edit `SPEC.md` to change the brand-standards rubric, the accepted variant criteria, the word-count ceiling, or the retry cap.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CopywriterAgent** — AutonomousAgent that generates a marketing copy variant on a brief, observing a word-count ceiling and brand-voice guidelines; accepts prior reviewer feedback when revising.
- **BrandReviewerAgent** — AutonomousAgent that scores a variant against the brand rubric (tone, vocabulary, claim accuracy, visual-reference compliance), returns either `APPROVE` or `REVISE` with a typed `ReviewNotes` payload.
- **AlignmentWorkflow** — Workflow that runs the generate → compliance-check → review → revise loop up to a configurable retry ceiling, transitions the material to `APPROVED` on reviewer acceptance or to `REJECTED_FINAL` when the ceiling is hit.
- **MaterialEntity** — EventSourcedEntity that holds the material lifecycle, every variant draft, every review, and the final outcome.
- **BriefQueue** — EventSourcedEntity that logs each submission for replay and audit.
- **MaterialsView** — read-side projection that the UI lists and streams via SSE.
- **BriefConsumer** — Consumer that starts a workflow per inbound submission.
- **CampaignSimulator** — TimedAction that drips a sample brief every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **BrandEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Material` record fields (e.g., raise `maxAttempts`, lower the word-count ceiling).
- `prompts/copywriter.md` — narrow the brand voice to a specific product tier or region.
- `prompts/brand-reviewer.md` — change the rubric (e.g., add a restricted-claim list).
- `eval-matrix.yaml` — tighten enforcement on the compliance-check control (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a brief → material progresses `DRAFTING` → `REVIEWING` → `APPROVED` within the retry ceiling.
2. Force-fail rubric → material hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every variant and every review for audit.
3. The compliance check blocks a variant whose word count exceeds the ceiling, so the reviewer never sees over-length copy.
4. Each completed cycle emits a `BrandEvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
