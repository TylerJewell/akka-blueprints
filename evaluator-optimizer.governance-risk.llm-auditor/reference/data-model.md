# Data model — llm-auditor

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ChatbotResponse` | `text` | `String` | no | The raw response text to audit. |
| | `channel` | `String` | no | The channel the response came from (e.g., "support"). |
| | `receivedAt` | `Instant` | no | When the response was received. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail allowed the response into the audit loop. |
| | `reasonCode` | `String` | no | `OK`, `SEVERITY_EXCEEDS_THRESHOLD`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| | `initialSeverity` | `int` | no | Computed severity score (0–10) from the deterministic scanner. |
| `AuditFinding` | `dimension` | `String` | no | One of: `ACCURACY`, `TONE`, `POLICY_COMPLIANCE`, `HALLUCINATION_RISK`. |
| | `description` | `String` | no | One sentence describing the specific problem. |
| | `suggestedFix` | `String` | no | One sentence describing the required change. |
| `AuditFindings` | `findings` | `List<AuditFinding>` | no | 0–4 findings (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `AuditVerdict` | `decision` | `CriticDecision` | no | `PASS` or `REVISE`. |
| | `findings` | `AuditFindings` | no | See above. |
| | `severityScore` | `int` | no | 0–10 residual risk after audit. |
| | `auditedAt` | `Instant` | no | When the CriticAgent returned. |
| `RevisedResponse` | `text` | `String` | no | The revised response text. |
| | `revisedAt` | `Instant` | no | When the ReviserAgent returned. |
| `RevisionCycle` | `cycleNumber` | `int` | no | 1-indexed; monotonic across the loop. |
| | `input` | `ChatbotResponse` | no | The response text audited in this cycle. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict (cycle 1 only carries the original guardrail; subsequent cycles inherit `passed=true`). |
| | `verdict` | `Optional<AuditVerdict>` | yes | Empty when the guardrail blocked; populated otherwise. |
| | `revision` | `Optional<RevisedResponse>` | yes | Empty when verdict is `PASS` or guardrail blocked; populated on `REVISE`. |
| `Session` (entity state) | `sessionId` | `String` | no | Unique id. |
| | `channel` | `String` | no | Channel from the original response. |
| | `maxRevisions` | `int` | no | Per-session revision ceiling (default 3). |
| | `status` | `SessionStatus` | no | See enum. |
| | `cycles` | `List<RevisionCycle>` | no | Bounded at `maxRevisions`; starts empty. |
| | `approvedAtCycle` | `Optional<Integer>` | yes | Populated on `SessionApproved`. |
| | `approvedText` | `Optional<String>` | yes | Populated on `SessionApproved`. |
| | `escalationReason` | `Optional<String>` | yes | Populated on `SessionEscalated`. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal state. |

## Enums

`SessionStatus`: `AUDITING`, `REVISING`, `APPROVED`, `ESCALATED`.

`CriticDecision`: `PASS`, `REVISE`.

## Events (`SessionEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `SessionCreated` | `sessionId, channel, maxRevisions, createdAt` | Workflow `startStep` | → `AUDITING` |
| `CycleStarted` | `cycleNumber, input: ChatbotResponse` | Before `auditStep` for each cycle | (no status change; appends new `RevisionCycle` to `cycles[]`) |
| `GuardrailVerdictRecorded` | `cycleNumber, verdict: GuardrailVerdict` | After `guardrailStep` (cycle 1 only) | `passed=true` → remains `AUDITING`; `passed=false` → `ESCALATED` immediately |
| `AuditVerdictRecorded` | `cycleNumber, verdict: AuditVerdict` | After `auditStep` returns | `PASS` → stays `AUDITING` (approve step next); `REVISE` and cycles < max → `REVISING` |
| `RevisionProduced` | `cycleNumber, revision: RevisedResponse` | After `reviseStep` returns | → `AUDITING` (next cycle) |
| `SessionApproved` | `cycleNumber, approvedText` | `AuditVerdict.decision = PASS` | → `APPROVED`, `finishedAt = now` |
| `SessionEscalated` | `bestCycleNumber, bestText, escalationReason` | `cycles.size() == maxRevisions` AND last verdict is `REVISE`; OR guardrail blocked; OR `defaultStepRecovery` failover | → `ESCALATED`, `finishedAt = now` |
| `AuditRecorded` | `cycleNumber, decision, severityScore, guardrailBlocked, recordedAt` | `AuditSampler` per completed cycle; workflow on terminal transition | (no status change; appends to an internal `auditEvents[]` view-side projection) |

## Events (`ResponseQueue`)

| Event | Payload |
|---|---|
| `ResponseReceived` | `sessionId, text, channel, receivedAt` |

## View row

`SessionRow` is structurally identical to `Session` — the `cycles` list is bounded at `maxRevisions` (default 3) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `SessionEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllSessions` returning the full list. Callers filter by `status` client-side.
