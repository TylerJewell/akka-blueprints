# User journeys — medical-preauth

## J1 — Submit a covered request and receive an APPROVED determination

**Preconditions:** Service running on declared port (`http://localhost:9592/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded request `DEXA bone density scan` has a matching `src/main/resources/sample-data/procedure-codes/77080.json` and `src/main/resources/sample-data/policies/pol-dexa-001.json`.

**Steps:**
1. Open `http://localhost:9592/` → App UI tab.
2. From the **Pick a seeded request** dropdown, select `DEXA bone density scan`.
3. Click **Submit request**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s, then `VALIDATING` within 1 s more.
- Within ~25 s the card reaches `VALIDATED`. The right pane shows the Validation panel: `eligible: true`, `codeRecognized: true`, `icd10Linked: true`.
- Within ~25 s more the card reaches `POLICY_REVIEWED`. The right pane shows the Policy Review panel: `matchedArticles` contains `pol-dexa-001`; the criteria table shows `cr-age-risk` as met.
- Within ~25 s more the card reaches `DETERMINED`, then `EVALUATED` within 1 s. The right pane shows outcome chip `APPROVED`, rationale ≥ 15 words, `citedArticleIds: ["pol-dexa-001"]`, `coveredProcedureCodes: ["77080"]`, eval score chip 5/5.
- Total elapsed time: ≤ 90 s.

## J2 — PHI sanitizer fires and records the audit event

**Preconditions:** Service running (real or mock LLM). Submit any request with a non-empty `memberId`.

**Steps:**
1. Submit the seeded request `DEXA bone density scan`.
2. Wait for `VALIDATING` to appear.
3. Watch the right pane's sanitization-log strip.

**Expected:**
- Within a second of the card reaching `VALIDATING`, the sanitization-log strip shows at least one entry with `field: "memberId"`, a 12-character `hashedToken`, `tool: "checkMemberEligibility"`, and a `sanitizedAt` timestamp.
- The service log shows no raw MRN value passed to the LLM. The LLM call payload contains only the hashed token.
- The `SanitizationApplied` event is visible on the entity log endpoint (`GET /api/auth-requests/{id}`).

## J3 — DENIED determination triggers the human-review hold

**Preconditions:** Service running with mock LLM selected. The mock's `determine-outcome.json` includes one entry whose `classifyOutcome` result is `DENIED`.

**Steps:**
1. Submit any seeded request four times in a row (the DENIED mock entry fires on a deterministic cycle controlled by `seedFor(requestId)`).
2. Watch the live list until one card transitions to `PENDING_HUMAN_REVIEW`.

**Expected:**
- The card shows status `PENDING_HUMAN_REVIEW` in amber. The right pane shows the full `Determination` panel with outcome badge `DENIED` and the rationale text.
- The **Acknowledge denial** button is visible.
- Click **Acknowledge denial**, type a brief reviewer note, and click confirm.
- Within 1–2 s the card transitions to `EVALUATED`. The right pane shows a `HumanApprovalReceived` record with `receivedAt` and `reviewerNotes`.
- The timeline strip shows the hold duration (time between `HumanApprovalRequested` and `HumanApprovalReceived`).

## J4 — Hallucinated policy citation flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `determine-outcome.json` includes one entry whose `citedArticleIds` contains an article id absent from the paired `PolicyReview.matchedArticles`.

**Steps:**
1. Submit any seeded request six times. (The hallucinated-citation mock entry is selected once per six runs by the mock's `seedFor(requestId)` modulo.)
2. Watch the live list until a card border highlights red.

**Expected:**
- The flagged determination is structurally complete (valid outcome, non-empty rationale).
- The eval score chip shows **1** and the rationale reads something like: *"Policy citation failed: citedArticleId 'pol-ortho-999' does not appear in PolicyReview.matchedArticles."*
- The card border is highlighted red. The reviewer knows to inspect this determination before acting on it.
- The other five determinations in the run scored ≥ 4.

## J5 — Failed eligibility check produces a PENDING_ADDITIONAL_INFO outcome

**Preconditions:** Mock LLM mode. The mock's `validate-request.json` includes one entry with `activeEligibility: false`.

**Steps:**
1. Submit any seeded request five times in a row.
2. Watch the live list until one card reaches `DETERMINED`.

**Expected:**
- The flagged determination shows outcome `PENDING_ADDITIONAL_INFO`.
- The rationale explains that eligibility was not confirmed.
- `evalStep` scores the determination with a check against outcome validity — `PENDING_ADDITIONAL_INFO` is a valid enum value, so that check passes. If the rationale is ≥ 15 words and the citation and procedure code checks pass, the score is 4 or 5.
- The workflow does NOT trigger `humanHoldStep` because `outcome != DENIED`. The card reaches `EVALUATED` without a human hold.

## J6 — Phase-gate rejection fires and the agent self-corrects

**Preconditions:** Mock LLM mode. The mock's `validate-request.json` includes one entry whose `tool_calls` array starts with a DETERMINE-phase tool (`buildRationale`) called during the VALIDATE phase.

**Steps:**
1. Submit any seeded request three times in a row (the phase-violating entry fires deterministically on the first iteration of the third request per `seedFor` modulo).
2. Watch the third request's lifecycle in the SSE stream (`/api/auth-requests/sse`).

**Expected:**
- On the third request's `validateStep`, the agent's first iteration calls `buildRationale`. `PhiSanitizer`'s phase-gate rejects it; a `PhaseGuardrailRejected{phase: "VALIDATE", tool: "buildRationale", reason: "phase-violation: ..."}` event lands on the entity.
- The rejected call NEVER reaches `DetermineTools.buildRationale` — there is no log line from the tool body.
- The agent's second iteration calls `checkMemberEligibility` — a valid VALIDATE-phase tool. The card eventually reaches `EVALUATED` as in J1.
- The right pane's phase-rejection-log strip shows the one rejected call with its full structured reason.
