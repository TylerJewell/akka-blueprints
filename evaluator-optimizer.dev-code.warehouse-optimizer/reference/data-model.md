# Data model — warehouse-optimizer

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `OptimizationTarget` | `originalSql` | `String` | no | The query or DDL statement to optimize. |
| | `objective` | `String` | no | The optimization goal. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `Proposal` | `proposedSql` | `String` | no | The rewritten SQL or DDL statement. |
| | `kind` | `ProposalKind` | no | Category of the proposed change. |
| | `rationale` | `String` | no | Short paragraph explaining the change. |
| | `proposedAt` | `Instant` | no | When the Optimizer returned the proposal. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the proposal. |
| | `reasonCode` | `String` | no | `OK`, `DDL_DETECTED`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `DBADecision` | `approved` | `boolean` | no | Whether the DBA approved the DDL proposal. |
| | `decisionNote` | `String` | no | Free-text note from the DBA; may be empty. |
| | `decidedBy` | `String` | no | Identifier of the DBA who decided. |
| | `decidedAt` | `Instant` | no | When the DBA decision was recorded. |
| `EvaluationNotes` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `APPROVE`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `Evaluation` | `verdict` | `EvaluatorVerdict` | no | `APPROVE` or `REVISE`. |
| | `notes` | `EvaluationNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `evaluatedAt` | `Instant` | no | When the Evaluator returned. |
| `Attempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop. |
| | `proposal` | `Proposal` | no | The Optimizer's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `dbaDecision` | `Optional<DBADecision>` | yes | Empty for non-DDL proposals; populated after DBA decides. |
| | `evaluation` | `Optional<Evaluation>` | yes | Empty when DBA rejected or when the loop has not yet evaluated; populated otherwise. |
| `OptimizationRequest` (entity state) | `requestId` | `String` | no | Unique id. |
| | `originalSql` | `String` | no | Original submitted SQL. |
| | `objective` | `String` | no | Original optimization goal. |
| | `maxAttempts` | `int` | no | Per-request attempt ceiling (default 4). |
| | `status` | `RequestStatus` | no | See enum. |
| | `attempts` | `List<Attempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `approvedAttemptNumber` | `Optional<Integer>` | yes | Populated on `RequestApproved`. |
| | `approvedSql` | `Optional<String>` | yes | Populated on `RequestApproved`. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `RequestRejectedFinal`. |
| | `createdAt` | `Instant` | no | When `RequestCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the request reached a terminal state. |

## Enums

`RequestStatus`: `PROPOSING`, `AWAITING_DBA`, `EVALUATING`, `APPROVED`, `REJECTED_FINAL`.

`EvaluatorVerdict`: `APPROVE`, `REVISE`.

`ProposalKind`: `QUERY_REWRITE`, `INDEX_ADD`, `PARTITION_PRUNE`, `DDL_ALTER`, `DDL_CREATE`, `DDL_DROP`.

## Events (`OptimizationRequestEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RequestCreated` | `originalSql, objective, maxAttempts, createdAt` | Workflow `startStep` | → `PROPOSING` |
| `AttemptProposed` | `attemptNumber, proposal: Proposal` | After `proposeStep` returns | (no status change; appends to `attempts[]`) |
| `AttemptGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `EVALUATING`; `passed=false` (DDL) → `AWAITING_DBA` |
| `DBADecisionRecorded` | `attemptNumber, decision: DBADecision` | DBA submits via `POST /api/requests/{id}/dba-decision` | `approved=true` → `EVALUATING`; `approved=false` → `REJECTED_FINAL` |
| `AttemptEvaluated` | `attemptNumber, evaluation: Evaluation` | After `evaluateStep` returns | (no status change; populates `attempts[n].evaluation`) |
| `RequestApproved` | `attemptNumber, approvedSql` | `Evaluation.verdict = APPROVE` | → `APPROVED`, `finishedAt = now` |
| `RequestRejectedFinal` | `bestAttemptNumber, bestSql, rejectionReason` | `attempts.size() == maxAttempts` AND last evaluation is `REVISE`; OR DBA rejected; OR `defaultStepRecovery` failover | → `REJECTED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, score, ddlDetected, recordedAt` | `EvalSampler` per evaluated attempt; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `RequestSubmitted` | `requestId, originalSql, objective, submittedBy, submittedAt` |

## View row

`RequestRow` is structurally identical to `OptimizationRequest` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `OptimizationRequestEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllRequests` returning the full list. Callers filter by `status` client-side.
