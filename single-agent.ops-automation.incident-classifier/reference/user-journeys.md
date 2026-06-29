# User journeys — incident-classifier

## J1 — Submit a database-failure incident and get a valid classification

**Preconditions:** Service running on `http://localhost:9562/`; a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9562/` → App UI tab.
2. From the **Seeded examples** dropdown, pick `Database connection failure`.
3. Click **Load seeded example** to fill the short and long description fields.
4. Enter any string in **Caller ID** (e.g. `ops-engineer-01`). Leave **Priority hint** as P2.
5. Click **Submit incident**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `TAXONOMY_VALIDATED` within 1 s. The taxonomy scope tile in the right pane shows 6 categories, ~28 subcategories, ~24 CIs.
- Within 30 s the card reaches `CLASSIFICATION_RECORDED`. The right pane shows: a `Database` category chip, a `Connectivity` subcategory chip, an `PROD-DB-01` CI label, and the agent's rationale sentence.
- Within 1 s of `CLASSIFICATION_RECORDED`, the card reaches `EVALUATED` and shows an eval score chip of 5 (green) and the rationale "All three fields match the taxonomy exactly."

## J2 — On-decision evaluator flags an out-of-vocabulary classification

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `classify-incident.json` includes a deliberately out-of-vocabulary entry (category `UnknownService`).

**Steps:**
1. Submit any seeded incident four times in a row (J1 steps × 4, using the same seeded example each time).
2. Watch the fourth submission's card in the live list.

**Expected:**
- The fourth submission receives the out-of-vocabulary mock response (deterministic per `seedFor(incidentId)` modulo 4).
- The eval score chip shows **1** (red).
- The three boolean indicators in the right pane show: Category in taxonomy = false, Subcategory valid = false, CI in registry = false.
- The card border highlights red. The "Review classification" button appears.
- No `EVALUATED` transition is suppressed — the evaluation is non-blocking; the classification still lands in the entity log with the low score attached.

## J3 — Rolling accuracy window reflects classification outcomes

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit 5 incidents in sequence using different seeded examples (mix of database failure, VPN outage, file-share denial).
2. After all 5 reach `EVALUATED`, open the Eval Matrix tab.

**Expected:**
- The accuracy panel shows a percentage ≥ 60% (at least 3 of 5 are fully correct when using the real LLM; with the mock LLM, the rate reflects the mock's out-of-vocabulary ratio).
- The sparkline shows a bar for today's date with height proportional to the daily count.
- Submitting one more incident with the mock's out-of-vocabulary path decreases the percentage by one contribution unit.

## J4 — Mock LLM out-of-vocabulary path is reproducible

**Preconditions:** Service running with mock LLM. Use the exact same seeded incident (database connection failure) submitted multiple times.

**Steps:**
1. Submit the database-failure seeded incident exactly 4 times.
2. After each submission reaches `EVALUATED`, note the eval score.

**Expected:**
- Submissions 1, 2, 3 receive well-formed classification results with score ≥ 3.
- Submission 4 (index 3 modulo 4 = 0 in the seed) receives the out-of-vocabulary result with score 1.
- Restarting the service and resubmitting produces the same pattern — `seedFor(incidentId)` is deterministic across restarts.

## J5 — Workflow failure path lands in FAILED state

**Preconditions:** Service running. Mock LLM mode.

**Steps:**
1. Temporarily modify the taxonomy.json to have 0 categories (empty array), restart the service.
2. Submit any incident.

**Expected:**
- `VocabularyValidator` still emits `TaxonomyValidated` (it counted 0 categories), so the workflow starts.
- `classifyStep` calls the agent, which returns a response. `AccuracyEvaluator` scores it 0 (no category in taxonomy, no subcategory, no CI). Score 0 emits `AccuracyScored` with score 1.
- The incident reaches `EVALUATED` with score 1 and all three boolean indicators false. The card border is red.
- Restoring taxonomy.json and resubmitting produces a well-formed score.

## J6 — SSE client receives all state transitions without polling

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Open the browser's network panel. Navigate to the App UI tab.
2. Confirm the UI opens an `EventSource` connection to `/api/incidents/sse`.
3. Submit one incident and observe the network panel.

**Expected:**
- The SSE stream delivers exactly 5 events for a happy-path incident: `SUBMITTED`, `TAXONOMY_VALIDATED`, `CLASSIFYING`, `CLASSIFICATION_RECORDED`, `EVALUATED`.
- Each event carries the full incident row at that moment (no partial payload).
- No polling request (no repeated `GET /api/incidents` at an interval) appears in the network panel. The UI updates only on SSE push.
