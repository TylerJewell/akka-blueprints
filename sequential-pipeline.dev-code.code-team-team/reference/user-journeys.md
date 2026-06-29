# User journeys — Game Builder Team

Acceptance journeys. Each must pass against the generated system for the blueprint to be considered correctly generated.

## J1 — Submit a brief and watch it ship

**Preconditions:** service running on `http://localhost:9375/`; a model provider configured (or mock).
**Steps:**
1. Open the App UI tab, type a clean brief (e.g. "a number-guessing CLI game"), click Build.
2. Observe the SSE-driven list.
**Expected:** a new build appears in `QUEUED`, then `ENGINEERED` (with a non-empty `sourceCode` and a `.py` `filename`), then `QA` (with a `qaScore`), then `SHIPPED` with `shipped: true` and a `shipSummary`. End state: `SHIPPED`.

## J2 — A failing build is rejected, never shipped

**Preconditions:** service running; a brief the QA stage fails (or a mock entry returning `passed: false`).
**Steps:**
1. Submit the brief.
2. Watch the build through QA.
**Expected:** the build records `qaPassed: false`, the chief-review step calls reject, and the build lands in `REJECTED` with a `reworkReason`. `shipped` is never true. End state: `REJECTED`.

## J3 — Sandbox guardrail blocks unsafe code

**Preconditions:** service running; a candidate source containing a forbidden operation (network, filesystem escape, `os.system`, `subprocess`, or `eval`).
**Steps:**
1. The QA stage calls the `CodeRunner` tool on that source.
**Expected:** the before-tool-call guardrail rejects the call before any simulated execution; the QA report records `passed: false` with a note naming the blocked operation; the build does not reach `SHIPPED`.

## J4 — Background load from the simulator

**Preconditions:** service running; no UI interaction.
**Steps:**
1. Leave the App UI tab open.
**Expected:** every 30 seconds `BriefSimulator` enqueues the next canned brief from `game-briefs.jsonl`; `BriefConsumer` starts a fresh `GameBuildWorkflow`; new builds appear in the list and progress through the pipeline without any user action.

## J5 — Metadata tabs render

**Preconditions:** service running.
**Steps:**
1. Open the Eval Matrix tab, then the Risk Survey tab.
**Expected:** the Eval Matrix tab renders three controls (G1, E1, A1) in `matrix-card` / `matrix-row` style with colored mechanism pills; the Risk Survey tab renders the pre-filled answers, with `TO_BE_COMPLETED_BY_DEPLOYER` values muted.
