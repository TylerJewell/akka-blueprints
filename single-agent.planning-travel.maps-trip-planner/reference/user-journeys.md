# User journeys — maps-trip-planner

Acceptance bar for the generated system. Each journey lists preconditions, steps, and expected outcomes.

---

## J1 — Happy path: 3-day Paris itinerary

**Preconditions:** Service running. No blocked destinations in the request. Mock LLM (or real key) configured.

**Steps:**
1. Open the App UI tab.
2. Click "Load seeded example → Paris".
3. Verify the form fills: origin = London, destinations = Paris, dates = 3 days, party size = 2, preferences = "cultural landmarks, vegetarian meals".
4. Click **Plan my trip**.
5. Observe the new card appear in the live list in `SUBMITTED` state.
6. Within ~1 s, the card transitions to `SCREENED` (PASS).
7. Within ~30 s, the card transitions through `PLANNING` to `ITINERARY_RECORDED`.
8. Verify the itinerary shows exactly 3 `ItineraryDay` entries, each with at least 2 stops and at least 1 transit leg.
9. Within ~1 s of the itinerary, the card transitions to `EVALUATED` and shows an eval score chip.
10. Click the card; verify the right pane shows the day timeline, preferences coverage (cultural landmarks and vegetarian meals both ticked), and the eval score.

**Expected:** Trip status = EVALUATED; itinerary quality ∈ {EXCELLENT, GOOD}; eval score ≥ 3; no console errors.

---

## J2 — Guardrail retry: malformed itinerary on first iteration

**Preconditions:** Mock LLM configured. The mock is seeded to return a malformed entry on the first iteration of every 3rd trip (stops list empty on day 1).

**Steps:**
1. Submit enough trips to trigger the 3rd (or 6th, 9th…) trip where the mock returns the malformed entry first.
2. Observe the card transition to `PLANNING`.
3. Confirm the card does NOT briefly show a partial itinerary with empty stops; the UI should stay in `PLANNING` until a valid response arrives.
4. Within ~30 s, the card transitions to `ITINERARY_RECORDED` with a well-formed itinerary (agent retried on iteration 2).
5. Confirm the entity event log contains no `ItineraryRecorded` event carrying an itinerary with empty stops.

**Expected:** UI never displays a malformed itinerary; final status = EVALUATED; the guardrail rejection appears in the service log but not in any user-facing field.

---

## J3 — Blocked destination: restricted keyword in request

**Preconditions:** Service running. `blocked-destinations.txt` contains `demo-block-a`.

**Steps:**
1. In the App UI, set destination = `demo-block-a-city` (contains the blocked keyword).
2. Fill other fields with valid values and click **Plan my trip**.
3. Observe the card appear in `SUBMITTED` state.
4. Within ~1 s, the card transitions to `BLOCKED`.
5. Click the card; verify the screen result banner shows red BLOCKED and the reason string "Destination matches blocked keyword: demo-block-a".
6. Verify no `PlanningStarted` event appears in the entity log for this trip.

**Expected:** Trip status = BLOCKED; no agent call; the right pane shows the block reason; the trip is excluded from any "EVALUATED" or "PLANNING" filter on the list.

---

## J4 — Eval flags zero-duration transit legs

**Preconditions:** Mock LLM configured to produce a trip where all `TransitLeg.durationMinutes = 0` (a mock entry with this defect seeded in `plan-trip.json`).

**Steps:**
1. Trigger a trip that resolves to the zero-duration mock response (force by adjusting the mock seed, or by using a test endpoint that pins the mock index).
2. Observe the card reach `ITINERARY_RECORDED`.
3. Verify the guardrail did NOT block this response (durationMinutes = 0 is the guardrail's boundary; the guardrail rejects values that are strictly negative, not zero — this tests the eval path).
4. Observe the card transition to `EVALUATED` with eval score = 1.
5. Click the card; verify the eval rationale states that transit durations are implausible.
6. Verify the card border is highlighted red (score ≤ 2).

**Expected:** Trip status = EVALUATED; eval score = 1; rationale mentions transit duration; card border red.

---

## J5 — Partial preferences coverage downgrades quality

**Preconditions:** Mock LLM configured to produce a trip where `preferencesAddressed` contains only half the keywords from the original request (e.g., request had 4 preference keywords; itinerary addresses 2).

**Steps:**
1. Submit a trip with 4 distinct preference keywords.
2. Observe the card reach `ITINERARY_RECORDED` with `quality = PARTIAL`.
3. Click the card; verify the preferences coverage section shows 2 keywords ticked and 2 muted.
4. Verify the eval score reflects the partial coverage (score ≤ 3).

**Expected:** quality badge = PARTIAL (yellow); eval score ≤ 3; unaddressed preferences visually distinct in the UI.

---

## J6 — SSE late-join delivers full current state

**Preconditions:** Service running with at least one EVALUATED trip already in the system.

**Steps:**
1. Open a new browser tab pointing at the App UI.
2. Without submitting a new trip, observe the live list.
3. Verify the previously evaluated trip appears immediately (not after a state transition).

**Expected:** The SSE endpoint delivers the full row for all existing trips on connect; no polling gap; the list is populated within ~1 s of the tab opening.
