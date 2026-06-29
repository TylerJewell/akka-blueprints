# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a trip request and confirm the itinerary

**Preconditions:** service running on `http://localhost:9381/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Fill in destination, dates, budget, and traveller count. Submit.
2. Observe the new trip row via SSE.
3. Wait for status to reach `AWAITING_APPROVAL`.
4. Expand the row; click Confirm.

**Expected:** the trip progresses `PLANNING → RESEARCHING → AWAITING_APPROVAL` within ~60 s. The expanded row shows a `DestinationBrief` (3–5 highlights, visa requirement) and a `FareEstimate` (flight + accommodation in USD). After confirmation the trip moves to `CONFIRMED`. All transitions arrive via SSE without a page reload.

## J2 — Decline the itinerary at the approval step

**Preconditions:** at least one trip in `AWAITING_APPROVAL`.

**Steps:**
1. Expand the awaiting trip row. Click Decline.

**Expected:** the trip moves immediately to `DECLINED`. No booking tool call is issued. The App UI row's status pill updates via SSE. The `bookingReference` field remains null.

## J3 — Guardrail blocks an over-budget booking

**Preconditions:** submit a trip whose assembled fare exceeds the stated `budgetUsd` (set a low budget, e.g., $100, for a long international trip).

**Steps:**
1. Submit the trip.
2. Wait for `AWAITING_APPROVAL`.
3. Click Confirm.

**Expected:** `guardrailStep` detects the budget overage; the workflow calls `blockTrip`; the trip enters `BLOCKED` with a `failureReason`. No booking action fires. The App UI row shows BLOCKED status and the failure reason in the expanded view.

## J4 — Worker timeout degrades the trip

**Preconditions:** `DestinationScout` step timeout set to 1 s (test override).

**Steps:**
1. Submit a trip.
2. Watch the trip.

**Expected:** `scoutStep` times out; the workflow routes to `degradeStep`; `TripCoordinator` assembles from the `FareEstimate` alone. The trip enters `DEGRADED`; the itinerary's `warnings` field notes the missing destination brief. No infinite retry.

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TripRequestSimulator` drips a request from `trip-requests.jsonl` every 90 s; each becomes a trip that flows through the pipeline to `AWAITING_APPROVAL`. The App UI is non-empty on first load.
