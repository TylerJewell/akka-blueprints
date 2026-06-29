# Akka Sample: YouTube to Blog

A single `BlogAgent` walks a YouTube URL through four task phases — **TRANSCRIPT → SUMMARY → DRAFT → POLISH** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a URL and receives a polished `BlogPost`.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-agent-response` guardrail that moderates the final published output before it leaves the agent, and an `on-decision-eval` evaluator that scores every emitted blog post for quality, accuracy alignment, and source attribution at each pipeline step.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every transcript / summary / draft / polish tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.video-to-blog-pipeline  ~/my-projects/video-to-blog
cd ~/my-projects/video-to-blog
```

(Optional) Edit `SPEC.md` to point at a different set of seeded YouTube URLs, a different model provider, or a richer set of editorial tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BlogAgent** — one AutonomousAgent declaring four Task constants (`EXTRACT_TRANSCRIPT`, `SUMMARISE_VIDEO`, `DRAFT_POST`, `POLISH_POST`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **BlogPipelineWorkflow** — runs `transcriptStep → summaryStep → draftStep → polishStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `BlogPostEntity` before the next step starts.
- **BlogPostEntity** — an EventSourcedEntity holding the per-post lifecycle (`TranscriptExtracted`, `SummaryProduced`, `DraftWritten`, `PostPolished`, `EvaluationScored`).
- **TranscriptTools / SummaryTools / DraftTools / PolishTools** — four function-tool classes registered on the agent, one per phase.
- **PublishGuardrail** — a `before-agent-response` guardrail that runs on the final POLISH task's response, checking for prohibited content patterns and brand-safety rules before the `PostPolished` event is recorded.
- **EditorialScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `PostPolished` and emits a 1–5 quality score.
- **BlogPostView + BlogEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded URL set under `src/main/resources/sample-events/urls.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/blog-agent.md` — narrow the agent's role (e.g., constrain it to technical tutorials, to product-launch posts, to tutorial-style educational content) by tightening the system prompt and renaming the typed records (`Transcript`, `VideoSummary`, `BlogDraft`, `BlogPost`).
- `SPEC.md §5` — extend the typed outputs with domain-specific fields. The guardrail moderation rules do not need editing — they check the polished post's text, not field shapes.
- `eval-matrix.yaml` — wire a real quality evaluator (replace the deterministic stub with an LLM-as-judge call) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a YouTube URL → `TRANSCRIPT` runs → `SUMMARY` runs → `DRAFT` runs → `POLISH` runs → a typed `BlogPost` lands in the UI within ~90 s. Every transition is visible in real time.
2. The mock LLM's polish step response contains a prohibited phrase → the `before-agent-response` guardrail rejects the response → the workflow records the rejection event → the agent retries within its iteration budget → the pipeline completes with clean content.
3. Every `BlogPost` emitted has an on-decision eval score visible on the same UI card; posts whose draft section count does not match the summary section count receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the TRANSCRIPT task does not see the draft instructions, and the DRAFT task does not see the raw transcript — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
