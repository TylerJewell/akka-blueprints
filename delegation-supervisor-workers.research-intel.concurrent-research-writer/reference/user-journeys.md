# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1-J5 pass.

## J1 -- Submit a topic and watch parallel assembly

**Preconditions:** service running on `http://localhost:9432/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a topic and Submit.
2. Observe the new report row via SSE.

**Expected:** the report progresses `PLANNING -> IN_PROGRESS -> ASSEMBLED` within ~60 s. The expanded row shows a `SourceBundle` (3-5 sources), a `DraftSection` (3-5 key points), and an assembled report body. Sources and draft arrive close together because the workers ran in parallel.

## J2 -- Worker timeout degrades the report

**Preconditions:** `SourceResearcher` step timeout set to 1 s (test override).

**Steps:**
1. Submit a topic.
2. Watch the report.

**Expected:** the `researchStep` times out, the workflow routes to `degradeStep`, and the Coordinator assembles from the DraftSection alone. The report enters `DEGRADED`; the body notes the missing side. No infinite retry.

## J3 -- Guardrail blocks a flawed report

**Preconditions:** Coordinator returns a report with a fabricated reference (test fixture).

**Steps:**
1. Submit the fixture topic.
2. Watch the report.

**Expected:** `guardrailStep` flags the assembled content; the workflow calls `block`; the report enters `BLOCKED` with a `failureReason`. The report is never surfaced as a finished result in the App UI list's done state.

## J4 -- Quality score appears beside an assembled report

**Preconditions:** at least one `ASSEMBLED` report without a `qualityScore`.

**Steps:**
1. Wait for `QualitySampler` to run (every 5 minutes), or trigger it.
2. Refresh the report row.

**Expected:** the report gains a `qualityScore` (1-5) and a `qualityRationale`; the App UI row shows the score. Delivery was never blocked by the eval (non-blocking).

## J5 -- Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `TopicSimulator` drips a topic from `writing-topics.jsonl` every 60 s; each becomes a report that flows through the full pipeline. The App UI is non-empty on first load.
