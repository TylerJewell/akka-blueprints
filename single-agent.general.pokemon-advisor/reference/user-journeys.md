# User journeys — pokemon-advisor

## J1 — Submit a VGC roster and get a recommendation

**Preconditions:** Service running on declared port (`http://localhost:9830/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9830/` → App UI tab.
2. From the **Battle format** dropdown, pick `VGC`.
3. Click **Load seeded example** to fill the trainer name and all six species slots with the seeded VGC roster.
4. Click **Get advice**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `VALIDATED` within 1 s. The right-pane detail shows the submitted roster table with no validation warnings.
- Within 30 s the card reaches `RECOMMENDATION_RECORDED`. The right pane shows: a verdict badge (STRONG / VIABLE / NEEDS_ADJUSTMENT), the summary paragraph, and one slot-recommendation row for each of the 6 submitted slots. Every slot row has a non-empty role chip, at least one suggested move, and a one-sentence rationale.
- Within 1 s of `RECOMMENDATION_RECORDED`, the card reaches `SCORED` and shows a coverage score chip (1–5) plus a one-line rationale.

## J2 — Guardrail blocks a malformed recommendation

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `advise-team.json` includes deliberately malformed entries (a slot whose role is outside the enum; a missing slots array).

**Steps:**
1. Submit any seeded roster three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the network panel of the browser dev tools (`/api/advisories/sse`).

**Expected:**
- The third submission's first agent iteration produces a malformed recommendation.
- The `before-agent-response` guardrail rejects it. The malformed recommendation NEVER lands in `RosterEntity` — there is no `RecommendationRecorded` event with the malformed payload.
- The agent loop retries on iteration 2 (and 3 if needed) and produces a well-formed recommendation. The card transitions to `RECOMMENDATION_RECORDED` with a recommendation that satisfies all five guardrail checks.
- The service log shows one `guardrail.reject` line per rejected iteration with the structured-error code naming which check failed.

## J3 — Low-coverage team flags coverage score 1

**Preconditions:** Mock LLM mode. The mock response entry for the casual 4-slot roster produces a recommendation whose `gaps` list is empty and whose `suggestedMoves` across all slots cover only two damage types.

**Steps:**
1. In the App UI, pick format `Casual` and load the seeded casual roster (Charizard, Blastoise, Venusaur, Pikachu).
2. Adjust the instruction set or trigger the specific mock entry that returns narrow coverage.
3. Submit the roster.

**Expected:**
- The recommendation lands well-formed (the guardrail only checks structural validity, not coverage breadth).
- The coverage score chip shows **1** and the rationale reads something like "Team's suggested moves cover only Fire and Water; four primary types are unaddressed."
- The card's border highlights red. The trainer knows to re-evaluate the roster.

## J4 — Duplicate species triggers validation failure

**Preconditions:** Service running. Any model provider (the validator runs before the agent).

**Steps:**
1. In the App UI, manually enter two slots with the species `Charizard` (e.g., slot 1 and slot 3).
2. Enter any format and trainer name.
3. Click **Get advice**.

**Expected:**
- `RosterValidator` detects the duplicate species and calls `RosterEntity.fail("DUPLICATE_SPECIES")`.
- No `RosterValidated` event is emitted; no `AdvisoryWorkflow` is started; `TeamAdvisorAgent` is never called.
- The card reaches `FAILED` state within 1 s with a reason string of `DUPLICATE_SPECIES` visible in the right-pane detail.
- The submission panel remains enabled so the trainer can correct the roster and resubmit.

## J5 — Slot count outside format limits triggers validation failure

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the App UI, pick format `VGC`.
2. Enter only 2 species slots (VGC requires 4–6 slots).
3. Click **Get advice**.

**Expected:**
- `RosterValidator` detects the slot count is below the VGC minimum and calls `RosterEntity.fail("INVALID_SLOT_COUNT:VGC_REQUIRES_4_TO_6")`.
- The card reaches `FAILED` state with the reason visible in the detail pane.
- The agent is never invoked.
