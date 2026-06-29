# Akka Sample: Invoice Processing

An `InvoiceAgent` walks each invoice through three task phases — **EXTRACT → VALIDATE → POST** — wired by explicit typed task dependencies. Vendor PII is scrubbed before the extracted data leaves the EXTRACT phase; invoices above a configurable amount threshold pause at a human-in-the-loop approval gate before the POST phase commits the journal entry. The submitter sends a document and receives a structured `PostedInvoice`.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-posting` sanitizer that strips vendor PII from extracted line items, and a human-in-the-loop (HITL) approval gate that holds high-value invoices for a reviewer before the POST phase executes.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — all EXTRACT, VALIDATE, and POST tools are implemented in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.finance-analysis.invoice-pipeline  ~/my-projects/invoice-pipeline
cd ~/my-projects/invoice-pipeline
```

(Optional) Edit `SPEC.md` to point at a different amount threshold for the HITL gate, a different model provider, or a richer set of validation rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **InvoiceAgent** — one AutonomousAgent declaring three Task constants (`EXTRACT_INVOICE`, `VALIDATE_LINE_ITEMS`, `POST_JOURNAL_ENTRY`); the workflow runs them in order, feeding each typed output forward as the next task's instruction context.
- **InvoicePipelineWorkflow** — runs `extractStep → validateStep → approvalStep → postStep → evalStep`. The `approvalStep` pauses the workflow when the invoice total exceeds the configured threshold and resumes only after a reviewer calls the approval endpoint.
- **InvoiceEntity** — an EventSourcedEntity holding the per-invoice lifecycle (`InvoiceCreated`, `ExtractionCompleted`, `ValidationCompleted`, `ApprovalRequested`, `ApprovalGranted`, `ApprovalDenied`, `PostingCompleted`, `SanitizationApplied`, `InvoiceFailed`).
- **ExtractTools / ValidateTools / PostTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase.
- **PhaseGuardrail** — the runtime check that backs the dependency contract; rejects any tool called outside its declared phase.
- **VendorPiiSanitizer** — a deterministic sanitizer that runs at the extract→validate boundary, replacing vendor contact fields with redacted placeholders before the data is written to `InvoiceEntity`.
- **PostingQualityScorer** — deterministic, rule-based on-decision evaluator that runs after `PostingCompleted` and emits a 1–5 score for GL account coverage and line-item balance.
- **InvoiceView + InvoiceEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded invoice set under `src/main/resources/sample-data/invoices/*.json` to match your vendor and GL account structure.
- `SPEC.md §4` and `prompts/invoice-agent.md` — narrow the agent's role (e.g., restrict to specific vendor classes, require a purchase-order match) by tightening the system prompt.
- `SPEC.md §5` — extend the typed outputs (`ExtractedInvoice`, `ValidatedInvoice`, `PostedInvoice`) with industry-specific fields such as cost-centre codes or multi-currency amounts.
- `eval-matrix.yaml` — adjust the `H1` HITL threshold or swap the approval trigger to a rule set (e.g., vendor-risk score ≥ HIGH instead of raw amount).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a standard invoice → `EXTRACT` runs → `VALIDATE` runs → the invoice total is below the threshold → `POST` runs → a typed `PostedInvoice` lands in the UI within ~60 s.
2. A high-value invoice is submitted → the workflow pauses at `APPROVAL_REQUESTED` → a reviewer calls the approval endpoint → the workflow resumes and `POST` executes correctly.
3. A submitted invoice whose vendor data contains PII fields has those fields redacted in the `ValidatedInvoice` record; the raw PII never appears in the entity log after the sanitizer fires.
4. Every `PostedInvoice` has an on-decision eval score visible on the same UI card; postings with unbalanced line items receive a score ≤ 2 and are flagged for inspection.

## License

Apache 2.0.
