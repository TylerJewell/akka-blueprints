# Data model — self-improving-agent

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Task` | `taskId` | `String` | no | Unique identifier assigned at enqueue time. |
| | `instruction` | `String` | no | The user-submitted task instruction. |
| | `expectedHint` | `String` | no | Optional output form hint; empty string when absent. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `ExecutionResult` | `outputText` | `String` | no | The executor's output. |
| | `qualityScore` | `int` | no | Self-assessed quality, 1–5. |
| | `latencyMs` | `long` | no | Time from task receipt to output ready. |
| | `executedAt` | `Instant` | no | When the executor returned the result. |
| `PerformanceRecord` | `configId` | `String` | no | The configuration used for this batch. |
| | `results` | `List<ExecutionResult>` | no | All execution results in the batch. |
| | `meanQualityScore` | `double` | no | Arithmetic mean of `qualityScore` across results. |
| | `collectedAt` | `Instant` | no | When the batch was closed. |
| `PromptRevision` | `proposedSystemPrompt` | `String` | no | The full revised system-prompt text. |
| | `rationale` | `String` | no | One-paragraph explanation of the change. |
| | `revisionAttempt` | `int` | no | 1-indexed cycle number. |
| `AttestationVerdict` | `passed` | `boolean` | no | Whether the gate passed. |
| | `scoreDelta` | `double` | no | `meanRegressionScore - baselineScore`; positive is better. |
| | `detail` | `String` | no | Human-readable detail about the gate decision. |
| `RevisionCycle` | `cycleNumber` | `int` | no | 1-indexed; monotonic across the loop. |
| | `proposal` | `PromptRevision` | no | The optimizer's proposed revision for this cycle. |
| | `attestation` | `AttestationVerdict` | no | The gate's verdict for this cycle. |
| | `appliedConfigId` | `Optional<String>` | yes | The new config ID if the revision was applied; empty on failure. |
| `AgentConfig` (entity state) | `configId` | `String` | no | Unique id. |
| | `systemPrompt` | `String` | no | The current active system prompt. |
| | `status` | `AgentConfigStatus` | no | See enum. |
| | `cycles` | `List<RevisionCycle>` | no | Bounded at `maxRevisions`; starts empty. |
| | `parentConfigId` | `Optional<String>` | yes | ID of the predecessor config; empty for the initial config. |
| | `rejectionReason` | `Optional<String>` | yes | Populated on `RevisionRejected`. |
| | `createdAt` | `Instant` | no | When `ConfigInitialized` emitted. |
| | `finalizedAt` | `Optional<Instant>` | yes | When the config reached a terminal state. |

## Enums

`AgentConfigStatus`: `ACTIVE`, `IMPROVING`, `APPLIED`, `REVISION_REJECTED`.

`AttestationResult`: `PASSED`, `FAILED`.

## Events (`AgentConfigEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ConfigInitialized` | `configId, systemPrompt, createdAt` | Workflow `collectStep` | → `IMPROVING` |
| `RevisionProposed` | `cycleNumber, proposal: PromptRevision` | After `proposeStep` returns | (no status change; appends to `cycles[]`) |
| `AttestationRecorded` | `cycleNumber, verdict: AttestationVerdict` | After `attestStep` completes | `passed=true` → advances to `applyStep`; `passed=false` → re-enters `proposeStep` or `rejectStep` |
| `RevisionApplied` | `cycleNumber, appliedConfigId, finalizedAt` | `AttestationVerdict.passed = true` | → `APPLIED`, `finalizedAt = now` |
| `RevisionRejected` | `bestCycleNumber, bestProposedPrompt, rejectionReason, finalizedAt` | `cycles.size() == maxRevisions` AND last attestation `passed=false`; OR `defaultStepRecovery` failover | → `REVISION_REJECTED`, `finalizedAt = now` |
| `EvalRecorded` | `cycleNumber, attestationResult, scoreDelta, applied, recordedAt` | `AttestationSampler` per completed cycle; workflow on terminal transition | (no status change; appended to an internal `evalEvents[]` view-side projection) |

## Events (`TaskQueueEntity`)

| Event | Payload |
|---|---|
| `TaskSubmitted` | `taskId, instruction, expectedHint, submittedBy, submittedAt` |

## View row

`ConfigRow` is structurally identical to `AgentConfig` — the `cycles` list is bounded at `maxRevisions` (default 3) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `AgentConfigEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllConfigs` returning the full list. Callers filter by `status` client-side.
