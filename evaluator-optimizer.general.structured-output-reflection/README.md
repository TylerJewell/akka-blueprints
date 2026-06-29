# Akka Sample: Reliable Structured Generation (Reflection Loop)

A generator agent produces structured JSON output against a schema; a critic agent validates and critiques it; the two exchange feedback until the output passes schema validation and quality checks, or the loop reaches its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance controls.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **None**. The blueprint runs out of the box; the submission pipeline and output surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.general.structured-output-reflection  ~/my-projects/structured-output-reflection
cd ~/my-projects/structured-output-reflection
```

(Optional) Edit `SPEC.md` to change the target schema, the critic rubric, or the retry ceiling.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **GeneratorAgent** — AutonomousAgent that produces a structured JSON document conforming to a target schema, accepting prior critic feedback when revising.
- **CriticAgent** — AutonomousAgent that validates the JSON against the schema, checks semantic quality, and returns either `PASS` or `REVISE` with a typed `ValidationNotes` payload.
- **ReflectionWorkflow** — Workflow that runs the generate → validate → revise loop up to a configurable retry ceiling, transitions the generation to `PASSED` on critic approval or to `FAILED_FINAL` when the ceiling is hit.
- **GenerationEntity** — EventSourcedEntity that holds the generation lifecycle, every attempt's output, every validation report, and the final outcome.
- **SubmissionQueue** — EventSourcedEntity that logs each request for replay and audit.
- **GenerationsView** — read-side projection that the UI lists and streams via SSE.
- **SubmissionConsumer** — Consumer that starts a workflow per inbound request.
- **RequestSimulator** — TimedAction that drips a sample generation request every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **GenerationEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample schemas the simulator uses, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Generation` record fields (e.g., raise `maxAttempts`, change the target schema).
- `prompts/generator.md` — narrow the output domain to a specific schema or output type.
- `prompts/critic.md` — change the validation rubric (e.g., add a business-rule check).
- `eval-matrix.yaml` — tighten enforcement on the schema guardrail (currently blocking).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a schema request → generation progresses `GENERATING` → `VALIDATING` → `PASSED` within the retry ceiling.
2. Force-fail validation → generation hits `FAILED_FINAL` after the configured number of attempts; the entity preserves every attempt and every critique for audit.
3. The schema guardrail rejects a structurally malformed output before the critic runs semantic checks.
4. Each completed cycle emits an `EvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
