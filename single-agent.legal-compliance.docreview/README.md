# Akka Sample: DocReview

A single document-reviewer agent reads a document against a list of review instructions and returns a structured compliance verdict: PASS / FAIL / NEEDS_REVISION plus a finding per instruction. The document rides into the agent as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a PII sanitizer that runs before the agent ever sees the document, a `before-agent-response` guardrail that validates the verdict's structure, and an on-decision evaluator that scores every verdict for evidence quality.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the document corpus lives in-process and the agent's "tool calls" are simulated.

## Generate the system

```sh
cp -r ./single-agent.legal-compliance.docreview  ~/my-projects/docreview
cd ~/my-projects/docreview
```

(Optional) Edit `SPEC.md` to point at a different review-instruction set (e.g., switch from a generic data-processing-agreement checklist to an MSA checklist or a SOC 2 evidence checklist).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DocumentReviewerAgent** — an AutonomousAgent that accepts review instructions plus a document attachment and returns a typed `ReviewVerdict`.
- **ReviewWorkflow** — orchestrates sanitize-wait → review → eval per submitted document.
- **ReviewEntity** — an EventSourcedEntity holding the per-review lifecycle.
- **DocumentSanitizer** — a Consumer that subscribes to `DocumentSubmitted` events, redacts PII into a `SanitizedDocument`, and emits `DocumentSanitized` back to the entity.
- **ReviewView + ReviewEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded review-instruction set for your own (the JSONL file under `src/main/resources/sample-events/review-instructions.jsonl` after generation).
- `SPEC.md §5` — extend `ReviewVerdict` with industry-specific fields (e.g., `riskTier`, `jurisdiction`, `clauseNumber`).
- `prompts/document-reviewer.md` — narrow the agent's role (a healthcare deployer would constrain it to HIPAA BAA review; a financial deployer to SOC 1 control narratives).
- `eval-matrix.yaml` — wire a real PII redactor (e.g., a presidio-style library) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a document + instructions → it is sanitized → reviewed → the verdict appears in the UI.
2. The agent returns a malformed verdict on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed verdict lands.
3. Every recorded verdict has an on-decision eval score visible on the same UI card.
4. PII strings submitted in the document never appear in the LLM call log; only the redacted form does.

## License

Apache 2.0.
