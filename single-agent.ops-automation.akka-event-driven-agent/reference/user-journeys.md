# User journeys — ambient-durable-agent-pubsub

## J1 — Publish a valid request and receive a completed forecast

**Preconditions:** Service running on declared port (`http://localhost:9390/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9390/` → App UI tab.
2. From the **Seeded payload** dropdown, pick `standard-daily-request`.
3. Click **Load seeded example** to fill location, horizon, and alert categories.
4. Click **Publish**.

**Expected:**
- The new card appears in the live list with status `RECEIVED` within 1 s.
- The card transitions to `VALIDATED` within ~1 s. The right-pane detail shows the green "VALID" chip.
- The card transitions to `SANITIZED` within ~1 s. The PII category chips appear (for the standard-daily-request, `email` at minimum if `requesterContact` was populated).
- Within 30 s the card reaches `COMPLETED`. The right pane shows: conditionsSummary paragraph, temperature range bar, wind chip, precipitation chip. `hazardFlag` is false for the standard-daily-request; no hazard badge appears.

## J2 — Guardrail blocks a malformed payload

**Preconditions:** Service running. Any model provider or mock.

**Steps:**
1. In the App UI's Publish panel, clear the **Location** field entirely.
2. Set **Horizon** to `DAILY_3D` and leave all other fields at defaults.
3. Click **Publish**.

**Expected:**
- The new card appears with status `RECEIVED` then immediately transitions to `FAILED` (within ~1 s).
- The right-pane validation result shows a red chip with violation code `location.required`.
- No agent task was started: the service log shows no `agent.task.start` entry for this `requestId`.
- The card remains in `FAILED` state permanently — there is no retry for a validation failure.

## J3 — PII is scrubbed from the agent task attachment

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the agent task attachment is logged.

**Steps:**
1. In the Publish panel, set **Requester contact** to `jane.ops@example.com`, **Location** to `Denver, CO`, **Horizon** to `DAILY_3D`.
2. Click **Publish**.
3. Wait for `COMPLETED`.
4. Inspect the service log for the agent task attachment (`debug:agent.task.attachment`).
5. Fetch `GET /api/requests/{id}` and read `rawPayload.requesterContact`.

**Expected:**
- The logged task attachment contains `[REDACTED-EMAIL]` in place of `jane.ops@example.com`. The raw string does not appear anywhere in the attachment.
- `rawPayload.requesterContact` in the JSON response still contains `jane.ops@example.com` — the audit log retains the original.
- `sanitized.piiCategoriesFound` lists `email` (at minimum).

## J4 — Two concurrent messages produce independent workflow executions

**Preconditions:** Service running. Any model provider or mock.

**Steps:**
1. Click **Publish** twice in rapid succession (within ~500 ms) using the same seeded payload (different `requestId` values will be minted by the endpoint).
2. Watch the live list.

**Expected:**
- Two cards appear, each with its own `requestId`.
- Both progress through their lifecycle independently; neither card's status is blocked on the other.
- Both cards reach `COMPLETED` (or `FAILED` if the mock selection returns an error path) with separate `WeatherReport` entries.
- The service log shows two distinct `forecast-<requestId>` workflow traces with no interleaving of state writes.

## J5 — Hazard flag surfaces on a storm-alert request

**Preconditions:** Service running. Mock LLM selected (so the tropical-storm-alert mock entry — which has `hazardFlag = true` — is returned).

**Steps:**
1. From the **Seeded payload** dropdown, pick `tropical-storm-alert`.
2. Click **Load seeded example**, then **Publish**.
3. Wait for `COMPLETED`.

**Expected:**
- The completed card shows a red `HAZARD` badge.
- The right-pane weather report section shows a red-bordered hazard panel with the `hazardDetail` string from the mock response (describing the storm category and expected gusts).
- `report.hazardFlag` is `true` in `GET /api/requests/{id}`.

## J6 — Invalid horizon enum value is rejected

**Preconditions:** Service running. Any model provider.

**Steps:**
1. POST directly to `/api/requests/publish` with a body that includes `"horizon": "MONTHLY"` (not a valid `ForecastHorizon` value).
2. Observe the resulting request card.

**Expected:**
- The request card appears in `FAILED` state.
- `validation.valid` is `false` and `validation.violationCodes` contains `horizon.invalid`.
- No agent task is present in the service log for this `requestId`.
