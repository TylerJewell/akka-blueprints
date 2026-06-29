# User journeys — medical-document-pipeline

## J1 — Upload a cardiology discharge summary and get a clinical summary

**Preconditions:** Service running on declared port (`http://localhost:9837/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded document `discharge-summary-cardiology.txt` exists under `src/main/resources/sample-data/documents/`.

**Steps:**
1. Open `http://localhost:9837/` → App UI tab.
2. From the **Document type** dropdown, select `discharge-summary`. Click **Pick a sample** — the text area fills with the seeded cardiology content.
3. Click **Process document**.
4. Watch the card transition to `SANITIZING`, then `SANITIZED`, then `EXTRACTING`, then `EXTRACTED`, then `AWAITING_REVIEW`.
5. In the right pane, review the extracted fields table. Confirm the Diagnoses sub-section shows at least one entry with a confidence ≥ 0.8.
6. Click **Approve** (no corrections needed on this seeded document).
7. Watch the card reach `SUMMARIZING`, then `SUMMARIZED`, then `EVALUATED`.

**Expected:**
- The card reaches `SANITIZED` within 3 s. The masked text preview in the right pane shows `[PATIENT_NAME]`, `[DOB]`, `[MRN]` tokens.
- The card reaches `EXTRACTED` within 25 s. The Extracted fields panel shows non-empty Demographics, Diagnoses, Medications, and Vitals sub-sections. Every row has a confidence bar and a source span reference.
- After clicking Approve, the card reaches `EVALUATED` within 30 s. The Clinical summary panel shows a patient context block, chief complaint, ≥ 1 section with source field chips, clinical impression, and recommendations.
- The eval score chip shows ≥ 4/5. The rationale reads "all checks passed" or names a minor gap (e.g., short section body).
- No red dot on the card (no guardrail rejections on the happy path).
- Total elapsed time (excluding review): ≤ 60 s.

## J2 — Phase-gate guardrail blocks a misordered tool call

**Preconditions:** Service running with mock LLM. The mock's `extract-fields.json` includes one entry whose `tool_calls` array starts with `buildClinicalImpression` (a SUMMARIZE-phase tool called during EXTRACT).

**Steps:**
1. Submit any seeded document three times in a row (J1 steps × 3, clicking Approve each time).
2. On the third submission, watch the card's status in the live list and the network panel (`/api/documents/sse`).

**Expected:**
- On the third document's `extractStep`, the agent's first iteration calls `buildClinicalImpression`. `SanitizationGuardrail` rejects it; a `GuardrailRejected{phase: "EXTRACT", tool: "buildClinicalImpression", reason: "phase-violation: ..."}` event lands on the entity.
- The tool body is never executed — there is no log line from `SummarizeTools.buildClinicalImpression`.
- The agent's second iteration calls `extractDiagnoses` correctly. The card eventually reaches `AWAITING_REVIEW` as in J1, and then `EVALUATED` after approval.
- The card in the live list shows the small red dot indicating a rejection fired. The rejection-log strip in the right pane shows the one rejected call with its structured reason.

## J3 — Hallucinated source field flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `write-summary.json` includes one entry whose first `ClinicalSection.sourceFieldIds` contains a `fieldId` not present in the paired `ValidationResult.approvedFields`.

**Steps:**
1. Submit any seeded document six times (approving each). The hallucinated-source entry is selected once in every six runs by the mock's `seedFor(documentId)` modulo.
2. Watch the live list until a card border highlights red.

**Expected:**
- The flagged document lands well-formed (the guardrail checks phase order and sanitization, not source provenance).
- The eval score chip shows **1** and the rationale reads: *"Value provenance failed: section 'sec-01' cites fieldId f-zzzzzzzz which does not appear in ValidationResult.approvedFields."*
- The card border highlights red. The clinician knows to re-examine this summary before submitting it to the chart.
- The other five documents in the run scored ≥ 4.

## J4 — PHI-not-sanitized guardrail fires before extraction

**Preconditions:** Mock LLM mode. The mock is configured to simulate a document that arrives with `sanitized = false` (the sanitize step is bypassed or returns an error).

**Steps:**
1. Submit the seeded document `radiology-report-chest-ct.txt`.
2. Observe the card and the service log.

**Expected:**
- When `extractStep` begins, the agent attempts `extractDiagnoses`. `SanitizationGuardrail` reads `DocumentEntity.sanitized = false` and rejects the call with `"phi-not-sanitized"` before the tool body runs.
- A `GuardrailRejected{reason: "phi-not-sanitized"}` event is recorded on the entity. The rejection is visible in the audit log.
- The workflow's step retry budget is exhausted (no retry can succeed without sanitization); the document transitions to `FAILED` with `reason = "phi-not-sanitized"`.
- The card shows `FAILED` status and the rejection-log strip shows the rejected call.

## J5 — Clinician corrects an extracted field; summary uses corrected value

**Preconditions:** Service running. Any model provider. The seeded `referral-letter-orthopedics.txt` has a medication field that the extraction tool returns with a minor error (detectable from the mock's `extract-fields.json`).

**Steps:**
1. Submit `referral-letter-orthopedics.txt`.
2. When the card reaches `AWAITING_REVIEW`, find the Medications sub-section in the extracted fields table. Edit the value of the medication field to correct the dosage.
3. Click **Submit corrections**.
4. Wait for `EVALUATED`.

**Expected:**
- The `ValidationResult.approvedFields` in the entity log contains the corrected value, not the originally extracted value.
- The clinical summary's medication-relevant section body reflects the corrected dosage.
- The eval score confirms value provenance: the summary's `sourceFieldIds` reference the field ID that was corrected, and the approved field's corrected value matches what appears in the summary body.
- Eval score ≥ 4. No hallucinated-source finding.

## J6 — Empty document produces an honest empty summary

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the App UI, clear the text area and click **Process document** with an empty (or whitespace-only) document body.

**Expected:**
- `sanitizeStep` completes (nothing to mask). `ExtractedFields` arrives with all four field groups empty (`diagnoses = []`, `medications = []`, `vitals = []`, `demographics = []`).
- The card reaches `AWAITING_REVIEW`. The clinician sees empty field tables and clicks **Approve**.
- The agent's WRITE_SUMMARY task returns a `ClinicalSummary` with `chiefComplaint = "(no diagnoses extracted)"`, empty `sections`, and a one-sentence `clinicalImpression` explaining the gap.
- The eval score is 1 (no mandatory fields to cover, but section completeness check fires on the empty sections list). The rationale reads "mandatory field coverage failed: no diagnosis fields to check."
- Nothing crashes; the pipeline completes with an honestly empty summary.
