# Akka Sample: Short Movie Builder

A single `MovieAgent` carries a user intent description through four production phases — **SCRIPT → STORYBOARD → ASSEMBLE → REVIEW** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a phase-specific tool set. The user submits a creative brief and receives a composed `MoviePackage` ready for downstream rendering.

Demonstrates the **sequential-pipeline** coordination pattern with one governance mechanism: an `after-llm-response` guardrail that inspects every phase output for content-safety violations before the result is committed to the entity.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every script, storyboard, assembly, and review tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.short-movie-builder  ~/my-projects/short-movie-builder
cd ~/my-projects/short-movie-builder
```

(Optional) Edit `SPEC.md` to point at a different creative brief seed list, a different model provider, or a stricter content-safety ruleset in `prompts/content-safety-guard.md`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MovieAgent** — one AutonomousAgent declaring four Task constants (`WRITE_SCRIPT`, `DESIGN_STORYBOARD`, `ASSEMBLE_PACKAGE`, `REVIEW_PACKAGE`); the workflow runs them in order, feeding each typed output forward as the next task's instruction context.
- **MovieProductionWorkflow** — runs `scriptStep → storyboardStep → assembleStep → reviewStep`. Each step calls `runSingleTask`, writes the typed result back onto `MovieEntity`, then advances.
- **MovieEntity** — an EventSourcedEntity holding the per-production lifecycle (`ScriptWritten`, `StoryboardDesigned`, `PackageAssembled`, `ReviewCompleted`).
- **ScriptTools / StoryboardTools / AssemblyTools / ReviewTools** — four function-tool classes registered on the agent, one per phase. The `after-llm-response` content-safety guardrail inspects every task output before it is committed.
- **ContentSafetyGuard** — the runtime check that backs the safety contract. Every LLM response from every phase is evaluated against a policy before the workflow writes the result onto the entity.
- **PackageScorer** — deterministic, rule-based review evaluator that runs immediately after `PackageAssembled` and emits a 1–5 coherence score.
- **MovieView + MovieEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded creative brief set under `src/main/resources/sample-events/briefs.jsonl` to fit your demo genre preferences.
- `SPEC.md §4` and `prompts/movie-agent.md` — narrow the agent's role (e.g., constrain it to documentary shorts, to product-launch teasers, to training videos) by tightening the system prompt and renaming the typed records (`MovieScript`, `Storyboard`, `MoviePackage`).
- `SPEC.md §5` — extend the typed outputs (`MovieScript`, `Storyboard`, `AssembledPackage`) with genre-specific fields. The content-safety guardrail does not need editing — it evaluates any text payload against the same policy rules.
- `eval-matrix.yaml` — wire a real content-safety evaluator (replace the deterministic stub with a moderation-API call) by editing the `H1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a creative brief → `SCRIPT` runs → `STORYBOARD` runs → `ASSEMBLE` runs → `REVIEW` runs → a typed `MoviePackage` lands in the UI within ~80 s. Every transition is visible in real time.
2. The agent produces a script containing a policy-violating phrase (forced via the mock LLM) → the `after-llm-response` guardrail rejects the response → the workflow records the safety-block event → the agent retries within its iteration budget → the pipeline completes with a clean script.
3. Every `MoviePackage` emitted has a coherence score visible on the same UI card; packages whose scenes reference shot types absent from the storyboard receive a score ≤ 2 and are flagged.
4. Each task receives only its own typed inputs; the SCRIPT task does not see assembly instructions, and the ASSEMBLE task does not see the raw brief directly — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
