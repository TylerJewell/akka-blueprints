# User journeys

Acceptance tests as numbered journeys. Passing all of these means the blueprint generated correctly.

## J1 — Submit a topic and watch a full campaign complete

**Preconditions:** service running on `http://localhost:9293/`; a model provider key set or mock provider selected.
**Steps:**
1. Open the App UI tab, enter a clearly on-brand topic, click Submit.
2. Observe the campaign appear in `RECEIVED`.
**Expected:** the campaign moves `RESEARCHING → DRAFTING → REVIEWING → EVALUATING → COMPLETED` within ~60s. At `COMPLETED` the research report, blog post, and LinkedIn post are all present, `brandVerdict` is `PASS`, and `qualityScore` is a number in `[0,1]`. The SSE stream updates the card in place at each transition.

## J2 — Off-brand topic is blocked

**Preconditions:** as J1.
**Steps:**
1. Submit a topic the brand reviewer flags (the canned topics include at least one; in mock mode the brand-reviewer responses include a block entry).
2. Watch the campaign progress.
**Expected:** the campaign reaches `REVIEWING`, then moves to `BLOCKED` with a non-empty `blockReason`. The public campaign list and SSE projection withhold `researchReport`, `blogPost`, and `linkedInPost`. The workflow ends; no `qualityScore` is recorded.

## J3 — Background load from the simulator

**Preconditions:** service running; no UI interaction.
**Steps:**
1. Wait one or more 30s ticks.
**Expected:** `TopicSimulator` enqueues a canned topic; `TopicConsumer` starts a `ContentWorkflow`; a new campaign appears in the App UI and runs the same pipeline to `COMPLETED` (or `BLOCKED` for the off-brand topic).

## J4 — Submit via API and read back

**Preconditions:** service running.
**Steps:**
1. `POST /api/campaigns` with `{ "topic": "..." }`; capture `campaignId`.
2. `GET /api/campaigns/{campaignId}` after ~60s.
**Expected:** the returned `Campaign` is `COMPLETED` with all three outputs, or `BLOCKED` with a reason. `GET /api/campaigns?status=COMPLETED` includes it when complete.

## J5 — UI renders governance

**Preconditions:** service running.
**Steps:**
1. Open the Eval Matrix tab.
2. Open the Risk Survey tab.
**Expected:** the Eval Matrix tab shows two controls (G1 guardrail with a red pill, E1 eval-event with a blue pill), each with name, rationale, and implementation. The Risk Survey tab shows the pre-filled answers; fields marked `TO_BE_COMPLETED_BY_DEPLOYER` render muted.
