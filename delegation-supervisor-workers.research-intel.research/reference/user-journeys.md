# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J4 pass.

## J1 — Submit a topic and watch parallel synthesis

**Preconditions:** service running on `http://localhost:9762/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a topic and Submit.
2. Observe the new brief row via SSE.

**Expected:** the brief progresses `PLANNING → IN_PROGRESS → SYNTHESISED` within ~60 s. The expanded row shows a `FindingsBundle` (3–6 findings), an `AnalyticalReport` (3–6 implications), and a synthesised summary. Findings and analysis arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the brief

**Preconditions:** `Researcher` step timeout set to 1 s (test override).

**Steps:**
1. Submit a topic.
2. Watch the brief.

**Expected:** the `researchStep` times out, the workflow routes to `degradeStep`, and the Coordinator synthesises from the Analyst output alone. The brief enters `DEGRADED`; the summary notes the missing side. No infinite retry.

## J3 — Guardrail blocks a flawed brief

**Preconditions:** Coordinator returns a brief with a fabricated citation (test fixture).

**Steps:**
1. Submit the fixture topic.
2. Watch the brief.

**Expected:** `guardrailStep` flags the synthesised content; the workflow calls `block`; the brief enters `BLOCKED` with a `failureReason`. The brief is never surfaced as a finished result in the App UI list's done state.

## J4 — Eval score appears beside a synthesised brief

**Preconditions:** at least one `SYNTHESISED` brief without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it.
2. Refresh the brief row.

**Expected:** the brief gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score. Delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `RequestSimulator` drips a topic from `research-topics.jsonl` every 60 s; each becomes a brief that flows through the full pipeline. The App UI is non-empty on first load.
