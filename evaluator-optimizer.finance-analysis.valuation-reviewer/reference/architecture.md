# Architecture: Valuation Reviewer

## Overview

The Valuation Reviewer baseline implements the evaluator-optimizer coordination pattern: one agent drafts the valuation report, a second agent critiques it against comparable transaction data and review standards, and the workflow drives repeated revision cycles until all checks pass or a budget limit is reached.

All durable state — iteration count, current report, critique results, status — lives in `ValuationReviewWorkflow`. The two agent actions (`ValuationDrafterAction`, `ValuationCriticAction`) are stateless; they receive inputs from the workflow, make a single LLM call, and return a typed result. This separation means the loop can survive service restarts mid-iteration without losing progress.

## Component roles

**ValuationReviewWorkflow** is the orchestrator. It holds the iteration counter, drives the draft → critique → revise cycle, enforces the iteration cap, and owns the sign-off gate transition. Every state change is persisted as a workflow event before the next step begins.

**ValuationDrafterAction** wraps the LLM model call for both initial report generation and targeted revisions. It receives the current report and a list of failing critique checks; it returns the revised report text. Retries (up to 3×) with exponential backoff are handled inside the action.

**ValuationCriticAction** wraps the LLM model call for standards-based critique. It receives the report text, the comparable set, and the applicable review standards, then returns a structured `CritiqueResult` with per-dimension scores and a boolean `allPassed` flag. The prompt is engineered to return JSON only, avoiding free-text parsing.

**ValuationReportView** is a key-value projection over workflow events. One row per valuation. The view is the single source of truth for read queries — the workflow itself is never queried directly for current state by external clients.

**ValuationEndpoint** is the HTTP + SSE surface. It routes incoming commands to the workflow, serves the view for GET requests, and streams view updates to SSE clients. It validates API key auth on all routes.

## Data flow

1. A client posts `CreateValuationCommand` to `ValuationEndpoint`.
2. The endpoint forwards to `ValuationReviewWorkflow`, which persists `ValuationCreated` and enters `DRAFTING`.
3. The workflow calls `ValuationDrafterAction`; on success, persists `ReportDrafted` and enters `CRITIQUING`.
4. The workflow calls `ValuationCriticAction`; on success, persists `CritiqueCompleted`.
5. If `allPassed` is false and `iterationCount < iterationCap`, the workflow enters `REVISING`, calls `ValuationDrafterAction` with failing checks, persists `ReportRevised`, increments the counter, and returns to step 4.
6. If `allPassed` is true, or `iterationCount == iterationCap`, the workflow persists `SignOffRequested` (or `IterationCapReached` first) and enters `AWAITING_SIGNOFF`.
7. The human reviewer calls `POST /sign-off` or `POST /reject`. The workflow transitions to `SIGNED_OFF` or `REJECTED`.
8. `ValuationReportView` projects all events in real time. SSE clients subscribed to the stream receive `valuation-updated` events on each projection update.

## Governance integration

The two governance controls are structural workflow properties, not policy add-ons:

- **HITL (application gate):** The `AWAITING_SIGNOFF` state is a hard pause — no further workflow step executes until a human command arrives. The 72-hour auto-reject timeout prevents indefinite suspension while preserving the human-in-the-loop guarantee.
- **HALT (budget cap):** The iteration counter is checked inside the workflow step, not by the caller or the agent. This means the cap is enforced even if the calling code is buggy or retried.
