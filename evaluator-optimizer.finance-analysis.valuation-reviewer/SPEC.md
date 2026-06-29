# Spec: Valuation Reviewer

## 1. Purpose

An agent-driven workflow that produces a reviewed, standards-compliant valuation report. A user submits an asset description, a set of comparable transactions, and optional review constraints. The system drafts an initial valuation report, critiques it against the comparables and applicable review standards, revises sections that fail checks, and iterates until all checks pass or an iteration budget is reached. A human reviewer signs off on the final report before it is released.

---

## 2. User stories

| ID | As a… | I want to… | So that… |
|---|---|---|---|
| US-1 | Valuation analyst | Submit an asset and comps and receive a reviewed draft report | I don't have to structure the analysis from scratch |
| US-2 | Valuation analyst | See the check results for each critique iteration | I understand which standards failed and why |
| US-3 | Senior reviewer | Approve or reject the final valuation report | I retain sign-off authority before release |
| US-4 | Platform operator | Cap the number of revision cycles | LLM costs stay bounded |
| US-5 | Valuation analyst | Be notified when the loop reaches its iteration cap | I can intervene and decide whether to approve the best available draft |

---

## 3. Functional requirements

| ID | Requirement |
|---|---|
| FR-1 | Accept an asset description, comparable transaction set, review standard identifiers, and an optional revision focus |
| FR-2 | Generate an initial valuation report using an LLM agent |
| FR-3 | Critique the report against comparables and review standards (comp alignment, methodology, supportability, disclosure) |
| FR-4 | Revise report sections that fail critique checks |
| FR-5 | Repeat critique–revise until all checks pass or iteration cap is reached |
| FR-6 | Pause the workflow and require explicit human sign-off before releasing the report |
| FR-7 | Emit an `IterationCapReached` event and halt the loop when the iteration budget is exhausted |
| FR-8 | Expose current report state via a streaming SSE view |
| FR-9 | Support cancellation at any point in the loop |

---

## 4. Component design

### 4.1 Workflow — `ValuationReviewWorkflow`

Drives the critique–revise lifecycle. Persists iteration count, current report, and critique results as workflow state. Pauses on `AWAITING_SIGNOFF`. Terminates on `SIGNED_OFF`, `REJECTED`, `CANCELLED`, or `ITERATION_CAP_REACHED`.

**Inputs:** `CreateValuationCommand` (valuationId, assetDescription, comparables, reviewStandards, iterationCap, checkThreshold)
**Outputs:** workflow-level events consumed by `ValuationReportView`

### 4.2 Agent action — `ValuationDrafterAction`

Wraps an LLM model call for both initial report generation and targeted revisions. Receives the current report and a list of failing critique checks; returns the revised report.

**Agent role:** drafter
**Called from:** `ValuationReviewWorkflow`

### 4.3 Agent action — `ValuationCriticAction`

Checks a valuation report against four review dimensions (comp alignment, methodology, supportability, disclosure) on a 0–10 scale. Returns a `CritiqueResult` record containing per-dimension scores and a boolean `allPassed` flag.

**Agent role:** critic
**Called from:** `ValuationReviewWorkflow`

### 4.4 View — `ValuationReportView`

Key-value view over workflow events. One row per valuation ID. Exposes current report text, iteration count, critique results, and status. Streamed to clients over SSE.

### 4.5 Endpoint — `ValuationEndpoint`

HTTP + SSE surface. Validates requests, routes commands to the workflow, serves the view, and exposes the sign-off gate.

---

## 5. Data model

### Commands

| Record | Fields |
|---|---|
| `CreateValuationCommand` | `valuationId: String`, `assetDescription: String`, `comparables: List<String>`, `reviewStandards: List<String>`, `iterationCap: int`, `checkThreshold: double` |
| `SignOffValuationCommand` | `valuationId: String`, `reviewerNotes: String` |
| `RejectValuationCommand` | `valuationId: String`, `reviewerNotes: String` |
| `CancelValuationCommand` | `valuationId: String` |

### Events

| Event | Trigger |
|---|---|
| `ValuationCreated` | Workflow initialized |
| `ReportDrafted` | `ValuationDrafterAction` completed initial draft |
| `CritiqueCompleted` | `ValuationCriticAction` completed a critique pass |
| `ReportRevised` | `ValuationDrafterAction` completed a revision cycle |
| `IterationCapReached` | Iteration count hit the configured cap |
| `SignOffRequested` | Loop exited; awaiting human sign-off |
| `ValuationSignedOff` | Human approved |
| `ValuationRejected` | Human rejected |
| `ValuationCancelled` | Cancellation requested mid-loop |

### Status enum

```
PENDING → DRAFTING → CRITIQUING → REVISING → AWAITING_SIGNOFF
         → SIGNED_OFF | REJECTED | CANCELLED | ITERATION_CAP_REACHED
```

### View row — `ValuationReportRow`

