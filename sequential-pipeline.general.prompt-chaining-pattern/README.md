# Akka Sample: Prompt Chaining Workflow

A single `DraftingAgent` walks a writing task through three chained LLM steps — **OUTLINE → DRAFT → REFINE** — where each step receives the prior step's typed output as its instruction context. A `before-agent-response` guardrail validates the composite result before it leaves the system, and an `on-decision-eval` evaluator scores every refined document for structural completeness.

Demonstrates the **sequential-pipeline** coordination pattern with a response guardrail at the output boundary and an on-decision evaluator at the task junction that gates whether the pipeline advances or retries.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every outline, draft, and refine step runs in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.general.prompt-chaining-pattern  ~/my-projects/prompt-chaining
cd ~/my-projects/prompt-chaining
```

(Optional) Edit `SPEC.md` to point at a different prompt seed list, a different model provider, or a richer set of quality gate criteria.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DraftingAgent** — one AutonomousAgent declaring three Task constants (`OUTLINE_DOCUMENT`, `DRAFT_DOCUMENT`, `REFINE_DOCUMENT`); the workflow runs them in order, feeding each typed output forward as the next step's instruction context.
- **DraftingWorkflow** — runs `outlineStep → draftStep → refineStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `DraftEntity` before the next step starts.
- **DraftEntity** — an EventSourcedEntity holding the per-draft lifecycle (`OutlineProduced`, `DraftWritten`, `RefinementApplied`, `QualityScored`).
- **OutlineTools / DraftTools / RefineTools** — three function-tool classes registered on the agent, one per step. The `before-agent-response` guardrail validates the final `RefinedDocument` before it is returned to the caller.
- **OutputGuardrail** — the runtime check that runs after `refineStep` returns and before the response reaches the endpoint. Enforces that the refined document meets minimum structural requirements (section count, word count, citation presence).
- **QualityScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `RefinementApplied` and emits a 1–5 score.
- **DraftView + DraftEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded prompt set under `src/main/resources/sample-events/prompts.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/drafting-agent.md` — narrow the agent's role (e.g., constrain it to technical specifications, legal briefs, or product announcements) by tightening the system prompt and renaming the typed records (`Outline`, `Draft`, `RefinedDocument`).
- `SPEC.md §5` — extend the typed outputs with domain-specific fields. The response guardrail does not need editing — it checks recorded structural properties, not field shapes.
- `eval-matrix.yaml` — wire a real quality evaluator (replace the deterministic scorer with an embedding-similarity check against a reference corpus) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prompt → `OUTLINE` runs → `DRAFT` runs → `REFINE` runs → a typed `RefinedDocument` lands in the UI within ~90 s. Every transition is visible in real time.
2. The `before-agent-response` guardrail intercepts a refined document that falls below the minimum section count (forced via the mock LLM), rejects it with a structured reason, and the workflow retries `refineStep` — demonstrating that the guardrail is the last line of defence before the output reaches the caller.
3. Every `RefinedDocument` emitted has an on-decision quality score visible on the same UI card; documents that fail the citation check receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the OUTLINE task does not see the raw prompt instructions handed to REFINE, and the REFINE task does not see the outline structure — the workflow's step-chaining is the only path information travels between steps.

## License

Apache 2.0.
