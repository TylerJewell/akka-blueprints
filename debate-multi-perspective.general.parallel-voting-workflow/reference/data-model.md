# Data model — parallel-voting-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskSubmission` | `title` | `String` | no | The task's title. |
| | `description` | `String` | no | The task description text. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `VotingBrief` | `feasibilityFocus` | `String` | no | Dispatcher's brief for the feasibility voter. |
| | `riskFocus` | `String` | no | Dispatcher's brief for the risk voter. |
| | `alignmentFocus` | `String` | no | Dispatcher's brief for the alignment voter. |
| `VoteReason` | `weight` | `String` | no | `LOW` / `MEDIUM` / `HIGH`. |
| | `text` | `String` | no | The concrete reason text. |
| `Vote` | `dimension` | `String` | no | `FEASIBILITY` / `RISK` / `ALIGNMENT`. |
| | `ballot` | `String` | no | `APPROVE` / `REJECT` / `ABSTAIN`. |
| | `confidence` | `int` | no | 1–5 confidence score. |
| | `reasons` | `List<VoteReason>` | no | 2–4 weighted reasons. |
| | `votedAt` | `Instant` | no | When the voter completed. |
| `AggregatedDecision` | `decision` | `String` | no | `PROCEED` / `HOLD` / `DECLINE`. |
| | `summary` | `String` | no | 60–100 word reconciliation. |
| | `votes` | `List<Vote>` | no | The votes aggregated. |
| | `quorumMet` | `boolean` | no | Whether at least two votes shared the plurality ballot. |
| | `decidedAt` | `Instant` | no | When the dispatcher aggregated. |
| `AlignmentVerdict` | `score` | `int` | no | 1–5 cross-vote alignment score. |
| | `rationale` | `String` | no | One-sentence reason. |
| `Task` (entity state, View row source) | `taskId` | `String` | no | Unique id. |
| | `title` | `String` | no | Task title. |
| | `status` | `TaskStatus` | no | See enum. |
| | `description` | `Optional<String>` | yes | Populated after TaskCreated. |
| | `brief` | `Optional<VotingBrief>` | yes | Populated after BriefingCompleted. |
| | `feasibility` | `Optional<Vote>` | yes | Populated after FeasibilityVoteAttached. |
| | `risk` | `Optional<Vote>` | yes | Populated after RiskVoteAttached. |
| | `alignment` | `Optional<Vote>` | yes | Populated after AlignmentVoteAttached. |
| | `decision` | `Optional<AggregatedDecision>` | yes | Populated after DecisionAggregated. |
| | `failureReason` | `Optional<String>` | yes | Populated on VotingPartial / VotingInconclusive. |
| | `alignmentScore` | `Optional<Integer>` | yes | Populated on AlignmentScored. |
| | `alignmentRationale` | `Optional<String>` | yes | Populated on AlignmentScored. |
| | `createdAt` | `Instant` | no | When TaskCreated emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the task reached a terminal state. |

Every nullable lifecycle field is `Optional<T>` because the View materializer rejects a row record with a non-optional `null` field (Lesson 6).

## Enums

`TaskStatus`: `SUBMITTED`, `VOTING`, `DECIDED`, `PARTIAL`, `INCONCLUSIVE`.

## Events (`TaskEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `TaskCreated` | `taskId, title, description, createdAt` | Workflow `createStep` | → SUBMITTED |
| `BriefingCompleted` | `brief` | Dispatcher returned `VotingBrief` | → VOTING |
| `FeasibilityVoteAttached` | `vote` | `FeasibilityVoter` returned | (no status change; populates `feasibility`) |
| `RiskVoteAttached` | `vote` | `RiskVoter` returned | (no status change; populates `risk`) |
| `AlignmentVoteAttached` | `vote` | `AlignmentVoter` returned | (no status change; populates `alignment`) |
| `DecisionAggregated` | `decision` | Dispatcher aggregated; quorum met | → DECIDED, `finishedAt = now` |
| `VotingPartial` | `decision(partial), failureReason` | A voter timed out | → PARTIAL, `finishedAt = now` |
| `VotingInconclusive` | `decision(no quorum), failureReason` | Quorum not met after all votes returned | → INCONCLUSIVE, `finishedAt = now` |
| `AlignmentScored` | `score, rationale` | `EvalSampler` → `AggregationJudge` | (no status change; populates `alignmentScore` + `alignmentRationale`) |

## Events (`TaskQueue`)

| Event | Payload |
|---|---|
| `TaskReceived` | `taskId, title, description, submittedBy, receivedAt` |

The `description` on `TaskReceived` is handed to the workflow's start command and persisted on `TaskEntity` after `TaskCreated`. There is no sanitizer in this blueprint; if a deployer's use case involves sensitive input, they can add a sanitizer step before briefing.

## View row

`TaskRow` mirrors `Task` minus the heavy `description` text (the row keeps the voting brief, voter ballots/confidence, the aggregated decision, and the alignment score; it truncates each voter's `reasons` to a count to keep the SSE stream small). The UI fetches the full task by id on expand.
