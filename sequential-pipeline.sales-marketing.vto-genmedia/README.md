# Akka Sample: GenMedia for Commerce

A single `VtoAgent` walks a try-on request through three task phases — **PREPARE → GENERATE → VALIDATE** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The shopper submits a garment and a model image reference; the pipeline produces a try-on composite and a short-form video clip, then scores the output for safety before surfacing it to the UI.

Demonstrates the **sequential-pipeline** coordination pattern wired with one governance mechanism: an `after-llm-response` guardrail that inspects every generated image or video frame for unsafe content before the output is returned to the caller.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every PREPARE / GENERATE / VALIDATE tool is implemented in-process inside the same Akka service; garment and model images are loaded from bundled sample assets.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.vto-genmedia  ~/my-projects/vto-genmedia
cd ~/my-projects/vto-genmedia
```

(Optional) Edit `SPEC.md` to point at a different garment catalogue, a different model provider, or a richer set of generation tools.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **VtoAgent** — one AutonomousAgent declaring three Task constants (`PREPARE_ASSETS`, `GENERATE_MEDIA`, `VALIDATE_OUTPUT`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **VtoPipelineWorkflow** — runs `prepareStep → generateStep → validateStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `TryOnEntity` before the next step starts.
- **TryOnEntity** — an EventSourcedEntity holding the per-request lifecycle (`AssetsResolved`, `MediaGenerated`, `SafetyChecked`, `EvaluationScored`).
- **PrepareTools / GenerateTools / ValidateTools** — three function-tool classes registered on the agent, one per phase. The `after-llm-response` guardrail enforces that every generated image passes a content-safety check before leaving the GENERATE phase.
- **ImageSafetyGuardrail** — the runtime check that intercepts generated media after the LLM response and before the output is persisted. Unsafe outputs are rejected with a structured reason; the agent retries within its iteration budget.
- **RenderQualityScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `SafetyChecked` and emits a 1–5 quality score.
- **TryOnView + TryOnEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded garment catalogue under `src/main/resources/sample-events/garments.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/vto-agent.md` — narrow the agent's role (e.g., constrain it to outerwear only, or to a single brand's catalogue) by tightening the system prompt.
- `SPEC.md §5` — extend the typed outputs (`AssetBundle`, `MediaResult`, `ValidatedMedia`) with retailer-specific fields. The safety guardrail does not need editing — it inspects output content, not field shapes.
- `eval-matrix.yaml` — wire a real quality scorer (replace the deterministic stub with a perceptual hash comparison against reference composites) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a garment + model reference → `PREPARE` runs → `GENERATE` runs → `VALIDATE` runs → a `ValidatedMedia` record with a composite image URL and a video clip URL lands in the UI within ~60 s.
2. A generated image flagged unsafe by the `after-llm-response` guardrail → the guardrail rejects the output → the workflow records the rejection event → the agent retries → the pipeline completes with a clean output.
3. Every `ValidatedMedia` emitted has a quality eval score visible on the same UI card; outputs missing garment-colour fidelity or out-of-bounds aspect ratios receive a score ≤ 2 and are flagged for review.
4. Each task receives only its own typed inputs; the PREPARE task does not see generation parameters, and the GENERATE task does not see safety rules — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
