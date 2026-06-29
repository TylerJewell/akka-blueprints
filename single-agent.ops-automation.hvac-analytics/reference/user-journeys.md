# User journeys — hvac-analytics

## J1 — Submit a cooling question and get a structured answer

**Preconditions:** Service running on declared port (`http://localhost:9151/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9151/` → App UI tab.
2. From the **Zone** dropdown, pick `Zone-A`.
3. Click the seeded question link **"Is Zone-A running above its cooling setpoint?"** to fill the question textarea.
4. Click **Ask**.

**Expected:**
- The new card appears in the live list with status `INITIATED` within 1 s.
- The card transitions to `SNAPSHOT_READY` within 1 s. The right-pane detail shows the telemetry snapshot table with supply-air-temp, return-air-temp, and other Zone-A metrics.
- Within 30 s the card reaches `ANSWER_RECORDED`. The right pane shows: a trend badge (IMPROVING / STABLE / DEGRADING / UNKNOWN), the assessment paragraph, a supporting data-points table with at least one non-empty significance note, and a recommended action beginning with an actionable verb.
- Within 1 s of `ANSWER_RECORDED`, the card reaches `EVALUATED` and shows a quality score chip (1–5) plus a one-line rationale.

## J2 — Multiple questions accumulate distinct eval scores

**Preconditions:** Service running. Any model provider (real or mock).

**Steps:**
1. Submit three different seeded questions — one for Zone-A, one for Zone-B, one for Zone-C — in sequence using J1 steps.
2. Wait for all three cards to reach `EVALUATED`.

**Expected:**
- All three cards display a quality score chip.
- The scores are not all identical (the seeded telemetry is varied: Zone-A is above setpoint, Zone-B is normal, Zone-C is degrading — the answers and their evidence quality differ).
- The Eval Matrix tab's rolling-average chart updates to reflect the three scores.

## J3 — Evidence-thin answer flags eval score 1

**Preconditions:** Mock LLM mode. The mock's `analyse-telemetry.json` includes deliberately thin entries with an empty `dataPoints` list.

**Steps:**
1. Submit any question in the App UI.
2. The mock selects the thin entry (every 4th query in the seed sequence).
3. Wait for `EVALUATED`.

**Expected:**
- The answer lands with `status = ANSWER_RECORDED` (the evaluator is non-blocking — the answer is recorded regardless of score).
- The quality score chip shows **1** and the rationale reads "dataPoints list is empty; answer is not anchored in the telemetry snapshot."
- The card's border highlights red.
- The engineer knows to verify this answer independently before acting.

## J4 — Equipment identifiers never reach the agent

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the agent task attachment is logged. Any model provider.

**Steps:**
1. Inspect `src/main/resources/sample-events/telemetry-data.jsonl` — confirm it contains an `equipmentSerialNumber` field on each raw telemetry record.
2. Submit a Zone-B question.
3. Wait for `SNAPSHOT_READY`.
4. Inspect the service log for the telemetry attachment sent to the agent (`debug:agent.task.attachment`).

**Expected:**
- The logged attachment does not contain any `equipmentSerialNumber` field — `TelemetryStore` strips that field before calling `attachSnapshot`.
- The `snapshot.points` array in `GET /api/queries/{id}` also has no serial-number field — the stripped form is what the entity stores.
- The in-process data store retains the full fidelity records (accessible only from within the `TelemetryStore` implementation).

## J5 — Zone-C degrading trend is detected and recommended action is specific

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit the seeded question **"Is Zone-C showing signs of a degrading trend?"** for `Zone-C`.
2. Wait for `EVALUATED`.

**Expected:**
- The answer's `trend` badge is `DEGRADING`.
- The supporting data points include at least two return-air-temp readings showing an upward progression over time.
- The recommended action begins with one of the recognised actionable verbs ("Inspect", "Schedule", "Verify", etc.) and specifically names Zone-C.
- The eval score is ≥ 3 (evidence quality is good when the seeded data is used correctly).

## J6 — Multi-zone question scopes the snapshot correctly

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the **Zone** dropdown, pick `custom` and enter `Zone-A,Zone-B`.
2. Type the question "Compare Zone-A and Zone-B cooling energy consumption."
3. Click **Ask**.
4. Wait for `EVALUATED`.

**Expected:**
- The telemetry snapshot displayed in the right pane includes data points for both Zone-A and Zone-B (cooling-energy-kwh metric for both zones).
- The assessment paragraph references both zones by name.
- The supporting data points table includes at least one entry per zone.
- No Zone-C data appears in the snapshot — the scope is correctly limited to the requested zones.
