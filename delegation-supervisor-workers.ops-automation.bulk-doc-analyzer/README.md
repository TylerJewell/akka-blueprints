# Akka Sample: High-Volume Document Analyzer

A document intake coordinator fans batches of incoming documents out to two specialist agents running **in parallel** — an Extractor that pulls structured fields and a Classifier that assigns risk categories — then merges their outputs into one unified document record. Demonstrates the **delegation-supervisor-workers** coordination pattern with a PII sanitizer guardrail embedded at the extraction stage.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the document batch feed and the processing tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.ops-automation.bulk-doc-analyzer  ~/my-projects/bulk-doc-analyzer
cd ~/my-projects/bulk-doc-analyzer
```

(Optional) Edit `SPEC.md` to change the document types the simulator drips, or to narrow the PII categories the sanitizer targets.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BatchCoordinator** — AutonomousAgent that partitions an incoming batch into per-document work items and merges the Extractor and Classifier outputs into a `ProcessedDocument` record.
- **Extractor** — AutonomousAgent that pulls structured fields (date, author, reference number, body text) from raw document content.
- **Classifier** — AutonomousAgent that assigns a risk category and sensitivity level to a document.
- **BatchWorkflow** — Workflow that fans each document out to Extractor and Classifier in parallel, then calls BatchCoordinator for merging and PII sanitization.
- **DocumentEntity** — EventSourcedEntity holding the full document processing lifecycle.
- **BatchEntity** — EventSourcedEntity tracking the status of an entire submission batch.
- **DocumentView** — projection the UI streams via SSE.
- **DocumentEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the document types the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ExtractedFields` record (e.g., add `jurisdiction` or `contractValue`).
- `prompts/extractor.md` — narrow the Extractor to a specific document family (contracts, invoices, regulatory filings).
- `eval-matrix.yaml` — extend the PII sanitizer to cover additional entity types (e.g., account numbers).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a document batch → each document enters `QUEUED`, then `PROCESSING`, then `PROCESSED`; the batch entity moves to `COMPLETE` when all documents are done.
2. Extractor or Classifier times out → the affected document enters `DEGRADED`; the batch continues processing remaining documents.
3. PII sanitizer strips a detected PII field → the processed record omits the raw value; the audit log records the redaction event.
4. Wait after successful processing; the document's row in the App UI shows a quality eval score.

## License

Apache 2.0.
