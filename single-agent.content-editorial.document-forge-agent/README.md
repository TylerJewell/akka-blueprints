# Akka Sample: Document Forge (Claude Skills Demo)

A single document-forging agent accepts a user prompt plus an optional style template, calls Claude's skill tools to draft structured documents, validates the generated file path and content before writing, and returns a completed `ForgeResult` containing the document text and a short audit trail.

Demonstrates the **single-agent** coordination pattern wired in the content-editorial domain. One `DocumentForgeAgent` (AutonomousAgent) drives the entire generation loop; a `before-tool-call` guardrail intercepts every file-writing tool invocation to validate the target path and content before any write is committed.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No integration-tier host software. The blueprint runs out of the box — the document store is in-process and the agent's tool calls target an in-memory file registry.

## Generate the system

```sh
cp -r ./single-agent.content-editorial.document-forge-agent  ~/my-projects/document-forge
cd ~/my-projects/document-forge
```

(Optional) Edit `SPEC.md` to add domain-specific document types (e.g., switch the seeded templates from generic business documents to legal briefs, technical specifications, or marketing copy).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DocumentForgeAgent** — an AutonomousAgent that accepts a forge request (prompt + optional template), calls a `write_document` tool to persist the result, and returns a typed `ForgeResult`.
- **ForgeWorkflow** — orchestrates validate-wait → forge → audit steps per submitted request.
- **ForgeEntity** — an EventSourcedEntity holding the per-forge lifecycle.
- **ForgeOutputConsumer** — a Consumer that subscribes to `ForgeCompleted` events, verifies the output document against quality rules, and emits `OutputVerified` back to the entity.
- **ForgeView + ForgeEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded template set for your own (the JSONL file under `src/main/resources/sample-events/forge-templates.jsonl` after generation).
- `SPEC.md §5` — extend `ForgeResult` with domain-specific fields (e.g., `wordCount`, `readabilityScore`, `targetAudience`).
- `prompts/document-forge.md` — narrow the agent's role (a technical writer would constrain it to API reference docs; a marketing team to campaign briefs).
- `eval-matrix.yaml` — wire real path-sanitization logic by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prompt + template → the agent generates the document → the result appears in the UI with full audit trail.
2. The agent attempts to write to a disallowed path → the `before-tool-call` guardrail blocks it → the agent retries with a valid path → a completed document lands.
3. An output document that fails quality rules receives an audit score of 1 and the card is flagged for review.
4. Path traversal strings submitted in the forge prompt never result in a file write outside the allowed directory; only sanitized paths reach the in-memory store.

## License

Apache 2.0.
