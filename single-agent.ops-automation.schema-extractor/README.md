# Akka Sample: Structured Extraction Agent

A single extraction agent reads an unstructured document and returns a typed, schema-bound record. The document rides into the agent as a task attachment, never as inline prompt text. If the agent's output fails schema validation, the system retries within the same task rather than propagating a bad record.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that runs before the agent ever sees the document, and an `after-llm-response` guardrail that validates the extracted record's shape and field constraints before it is persisted.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the document corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.schema-extractor  ~/my-projects/schema-extractor
cd ~/my-projects/schema-extractor
```

(Optional) Edit `SPEC.md` to target a different extraction schema — for example, switch from the seeded invoice schema to a purchase-order schema or a support-ticket schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ExtractionAgent** — an AutonomousAgent that accepts a target schema plus a document attachment and returns a typed `ExtractedRecord`.
- **ExtractionWorkflow** — orchestrates sanitize-wait → extract → validate per submitted document.
- **ExtractionJobEntity** — an EventSourcedEntity holding the per-job lifecycle.
- **DocumentSanitizer** — a Consumer that subscribes to `DocumentSubmitted` events, redacts PII into a `SanitizedDocument`, and emits `DocumentSanitized` back to the entity.
- **ExtractionView + ExtractionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded extraction schema for your own (the JSON schema file under `src/main/resources/schemas/` after generation).
- `SPEC.md §5` — extend `ExtractedRecord` with domain-specific fields (e.g., `lineItems`, `currency`, `jurisdiction`).
- `prompts/extraction-agent.md` — narrow the agent's role (a finance deployer would constrain it to invoice line-item extraction; a procurement deployer to purchase-order header fields).
- `eval-matrix.yaml` — wire a real PII redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a document + target schema → it is sanitized → extracted → the record appears in the UI with all required fields present.
2. The agent returns a malformed record on first try → the `after-llm-response` guardrail rejects it → the agent retries → a valid record lands.
3. A document submitted with known PII strings never exposes those strings in the LLM call log; only the redacted form appears.
4. Required fields missing from the extracted record cause the guardrail to reject the response, and the retry produces a complete record.

## License

Apache 2.0.
