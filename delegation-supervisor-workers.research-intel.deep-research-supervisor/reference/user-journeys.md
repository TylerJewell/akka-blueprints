# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1-J5 pass.

## J1 -- Submit a question and watch multi-subquery synthesis

**Preconditions:** service running on `http://localhost:9390/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a research question and Submit.
2. Observe the new report row via SSE.

**Expected:** the report progresses `PLANNING -> IN_PROGRESS -> SYNTHESISED` within ~120 s. The expanded row shows 2-4 per-subquery summaries, an executive summary, and a citation list. The subquery summaries arrive close together because the subquery pipelines ran in parallel.

## J2 -- SearchWorker timeout degrades the report

**Preconditions:** `SearchWorker` step timeout set to 1 s (test override).

**Steps:**
1. Submit a research question.
2. Watch the report.

**Expected:** the affected subquery's `searchStep` times out, that subquery is marked partial, and the workflow continues with the remaining subqueries. The supervisor synthesises from the available summaries. The report enters `DEGRADED`; the executive summary notes the missing subquery. No infinite retry.

## J3 -- Guardrail blocks a report with unverifiable citations

**Preconditions:** ResearchSupervisor returns a report citing a source not present in the collected PassageBundles (test fixture).

**Steps:**
1. Submit the fixture question.
2. Watch the report.

**Expected:** `guardrailStep` detects the unverifiable citation; the workflow calls `block`; the report enters `BLOCKED` with a `failureReason` naming the first unverifiable citation. The report never appears as a finished result in the App UI done state.

## J4 -- Citation-grounding eval score appears beside a synthesised report

**Preconditions:** at least one `SYNTHESISED` report without an `evalScore`.

**Steps:**
1. Wait for `CitationEvalSampler` to run (every 5 minutes), or trigger it directly.
2. Refresh the report row.

**Expected:** the report gains an `evalScore` (1-5) and an `evalRationale`; the App UI row shows the score alongside the status pill. Delivery was never blocked by the eval (non-blocking).

## J5 -- Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `QuestionSimulator` drips a question from `research-questions.jsonl` every 60 s; each becomes a report that flows through the full pipeline. The App UI is non-empty on first load.

## J6 -- Subquery decomposition produces non-overlapping queries

**Preconditions:** service running with a real model provider.

**Steps:**
1. Submit a broad research question (e.g., "How does the EU AI Act treat foundation models differently from narrow AI systems?").
2. Wait for `SYNTHESISED`.
3. Expand the report and read the per-subquery summaries.

**Expected:** each `SubquerySummary.queryText` addresses a distinct facet of the original question with no two summaries covering the same ground. The citation list contains only citations whose `quote` appears verbatim in a `SubquerySummary.supportingClaims` entry.
