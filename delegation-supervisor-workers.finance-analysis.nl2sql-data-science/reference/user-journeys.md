# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a financial question and watch parallel analysis

**Preconditions:** service running on `http://localhost:9814/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a financial question such as "What were the top 5 revenue-generating product lines last quarter?" and Submit.
2. Observe the new job row via SSE.

**Expected:** the job progresses `TRANSLATING → RUNNING → COMPLETE` within ~90 s. The expanded row shows a SQL plan (SELECT statement, target table, safetyVerdict "ok"), a DataSet (column headers and data rows), a ModelResult (model type and 2–4 metrics), and a synthesised summary. DataSet and ModelResult arrive close together because the workers ran in parallel.

## J2 — Worker timeout degrades the job

**Preconditions:** `DataRetriever` step timeout set to 1 s (test override).

**Steps:**
1. Submit a financial question.
2. Watch the job row.

**Expected:** the `retrieveStep` times out, the workflow routes to `degradeStep`, and the Coordinator synthesises from the ModelResult alone. The job enters `DEGRADED`; the summary notes the missing DataSet. No infinite retry. The ModelResult is still visible in the expanded row.

## J3 — SQL guardrail blocks a destructive query

**Preconditions:** the test fixture submits a question phrased to trigger a destructive SQL keyword (e.g., "Delete all expense records for Q1").

**Steps:**
1. Submit the fixture question via the App UI or `POST /api/analysis`.
2. Watch the job row.

**Expected:** `guardrailStep` detects a destructive keyword in the generated SQL; the workflow calls `blockJob`; the job enters `BLOCKED` with a `failureReason` naming the rejected keyword. No DataRetriever or StatisticsModeler call is made. The job row's expanded view shows the failureReason.

## J4 — NL2SQL fidelity score appears beside a completed job

**Preconditions:** at least one `COMPLETE` job without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it directly.
2. Refresh or watch the job row via SSE.

**Expected:** the job gains an `evalScore` (1–5) and an `evalRationale` assessing how accurately the coordinator's SQL reflected the original question. The App UI row shows the score. Job status remains `COMPLETE` — delivery was never blocked by the eval (non-blocking).

## J5 — Background load from the question simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `QuestionSimulator` drips a financial question from `financial-questions.jsonl` every 60 s; each becomes a job that flows through the full pipeline (TRANSLATING → RUNNING → COMPLETE or DEGRADED). The App UI is non-empty on first load.

## J6 — Status filtering narrows the job list

**Preconditions:** at least one COMPLETE and one BLOCKED job present in the view.

**Steps:**
1. Call `GET /api/analysis?status=COMPLETE`.
2. Call `GET /api/analysis?status=BLOCKED`.

**Expected:** each call returns only jobs matching the requested status (client-side filter applied by the endpoint). No cross-contamination between status values. DEGRADED jobs do not appear in the COMPLETE list.
