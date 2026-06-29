# User journeys

Acceptance tests for the generated system. The blueprint generated correctly when J1–J5 pass.

## J1 — Submit a question and watch parallel retrieval and synthesis

**Preconditions:** service running on `http://localhost:9206/`; a model provider configured (real or mock).

**Steps:**
1. Open the App UI tab. Enter a research question and Submit.
2. Observe the new run row via SSE.

**Expected:** the run progresses `QUEUED → IN_PROGRESS → ANSWERED` within ~90 s. The expanded row shows a `WebBundle` (2–4 pages with extracted text), a `DocumentBundle` (3–5 sections), and a `SynthesisedAnswer` with a summary and citations. Web content and document content arrive close together because the workers ran in parallel. The `piiSanitized` flag is true on the answer.

## J2 — Worker timeout degrades the run

**Preconditions:** `WebBrowsingAgent` step timeout set to 1 s (test override).

**Steps:**
1. Submit a question.
2. Watch the run.

**Expected:** `fetchWebStep` times out, the workflow routes to `degradeStep`, and the manager synthesises from the document content alone. The run enters `DEGRADED`; the summary notes the missing web content in one sentence. No infinite retry.

## J3 — Guardrail blocks a disallowed URL

**Preconditions:** retrieval plan includes a URL not on the allow-list (test fixture).

**Steps:**
1. Submit a question whose retrieval plan is seeded with a disallowed URL.
2. Watch the run.

**Expected:** the `before-tool-call` guardrail blocks the disallowed URL before the fetch tool executes; the URL is recorded as `blockedUrl` on the run. The remaining allowed URLs are fetched normally. The run continues to `ANSWERED` (not `BLOCKED`) unless all URLs were blocked and no document content came back either.

## J4 — Eval score appears beside an answered run

**Preconditions:** at least one `ANSWERED` run without an `evalScore`.

**Steps:**
1. Wait for `EvalSampler` to run (every 5 minutes), or trigger it manually.
2. Observe the run row.

**Expected:** the run gains an `evalScore` (1–5) and an `evalRationale`; the App UI row shows the score badge. Delivery was never blocked by the eval (non-blocking).

## J5 — Retry-budget breach raises the oversight flag

**Preconditions:** configure the retry budget to 1 (test override); use a model configuration that triggers at least one retry.

**Steps:**
1. Submit a question.
2. Watch the monitoring endpoint and the run row.

**Expected:** `monitorStep` detects the budget breach; `OversightFlagged` event is emitted; `oversightFlagged` is true on the run; the run still reaches `ANSWERED`. `GET /api/research/monitoring` returns `flaggedCount >= 1` with a recent `lastFlaggedAt` timestamp.

## J6 — Background load from the simulator

**Preconditions:** no UI interaction.

**Steps:**
1. Leave the service running.

**Expected:** `QuestionSimulator` drips a question from `research-questions.jsonl` every 60 s; each becomes a run that flows through the full pipeline. The App UI is non-empty on first load.
