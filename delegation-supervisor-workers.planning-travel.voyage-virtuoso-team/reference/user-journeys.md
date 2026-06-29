# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a travel request and watch parallel specialist execution

**Preconditions:** service running on `http://localhost:9798/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Fill in destination, origin, departure date, return date, traveller count, and tier. Submit.
2. Observe the new request row via SSE.

**Expected:** the request progresses `PLANNING → IN_PROGRESS → ASSEMBLED` within ~90 s. The expanded row shows a `FlightPlan` (outbound + return routing, fare class), a `LodgingPlan` (property name and room type), an `ExperiencePlan` (3–5 activities, 2–4 dining items), a `LogisticsPlan` (transfers and entry requirements), and an itinerary synopsis. The four pillar plans arrive close together because the specialists ran in parallel.

## J2 — Specialist timeout produces a PARTIAL itinerary

**Preconditions:** `FlightSpecialist` step timeout set to 1 s (test override).

**Steps:**
1. Submit a travel request.
2. Watch the request row.

**Expected:** the `flightStep` times out, the workflow routes to `partialStep`, and the ItineraryDirector assembles from the three remaining specialist outputs. The request enters `PARTIAL`; the synopsis notes the missing flight pillar in one sentence. No infinite retry; the other three plans are visible in the expanded row.

## J3 — Guardrail blocks a fabricated itinerary

**Preconditions:** ItineraryDirector is configured to return a `guardrailVerdict` other than `"ok"` (test fixture — e.g., a mocked plan containing an invalid IATA code).

**Steps:**
1. Submit the fixture travel request.
2. Watch the request row.

**Expected:** `guardrailStep` detects the fabricated datum; the workflow calls `block`; the request enters `BLOCKED` with a `failureReason` naming which pillar triggered the rejection. The request row shows the BLOCKED status pill. The itinerary is never surfaced as an ASSEMBLED result.

## J4 — Eval score appears beside an assembled itinerary

**Preconditions:** at least one `ASSEMBLED` request without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the request row.

**Expected:** the request gains an `evalScore` (1–5) and an `evalRationale`; the App UI row displays the score badge. Delivery of the itinerary was never blocked by the eval (non-blocking enforcement).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction; service freshly started.

**Steps:**
1. Leave the service running for 60 seconds.

**Expected:** `RequestSimulator` drips one travel scenario from `travel-scenarios.jsonl` every 60 s; each becomes a request that flows through the full pipeline. The App UI is non-empty on first load without any user submission.

## J6 — Multi-traveller premium request assembles all four pillars

**Preconditions:** service running; model provider configured.

**Steps:**
1. Submit a request with `travellers: 4`, `tier: "first"`, and a long-haul destination.
2. Expand the assembled row.

**Expected:** `FlightPlan.cabinClass` is `"first"`, `LodgingPlan.propertyCategory` is `"urban luxury hotel"` or `"resort"`, `ExperiencePlan.activities` includes at least one private or exclusive-access item, and `LogisticsPlan.groundTransfers` includes a private chauffeur option. The synopsis reads as a sequenced journey.
