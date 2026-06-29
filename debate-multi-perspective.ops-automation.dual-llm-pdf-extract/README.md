# Akka Sample: Dual-LLM PDF Extractor

Upload a PDF; two specialist extraction agents — one backed by Claude, one backed by Gemini — independently extract structured data from the same document, then a reconciler combines their outputs into a single authoritative extraction. Cross-model disagreement is surfaced as an eval signal. Demonstrates the **debate-multi-perspective** coordination pattern applied to document operations.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — every external surface is modelled inside the service with Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./debate-multi-perspective.ops-automation.dual-llm-pdf-extract  ~/my-projects/dual-llm-pdf-extract
cd ~/my-projects/dual-llm-pdf-extract
```

(Optional) Edit `SPEC.md` to change the extraction schema, the field set, or the reconciliation strategy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ClaudeExtractor** — AutonomousAgent that extracts structured fields from the sanitized PDF text using the Anthropic model.
- **GeminiExtractor** — AutonomousAgent that extracts the same structured fields from the same text using the Google AI Gemini model.
- **ExtractionReconciler** — AutonomousAgent that receives both raw extractions, identifies field-level disagreements, and produces a merged authoritative result with a confidence map.
- **EvalJudge** — AutonomousAgent that scores how far apart the two raw extractions were, as a signal of per-document model agreement.
- **ExtractionWorkflow** — Workflow that sanitizes the document, fans extraction out to both agents in parallel, then calls the reconciler for synthesis.
- **ExtractionEntity** — EventSourcedEntity holding the full extraction lifecycle.
- **ExtractionView** — projection used by the UI.
- **ExtractionEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample PDFs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ExtractedFields` record (e.g., add a `totalAmount` field for invoice processing).
- `prompts/extraction-reconciler.md` — tighten the merge strategy for your field types.
- `eval-matrix.yaml` — broaden the PII sanitizer's pattern table to match your document corpus.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Upload a PDF → extraction enters `INTAKE`, then `EXTRACTING`, then `RECONCILED` with two raw extractions and one merged result; UI reflects each transition via SSE.
2. PII in the extracted text is redacted before either agent reads it; the redaction count surfaces in the UI.
3. One extractor timeout drives the extraction to `DEGRADED` with the reconciler working from the single available extraction.
4. A completed extraction with no `agreementScore` receives one within the next eval interval; the score is visible on the App UI.
5. The disagreement score on a reconciled extraction reflects the number of fields where Claude and Gemini disagreed.

## License

Apache 2.0.
