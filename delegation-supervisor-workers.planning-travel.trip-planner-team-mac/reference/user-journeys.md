# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J6 pass.

## J1 — Submit a trip request and approve the plan

**Preconditions:** service running on `http://localhost:9621/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Fill in destination, dates, traveller count, and budget. Click Submit.
2. Observe the new trip row via SSE.
3. Wait for the trip to reach `AWAITING_APPROVAL` (typically within 90 s). The row expands to show destination notes, a day-by-day itinerary, and booking proposals.
4. Click Approve.

**Expected:** the trip progresses `PLANNING → IN_PROGRESS → AWAITING_APPROVAL` then `CONFIRMED` in that order. The expanded row after confirmation shows the compiled plan. No external booking was called — proposals are modelled in-process. All three worker outputs (destination notes, itinerary, booking proposals) appear in the expanded row, confirming the parallel fan-out completed before compilation.

## J2 — Worker timeout degrades the trip

**Preconditions:** `ItineraryPlanner` step timeout set to 1 s (test override).

**Steps:**
1. Submit a trip request.
2. Watch the trip row.

**Expected:** the `planStep` times out, the workflow routes to `degradeStep`, and TripSupervisor compiles from the DestinationResearcher and BookingAgent outputs alone. The trip enters `DEGRADED`; the compiled plan's `compilationNotes` mentions the missing itinerary. No infinite retry.

## J3 — Budget guardrail blocks an over-budget plan

**Preconditions:** submit a trip with a very low budget (e.g., `budgetUsd: 50`) for a multi-day international destination. The BookingAgent will produce realistic proposals that exceed it.

**Steps:**
1. Submit the low-budget request.
2. Watch the trip row.

**Expected:** `guardrailStep` fires — at least one booking proposal exceeds the budget ceiling — the workflow calls `TripEntity.block`, and the trip enters `BLOCKED` with a `failureReason`. No `ApprovalEntity` token is issued. The trip never reaches `AWAITING_APPROVAL`.

## J4 — User rejects the draft

**Preconditions:** at least one trip in `AWAITING_APPROVAL`.

**Steps:**
1. Click Reject on an `AWAITING_APPROVAL` row.
2. Observe the row.

**Expected:** the trip transitions to `REJECTED`. The row no longer shows Approve or Reject buttons. No booking payload is executed. The `finishedAt` timestamp is set.

## J5 — Eval score appears beside a confirmed trip

**Preconditions:** at least one `CONFIRMED` trip without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 10 minutes), or trigger it directly.
2. Observe the trip row.

**Expected:** the trip gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery was never blocked by the eval (non-blocking). The `CONFIRMED` status does not change.

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TripRequestSimulator` drips a request from `trip-requests.jsonl` every 90 s; each becomes a trip that flows through the full pipeline. The App UI is non-empty on first load. Simulated trips go through the guardrail and reach `AWAITING_APPROVAL`; without an explicit approve call they remain there until a user interacts.
