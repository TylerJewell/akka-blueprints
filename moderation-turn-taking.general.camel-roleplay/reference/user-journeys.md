# User journeys

The acceptance bar. The generated system is correct when these pass. Journeys J1–J4 are the inline acceptance set named in SPEC.md Section 10; J5 and J6 cover the simulator and the operator halt.

## J1 — Start a collaboration and see the first turn

**Preconditions:** the service is running on `http://localhost:9100`; a model provider is wired (or the mock provider is active).

**Steps:**
1. Open the App UI tab.
2. Enter task description `Write a Python function that returns the nth Fibonacci number`, assistant role `Python developer`, user role `software engineer giving requirements`. Click Start.

**Expected:**
- The response carries a `collaborationId`.
- Within a few seconds the collaboration appears in the list in `COLLABORATING` status with `currentRound` 1.
- The turn history holds at least the User's round-1 `TurnLine` with a non-empty `message` and `intent`.
- `GET /api/collaborations/{id}` returns the same collaboration.

## J2 — A solvable task reaches SOLVED with a complete answer

**Preconditions:** the task is well-posed and achievable within the assigned roles.

**Steps:**
1. Start a collaboration with task `Explain the difference between a stack and a queue with a code example`, assistant role `computer science tutor`, user role `student asking for clarification`.
2. Watch the SSE-fed list.

**Expected:**
- Across successive rounds the Assistant produces an explanation and the User requests refinements; the explanation grows toward completeness.
- The collaboration reaches `CONCLUDED` with `outcome = SOLVED` at or before round 12.
- `solutionSummary` is non-empty and summarises what was produced.
- `finalAnswer` is non-empty and contains the complete explanation extracted from the Assistant's turn.
- No `TurnLine` for the User agent contains a direct answer to the task (the output guardrail held the role boundary).

## J3 — An unsolvable task reaches IMPASSE

**Preconditions:** the task contains conflicting constraints with no satisfying solution.

**Steps:**
1. Start a collaboration with task `Design a sorting algorithm that is both O(1) in time and O(1) in space for arbitrary inputs`, assistant role `algorithms expert`, user role `researcher specifying requirements`.
2. Watch the list.

**Expected:**
- The collaboration reaches `CONCLUDED` with `outcome = IMPASSE` no later than round 12.
- `solutionSummary` is empty or null and `finalAnswer` is empty or null.
- The Coordinator declares impasse once the impossibility of the constraints is evident, rather than running all twelve rounds.

## J4 — Every concluded collaboration is scored

**Preconditions:** at least one collaboration from J2 or J3 has reached `CONCLUDED`.

**Steps:**
1. Inspect a concluded collaboration in the list (or `GET /api/collaborations/{id}`).

**Expected:**
- Within a few seconds of concluding, `outcomeScore` is non-null and `outcomeNotes` is non-empty.
- For a `SOLVED` collaboration, the score reflects solution presence, completeness indicators, and rounds used.
- For an `IMPASSE`, the score reflects that no solution was produced.

## J5 — Background load from the simulator

**Preconditions:** the service has been running for at least one minute with no user input.

**Steps:**
1. Leave the App UI tab open without starting anything.

**Expected:**
- `TaskSimulator` drips a canned task from `task-requests.jsonl` every thirty seconds.
- New collaborations appear and conclude on their own; the canned set includes both solvable and unsolvable tasks, so both outcomes show up over time.

## J6 — Operator halt pauses new collaborations

**Preconditions:** the service is running.

**Steps:**
1. Click Halt in the operator control (or `POST /api/system/halt`).
2. Submit a new collaboration (or wait for the simulator's next tick).
3. Click Resume (or `POST /api/system/resume`).

**Expected:**
- While halted, `GET /api/system/status` returns `{ "halted": true }`, the Start button is disabled, and no new collaboration starts — queued tasks are skipped by `TaskRequestConsumer`.
- Any collaboration already in flight keeps running to its conclusion; halt gates starts only.
- After Resume, new collaborations start again on the next request or simulator tick.
