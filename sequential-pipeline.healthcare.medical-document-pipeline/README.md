# Akka Sample: Medical Document Processing Assistant

A single `MedicalDocumentAgent` walks an ingested document through three task phases — **EXTRACT → VALIDATE → SUMMARIZE** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a targeted set of phase-specific tools. A clinician uploads a document and receives a structured `ClinicalSummary` ready for chart review.

Demonstrates the **sequential-pipeline** coordination pattern wired with four governance mechanisms: a `before-tool-call` guardrail that blocks PHI/PII-touching tools unless the document has passed sanitization, application-level human-in-the-loop review where a clinician approves extracted facts before the summary is finalized, and an `on-decision-eval` evaluator that scores every emitted summary for clinical field completeness and value accuracy.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every extract / validate / summarize tool is implemented in-process inside the same Akka service. Sample documents are bundled under `src/main/resources/sample-data/documents/`.

## Generate the system

```sh
cp -r ./sequential-pipeline.healthcare.medical-document-pipeline  ~/my-projects/medical-doc-pipeline
cd ~/my-projects/medical-doc-pipeline
```

(Optional) Edit `SPEC.md` to adjust the seeded document set, point at a different model provider, or extend the set of extractable clinical fields.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MedicalDocumentAgent** — one AutonomousAgent declaring three Task constants (`EXTRACT_FIELDS`, `VALIDATE_FIELDS`, `WRITE_SUMMARY`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **DocumentPipelineWorkflow** — runs `extractStep → validateStep → summarizeStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `DocumentEntity` before the next step starts.
- **DocumentEntity** — an EventSourcedEntity holding the per-document lifecycle (`FieldsExtracted`, `ValidationCompleted`, `SummaryWritten`, `EvaluationScored`).
- **ExtractTools / ValidateTools / SummarizeTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces phase discipline and blocks PHI-touching extract tools from running when the document has not yet been sanitized.
- **SanitizationGuardrail** — runs before every tool call. Checks that the document's PHI/PII fields have been masked before any extract tool accesses raw document text, and enforces phase ordering in the same hook.
- **ClinicalAccuracyScorer** — deterministic, rule-based on-decision evaluator that runs immediately after `SummaryWritten` and emits a 1–5 score based on field completeness, value plausibility, and source traceability.
- **DocumentView + DocumentEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded document set under `src/main/resources/sample-data/documents/` to fit your specialty (e.g., cardiology discharge summaries, radiology reports, referral letters).
- `SPEC.md §4` and `prompts/medical-document-agent.md` — narrow the agent's role (e.g., constrain it to structured FHIR extraction, to ICD-10 code suggestion, to medication reconciliation) by tightening the system prompt and renaming the typed records.
- `SPEC.md §5` — extend the typed outputs (`ExtractedFields`, `ValidationResult`, `ClinicalSummary`) with specialty-specific fields. The phase-gating guardrail does not need editing — it checks recorded phase preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real clinical accuracy evaluator (replace the deterministic stub with a terminology-lookup check against a SNOMED/ICD vocabulary) by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A clinician uploads a document → `EXTRACT` runs → `VALIDATE` runs → `SUMMARIZE` runs → a typed `ClinicalSummary` lands in the UI within ~60 s. Every transition is visible in real time via SSE.
2. An extract tool is called before the document's PHI fields have been masked (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries correctly → the pipeline completes.
3. Every `ClinicalSummary` emitted has an on-decision eval score visible on the same UI card; summaries missing mandatory clinical fields or citing values absent from the extracted source receive a score ≤ 2 and are flagged for clinician review.
4. The human-in-the-loop gate at `validateStep` surfaces extracted fields to the clinician before the summary is written; the agent does not advance to `WRITE_SUMMARY` until the clinician approves or the timeout escalates the decision.

## License

Apache 2.0.
