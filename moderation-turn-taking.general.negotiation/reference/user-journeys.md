# User journeys

The acceptance bar. The generated system is correct when these pass. Journeys J1–J4 are the inline acceptance set named in SPEC.md Section 10; J5 and J6 cover the simulator and the operator halt.

## J1 — Start a negotiation and see the first offer

**Preconditions:** the service is running on `http://localhost:9375`; a model provider is wired (or the mock provider is active).

**Steps:**
1. Open the App UI tab.
2. Enter item `vintage road bike`, buyer budget `500`, seller floor `300`. Click Start.

**Expected:**
- The response carries a `negotiationId`.
- Within a few seconds the negotiation appears in the list in `NEGOTIATING` status with `currentRound` 1.
- The offer history holds at least the Buyer's round-1 `OfferLine`, with `price ≤ 500`.
- `GET /api/negotiations/{id}` returns the same negotiation.

## J2 — Convergence to a deal

**Preconditions:** budget comfortably above floor (overlap exists).

**Steps:**
1. Start a negotiation with item `office chair`, buyer budget `450`, seller floor `300`.
2. Watch the SSE-fed list.

**Expected:**
- Across successive rounds the Buyer's price rises and the Seller's price falls; the gap narrows.
- The negotiation reaches `CONCLUDED` with `outcome = CONVERGED` at or before round 10.
- `finalPrice` is set and sits at or above `300` and at or below `450`.
- `finalTerms` is non-empty.
- No `OfferLine` ever has a Buyer price above `450` or a Seller price below `300` (the output guardrail held the bands).

## J3 — No deal when the bands do not overlap

**Preconditions:** budget below floor (no overlap).

**Steps:**
1. Start a negotiation with item `rare watch`, buyer budget `200`, seller floor `350`.
2. Watch the list.

**Expected:**
- The negotiation reaches `CONCLUDED` with `outcome = NO_DEAL` no later than round 10.
- `finalPrice` is not set (`null`) and `finalTerms` is empty or `null`.
- The Facilitator concludes early once the lack of overlap is clear, rather than burning all ten rounds.

## J4 — Every concluded negotiation is scored

**Preconditions:** at least one negotiation from J2 or J3 has reached `CONCLUDED`.

**Steps:**
1. Inspect a concluded negotiation in the list (or `GET /api/negotiations/{id}`).

**Expected:**
- Within a few seconds of concluding, `outcomeScore` is non-null and `outcomeNotes` is non-empty.
- For a `CONVERGED` deal, the score reflects the surplus split (where `finalPrice` sits between floor and budget) and the rounds used.
- For a `NO_DEAL`, the score reflects that no deal was reached.

## J5 — Background load from the simulator

**Preconditions:** the service has been running for at least one minute with no user input.

**Steps:**
1. Leave the App UI tab open without starting anything.

**Expected:**
- `RequestSimulator` drips a canned scenario from `negotiation-requests.jsonl` every thirty seconds.
- New negotiations appear and conclude on their own; the canned set includes both deal and no-deal cases, so both outcomes show up over time.

## J6 — Operator halt pauses new negotiations

**Preconditions:** the service is running.

**Steps:**
1. Click Halt in the operator control (or `POST /api/system/halt`).
2. Submit a new negotiation (or wait for the simulator's next tick).
3. Click Resume (or `POST /api/system/resume`).

**Expected:**
- While halted, `GET /api/system/status` returns `{ "halted": true }`, the Start button is disabled, and no new negotiation starts — queued requests are skipped by `NegotiationRequestConsumer`.
- Any negotiation already in flight keeps running to its conclusion; halt gates starts only.
- After Resume, new negotiations start again on the next request or simulator tick.
