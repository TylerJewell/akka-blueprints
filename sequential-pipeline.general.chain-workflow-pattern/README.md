# Akka Sample: Chain Workflow

A single `TransformAgent` walks a piece of content through three transformation stages — **EXTRACT → REFINE → FORMAT** — wired together by explicit typed handoffs. Each stage has its own typed input, typed output, and a stage-specific set of tools. The user submits raw content and receives a structured `Document`.

Demonstrates the **sequential-pipeline** coordination pattern via prompt chaining: each LLM call receives only the typed output of the previous call as its context. Two governance mechanisms are wired around the pipeline: an `after-llm-response` guardrail that validates every intermediate transform before it is passed downstream, and a deterministic output-quality evaluator that scores every emitted `Document` for structural completeness.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every extract / refine / format tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.general.chain-workflow-pattern  ~/my-projects/chain-workflow
cd ~/my-projects/chain-workflow
```

(Optional) Edit `SPEC.md` to point at a different content corpus, a different model provider, or a richer set of refinement tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TransformAgent** — one AutonomousAgent declaring three Task constants (`EXTRACT_STRUCTURE`, `REFINE_CONTENT`, `FORMAT_DOCUMENT`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **TransformPipelineWorkflow** — runs `extractStep → refineStep → formatStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `DocumentEntity` before the next step starts.
- **DocumentEntity** — an EventSourcedEntity holding the per-document lifecycle (`StructureExtracted`, `ContentRefined`, `DocumentFormatted`, `QualityScored`).
- **ExtractTools / RefineTools / FormatTools** — three function-tool classes registered on the agent, one per stage. The `after-llm-response` guardrail validates each stage's output before the workflow records it.
- **OutputGuardrail** — validates the typed output returned by each LLM call before the workflow advances to the next stage. An output that fails validation is rejected and the agent retries within its iteration budget.
- **QualityScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `DocumentFormatted` and emits a 1–5 score.
- **DocumentView + DocumentEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded content set under `src/main/resources/sample-events/inputs.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/transform-agent.md` — narrow the agent's role (e.g., constrain it to contract clauses, to release notes, to technical specifications) by tightening the system prompt and renaming the typed records (`ExtractedStructure`, `RefinedContent`, `Document`).
- `SPEC.md §5` — extend the typed outputs with domain-specific fields. The output guardrail does not need editing — it validates the recorded typed result, not field shapes.
- `eval-matrix.yaml` — wire a stricter quality scorer (replace the structural-completeness check with a domain-specific rubric) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits content → `EXTRACT` runs → `REFINE` runs → `FORMAT` runs → a typed `Document` lands in the UI within ~60 s. Every transition is visible in real time.
2. The `after-llm-response` guardrail rejects a malformed EXTRACT output (forced via the mock LLM) → the workflow records the rejection event → the agent retries → the pipeline completes correctly.
3. Every `Document` emitted has an on-decision quality score visible on the same UI card; documents whose sections reference no extracted facts receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the EXTRACT task does not see the format instructions, and the FORMAT task does not see raw extracted structure — the workflow's task-chaining is the only path information travels between stages.

## License

Apache 2.0.