| Field | Nullable |
|---|---|
| `valuationId` | No |
| `assetDescription` | No |
| `status` | No |
| `currentReport` | Yes (null until first draft) |
| `iterationCount` | No |
| `critiqueResults` | Yes (null until first critique) |
| `allPassed` | Yes |
| `reviewerNotes` | Yes |
| `createdAt` | No |
| `updatedAt` | No |

---

## 6. API contract

| Method | Path | Body / Params | Response |
|---|---|---|---|
| POST | `/valuations` | `CreateValuationCommand` | `201 { valuationId }` |
| GET | `/valuations/{valuationId}` | — | `200 ValuationReportRow` |
| GET | `/valuations/{valuationId}/stream` | — | SSE stream of `ValuationReportRow` |
| POST | `/valuations/{valuationId}/sign-off` | `SignOffValuationCommand` | `200 { status }` |
| POST | `/valuations/{valuationId}/reject` | `RejectValuationCommand` | `200 { status }` |
| DELETE | `/valuations/{valuationId}` | — | `204` |

**SSE event format**
```
event: valuation-updated
data: { "valuationId": "...", "status": "CRITIQUING", "iterationCount": 2, ... }
```

---

## 7. Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-1 | Workflow state survives service restart |
| NFR-2 | SSE view delivers updates within 500 ms of state change |
| NFR-3 | LLM call timeout: 30 s per action invocation |
| NFR-4 | Iteration cap max value: 10 |
| NFR-5 | Sign-off gate timeout: 72 h (workflow auto-rejects if no decision received) |

---

## 8. Security

| Control | Implementation |
|---|---|
| API key auth | Bearer token validated at `ValuationEndpoint` before any workflow command |
| LLM key isolation | API key read from environment / secret; never logged or returned in API responses |
| Input length cap | Asset description capped at 2000 chars; comparables list capped at 20 items |

---

## 9. Error handling

| Scenario | Handling |
|---|---|
| LLM call fails | Retry up to 3 times with exponential backoff; emit `ReportDraftFailed` and halt if all retries exhausted |
| Sign-off timeout | Auto-reject after 72 h; emit `ValuationRejected` with `reviewerNotes: "sign-off timeout"` |
| Invalid command | Return `400` with error detail; workflow state unchanged |
| Unknown valuationId | Return `404` |

---

## 10. Test plan

| Test | Type | Asserts |
|---|---|---|
| Happy-path loop | Integration | Report reaches `SIGNED_OFF` after ≤ N iterations |
| Cap enforcement | Unit | Loop stops at configured `iterationCap`; `ITERATION_CAP_REACHED` emitted |
| Sign-off gate | Integration | Workflow pauses at `AWAITING_SIGNOFF`; resumes on `POST /sign-off` |
| Rejection path | Integration | `POST /reject` transitions to `REJECTED` |
| Cancellation | Unit | `CancelValuationCommand` mid-loop transitions to `CANCELLED` |
| LLM retry | Unit | Simulated LLM failure triggers retry; third failure halts workflow |
| SSE stream | Integration | View delivers `CRITIQUING` event within 500 ms |

---

## 11. Identity

| Field | Value |
|---|---|
| Folder | `evaluator-optimizer.finance-analysis.valuation-reviewer` |
| Maven group | `io.akka.samples` |
| Artifact | `evaluator-optimizer-finance-analysis-valuation-reviewer` |
| Java package | `io.akka.samples.valuationreviewer` |
| Akka version | `3.6.0` |
| HTTP port | `9185` |

---

## 12. Constraints and Akka lesson references

- **Lesson 1** — Workflow as the unit of durable state for the critique–revise loop; no in-memory counters.
- **Lesson 4** — Agent actions (`ValuationDrafterAction`, `ValuationCriticAction`) are stateless; all state lives in the workflow.
- **Lesson 6** — Sign-off gate modelled as a workflow pause (`AWAITING_SIGNOFF`) with explicit resume command.
- **Lesson 7** — LLM retries (3×) with exponential backoff inside `ValuationDrafterAction`.
- **Lesson 8** — Iteration cap enforced inside the workflow step, not by the caller.
- **Lesson 9** — `ValuationCriticAction` prompt engineered to return structured `CritiqueResult`; no free-text parsing.
- **Lesson 10** — All secrets (API key) read from environment or Akka secret store; never hardcoded.
- **Lesson 11** — View (`ValuationReportView`) is the single source of truth for read queries; workflow state is write-only.
- **Lesson 12** — SSE endpoint streams `ValuationReportRow` updates; no polling.
- **Lesson 13** — Cancellation command handled in every workflow step to avoid stuck states.
- **Lesson 23** — `IterationCapReached` is a first-class event, not an error.
- **Lesson 24** — State machine diagrams use the Akka Mermaid theme with CSS label overrides.
- **Lesson 25** — API contract is OpenAPI-traceable; every endpoint documented with request/response schema.
- **Lesson 26** — UI tab switching is attribute-based (`data-tab`), not class-based.

---

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
