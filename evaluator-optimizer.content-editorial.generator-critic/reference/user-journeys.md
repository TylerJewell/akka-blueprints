# User journeys — generator-critic

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Convergence on or before the ceiling

**Preconditions:** Service running on port 9732; valid model-provider API key set, OR mock mode active.

**Steps:**
1. Open `http://localhost:9732/`. App UI tab is visible.
2. In the Topic field, type "the long-term effects of remote work on urban planning". Leave the word ceiling at the default 400. Click Submit.
3. A new essay card appears with status `DRAFTING`.

**Expected:**
- Within 1 s, status transitions to `REFLECTING` and the first round's draft appears.
- Within 60 s of submission, either:
  - Round 1's reflection is `ACCEPT` and the essay transitions to `ACCEPTED`, then `RELEASED`, OR
  - Round 1's reflection is `REVISE` with 1–3 bullets and round 2 appears shortly after; this continues until either an `ACCEPT`→`RELEASED` or the round ceiling.
- On `RELEASED`, the terminal block shows the released text and "released after N rounds."
- The expanded view shows every round's draft, reflector verdict, score, notes, and the final guardrail result.

## J2 — Halt at round ceiling

**Preconditions:** As J1, plus an override that forces the Reflector to always return `REVISE` (test mode — submit the literal topic `"test-force-reject"`, which the mock provider's seedFor logic always answers with `REVISE`).

**Steps:**
1. Submit the topic `"test-force-reject"` with the default word ceiling.

**Expected:**
- Essay progresses `DRAFTING` → `REFLECTING` → `DRAFTING` → `REFLECTING` → … for `maxRounds` cycles (default 4).
- After the 4th round ends in `REVISE`, the essay transitions to `REJECTED_FINAL` (not stuck in `REFLECTING`).
- The terminal block shows the highest-scoring round's text as the "best of 4 rounds" and the `rejectionReason` reads `"max rounds reached (4)"`.
- All 4 rounds are present in the expanded view, each with its draft, `REVISE` verdict, and score.
- `GET /api/essays/{id}` returns the full Essay with all 4 rounds in `rounds[]` and `status: "REJECTED_FINAL"`.

## J3 — Guardrail block on release

**Preconditions:** As J1, with at least one phrase added to `basic-reflection.policy.forbidden-phrases` (e.g., `"FORBIDDEN_PHRASE_TEST"`).

**Steps:**
1. Submit a topic that causes the Generator to produce a draft containing the configured forbidden phrase (in mock mode, the intentionally policy-violating entry in `generator.json` is selected via `seedFor` for this topic).

**Expected:**
- The reflector eventually returns `ACCEPT` for a round.
- The guardrail step runs on the accepted draft and detects the forbidden phrase.
- `RoundGuardrailVerdictRecorded` is emitted with `passed = false` and `reasonCode = "POLICY_VIOLATION"`.
- The Generator is called with `POLICY_REVISE` mode and a feedback string naming the offending phrase.
- The revised draft passes the guardrail and the essay transitions to `RELEASED`.
- The expanded view shows the round where the guardrail fired with a red `POLICY_VIOLATION` pill and the detail text, followed by a green `OK` pill on the revised draft.
- `GET /api/essays/{id}` includes the guardrail violation and the final clean release.

## J4 — Eval-event timeline

**Preconditions:** At least one essay has completed (any terminal state).

**Steps:**
1. Click the essay card to expand.

**Expected:**
- The timeline shows one `ReflectionRecorded` event per reflected round, with `verdict`, `score`, and `guardrailPassed` populated.
- The terminal transition (`EssayReleased` or `EssayRejectedFinal`) is also surfaced as a final `ReflectionRecorded` event carrying the loop-level outcome.
- `GET /api/essays/{id}` includes an `evalEvents[]` array (or equivalent surfaced through the SSE update) with the same content. The UI does not require a separate fetch to render the timeline.
