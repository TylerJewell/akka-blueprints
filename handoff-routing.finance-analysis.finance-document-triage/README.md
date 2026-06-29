# Akka Sample: Finance Document Triage

A `ClassifierAgent` examines each incoming finance document, applies PII and sector-regulated redaction, then hands the same `PROCESS` task to the correct downstream handler — `InvoiceProcessor`, `LoanApplicationProcessor`, or `ComplianceReviewProcessor` — which owns the document end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with four governance mechanisms (PII sanitizer, sector-data sanitizer, before-agent-response guardrail on the processor draft, and an on-decision eval that grades every classification call).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound document stream and the outbound processing surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.finance-analysis.finance-document-triage  ~/my-projects/finance-document-triage
cd ~/my-projects/finance-document-triage
```

(Optional) Edit `SPEC.md` to point `DocumentSimulator` at a real document-ingestion source (S3 bucket, SFTP drop, webhook) or to add a new document type category.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DocumentSimulator** — TimedAction firing every 30 s that drips canned finance documents from a JSONL file into `DocumentQueue`.
- **DocumentQueue** — EventSourcedEntity append-only log of every inbound document (audit before redaction).
- **PiiSanitizer** — Consumer that redacts names, account numbers, tax ids, and national ids before any LLM call.
- **SectorSanitizer** — Consumer that applies financial-sector regulated redaction (AML flags, credit scores, fund codes) on top of PII redaction.
- **ClassifierAgent** — typed Agent that classifies the sanitized document into `INVOICE`, `LOAN_APPLICATION`, or `COMPLIANCE_REVIEW`.
- **InvoiceProcessor** — AutonomousAgent that owns the `PROCESS` task for invoice documents. Returns a typed `ProcessingResult`.
- **LoanApplicationProcessor** — AutonomousAgent that owns the `PROCESS` task for loan application documents.
- **ComplianceReviewProcessor** — AutonomousAgent that owns the `PROCESS` task for compliance-review documents.
- **ClassificationJudge** — typed Agent used by `ClassificationEvalScorer` to grade every classification decision against a 1–5 rubric.
- **OutputGuardrail** — typed Agent: before-agent-response guardrail on every processor draft.
- **DocumentWorkflow** — Workflow per document: sanitize → classify → route → process → guardrail → publish.
- **DocumentEntity** — EventSourcedEntity holding each document's lifecycle.
- **DocumentView + DocumentEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **ClassificationEvalScorer** — Consumer that listens for `DocumentClassified` events and writes an inline eval score.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the document taxonomy (add `TRADE_CONFIRMATION`, `EXPENSE_REPORT`, etc.) and add the matching processor agent.
- `SPEC.md §5` — extend the `Document` record with deployer-specific fields (`sourceSystem`, `currencyCode`, `jurisdictionCode`).
- `prompts/classifier-agent.md` — narrow the classifier rules (confidence thresholds, document-type keywords, language detection).
- `prompts/invoice-processor.md` / `prompts/loan-application-processor.md` / `prompts/compliance-review-processor.md` — encode business rules, approval authority limits, regulatory references.
- `eval-matrix.yaml` — swap the in-process PII regex for a dedicated redaction service; swap the sector sanitizer for a vendor library (make it Real-service-via-env-var if you do).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips an invoice-flavoured document → it is sanitized, classified as `INVOICE`, handed off to `InvoiceProcessor`, and a processing result is published.
2. Simulator drips a loan-application document → classified as `LOAN_APPLICATION`, handed off to `LoanApplicationProcessor`, result published.
3. An ambiguous document classifies as `COMPLIANCE_REVIEW` and is routed to `ComplianceReviewProcessor` for human-assisted disposition.
4. A processor draft that cites an invented account balance or approval amount is blocked by the guardrail; the document lands in `BLOCKED` for operator review.
5. The classification score (1–5) and rationale appear on every classified document within a few seconds of the decision.

## License

Apache 2.0.
