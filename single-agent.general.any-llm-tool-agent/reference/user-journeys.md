# User journeys — any-llm-tool-agent

## J1 — Submit a weather query and receive a report

**Preconditions:** Service running on declared port (`http://localhost:9599/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9599/` → App UI tab.
2. Click **Load example** next to "Tokyo" to fill the Query textarea.
3. Leave the Backend dropdown on `mock`.
4. Click **Ask**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `INVOKING` within 1 s. The right pane shows the spinner "Agent is calling get_weather…".
- Within 30 s the card reaches `RESULT_RECORDED`. The right pane shows: the location header "Tokyo", a condition badge, three stat tiles (temperature, humidity, wind), and a one-sentence narrative.
- The `report.data.condition` value is one of `SUNNY`, `CLOUDY`, `RAINY`, `SNOWY`, or `UNKNOWN` — never null.
- The `report.narrative` is non-empty and mentions the location by name.

## J2 — Guardrail rejects a blank location; agent returns a clarification report

**Preconditions:** Service running with mock LLM. The mock's `weather-report.json` includes a guardrail-path entry that triggers when the agent produces an empty-string `location` argument.

**Steps:**
1. In the Query textarea, type "What's the weather?" (no location).
2. Click **Ask**.
3. Watch the card lifecycle and the network panel for `/api/queries/sse`.

**Expected:**
- The card reaches `RESULT_RECORDED`.
- The `WeatherReport.data.condition` is `UNKNOWN`.
- The `WeatherReport.narrative` contains the text "I could not determine a location from your query."
- The service log shows one `guardrail.reject` line with error code `invalid-tool-input` for the empty-location attempt.
- The `get_weather` tool method body is never executed with the blank string — confirmed by the absence of any `WeatherTools.get_weather` log entry for that invocation.

## J3 — Same query, different backend declarations

**Preconditions:** Service running. At least one real API key is configured; mock is also available.

**Steps:**
1. Submit the Tokyo query with Backend = `anthropic`.
2. Submit the same Tokyo query with Backend = `mock`.
3. Wait for both cards to reach `RESULT_RECORDED`.

**Expected:**
- Both cards show a valid `WeatherReport` with a non-null location and a condition badge.
- The backend badge on each card shows the declared backend correctly.
- Neither card shows an error state. The Java source for `WeatherAgent`, `WeatherTools`, and `WeatherToolGuardrail` is identical for both runs — no code branch on backend.

## J4 — SSE stream reflects every status transition

**Preconditions:** Service running. Browser dev tools open on the Network panel.

**Steps:**
1. Open the `/api/queries/sse` endpoint in the Network panel (or observe the App UI tab's live list).
2. Submit a query.
3. Observe the SSE events without refreshing the page.

**Expected:**
- At least two SSE events arrive for the query: one with `"status": "SUBMITTED"` and one with `"status": "RESULT_RECORDED"` (and optionally one with `"status": "INVOKING"`).
- Each event carries the full query row at the moment of transition — including `report: null` for the `SUBMITTED` event and a populated `report` for `RESULT_RECORDED`.
- The App UI's live list updates in real time; no page refresh is required.
- A second browser tab opened after the first query completes shows the completed query immediately (the view serves the full history, not just live events).
