# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a startup brief and watch parallel GTM synthesis

**Preconditions:** service running on `http://localhost:9603/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a startup brief description and Submit.
2. Observe the new session row via SSE.

**Expected:** the session progresses `SCOPING → IN_PROGRESS → ADVISED` within ~60 s. The expanded row shows a `MarketSnapshot` (TAM, 3–5 competitors, 2–4 buyer segments), a `ChannelPlan` (3–5 channel recommendations with effort ratings and a designated primary channel), and a GTM guidance summary. The market snapshot and channel plan arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the session

**Preconditions:** `MarketResearcher` step timeout set to 1 s (test override).

**Steps:**
1. Submit a startup brief.
2. Watch the session.

**Expected:** the `researchStep` times out, the workflow routes to `degradeStep`, and the AdvisorCoordinator synthesises from the GrowthAnalyst output alone. The session enters `DEGRADED`; the guidance summary notes the missing market snapshot in one sentence. No infinite retry.

## J3 — MCP tool allow-list blocks a disallowed tool call

**Preconditions:** a test fixture configured so that `MarketResearcher` attempts to call a tool not on the allow-list.

**Steps:**
1. Submit the fixture brief.
2. Watch the session.

**Expected:** the before-tool-call guardrail fires before the disallowed tool executes; the call is rejected with a structured error; the session records the `failureReason`; the workflow routes to `degradeStep` and the session enters `DEGRADED`. No tool side-effects occur.

## J4 — Eval score appears beside an advised session

**Preconditions:** at least one `ADVISED` session without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it directly.
2. Observe the session row.

**Expected:** the session gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score alongside the status pill. Guidance delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `BriefSimulator` drips a startup brief from `startup-briefs.jsonl` every 60 s; each becomes a session that flows through the full pipeline. The App UI is non-empty on first load.
