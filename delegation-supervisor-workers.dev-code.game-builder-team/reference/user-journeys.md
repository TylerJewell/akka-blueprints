# User journeys — game-builder-team

Acceptance journeys. Passing all of these means the blueprint generated correctly.

## J1 — Submit a game idea and watch it deliver
**Preconditions:** service running on `http://localhost:9489/`; a model provider key set or the mock provider selected.
**Steps:**
1. Open the App UI tab; type "a snake game where the snake speeds up over time"; click Build.
2. Observe the new project appear via SSE in `QUEUED`, then `DESIGNING`, `CODING`, `TESTING`.
**Expected:** within ~60 s the project reaches `DELIVERED` with a non-empty `spec` and `code`. The Play button loads the game into a sandboxed iframe and it renders.

## J2 — Sandbox guardrail blocks unsafe code
**Preconditions:** a build whose generated `code.html` references a forbidden API (network, filesystem, `eval`, dynamic import). With the mock provider, use the seeded "unsafe" mock entry.
**Steps:**
1. Submit the idea that maps to the unsafe mock (or let the simulator seed it).
2. The workflow reaches `testStep`; the before-tool-call guardrail screens the code.
**Expected:** the project moves to `BLOCKED` with a populated `blockReason`; no run tool executes; the App UI shows the reason.

## J3 — Test gate fails and the build settles in TEST_FAILED
**Preconditions:** a build whose code fails the deterministic smoke checks (missing entry point / no canvas).
**Steps:**
1. Submit the idea that maps to the failing mock.
2. `testStep` runs the smoke checks, fails, retries `codeStep` up to twice.
**Expected:** after the retry budget is spent the project settles in `TEST_FAILED` with `testReport.failures` populated; `testAttempts` reflects the retries.

## J4 — Simulator seeds a build with no user interaction
**Preconditions:** service running; App UI not touched.
**Steps:**
1. Wait up to 60 s.
**Expected:** `RequestSimulator` enqueues an idea from `game-ideas.jsonl`, `BuildRequestConsumer` starts a workflow, and a new project appears in the SSE list.

## J5 — Metadata tabs render
**Preconditions:** service running.
**Steps:**
1. Open the Eval Matrix tab, then the Risk Survey tab.
**Expected:** the Eval Matrix tab shows two controls (G1, C1) in `matrix-card`/`matrix-row` style with mechanism pills; the Risk Survey tab shows the pre-filled answers with deployer placeholders muted.
