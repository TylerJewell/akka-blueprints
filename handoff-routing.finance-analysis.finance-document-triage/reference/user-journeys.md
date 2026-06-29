# User journeys — finance-document-triage

## J1 — Invoice document: end-to-end handoff to InvoiceProcessor

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `DocumentSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated invoice-flavoured document to drop.

**Expected:**
- The document appears with status `RECEIVED` and within ~2 s transitions through `PII_SANITIZED` to `SECTOR_SANITIZED`. The redacted subject is visible; raw content is not.
- Within ~10 s the classification block shows `docType = INVOICE`, `confidence = high`, and a one-sentence reason. The status pill changes to `CLASSIFIED` then `ROUTED_INVOICE`.
- Within ~5 s of routing, the right column populates with the `InvoiceProcessor` draft: a one-sentence summary, a 2–4 paragraph detail, an action chip (typically `APPROVED` or `INFORMATION_REQUESTED`), and an `invoice` processor tag.
- The guardrail block shows a green "allowed" check.
- The status pill transitions to `PROCESSED`. `finishedAt` is set.
- Within ~10 s of the classification decision, the classification score chip shows a number 1–5 with a rationale.

## J2 — Loan-application document: end-to-end handoff to LoanApplicationProcessor

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a loan-application-flavoured document (the seeded JSONL covers all three types in rotation).

**Expected:**
- Dual-sanitization pipeline runs; `sectorFieldsRedacted` shows credit-score tokens masked.
- Classification emits `docType = LOAN_APPLICATION`. Status `ROUTED_LOAN`.
- `LoanApplicationProcessor` returns a `ProcessingResult` with action `FLAGGED_FOR_REVIEW` or `INFORMATION_REQUESTED`.
- Guardrail allows (the draft does not issue a credit conclusion). Status `PROCESSED`.
- `InvoiceProcessor` and `ComplianceReviewProcessor` are never invoked for this document.

## J3 — Compliance-review document: forwarded to compliance officer

**Preconditions:** The seeded JSONL includes a compliance-sensitive document (AML alert or suspicious-activity description).

**Steps:**
1. Wait for the compliance-sensitive document to drop.

**Expected:**
- `SectorSanitizer` redacts AML flags; `sectorFieldsRedacted` contains "aml-flag".
- Classification emits `docType = COMPLIANCE_REVIEW`. Status `ROUTED_COMPLIANCE`.
- `ComplianceReviewProcessor` returns a `ProcessingResult` with `action = FORWARDED_TO_COMPLIANCE` and a detail that summarises the redacted categories without issuing a regulatory conclusion.
- Guardrail allows. Status `PROCESSED`.
- The right column shows the `compliance` processor tag and a "Forwarded to compliance officer" indicator.
- A classification score still fires; the score chip appears.

## J4 — Guardrail blocks a draft with an invented approval amount

**Preconditions:** The seeded JSONL includes one invoice engineered to produce a draft citing a fabricated approval reference number or invented amount (and the mock-responses file includes a matching entry).

**Steps:**
1. Wait for the trip document to drop and be classified as `INVOICE`.
2. Wait for `InvoiceProcessor` to produce a draft.

**Expected:**
- Status reaches `RESULT_DRAFTED`.
- The guardrail block in the UI shows a red badge with the violation `invented-reference-id` or `invented-approval-amount`.
- Status transitions to `BLOCKED`. `finishedAt` remains null.
- The right column shows the blocked draft and an Unblock button.

## J5 — Operator unblocks a previously-blocked document

**Preconditions:** A document in `BLOCKED`.

**Steps:**
1. Click the Unblock button on the blocked document.
2. Enter a note ("operator override — verified by finance lead").
3. Confirm.

**Expected:**
- A `POST /api/documents/{id}/unblock` is sent.
- The document transitions to `PROCESSED`. The previously-blocked draft is now the published `ProcessingResult`.
- The guardrail block continues to show the original violation list (audit record preserved); a small note displays the operator's override.

## J6 — Classification score appears on every classified document

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new document through the classification step.

**Expected:**
- Within ~10 s of `DocumentClassified`, the per-document classification score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not block the result from being published.
