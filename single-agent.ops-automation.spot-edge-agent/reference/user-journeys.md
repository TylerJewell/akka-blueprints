# User journeys — spot-edge-agent

Six acceptance journeys. Each defines a precondition set, a sequence of steps, and an expected outcome. All journeys run against the local dev service started by `/akka:build`.

---

## J1 — Trivial patrol mission completes end-to-end

**Preconditions**
- Service is running.
- The 3-waypoint indoor patrol seed route is available (`wp-entrance`, `wp-server-room`, `wp-dock` — all `terrainType = flat`).
- No simulated faults configured (battery injection disabled or beyond the 3-waypoint traversal distance).

**Steps**
1. Open the App UI tab and select "Indoor Patrol" from the Route dropdown.
2. Accept the pre-filled mission name and payload instructions ("Capture thermal image at each waypoint").
3. Enter `ops-test-1` as the operator callsign and click **Submit mission**.
4. Observe the mission card appear in `SUBMITTED` → `PLANNING` within 1 s.
5. Confirm that the mission skips the approval gate (all commands are `TRIVIAL`) and transitions to `EXECUTING`.
6. Observe the waypoint list progress: each waypoint flips from pending to reached as the agent dispatches `spot.navigate_to` and `spot.capture_image` calls.
7. Wait up to 30 s for the mission to reach `COMPLETED`.

**Expected**
- Final status: `COMPLETED`.
- `MissionOutcome.status = COMPLETED`.
- `waypointsReached` contains all three waypoint IDs.
- `anomaliesObserved` is empty.
- No approval panel was displayed.

---

## J2 — Motion-command guardrail blocks a terrain-incompatible command

**Preconditions**
- Service is running.
- The mock LLM's guardrail-triggering entry fires on the first iteration of a 4th mission (modulo seed). Submit three trivial missions first, then trigger J2 on the fourth.
- Alternatively: mock LLM is configured to inject a stair-gait command on a flat-terrain route.

**Steps**
1. Submit the Indoor Patrol seed route (all flat terrain).
2. The mock LLM proposes `spot.change_gait(gait="stair")` as its first planning command.
3. `CommandGuardrail` fires and rejects the command with reason "stair gait incompatible with flat terrain".
4. The agent receives the rejection, removes the stair-gait command from its plan, and proposes `spot.navigate_to` with `walk` gait instead.
5. The replanned command passes the guardrail.
6. The mission proceeds to `EXECUTING` and completes.

**Expected**
- The command log (visible in the JSON response at `GET /api/missions/{id}`) contains no `spot.change_gait(gait=stair)` call in the final executed sequence.
- The mission status reaches `COMPLETED`.
- The guardrail rejection is visible in the agent's iteration history (accessible via the Akka local console at port 9889) — the first iteration was rejected; the second passed.

---

## J3 — Automatic safety halt fires on low battery

**Preconditions**
- Service is running.
- `simulation.battery-inject-after-calls = 6` in application.conf (default). This means the 7th `SimulatedSpotMcp` call returns a battery reading of 10 %.
- Submit the 5-waypoint outdoor inspection seed route, which requires more than 6 MCP tool calls.

**Steps**
1. Submit the Outdoor Inspection seed route.
2. Approve the non-trivial stair-climb maneuver when the approval panel appears (J3 assumes the approval gate is traversed successfully).
3. The mission enters `EXECUTING`.
4. After 6 tool calls (approximately waypoint 3 of 5), `TelemetryMonitor` reads `batteryPct = 10.1`.
5. `TelemetryMonitor` calls `MissionEntity.triggerSafetyHalt` and dispatches `SimulatedSpotMcp.halt()`.
6. The mission transitions to `HALTED`.

**Expected**
- Final status: `HALTED`.
- The halt reason on the card shows `cause = "low-battery"` and a sensor snapshot with `batteryPct ≤ 15`.
- The battery telemetry tile turns red in the UI.
- `MissionOutcome` is absent (the mission did not complete).
- `waypointsReached` in the halt detail shows only the waypoints reached before halt.

---

## J4 — Operator rejects a non-trivial maneuver

**Preconditions**
- Service is running.
- The Payload Drop seed route includes a tipping maneuver classified as `NON_TRIVIAL`.

**Steps**
1. Submit the Payload Drop seed route.
2. The mission enters `PENDING_APPROVAL`. The approval panel shows the plain-English summary: "The mission includes a payload tipping maneuver..."
3. The operator clicks **Reject** in the approval panel.
4. `POST /api/missions/{id}/reject` is called.
5. The mission transitions to `FAILED`.

**Expected**
- Final status: `FAILED`.
- `approvalDecision = "REJECTED"` visible on the mission card detail.
- No `MissionStarted` event was emitted (the robot never moved).
- The approve button is no longer displayed after rejection.

---

## J5 — Approval gate times out and auto-rejects

**Preconditions**
- Service is running.
- The Payload Drop seed route triggers `PENDING_APPROVAL`.
- The `approvalGateStep` timeout is set to 60 s (override `WorkflowSettings.stepTimeout` in test mode to shorten the default 10-minute timeout).

**Steps**
1. Submit the Payload Drop seed route.
2. The mission enters `PENDING_APPROVAL`.
3. Do not click Approve or Reject.
4. Wait for the `approvalGateStep` timeout to fire (60 s in test mode).

**Expected**
- The workflow fires the timeout branch and calls `decideApproval("REJECTED")`.
- Final status: `FAILED`.
- `approvalDecision = "REJECTED"` visible on the card.
- No `MissionStarted` event was emitted.

---

## J6 — Partial mission with an obstacle anomaly

**Preconditions**
- Service is running.
- The mock LLM is configured to return `status = PARTIAL` with an `anomaliesObserved` entry for waypoint 4 of 5 ("unexpected-obstacle at wp-loading-bay").

**Steps**
1. Submit the Outdoor Inspection seed route.
2. Approve the non-trivial maneuver when the approval panel appears.
3. The mission enters `EXECUTING`.
4. The agent navigates to waypoints 1–3 successfully. At waypoint 4, `spot.navigate_to` returns an obstacle-detected signal.
5. The agent records the anomaly, skips waypoint 4, and completes waypoint 5.
6. The agent returns `MissionOutcome(status=PARTIAL, waypointsReached=["wp1","wp2","wp3","wp5"], anomaliesObserved=["unexpected-obstacle at wp-loading-bay"])`.

**Expected**
- Final status: `COMPLETED` (the entity uses MissionCompleted for all non-halted terminal states; the partial outcome is encoded in `MissionOutcome.status`).
- Outcome badge in the UI shows `PARTIAL` (from `MissionOutcome.status`).
- `waypointsReached` has 4 of 5 waypoints.
- `anomaliesObserved` lists the obstacle event.
