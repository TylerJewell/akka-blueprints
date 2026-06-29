# User journeys — smart-home-agent

## J1 — Non-sensitive command dispatches without confirmation

**Preconditions:** Service running on `http://localhost:9661/`; a valid model-provider API key set, or the mock LLM selected at scaffold time. Device registry seeded with at least `kitchen-light` (DeviceClass: LIGHT).

**Steps:**
1. Open `http://localhost:9661/` → App UI tab.
2. In the **Your command** input, type `Turn on the kitchen lights`.
3. Click **Send command**.

**Expected:**
- The new command card appears in the live list with status `RECEIVED` within 1 s.
- Within 30 s the card reaches `GUARD_PASSED`, then transitions immediately to `DISPATCHED` (no confirmation step for non-sensitive actions).
- The right pane shows: guard result "PASSED", proposed action `TURN_ON` on `kitchen-light`, rationale sentence.
- The device grid updates: `kitchen-light` shows the `on` badge.
- No confirmation panel appears at any point.

## J2 — Sensitive command pauses for resident confirmation

**Preconditions:** Service running. Device registry seeded with `front-door-lock` (DeviceClass: LOCK).

**Steps:**
1. In the **Your command** input, type `Lock the front door`.
2. Click **Send command**.
3. Wait for the status card to reach `AWAITING_CONFIRMATION`.
4. In the right pane's confirmation panel, click **Confirm**.

**Expected:**
- The card reaches `AWAITING_CONFIRMATION` within 30 s and displays the confirmation panel with the proposed LOCK action and `Confirm` / `Cancel` buttons.
- After clicking **Confirm**, the card transitions to `DISPATCHED` within 2 s.
- The device grid updates: `front-door-lock` shows the locked icon.
- A second scenario: repeat steps 1–3, then click **Cancel**. The card transitions to `CANCELLED` and the device state does not change.

## J3 — Guardrail blocks out-of-range thermostat command

**Preconditions:** Service running. Device registry seeded with `thermostat-main` (DeviceClass: THERMOSTAT, minValue: 60, maxValue: 80).

**Steps:**
1. In the **Your command** input, type `Set the thermostat to 95 degrees`.
2. Click **Send command**.

**Expected:**
- The card reaches `BLOCKED` within 30 s.
- The right pane shows: guard result "BLOCKED" badge, rejection reason `parameter-out-of-range: numericParam=95 is outside [60, 80] for device thermostat-main`.
- No confirmation panel appears; no `DISPATCHED` event is emitted; `DeviceRegistry` state for `thermostat-main` is unchanged.
- The service log shows one `guardrail.reject` line per rejected iteration naming `parameter-out-of-range`.

## J4 — Guardrail blocks command for unknown device

**Preconditions:** Service running. The device registry does NOT contain a garage door device.

**Steps:**
1. In the **Your command** input, type `Open the garage door`.
2. Click **Send command**.

**Expected:**
- The card reaches `BLOCKED` within 30 s.
- The rejection reason reads `device-not-found: deviceId=garage-door (or similar) is not in the device registry`.
- `DeviceRegistry` state is unchanged.

## J5 — Confirmation timeout cancels the workflow

**Preconditions:** Service running with `confirmStep` stepTimeout set to a low value for this test (or wait 5 minutes if using the default 300 s). Device registry seeded with `front-door-lock`.

**Steps:**
1. Submit `Lock the front door`.
2. Wait for the card to reach `AWAITING_CONFIRMATION`.
3. Do NOT click Confirm or Cancel. Wait for the step timeout to elapse.

**Expected:**
- After the timeout elapses, the card transitions to `FAILED` with `reason = "confirmation-timeout"`.
- The device state for `front-door-lock` is unchanged.
- The service log shows a `WorkflowStepTimeout` event on the `confirmStep`.

## J6 — Brightness dim dispatches with correct numeric param

**Preconditions:** Service running. Device registry seeded with `living-room-light` (DeviceClass: LIGHT, minValue: 0, maxValue: 100).

**Steps:**
1. From the **Seeded commands** dropdown, pick `Dim the living room lights to 40%`.
2. Click **Send command**.

**Expected:**
- The card dispatches without confirmation (brightness is not in the sensitive action set).
- The dispatched action shows `SET_BRIGHTNESS` on `living-room-light` with `numericParam = 40`.
- The device grid updates: `living-room-light` shows brightness `40%`.
- The guard result shows `PASSED` with no rejection reason.
