# Akka Sample: HITL PDF Data Extraction

Structured fields are extracted from uploaded PDFs by an agent. Low-confidence results pause at a human review gate before the validated data is posted downstream.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.ops-automation.hitl-pdf-extract  ~/my-projects/hitl-pdf-extract
cd ~/my-projects/hitl-pdf-extract
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `ExtractionAgent` (AutonomousAgent) — extracts structured fields from a PDF document, returns a typed `ExtractionResult{fields, confidence}`.
- `RedactionAgent` (AutonomousAgent) — redacts PII from extracted field values before human review, returns a typed `RedactedResult{fields}`.
- `ExtractionWorkflow` (Workflow) — a 4-task graph: extract → redact → await review → post downstream.
- `DocumentEntity` (EventSourcedEntity) — the document lifecycle and its events.
- `DocumentsView` (View) — a read model the UI queries and streams over SSE.
- `ExtractionEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/extraction-agent.md` — the field extraction instructions and output schema.

## What gets validated

- Submitting a document URL triggers extraction that appears in `EXTRACTED` with non-empty `fields`.
- High-confidence results auto-advance to `POSTED`; low-confidence results pause in `PENDING_REVIEW`.
- Approving a `PENDING_REVIEW` document posts the data downstream and transitions to `POSTED`.
- Rejecting a `PENDING_REVIEW` document moves it to terminal `REJECTED` with the reason shown.
- The post step never runs on a document that is not `APPROVED` or auto-approved by the confidence gate.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
