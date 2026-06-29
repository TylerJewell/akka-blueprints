# Data model — mr-reviewer

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MrWebhook` | `projectPath` | `String` | no | GitLab project path (e.g. `acme/backend`). |
| | `mrIid` | `int` | no | MR IID within the project. |
| | `targetBranch` | `String` | no | Branch the MR targets. |
| | `diffText` | `String` | no | Raw unified diff string. |
| | `submittedBy` | `String` | no | UI identifier of the submitter; defaults to `"anonymous"`. |
| `SanitizerVerdict` | `passed` | `boolean` | no | Whether the diff passed all secret-detection patterns. |
| | `reasonCode` | `String` | no | `OK` or `SECRET_DETECTED`. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `Finding` | `filePath` | `String` | no | File the finding concerns. |
| | `lineNumber` | `int` | no | Diff line number; 0 if the finding is file-level. |
| | `severity` | `String` | no | `INFO`, `WARNING`, or `ERROR`. |
| | `message` | `String` | no | One concise actionable sentence. |
| `ReviewResult` | `findings` | `List<Finding>` | no | Per-file findings; empty if the diff is clean. |
| | `summary` | `String` | no | 2–5 sentence narrative. |
| | `qualityScore` | `int` | no | 0–100 reviewer self-assessment of thoroughness. |
| | `commentary` | `String` | no | Plain-text MR comment text; must not reproduce secrets. |
| | `reviewedAt` | `Instant` | no | When the reviewer returned. |
| `GateFeedback` | `bullets` | `List<String>` | no | 0–3 short bullets (empty on `PASS`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `GateVerdict` | `decision` | `GateDecision` | no | `PASS` or `REFINE`. |
| | `feedback` | `GateFeedback` | no | See above. |
| | `gateScore` | `int` | no | 0–100 rubric. |
| | `evaluatedAt` | `Instant` | no | When the gate returned. |
| `ReviewPass` | `passNumber` | `int` | no | 1-indexed; monotonic across the whole loop (includes guardrail-blocked passes). |
| | `sanitizer` | `SanitizerVerdict` | no | The diff pre-scan verdict. |
| | `review` | `Optional<ReviewResult>` | yes | Empty when sanitizer blocked; populated otherwise. |
| | `gate` | `Optional<GateVerdict>` | yes | Empty when guardrail blocked or sanitizer blocked; populated after gate runs. |
| `MergeRequest` (entity state) | `mrId` | `String` | no | Unique id. |
| | `projectPath` | `String` | no | GitLab project path. |
| | `mrIid` | `int` | no | MR IID within the project. |
| | `targetBranch` | `String` | no | Branch the MR targets. |
| | `maxPasses` | `int` | no | Per-MR retry ceiling (default 3). |
| | `status` | `MrStatus` | no | See enum. |
| | `passes` | `List<ReviewPass>` | no | Bounded at `maxPasses`; starts empty. |
| | `acceptedPassNumber` | `Optional<Integer>` | yes | Populated on `MrCiPassed`. |
| | `acceptedReview` | `Optional<ReviewResult>` | yes | Populated on `MrCiPassed`. |
| | `ciSignal` | `Optional<String>` | yes | `"CI_PASS"` or `"CI_FAIL"`; populated on terminal event. |
| | `receivedAt` | `Instant` | no | When `MrReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the MR reached a terminal state. |

## Enums

`MrStatus`: `RECEIVED`, `REVIEWING`, `GATE_CHECKING`, `CI_PASS`, `CI_FAIL`, `SANITIZER_BLOCKED`.

`GateDecision`: `PASS`, `REFINE`.

## Events (`MrEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `MrReceived` | `mrId, projectPath, mrIid, targetBranch, maxPasses, receivedAt` | Workflow `startStep` | → `RECEIVED` |
| `SanitizerVerdictRecorded` | `passNumber, verdict: SanitizerVerdict` | After `sanitizerStep` | `passed=true` → `REVIEWING`; `passed=false` → `SANITIZER_BLOCKED` |
| `ReviewPassDrafted` | `passNumber, review: ReviewResult` | After `reviewStep` returns | (no status change; appends to `passes[]`) |
| `ReviewGuardrailVerdictRecorded` | `passNumber, reasonCode, detail` | After `guardrailStep` | `passed=true` → `GATE_CHECKING`; `passed=false` → `REVIEWING` (re-review) |
| `ReviewPassGated` | `passNumber, gate: GateVerdict` | After `gateStep` returns | (no status change; populates `passes[n].gate`) |
| `MrCiPassed` | `passNumber, acceptedReview: ReviewResult` | `GateDecision = PASS` | → `CI_PASS`, sets `ciSignal = "CI_PASS"`, `finishedAt = now` |
| `MrCiFailed` | `bestPassNumber, bestReview: ReviewResult, reason` | `passes.size() == maxPasses` AND last gate is `REFINE`; OR `defaultStepRecovery` failover | → `CI_FAIL`, sets `ciSignal = "CI_FAIL"`, `finishedAt = now` |
| `ReviewEvalRecorded` | `passNumber, decision, gateScore, secretBlocked, recordedAt` | `EvalSampler` per gated pass; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`MrQueue`)

| Event | Payload |
|---|---|
| `WebhookReceived` | `mrId, projectPath, mrIid, targetBranch, diffText, submittedBy, receivedAt` |

## View row

`MrRow` is structurally identical to `MergeRequest` — the `passes` list is bounded at `maxPasses` (default 3) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `MrEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllMrs` returning the full list. Callers filter by `status` client-side.
