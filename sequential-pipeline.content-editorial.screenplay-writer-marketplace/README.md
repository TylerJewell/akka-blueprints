# Akka Sample: Screenplay Writer Team

A single `ScreenplayAgent` walks source material — plain text, email threads, or draft outlines — through three task phases: **PARSE → DEVELOP → FORMAT** — producing a properly structured screenplay document. Each phase has its own typed input, typed output, and a focused set of phase-specific tools. The user submits source material and receives a `Screenplay`.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-agent-response` guardrail that checks the final screenplay for PII before delivery, and a `pii` sanitizer that scrubs personally identifiable content found in source emails before it propagates into the output.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every parse / develop / format tool is implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.screenplay-writer-marketplace  ~/my-projects/screenplay-writer
cd ~/my-projects/screenplay-writer
```

(Optional) Edit `SPEC.md` to point at a different sample-email corpus, a different model provider, or additional formatting conventions.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ScreenplayAgent** — one AutonomousAgent declaring three Task constants (`PARSE_SOURCE`, `DEVELOP_SCENES`, `FORMAT_SCREENPLAY`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **ScreenplayPipelineWorkflow** — runs `parseStep → developStep → formatStep → deliverStep`. Each step calls `runSingleTask` and writes the typed result back onto `ScreenplayEntity` before the next step starts.
- **ScreenplayEntity** — an EventSourcedEntity holding the per-screenplay lifecycle (`SourceParsed`, `ScenesDevoped`, `ScreenplayFormatted`, `ScreenplayDelivered`).
- **ParseTools / DevelopTools / FormatTools** — three function-tool classes registered on the agent, one per phase.
- **PiiSanitizer** — runs immediately after source material ingestion; scrubs PII tokens from emails before they reach the agent.
- **DeliveryGuardrail** — the `before-agent-response` hook that checks the final screenplay for residual PII before it is returned to the caller.
- **ScreenplayView + ScreenplayEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded sample emails under `src/main/resources/sample-events/emails.jsonl` to fit your demo audience.
- `SPEC.md §4` and `prompts/screenplay-agent.md` — narrow the agent's role (e.g., constrain it to feature films, to TV pilots, to corporate training scripts) by tightening the system prompt.
- `SPEC.md §5` — extend the typed outputs (`ParsedSource`, `ScenePlan`, `Screenplay`) with genre-specific fields.
- `eval-matrix.yaml` — wire a real PII detection service (replace the in-process regex stub with a dedicated NLP classifier) by editing the `S1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an email thread → `PARSE` runs → `DEVELOP` runs → `FORMAT` runs → a typed `Screenplay` lands in the UI within ~60 s.
2. An email containing PII is submitted → the `pii` sanitizer redacts PII tokens before the agent sees them → the output screenplay contains no raw PII.
3. A finished screenplay that somehow retains a PII token is caught by the `before-agent-response` guardrail → delivery is blocked → a `DeliveryBlocked` event is recorded.
4. Each task receives only its own typed inputs; the PARSE task does not see the formatting instructions, and the FORMAT task does not see raw email text — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
