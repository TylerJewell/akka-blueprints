# Akka Sample: Blog Writer Pipeline

A single `BlogWriterAgent` walks a topic through three task phases — **RESEARCH → OUTLINE → DRAFT** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits a topic and style preferences and receives a structured `BlogPost`.

Demonstrates the **sequential-pipeline** coordination pattern wired with one governance mechanism: a `before-agent-response` guardrail that checks every generated draft for brand-voice conformance and content-quality standards before the post is recorded as complete.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every research, outline, and draft tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.blog-writer-pipeline  ~/my-projects/blog-writer-pipeline
cd ~/my-projects/blog-writer-pipeline
```

(Optional) Edit `SPEC.md` to point at a different topic seed list, a different model provider, or different brand-voice rules in the guardrail.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BlogWriterAgent** — one AutonomousAgent declaring three Task constants (`RESEARCH_TOPIC`, `OUTLINE_POST`, `DRAFT_POST`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **BlogWritingWorkflow** — runs `researchStep → outlineStep → draftStep → qualityCheckStep`. Each step calls `runSingleTask` and writes the typed result back onto `BlogPostEntity` before the next step starts.
- **BlogPostEntity** — an EventSourcedEntity holding the per-post lifecycle (`ResearchCollected`, `OutlineProduced`, `DraftWritten`, `QualityChecked`).
- **ResearchTools / OutlineTools / DraftTools** — three function-tool classes registered on the agent, one per phase. The `before-agent-response` guardrail enforces that the final draft output passes brand-voice and quality checks before recording.
- **BrandGuardrail** — the runtime check that backs the quality contract. A draft that fails minimum-word-count, forbidden-phrase, or required-CTA checks is rejected before the draft is recorded on the entity; the agent retries within its iteration budget.
- **QualityScorer** — deterministic, rule-based quality evaluator that runs immediately after `DraftWritten` and emits a 1–5 score.
- **BlogPostView + BlogPostEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded topic set under `src/main/resources/sample-events/topics.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/blog-writer-agent.md` — narrow the agent's role (e.g., constrain it to technical tutorials, to product-announcement posts, to thought-leadership pieces) by tightening the system prompt and renaming the typed records (`BlogPost`, `Outline`, `ResearchNotes`).
- `SPEC.md §5` — extend the typed outputs (`ResearchNotes`, `Outline`, `BlogPost`) with brand-specific fields. The brand guardrail's rule set does not need editing — it checks declared rules, not field shapes.
- `eval-matrix.yaml` — wire a real tone-analysis evaluator (replace the deterministic stub with an LLM-judge call) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a topic → `RESEARCH` runs → `OUTLINE` runs → `DRAFT` runs → a typed `BlogPost` lands in the UI within ~60 s. Every transition is visible in real time.
2. A draft that fails the minimum-word-count rule (forced via the mock LLM) → the `before-agent-response` guardrail rejects it → the agent retries with a longer draft → the pipeline completes correctly.
3. Every `BlogPost` emitted has an on-decision quality score visible on the same UI card; posts whose sections contain no call-to-action receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the RESEARCH task does not see the draft instructions, and the DRAFT task does not see raw research notes directly — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
