# User journeys — lead-qualifier

## J1 — Submit an inquiry and get a qualified lead

**Preconditions:** Service running on declared port (`http://localhost:9612/`); a valid model-provider API key set, or the mock LLM selected at scaffold time. The seeded inquiry `Enterprise SaaS pricing for 500 seats` has a matching `src/main/resources/sample-data/inquiries/enterprise-saas-pricing.json` file.

**Steps:**
1. Open `http://localhost:9612/` → App UI tab.
2. From the **Pick a seeded inquiry** dropdown, pick `Enterprise SaaS pricing for 500 seats`.
3. Click **Qualify lead**.

**Expected:**
- The new card appears in the live list with status `CREATED` within 1 s, then `CAPTURING` within 1 s more.
- Within ~20 s the card reaches `CAPTURED`. The right pane shows the Captured-form panel: name-initial, company, channel, productInterest, and budgetIndicator are all non-empty. No raw email or phone appears anywhere in the UI.
- Within ~20 s more the card reaches `QUALIFIED`. The right pane shows a fit score ≥ 60 (enterprise-seat inquiry), urgency `HIGH` or `MEDIUM`, and recommendedStage `SQL` or `MQL`.
- Within ~20 s more the card reaches `ENRICHED`, then `EVALUATED` within 1 s of that. The CRM entry panel shows a valid stage, a non-empty ownerCandidate, and a notes field. The data-quality score chip shows 5/5.
- Total elapsed time: ≤ 60 s on the happy path.

## J2 — Phase-gate guardrail blocks a premature CRM write

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `capture-inquiry.json` includes one entry whose `tool_calls` array starts with a `writeCrmStage` call — an ENRICH-phase tool called during the CAPTURE phase.

**Steps:**
1. Submit any seeded inquiry three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/leads/sse`).

**Expected:**
- On the third submission's `captureStep`, the agent's first iteration calls `writeCrmStage`. `CrmWriteGuardrail` rejects it with a `phase-violation`; a `GuardrailRejected{phase: "CAPTURE", tool: "writeCrmStage", reason: "phase-violation: ..."}` event lands on the entity.
- The misordered call NEVER reaches `EnrichTools.writeCrmStage` — there is no log line from the tool body.
- The agent's second iteration falls through to a normal capture sequence (`parseContact` + `detectIntent`). The card eventually reaches `EVALUATED` as in J1.
- The card in the App UI shows the small red dot indicating a rejection fired. The rejection-log strip on the right pane shows the one rejected call with its full structured reason.

## J3 — Schema guardrail blocks an invalid CRM stage

**Preconditions:** Mock LLM mode. The mock's `enrich-crm.json` includes one entry where `stage = "Prospect"` — a value not in the allowed enum.

**Steps:**
1. Submit any seeded inquiry enough times that the invalid entry is selected (the mock's `seedFor(leadId)` selects it deterministically on a specific modulo — check `MockModelProvider.java` for the exact sequence).
2. Watch the lead's lifecycle in the App UI.

**Expected:**
- On `enrichStep`, the agent's first iteration calls `writeCrmStage` with `stage = "Prospect"`. `CrmWriteGuardrail` rejects it with a `schema-violation`; a second `GuardrailRejected` event lands on the entity.
- The agent corrects to `stage = "New"` on its next iteration; `writeCrmStage` is accepted.
- The card eventually reaches `EVALUATED`. The rejection-log strip shows exactly one schema-violation rejection.
- The data-quality score chip shows 4/5 (stage was corrected but the corrected stage `New` may not match the fit score's recommended stage, losing the fit-score-consistency check — or 5/5 if corrected stage is consistent).

## J4 — PII never appears in the API or UI

**Preconditions:** Service running. Any model provider. The seeded inquiry for `Integration partner referral – ERP migration project` has a contact with a real-looking email and phone in the sample data.

**Steps:**
1. Submit the seeded inquiry `Integration partner referral – ERP migration project`.
2. Wait for `EVALUATED`.
3. Inspect every SSE event payload received on `/api/leads/sse` (filter by leadId in the browser dev tools network panel).
4. Call `GET /api/leads/{id}` and examine the full response body.

**Expected:**
- No SSE event payload contains `rawEmail` or `rawPhone` fields.
- The `GET /api/leads/{id}` response body contains `maskedEmail` (8 hex chars + `@masked`) and `maskedPhone` (`[REDACTED]`) in the `crmEntry` block.
- The `form.contact` block in the response contains `nameInitial`, `company`, and `channel` only — no `rawEmail` or `rawPhone` keys.
- The service log at INFO level contains no lines including the literal email address or phone number from the sample data file.

## J5 — Disqualified lead routed correctly

**Preconditions:** Service running. Any model provider. The seeded inquiry `Demo request: compliance reporting module` has a `budgetIndicator = "<10k"` and a generic company name — the scoring heuristic classifies it as `DISQUALIFIED`.

**Steps:**
1. Submit the seeded inquiry `Demo request: compliance reporting module`.
2. Wait for `EVALUATED`.

**Expected:**
- The `LeadScore` panel shows `urgency = DISQUALIFIED` and `recommendedStage = "Disqualified"`.
- The `disqualificationReason` field is non-empty.
- The `CrmEntry` panel shows `stage = "Disqualified"` and a `notes` field explaining the disqualification.
- The data-quality score chip shows ≥ 3 (stage valid, owner assigned or explicitly skipped with reason, notes present).
- The pipeline completes without error; disqualification is a valid terminal outcome, not a failure.

## J6 — Custom inquiry with no matching sample file

**Preconditions:** Service running. Any model provider. No `src/main/resources/sample-data/inquiries/<slug>.json` exists for the user's text.

**Steps:**
1. In the App UI, type a novel inquiry (e.g., `Interested in your machine learning ops platform`) into the inquiry text field.
2. Click **Qualify lead**.

**Expected:**
- `CaptureTools.parseContact` returns a `ContactFields` with best-effort extraction from the raw text (or defaults if the text contains no parseable contact info).
- The pipeline progresses through all three phases without crashing.
- If the extracted `InquiryForm` contains no company and no budget indicator, `QualifyTools.scoreFit` returns a low score and `classifyUrgency` returns `LOW` or `DISQUALIFIED`.
- The pipeline reaches `EVALUATED`; the partial or disqualified result is honestly represented in the UI. Nothing crashes on an atypical input.
