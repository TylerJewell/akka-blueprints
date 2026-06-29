# User journeys — weather-agent

Acceptance bar for the generated system. Each journey defines preconditions, steps, and the expected outcome that signals the journey passed.

---

## J1 — Current conditions, known city

**Preconditions:** The service is running. The stub data includes Tokyo. No prior queries in the live list.

**Steps:**
1. Open the App UI tab.
2. Type "What is the weather in Tokyo right now?" into the Question textarea.
3. Select Metric units.
4. Click **Ask**.

**Expected:**
- A query card appears in the live list in PROCESSING state within 1 s.
- Within 30 s the card transitions to ANSWERED.
- The right pane shows a tool-call timeline with exactly two entries: `geocode(location="Tokyo") → success` and `getWeather(lat=35.68, lon=139.65, units="metric", forecastDays=1) → success`.
- The current conditions card displays temperature, feels-like, wind speed, humidity, and a description string.
- `answer.narrative` is a 1–3 sentence paragraph that names the city and conditions.
- No forecast table appears (forecastDays was 1).

---

## J2 — Guardrail rejects empty location on first attempt

**Preconditions:** Mock LLM mode is active. The mock is seeded to produce an empty-location `geocode` call on the first iteration of every 4th query.

**Steps:**
1. Submit enough queries to reach the 4th (or configure the mock seed to trigger on the first query in a fresh session).
2. Watch the tool-call timeline in the right pane.

**Expected:**
- The first `ToolCallRecorded` event for the `geocode` tool shows `outcome = "rejected"` with a parameter chip `location=""`.
- A second `ToolCallRecorded` event for `geocode` follows with a non-empty location and `outcome = "success"`.
- The query still reaches ANSWERED status (the agent corrected itself within its 4-iteration budget).
- The UI never displays the rejected call as the final result — only the successful answer appears in the weather answer block.

---

## J3 — Forecast request capped at 16 days

**Preconditions:** The service is running.

**Steps:**
1. Type "Give me a 20-day forecast for London." into the Question textarea.
2. Select Metric units.
3. Click **Ask**.

**Expected:**
- The tool-call timeline shows a `getWeather` call with `forecastDays` parameter attempting 20.
- `ToolCallGuardrail` rejects it with a rejection naming the `forecastDays > 16` check.
- A second `getWeather` call follows with `forecastDays=16` and `outcome = "success"`.
- The forecast table in the right pane shows 16 rows (one per day).
- The query reaches ANSWERED status.

---

## J4 — Unknown location transitions to FAILED

**Preconditions:** The service is running. The stub geocoding data does not include "Zyxwvut City".

**Steps:**
1. Type "What is the weather in Zyxwvut City?" into the Question textarea.
2. Click **Ask**.

**Expected:**
- The tool-call timeline shows a single `geocode(location="Zyxwvut City") → success` entry (geocode ran; it just returned no results).
- The query card transitions to FAILED within 10 s.
- The right pane shows the failure callout with text: `"Geocoding returned no results for: Zyxwvut City"`.
- No weather conditions block or forecast table appears.

---

## J5 — Multi-location comparison question

**Preconditions:** The service is running. Both New York and Los Angeles are in the stub data.

**Steps:**
1. Type "How does the weather in New York compare to Los Angeles today?" into the Question textarea.
2. Select Imperial units.
3. Click **Ask**.

**Expected:**
- The tool-call timeline shows four entries: `geocode("New York") → success`, `geocode("Los Angeles") → success`, `getWeather(NewYork coords, "imperial", 1) → success`, `getWeather(LosAngeles coords, "imperial", 1) → success`.
- The weather answer block shows conditions for the primary location (New York) with coordinates from the geocoding result.
- `answer.narrative` mentions both cities and compares their conditions.
- The query reaches ANSWERED status.

---

## J6 — Seeded example loads and answers correctly

**Preconditions:** The service is running.

**Steps:**
1. Click one of the four seeded-example buttons (e.g., "5-day forecast for London").
2. Verify the Question textarea fills in automatically.
3. Click **Ask**.

**Expected:**
- The question text matches the seeded example exactly.
- The query follows the normal PROCESSING → ANSWERED path.
- For the "5-day forecast" example, the forecast table shows at least 5 rows.
- Unit system is pre-set to Metric by the seeded example button.
