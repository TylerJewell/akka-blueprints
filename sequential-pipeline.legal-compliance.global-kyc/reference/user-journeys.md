# User journeys — global-kyc-agent

## J1 — Submit a PASS-outcome applicant and receive a KYC decision

**Preconditions:** Service running on declared port (`http://localhost:9356/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded applicant `CORP-001` has a matching `src/main/resources/sample-data/applicants/CORP-001.json` with a valid passport and jurisdiction `GB` rules loaded from `src/main/resources/sample-data/rules/GB.json`.

**Steps:**
1. Open `http://localhost:9356/` → App UI tab.
2. From the **Pick a seeded profile** dropdown, pick `CORP-001 (GB Passport)`.
3. Click **Start KYC**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `COLLECTING` within 1 s more.
- Within ~20 s the card reaches `COLLECTED`. The right pane shows the Collected-documents table with at least 1 row; all three PII columns (name, DOB, document number) show tokenised values (`NAME-…`, `DOB-…`, `DOC-****-…`). No raw PII is visible.
- Within ~20 s more the card reaches `VERIFIED`. The verification panel shows every document with `status = VERIFIED` and every rule check with `passed = true`.
- Within ~20 s more the card reaches `DECIDED`, then `EVALUATED` within 1 s. The decision card shows `outcome = PASS`, ≥ 2 cited rule IDs, and the eval score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — DECLINE outcome triggers HITL gate and pauses for compliance officer review

**Preconditions:** Service running with the mock LLM selected. The mock's `render-decision.json` includes a DECLINE entry. The seeded applicant `CORP-002` is wired to a DECLINE trajectory by the mock's `seedFor(caseId)` logic.

**Steps:**
1. Pick seeded profile `CORP-002 (US-NY Expired Passport)` and click **Start KYC**.
2. Watch the card in the live list until it reaches `PENDING_REVIEW` (status pill turns amber).
3. Click the card to open the detail pane. The HITL review panel is visible with the decision card showing `outcome = DECLINE`.
4. Enter notes in the review panel and click **Approve as submitted**.
5. Watch the card in the live list.

**Expected:**
- The card pauses at `PENDING_REVIEW` and does not advance automatically.
- The workflow is parked; no additional LLM call is made during the review pause.
- After clicking **Approve as submitted**, the POST to `/api/cases/{id}/review` triggers. Within 5 s the card advances from `PENDING_REVIEW → REVIEWED → DECIDED → EVALUATED`.
- The review record strip in the right pane shows the reviewer ID, `resolution = APPROVED_AS_SUBMITTED`, the entered notes, and the resolved timestamp.
- The eval score chip appears with the scorer's result.

## J3 — Invalid rule citation flags eval score ≤ 2

**Preconditions:** Mock LLM mode. The mock's `render-decision.json` includes one entry whose `citedRuleIds` contains a rule ID absent from `DocumentSet.applicableRules`.

**Steps:**
1. Submit applicant profiles until a card appears with its border highlighted and a caution dot in the live list. (The hallucinated-rule entry fires once in the mock's seeded rotation.)
2. Click the flagged card.

**Expected:**
- The decision card shows the cited rule IDs, including one that does not match any entry in the `applicableRules` list shown in the verification panel.
- The eval score chip shows 1 or 2. The scorer rationale reads: *"Rule ID validity failed: citedRuleId '<phantom-id>' does not exist in applicableRules."*
- The card border is highlighted and a caution dot appears on the live-list entry.
- The pipeline completed without errors — the evaluator flagged the issue non-blockingly, as designed.

## J4 — PII never appears in the UI or SSE stream

**Preconditions:** Service running. Any model provider. At least one case in `COLLECTED` or later state.

**Steps:**
1. Submit any seeded applicant profile and wait for `COLLECTED`.
2. Open the browser dev tools Network tab. Filter to `case-update` events on the `/api/cases/sse` stream.
3. Inspect all `case-update` event payloads. Search for the applicant's expected raw full name (e.g., `Jane Thornton`).
4. Inspect the right pane in the UI. Check the documents table's name, DOB, and document-number columns.

**Expected:**
- No raw PII string (`Jane Thornton`, `1982-04-15`, `GB1234567`) appears in any SSE event payload.
- All three PII columns in the UI show tokenised values (`NAME-7f3a1b22`, `DOB-9c0e44d1`, `DOC-****-4567`).
- The service log contains a `SanitizerApplied` event entry listing `fieldsRedacted: [fullName, dateOfBirth, documentNumber]` at the timestamp of the `DocumentsCollected` projection.
- The entity event log (inspected via `/actuator` or the Akka dev console) retains the original `DocumentSet` with raw values — the sanitizer runs only on the view-projection path, not on the entity itself.

## J5 — Empty applicant corpus returns an honest PENDING_DOCUMENTS outcome

**Preconditions:** Service running. A custom applicant ID that has no matching file in `src/main/resources/sample-data/applicants/`.

**Steps:**
1. In the App UI, type a custom applicant ID (e.g., `UNKNOWN-999`) in the Applicant ID field. Select jurisdiction `GB`. Click **Start KYC**.
2. Wait for the card to reach a terminal state.

**Expected:**
- `CollectTools.fetchDocument` returns no documents for the unknown applicant ID.
- The agent's COLLECT task returns a `DocumentSet` with `documents = []`.
- The workflow advances to `verifyStep`. The VERIFY task returns a `VerificationResult` with empty lists.
- The workflow advances to `decideStep`. The DECIDE task returns `outcome = PENDING_DOCUMENTS`, `citedRuleIds = []`, and a rationale stating no documents were available.
- Because the outcome is `PENDING_DOCUMENTS` (non-PASS), the HITL gate triggers. The case pauses at `PENDING_REVIEW`.
- The eval score chip eventually shows 1 (no document verification, no rule citation, no rule ID validity — only outcome validity passes). The rationale names "no documents to verify."
- Nothing crashes; the empty-documents path is handled honestly throughout the pipeline.

## J6 — Per-task tool isolation (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit any seeded applicant profile.
2. Wait for `EVALUATED`.
3. Inspect the service log for tool-call lines (`debug:agent.tool.call`). Filter to entries tagged with the `caseId`.

**Expected:**
- The COLLECT task's log entries show only `fetchDocument` and `lookupJurisdictionRules` calls.
- The VERIFY task's log entries show only `checkDocumentAuthenticity` and `evaluateRule` calls.
- The DECIDE task's log entries show only `compileDecision` and `buildRationale` calls.
- No cross-phase calls appear.
- The order of tasks in the log is COLLECT → VERIFY → DECIDE. Each task's first log line is preceded by a `workflow.step.start` line for the matching step.
- The `SanitizerApplied` audit log line appears immediately after the `DocumentsCollected` projection write — before any VERIFY-phase log lines.
