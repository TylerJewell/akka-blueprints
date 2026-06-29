# User journeys — dual-llm-pdf-extract

Acceptance criteria. The generated system passes when all five journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on port 9445; valid model-provider API keys set for at least one provider (or mock LLM selected). Both `ANTHROPIC_API_KEY` and `GOOGLE_AI_GEMINI_API_KEY` set for full dual-model behavior.

**Steps:**
1. Open `http://localhost:9445/`. App UI tab is visible.
2. In the Filename field type "quarterly-report.pdf"; in the PDF text textarea paste two paragraphs of plain business text. Click Submit.
3. A new extraction card appears with status INTAKE.

**Expected:**
- Within 1 s, status transitions to EXTRACTING via SSE.
- Within 60 s, status transitions to RECONCILED.
- The expanded view shows two raw-extraction blocks (Claude and Gemini), each with a model family badge, a list of extracted fields with confidence values, and extractor notes; and one merged extraction block with the authoritative field list, the confidence map, the disagreement field table, and reconciliation notes. No failure reason.

## J2 — PII redaction at intake

**Preconditions:** As J1.

**Steps:**
1. Submit a document whose text contains the line "Please send the signed contract to john.smith@acmecorp.com or call 555-987-6543 before Friday."

**Expected:**
- `GET /api/extractions/{id}` returns a `redactedText` in which the email and phone number are replaced with `[EMAIL]` and `[PHONE]` (and the name with `[NAME]`); the raw email and phone strings appear nowhere in the response.
- `redactionCount` is at least 2.
- The extraction card shows a redaction-count chip.
- Neither the Claude nor the Gemini extractor's notes or field values contain the raw PII strings.
- The persisted extraction never contains the raw text — confirmable by inspecting the entity via the backoffice.

## J3 — Degraded extractor

**Preconditions:** As J1, plus one extractor's step timeout set to 1 s (configurable via env var `GEMINI_TIMEOUT_MS=1000` or by editing `application.conf`).

**Steps:**
1. Submit any document text.

**Expected:**
- Extraction progresses INTAKE → EXTRACTING → DEGRADED.
- The expanded view shows the extractor that returned populated; the timed-out extractor block is empty or shows a timeout notice; the merged extraction carries reconciliation notes acknowledging the single-model path; `failureReason` names the missing extractor (e.g., "extractor timeout: GeminiExtractor").
- `disagreementCount` is 0 in the merged extraction (single-model path).

## J4 — Agreement score assigned

**Preconditions:** At least one RECONCILED extraction exists with no `agreementScore`. The `AgreementSampler` schedule reduced to 30 s for the test (`AGREEMENT_SAMPLER_SECONDS=30`).

**Steps:**
1. Submit any document text; wait for RECONCILED.
2. Wait 30 s.

**Expected:**
- The extraction card shows an agreement-score chip (1–5) and the rationale is visible in the expanded view.
- `GET /api/extractions/{id}` includes `agreementScore` and `agreementRationale` populated.
- The extraction's status is unchanged (still RECONCILED); only the agreement fields are added.
- If `disagreementCount` was 0, the score should be 5 or close to it.

## J5 — Disagreement count reflects field-level divergence

**Preconditions:** As J1 with both model keys configured (not mock).

**Steps:**
1. Submit a document that is deliberately ambiguous in one field — for example, a date written as "Q2 2026" with no specific day.
2. Wait for RECONCILED.

**Expected:**
- The merged extraction block shows a `disagreementCount` of at least 1.
- The disagreement field table names the ambiguous field and shows both the Claude value and the Gemini value.
- A second, unambiguous document submitted afterward shows a lower `disagreementCount` than the first.
