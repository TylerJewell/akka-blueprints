# User journeys — multi-agent-handoff-workflow

## J1 — Data-analysis task: end-to-end handoff to DataAnalyst

**Preconditions:** Service running on declared port; valid model-provider API key set (or mock LLM); `TaskSimulator` enabled.

**Steps:**
1. Open `http://localhost:<port>/` → App UI tab.
2. Wait up to 30 s for the first simulated request seeded as data-analysis-flavoured to drop.

**Expected:**
- The task appears with status `RECEIVED` and within ~1 s transitions to `ADMITTED`. The admission block shows `admitted=true` with no rejection reasons.
- Within ~10 s the routing block shows `domain = DATA_ANALYSIS`, `confidence = high`, and a one-sentence reason. The status pill changes to `ROUTED` then `ROUTED_DATA_ANALYSIS`.
- Within ~5 s of routing, the right column populates with the `DataAnalyst` result: a headline, a markdown-formatted body, and a `data-analyst` specialist tag.
- The status pill transitions to `COMPLETED`. `finishedAt` is set.
- Within ~10 s of the routing decision, the routing-score chip shows a number 1–5 with a rationale.

## J2 — Content-writing task: end-to-end handoff to ContentWriter

**Preconditions:** Same as J1.

**Steps:**
1. Open the App UI tab.
2. Wait for the simulator to drop a content-writing-flavoured request (the seeded JSONL covers the three domains in rotation).

**Expected:**
- Routing emits `domain = CONTENT_WRITING`. Status `ROUTED_CONTENT_WRITING`.
- `ContentWriter` returns a `TaskResult` with a draft body in `PLAIN_TEXT` or `MARKDOWN_REPORT` format.
- Validation passes. Status `COMPLETED`.
- The `DataAnalyst` and `CodeReviewer` are never invoked for this task — no result from them appears in any surface.

## J3 — Unroutable task: short-circuits to REJECTED

**Preconditions:** The seeded JSONL includes a deliberately ambiguous one-liner.

**Steps:**
1. Wait for the unroutable request to drop.

**Expected:**
- Routing emits `domain = UNROUTABLE` with `confidence = low`.
- The workflow terminates immediately with `TaskRejected`. Status `REJECTED`.
- No specialist is invoked; the right column shows a muted "Rejected — no specialist invoked" block with the `rejectionReason`.
- A `RoutingScored` event still fires; the score chip appears (typically high, since the classifier was correct to refuse).

## J4 — Task fails admission before any agent activates

**Preconditions:** The seeded JSONL includes a request with a prohibited-topic keyword or an empty description.

**Steps:**
1. Wait for the failing request to drop.

**Expected:**
- The admission block shows `admitted=false` with at least one rejection reason populated.
- Status transitions directly to `REJECTED` (terminal). No workflow is started.
- Neither the `RouterAgent` nor any specialist is ever invoked — the App UI shows "Rejected at admission" in the right column.
- No `RoutingDecided` event fires; there is no routing-score chip for this task.

## J5 — Before-tool-call guardrail blocks a disallowed tool

**Preconditions:** The seeded JSONL includes a request engineered to cause a specialist to attempt an external web fetch (and the mock-responses file has a matching trip-the-guardrail entry).

**Steps:**
1. Wait for the trip request to drop and be admitted and routed.
2. Wait for the specialist to begin executing and attempt the disallowed tool call.

**Expected:**
- Status reaches `EXECUTING`.
- The right column shows a red "Tool blocked" banner with the tool name (`fetch_external_url`) and block reason ("tool not in allowed-tool registry").
- Status transitions to `TOOL_BLOCKED`. `finishedAt` remains null.
- No `TaskResult` is produced; the `result` field remains absent.

## J6 — Routing score appears on every routed task

**Preconditions:** Service running at least 60 s with the simulator on.

**Steps:**
1. Watch any new task through the routing step.

**Expected:**
- Within ~10 s of `RoutingDecided`, the per-task routing-score chip is populated with a 1–5 value and a one-sentence rationale.
- The chip colour-grades the value: 1–2 red, 3 amber, 4–5 green.
- The chip is visible both on the list-row card and in the centre column detail.
- The chip never changes the workflow's flow — a low score does not block the result from being published.
