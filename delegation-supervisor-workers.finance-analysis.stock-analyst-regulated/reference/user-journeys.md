# User journeys — Stock Analysis Team

Acceptance journeys. Passing all of these means the blueprint generated correctly.

## J1 — Submit a ticker and watch a recommendation issue

- **Preconditions:** service running on `http://localhost:9956/`; one model provider configured (or mock).
- **Steps:** POST `/api/analyses` with `{ "ticker": "ACME" }`; subscribe to `/api/analyses/sse`.
- **Expected:** the analysis appears in `QUEUED`, advances through `RESEARCHING → SUMMARIZING → SANITIZING → RECOMMENDING`, and within ~60s reaches `ISSUED_PENDING_REVIEW` with a non-empty `recommendation`, `rationale`, and `disclaimer`.
- **Done when:** the issued analysis is visible in the App UI with its recommendation call.

## J2 — Disclaimer guardrail rejects an undisclosed recommendation

- **Preconditions:** a recommendation generated without a disclaimer (forced via a mock-response entry whose `disclaimer` is empty).
- **Steps:** run an analysis that produces the undisclosed recommendation.
- **Expected:** the before-agent-response guardrail rejects it; the recommendation never persists; the analysis ends in `FAILED` rather than `ISSUED_PENDING_REVIEW`.
- **Done when:** no `ISSUED_PENDING_REVIEW` analysis exists without a `disclaimer`.

## J3 — Live compliance review clears or retracts

- **Preconditions:** an analysis in `ISSUED_PENDING_REVIEW`.
- **Steps:** in the App UI, click Retract (with reviewer + note); repeat on another analysis with Clear.
- **Expected:** the first moves to `RETRACTED`, the second to `CLEARED`; both record `reviewedBy`, `reviewNote`, and `reviewedAt`; the review buttons disappear.
- **Done when:** the SSE stream reflects the terminal review state.

## J4 — Drift watch flags a skewed distribution

- **Preconditions:** several analyses issued with the same call (e.g. all `BUY`).
- **Steps:** let `DriftMonitor` run (every 60s).
- **Expected:** `driftFlag` is set on the affected analyses; the UI shows the flag.
- **Done when:** at least one analysis row shows the drift flag.

## J5 — Background load from the simulator

- **Preconditions:** service running, no UI interaction.
- **Steps:** wait through one or more `RequestSimulator` ticks (every 30s).
- **Expected:** new analyses appear seeded from `sample-events/analysis-requests.jsonl`, each starting a fresh workflow.
- **Done when:** analyses appear without any manual submission.
