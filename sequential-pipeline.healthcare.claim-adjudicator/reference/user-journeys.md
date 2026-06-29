# User journeys — claim-adjudication-agent

## J1 — Submit a routine lab claim and receive an approval

**Preconditions:** Service running on `http://localhost:9530/`; a valid model-provider API key set, or the mock LLM selected at scaffold time. Seeded claim `LAB-001` has a matching member file `PPO-100.json`, procedure-code file `lab.json`, and policy-rule file `lab-rules.json`.

**Steps:**
1. Open `http://localhost:9530/` → App UI tab.
2. From the **Pick a seeded claim** dropdown, pick `LAB-001 — Comprehensive metabolic panel`.
3. Click **Submit claim**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `VALIDATING` within 1 s more.
- Within ~20 s the card reaches `VALIDATED`. The right pane shows the Validation panel: eligibility chip shows "Eligible — PPO-100 Standard"; procedure code 80053 shows recognised = true, family = lab.
- Within ~20 s more the card reaches `EVALUATED_COVERAGE`. The Coverage panel shows ≥ 2 applicable rules (LAB-R1, LAB-R2), no exclusion rules, and covered percent ≥ 0.80.
- Within ~20 s more the card reaches `DECIDED` with outcome `APPROVED`, then `ADJUDICATION_EVALUATED` within 1 s. The Decision panel shows ≥ 2 cited rule IDs, a non-null approved amount, and a null denial code. The eval score chip shows ≥ 4.
- The PHI-redaction log strip shows at least two entries (`memberId`, `npi`). No raw PHI appears in any visible field.
- Total elapsed time: ≤ 90 s.

## J2 — Phase-gate guardrail blocks an out-of-phase tool call

**Preconditions:** Service running with the mock LLM selected. The mock's `validate-claim.json` includes one entry whose `tool_calls` array starts with `buildDecisionRationale` — an ADJUDICATE-phase tool called during VALIDATE.

**Steps:**
1. Submit any seeded claim three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the browser's network panel (`/api/claims/sse`).

**Expected:**
- On the third submission's `validateStep`, the agent's first iteration calls `buildDecisionRationale`. `PhiSanitizerGuardrail` first sanitises any PHI in the args (emitting `PhiRedacted` events), then detects the phase mismatch and rejects the call; a `GuardrailRejected{phase: "VALIDATE", tool: "buildDecisionRationale", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `AdjudicateTools.buildDecisionRationale` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal validate sequence (`checkEligibility` + `verifyProcedureCodes`). The card eventually reaches `ADJUDICATION_EVALUATED` as in J1.
- The card in the App UI shows the rejection-log strip with the one rejected call and its full structured reason.

## J3 — Denial claim triggers human-review hold

**Preconditions:** Service running. Seeded claim `DENIED-004` has an exclusion rule that sets `coveredPercent = 0.0` in the mock's `evaluate-coverage.json` entry.

**Steps:**
1. From the dropdown, pick `DENIED-004 — Excluded procedure`.
2. Click **Submit claim**.
3. Wait for the card to reach `PENDING_REVIEW`.
4. In the right pane's reviewer action panel, enter a reviewer ID and notes, then click **Approve denial**.

**Expected:**
- The card enters `PENDING_REVIEW` with an orange pill and a countdown timer showing the review deadline.
- The reviewer action panel appears with the decision summary, denial code, rationale narrative, and the two action buttons.
- After clicking **Approve denial**, the card transitions to `DENIED → ADJUDICATION_EVALUATED` within ~2 s.
- The Decision panel shows `outcome = DENIED` with a non-null denial code and the rationale citing the exclusion rule.
- The eval score reflects outcome-consistency and rule-citation checks; score ≥ 3 when both hold.

## J4 — Adjudication with insufficient rule citations scores low

**Preconditions:** Mock LLM mode. The mock's `adjudicate-claim.json` includes one entry that cites only one policy rule in `citedRuleIds`.

**Steps:**
1. Submit any seeded claim six times. (The single-citation entry is selected once in every six runs by the mock's `seedFor(claimId)` modulo.)
2. Watch the live list until a card's border highlights red.

**Expected:**
- The flagged claim's decision is otherwise well-formed (eligibility confirmed, outcome APPROVED).
- The eval score chip shows **2** and the rationale reads: `"Rule citation count below minimum: 1 cited rule (minimum 2)."`.
- The card border highlights red and the secondary-review badge appears.
- The other five claims in the run scored ≥ 4.

## J5 — PHI never appears in raw form in any API response

**Preconditions:** Service running. Any model provider. A browser dev-tools network panel open.

**Steps:**
1. Submit seeded claim `LAB-001`.
2. While the claim processes, open the Network tab and inspect every SSE event payload and every `GET /api/claims/{id}` response body.
3. After the claim reaches `ADJUDICATION_EVALUATED`, inspect the claim card's PHI-redaction log strip.

**Expected:**
- No response body contains a raw member name, date of birth, or unmasked NPI value.
- Every field that would carry PHI appears as `[REDACTED:<field>:<sha1-prefix>]`.
- The PHI-redaction log strip shows at least two entries with field names `memberId` and `npi`, each with a masked value and a timestamp.
- The `phiRedactions` array in the `ClaimRecord` JSON response mirrors the log strip exactly.

## J6 — Override-to-approve reverses a denial

**Preconditions:** Service running. Seeded claim `DENIED-004` to trigger the denial path.

**Steps:**
1. Submit `DENIED-004`.
2. Wait for `PENDING_REVIEW`.
3. In the reviewer action panel, click **Override to approve** with notes `"Manual review: exclusion does not apply to this member's plan tier."`.

**Expected:**
- The card transitions from `PENDING_REVIEW → APPROVED → ADJUDICATION_EVALUATED`.
- The Decision panel shows `outcome = APPROVED` (overridden), with the reviewer notes and reviewer ID recorded on the entity.
- The `DenialOverridden` event appears in the SSE stream before `EvaluationScored`.
- The eval score's outcome-consistency check passes for an overridden approval (the scorer treats an explicit override as a valid outcome regardless of the coverage evaluation's `coveredPercent`).
