# User journeys — gemma-food-tour-guide

## J1 — Submit a 3-day Tokyo tour and get a full itinerary

**Preconditions:** Service running on declared port (`http://localhost:9348/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9348/` → App UI tab.
2. From the **City** dropdown, pick `Tokyo`. Set **Duration** to `3`. Select `vegetarian` and `gluten-free` in **Dietary styles**. Set **Budget tier** to `Street food`.
3. Click **Plan my tour** (or click **Load seeded example** to prefill all fields, then submit).

**Expected:**
- The new card appears in the live list with status `REQUESTED` within 1 s.
- The card transitions to `PREFERENCES_VALIDATED` within 1 s. The right-pane detail shows the normalized category tag cloud: `vegetarian`, `gluten-free`.
- Within 30 s the card reaches `ITINERARY_RECORDED`. The right pane shows: a decision badge (`FULL_PLAN`), the summary paragraph, and 3 collapsible day sections. Each day section contains at least 3 venue cards. Every venue card has a non-empty `venueName`, `neighborhood`, `culturalNote`, and a `description` exceeding 20 characters.
- Within 1 s of `ITINERARY_RECORDED`, the card reaches `SCORED` and shows a coverage score chip (1–5) plus a one-line rationale.
- The itinerary contains at least one stop tagged `vegetarian` and at least one tagged `gluten-free` somewhere across the 3 days.

## J2 — Guardrail blocks a malformed itinerary

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `plan-food-tour.json` includes deliberately malformed entries (a stop with `dayIndex` outside the requested duration; a stop with an unrecognized `mealSlot`).

**Steps:**
1. Submit any seeded tour request three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/tours/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed itinerary.
- The `before-agent-response` guardrail rejects it. The malformed itinerary NEVER lands in `TourRequestEntity` — there is no `ItineraryRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed itinerary. The card transitions to `ITINERARY_RECORDED` with an itinerary that satisfies all four guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed (e.g., `day-index-out-of-range:4,max:3` or `invalid-meal-slot:BRUNCH`).

## J3 — Coverage-thin itinerary flags score 1

**Preconditions:** Mock LLM mode. A specific mock-response entry returns an itinerary where every stop shares the same `mealSlot` (LUNCH) and all `description` values are fewer than 20 characters.

**Steps:**
1. In the App UI, submit a 1-day Barcelona request with `pescatarian` dietary style. The mock selects the coverage-thin entry based on the deterministic seed.

**Expected:**
- The itinerary lands well-formed (the guardrail validates structure, not quality).
- The coverage score chip shows **1** and the rationale reads something like "All stops share the same meal slot; descriptions are too terse to guide a traveler."
- The card's border highlights red.
- The traveler knows to request a revised itinerary before booking.

## J4 — Health notes normalized before LLM sees them

**Preconditions:** Service running with `LOG_LEVEL=DEBUG`. Any model provider.

**Steps:**
1. Submit a Tokyo tour request with `notes` set to `"I have celiac disease and also lactose intolerance. I take metformin daily."` and `dietaryStyles` set to `[]` (no explicit checkboxes).
2. Wait for `ITINERARY_RECORDED`.
3. Inspect the service log for the agent task instructions payload.
4. Fetch `GET /api/tours/{id}` and read `preferences.notes` and `validated.normalizedCategories`.

**Expected:**
- The logged agent task instructions contain `normalizedCategories: ["gluten-free", "dairy-free"]`. The raw notes string does not appear in the agent's task payload.
- `preferences.notes` in the JSON still contains the original text — the entity audit log preserves it.
- `validated.normalizedCategories` lists `gluten-free` and `dairy-free` (at minimum).
- The generated itinerary contains at least one stop tagged `gluten-free` and at least one tagged `dairy-free`.

## J5 — Multi-day dietary coverage across all days

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a 5-day Barcelona tour with `["vegan", "pescatarian"]` dietary styles and `MID_RANGE` budget.
2. Wait for `SCORED`.

**Expected:**
- The itinerary's `days` array has exactly 5 entries (`dayIndex` 1–5).
- At least one stop tagged `vegan` and at least one stop tagged `pescatarian` appear somewhere across the 5 days.
- No individual day has all stops sharing the same `mealSlot`.
- The coverage score is ≥ 3.

## J6 — Unknown city returns NEEDS_MORE_INFO without refusing

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Submit a tour request for a city not in the seeded corpus (e.g., `"Reykjavik"`).

**Expected:**
- The agent does not refuse the task.
- The itinerary arrives with `decision = NEEDS_MORE_INFO`.
- The `summary` explains that the city profile is not in the corpus and directs the user to add a city profile entry.
- The card reaches `SCORED` (the scorer runs over whatever partial itinerary the agent returned).
- The UI displays the `NEEDS_MORE_INFO` badge in red on the card.
