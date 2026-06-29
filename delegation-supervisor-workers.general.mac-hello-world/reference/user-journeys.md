# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J3 pass.

## J1 — Submit a greeting request and watch parallel composition

**Preconditions:** service running on `http://localhost:9954/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a name (e.g., "Yuki") and context (e.g., "accepting a job offer") and click Submit.
2. Observe the new request row via SSE.

**Expected:** the request progresses `PENDING → IN_PROGRESS → COMPLETED` within ~30 s. The expanded row shows a draft message from GreetingWriter (20–50 words), a follow-up action from ActionAdvisor (15–35 words), and a composed response that integrates both. The draft and follow-up arrive close together because both workers ran in parallel.

## J2 — Worker timeout degrades the request

**Preconditions:** `GreetingWriter` step timeout set to 1 s (test override).

**Steps:**
1. Submit a name and context.
2. Watch the request row.

**Expected:** the `writeStep` times out, the workflow routes to `degradeStep`, and the Coordinator composes from the ActionAdvisor output alone. The request enters `DEGRADED`; the composed message notes the missing draft. No infinite retry.

## J3 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running for at least 60 seconds.

**Expected:** `RequestSimulator` drips a request from `greeting-requests.jsonl` every 60 s; each becomes a greeting request that flows through the full pipeline. The App UI is non-empty on first load — no manual submission needed to verify the pipeline end-to-end.

## J4 — Both workers succeed independently

**Preconditions:** service running; mock or real model provider.

**Steps:**
1. Submit several requests in quick succession.
2. Expand each completed row.

**Expected:** every completed row shows distinct draft messages and follow-up actions, reflecting the context provided. The two worker outputs (draft and followUp) are visible separately before the composed response, confirming the parallel fan-out captured both.

## J5 — App UI tab switching is attribute-based

**Preconditions:** service running; browser open at `http://localhost:9954/`.

**Steps:**
1. Click each of the five tabs in turn: Overview, Architecture, Risk Survey, Eval Matrix, App UI.

**Expected:** each tab activates by matching `data-tab` / `data-panel` attributes. No blank panels appear. The Eval Matrix tab shows the empty-state message (zero controls defined). All five panels are present in the DOM with distinct `data-panel` values.
