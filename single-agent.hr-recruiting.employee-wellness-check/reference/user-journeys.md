# User journeys — wellness-check-agent

## J1 — Launch a campaign and receive a standard morale analysis

**Preconditions:** Service running on declared port (`http://localhost:9338/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9338/` → App UI tab.
2. In the Campaign panel, select `Morale Pulse (5Q)` from the dropdown, enter a campaign name, and click **Launch campaign**.
3. In the check-in submission panel, select the new campaign, enter any `employeeRef` token (e.g. `emp-ref-0001`), and click **Load seeded example** to fill the answer fields.
4. Click **Submit response**.

**Expected:**
- The new check-in card appears in the live list with status `RECEIVED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the redacted answer map. If the seeded response contained any special-category marker terms, the category chips appear above the redacted answers.
- Within 30 s the card reaches `ANALYSIS_RECORDED`. The right pane shows: a morale-level badge (HIGH / MODERATE / LOW), a 1–3-sentence interpretation, and a non-empty recommendation beginning with an actionable verb.
- Within 1 s of `ANALYSIS_RECORDED`, the card reaches `EVALUATED` and the surveillance section shows `riskFlagRaised: false` with a ratio rationale.

## J2 — Guardrail blocks a malformed analysis

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `analyse-check-in.json` includes deliberately malformed entries (one with `moraleLevel` outside the enum; one with a null `recommendation`).

**Steps:**
1. Submit any seeded response three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/check-ins/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed analysis.
- The `before-agent-response` guardrail rejects it. The malformed analysis NEVER lands in `CheckInEntity` — there is no `AnalysisRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 and produces a well-formed analysis. The card transitions to `ANALYSIS_RECORDED` with a valid `moraleLevel`.
- The service log shows one `guardrail.reject` line per rejected iteration, naming which check failed.

## J3 — Crisis signal triggers escalation path

**Preconditions:** Service running with mock LLM. The mock's `analyse-check-in.json` includes one entry with `moraleLevel: CRISIS` and `crisisFlag: true`.

**Steps:**
1. Observe the mock selection pattern (the CRISIS entry is selected on a specific seed — consult `MockModelProvider.seedFor()` in the generated code to identify the `employeeRef` or iteration count that triggers it).
2. Submit a response that will trigger the CRISIS mock entry.

**Expected:**
- `ResponseGuardrail` intercepts `crisisFlag: true` in the `before-agent-response` hook and calls `CheckInEntity.escalateCrisis()` before accepting.
- `CheckInEntity` emits `CrisisEscalated` and transitions to `ESCALATED`.
- The check-in card disappears from the standard morale list and appears only in a dedicated crisis queue with the red banner: "Crisis signal detected — routed to human. This check-in is excluded from aggregate morale scores."
- The `crisisCount` on the parent campaign increments by 1. The check-in's `analysis.moraleLevel` is `CRISIS` but the card is never surfaced as a normal morale result in the UI.

## J4 — Special-category data never reaches the LLM

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. Submit a response whose answers contain the literal test tokens `[MENTAL-HEALTH-MARKER-TEST]`, `[DISABILITY-REF-TEST]`, and the phrase "anxiety and exhaustion" (a phrase the sanitizer's heuristic stage matches).
2. Wait for `ANALYSIS_RECORDED`.
3. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
4. Fetch `GET /api/check-ins/{id}` and read `response.answers`.

**Expected:**
- The logged LLM call body contains only the redacted forms. The raw test tokens and matched phrases do not appear.
- `response.answers` in the JSON still contains the raw strings — the audit log preserves them.
- `sanitized.specialCategoriesFound` lists at minimum `mental-health-marker` and `disability-marker`.

## J5 — Post-market-surveillance risk flag triggers on high LOW+CRISIS ratio

**Preconditions:** Service running with mock LLM. `MoraleSurveillance` threshold is 0.25.

**Steps:**
1. Launch a campaign.
2. Submit 8 check-in responses. Arrange (via mock seed control or manual submission) so that 3 of the 8 produce `LOW` and 1 produces `CRISIS` — giving a ratio of 4/8 = 0.50, above the 0.25 threshold.

**Expected:**
- After the 4th low-or-crisis response lands, the next `surveillanceStep` execution produces `SurveillanceEvaluated{riskFlagRaised: true}`.
- The campaign card in the left column shows the yellow **Risk review required** banner.
- The surveillance section on the selected check-in shows `riskFlagRaised: true` with a rationale that names the measured ratio (e.g. "Campaign LOW+CRISIS ratio is 0.50; exceeds threshold of 0.25.").
- No automated action is taken. The banner is advisory only — a human HR decision-maker must acknowledge it.
