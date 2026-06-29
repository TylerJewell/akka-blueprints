# Akka Sample: Deterministic Multi-Stage Agent Pipeline

A single `StoryAgent` walks a creative prompt through three fixed stages — **OUTLINE → BODY → ENDING** — wired together by explicit stage dependencies. Each stage has its own typed input, typed output, and a dedicated set of stage-specific tools. The user submits a prompt and receives a finished `Story`.

Demonstrates the **sequential-pipeline** coordination pattern with one governance mechanism: an `after-llm-response` guardrail that validates each stage's output before feeding it to the next stage. No stage's output enters the next stage's instruction context until it passes the structural check.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every outline / body / ending tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.akka-deterministic-stages  ~/my-projects/story-pipeline
cd ~/my-projects/story-pipeline
```

(Optional) Edit `SPEC.md` to point at a different prompt seed list, a different model provider, or richer stage-specific tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **StoryAgent** — one AutonomousAgent declaring three Task constants (`OUTLINE_STORY`, `WRITE_BODY`, `WRITE_ENDING`); the workflow runs them in order, feeding each output forward as the next stage's instruction context.
- **StoryPipelineWorkflow** — runs `outlineStep → bodyStep → endingStep → validateStep`. Each step calls `runSingleTask` and writes the typed result back onto `StoryEntity` before the next step starts.
- **StoryEntity** — an EventSourcedEntity holding the per-story lifecycle (`OutlineProduced`, `BodyWritten`, `EndingWritten`, `StoryValidated`).
- **OutlineTools / BodyTools / EndingTools** — three function-tool classes registered on the agent, one per stage. The `after-llm-response` guardrail enforces that each stage's output is structurally sound before the workflow advances.
- **StageOutputGuardrail** — the runtime check that validates each stage result. A stage output missing required fields or violating structural constraints is rejected; the agent retries within its iteration budget.
- **StoryStructureValidator** — deterministic, rule-based structural validator that runs immediately after `EndingWritten` and emits a 1–5 coherence score.
- **StoryView + StoryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded prompt set under `src/main/resources/sample-events/prompts.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/story-agent.md` — narrow the agent's role (e.g., constrain it to technical documentation, to marketing copy, to product announcements) by tightening the system prompt and renaming the typed records (`Outline`, `Body`, `Ending`).
- `SPEC.md §5` — extend the typed outputs (`Outline`, `Body`, `Ending`) with genre-specific fields. The stage-output guardrail does not need editing — it checks structural completeness on the typed result, not field shapes.
- `eval-matrix.yaml` — wire a real coherence scorer (replace the deterministic stub with a vector-similarity check against a reference corpus) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prompt → `OUTLINE` runs → `BODY` runs → `ENDING` runs → a typed `Story` lands in the UI within ~60 s. Every transition is visible in real time.
2. A stage output missing required fields (forced via the mock LLM) → the `after-llm-response` guardrail rejects it → the workflow records the rejection event → the agent retries → the pipeline completes correctly.
3. Every `Story` emitted has a structural-validation score visible on the same UI card; stories whose body does not reference any outline beat receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the OUTLINE task does not see the ending instructions, and the ENDING task does not see raw outline beats — the workflow's stage-chaining is the only path information travels between stages.

## License

Apache 2.0.
