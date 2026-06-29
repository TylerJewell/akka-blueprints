# User journeys — fun-facts

## J1 — Submit a topic and get a fact collection

**Preconditions:** Service running on declared port (`http://localhost:9195/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9195/` → App UI tab.
2. Click the **Octopus Intelligence** quick-pick button. The topic field fills automatically.
3. Enter any value in the **Requested by** field.
4. Click **Generate facts**.

**Expected:**
- The new card appears in the live list with status `PENDING` within 1 s.
- The card transitions to `GENERATING` within 1 s. The right pane shows the topic and a loading indicator.
- Within 30 s the card reaches `GENERATED`. The right pane shows: a headline fact in the prominent display block and at least 3 supporting facts, each with a non-empty `area` chip, a non-empty `statement`, and a `confidence` chip (HIGH / MEDIUM / LOW).
- No page reload is required; the update arrives via SSE.

## J2 — Same topic submitted twice produces independent results

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit `"Black Holes"` (J1 steps with that topic). Wait for `GENERATED`.
2. Submit `"Black Holes"` again.

**Expected:**
- Both cards appear in the live list with distinct `requestId` values.
- Both reach `GENERATED` independently.
- Selecting each card in turn shows two separate `FactCollection` objects (the mock LLM's `seedFor(requestId)` ensures different selections; a real LLM will produce naturally different outputs).
- Neither request interferes with the other's lifecycle.

## J3 — Live list updates without page reload

**Preconditions:** Service running. App UI tab open. SSE connection established.

**Steps:**
1. Submit `"Ancient Roman Engineering"`.
2. Do not reload the page.
3. Observe the live list and right pane.

**Expected:**
- The card status transitions from `PENDING` → `GENERATING` → `GENERATED` are all reflected in the live list in real time via SSE.
- The right pane updates automatically when the active card transitions to `GENERATED` — no user action required.
- The browser dev tools network panel shows one persistent `EventStream` connection to `/api/fact-requests/sse` carrying the transition events.

## J4 — Blank topic rejected at the endpoint

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Clear the topic field (or do not fill it).
2. Click **Generate facts**.

**Expected:**
- The endpoint returns `400` with body `{ "error": "topic must not be blank" }`.
- No card appears in the live list.
- No `FactRequested` event is emitted; no entity is created.
- The UI displays the error message inline beneath the topic field without navigating away.

## J5 — Failed generation is surfaced clearly

**Preconditions:** Service running with the mock LLM selected. A mock configuration that forces a failure on the `generate-facts` task (simulated by exhausting all retry iterations with parse errors).

**Steps:**
1. Submit any topic while the mock is configured to fail all iterations.
2. Wait for the lifecycle to complete.

**Expected:**
- The card transitions from `PENDING` → `GENERATING` → `FAILED`.
- The status pill on the card is red.
- The right pane shows: topic, status `FAILED`, and a reason string from `RequestFailed.reason`.
- No `CollectionGenerated` event is emitted; the entity's `collection` field remains `null` in the JSON response from `GET /api/fact-requests/{id}`.
