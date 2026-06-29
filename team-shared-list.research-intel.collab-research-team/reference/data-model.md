# Data model — collab-research-team

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ResearchQuestion` | `questionId` | `String` | no | Id assigned at submission. |
| | `title` | `String` | no | Question as phrased by the user. |
| | `context` | `String` | no | Additional scope notes (may be empty string). |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `SubTopicSpec` | `title` | `String` | no | Short noun phrase naming the sub-topic. |
| | `focusStatement` | `String` | no | One sentence the researcher should keep in mind. |
| | `keywords` | `List<String>` | no | Three to five search terms (may not be empty). |
| `ResearchPlan` | `planSummary` | `String` | no | One-sentence approach from the coordinator. |
| | `subTopics` | `List<SubTopicSpec>` | no | Three to five sub-topic specs. |
| `SourceRecord` | `url` | `String` | no | Well-formed HTTPS URL. |
| | `title` | `String` | no | Page or article title. |
| | `excerpt` | `String` | no | Verbatim excerpt (≤ 200 words). |
| `CoordinationRequest` | `toResearcher` | `String` | no | Researcher id the request is addressed to. |
| | `question` | `String` | no | Concrete scope-clarification question. |
| `FindingsReport` | `taskId` | `String` | no | The sub-topic task this report covers. |
| | `sources` | `List<SourceRecord>` | no | At least two distinct source records (empty when a coordination request is raised). |
| | `summary` | `String` | no | Two-to-four sentence synthesis of the sources. |
| | `coordinationRequest` | `Optional<CoordinationRequest>` | yes | Present only when the researcher is blocked on scope clarification. |
| `ReportConclusion` | `statement` | `String` | no | One declarative sentence. |
| | `citedUrls` | `List<String>` | no | At least one URL from the findings reports (empty only if the E1 eval fires). |
| `ResearchReport` | `questionId` | `String` | no | The question this report answers. |
| | `conclusions` | `List<ReportConclusion>` | no | Three to six conclusions. |
| | `executiveSummary` | `String` | no | Three to five sentence summary. |
| | `synthesizedAt` | `Instant` | no | When synthesis completed. |
| `CitationEvalResult` | `statement` | `String` | no | The conclusion that was evaluated. |
| | `grounded` | `boolean` | no | Whether at least one cited URL is well-formed and present. |
| | `missingCitations` | `List<String>` | no | Claims in the statement that could not be traced (empty on pass). |
| `CoordinationMessage` | `messageId` | `String` | no | Unique id. |
| | `fromResearcher` | `String` | no | Sender researcher id. |
| | `toResearcher` | `String` | no | Recipient researcher id. |
| | `taskId` | `String` | no | The blocked sub-topic task the message concerns. |
| | `question` | `String` | no | The scope-clarification question. |
| | `sentAt` | `Instant` | no | When posted. |
| | `reply` | `Optional<String>` | yes | Populated on reply. |
| | `repliedAt` | `Optional<Instant>` | yes | Populated on reply. |

## Entity state — `ResearchTask` (`ResearchTaskEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `taskId` | `String` | no | Deterministic id `questionId + "-t" + index`. |
| `questionId` | `String` | no | Owning question. |
| `title` | `String` | no | From the `SubTopicSpec`. |
| `focusStatement` | `String` | no | From the `SubTopicSpec`. |
| `keywords` | `List<String>` | no | From the `SubTopicSpec`. |
| `status` | `ResearchTaskStatus` | no | See enum. |
| `claimedBy` | `Optional<String>` | yes | Researcher that won the claim. |
| `claimedAt` | `Optional<Instant>` | yes | When claimed. |
| `findingsSummary` | `Optional<String>` | yes | Summary from the submitted `FindingsReport`. |
| `sourceCount` | `Optional<Integer>` | yes | Number of distinct sources in the report. |
| `blockedReason` | `Optional<String>` | yes | Why the task is blocked. |
| `completedAt` | `Optional<Instant>` | yes | When the task reached `DONE`. |
| `createdAt` | `Instant` | no | When `TaskCreated` emitted. |

## Entity state — `Question` (`QuestionEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `questionId` | `String` | no | Unique id. |
| `title` | `String` | no | Question text. |
| `context` | `String` | no | Scope notes. |
| `submittedBy` | `String` | no | Submitter id. |
| `status` | `QuestionStatus` | no | See enum. |
| `taskIds` | `List<String>` | no | Ids of sub-topic tasks. |
| `planSummary` | `Optional<String>` | yes | Populated on `QuestionPlanned`. |
| `report` | `Optional<ResearchReport>` | yes | Populated on `ReportReady`. |
| `reviewNote` | `Optional<String>` | yes | Populated on `ReportApproved` or `ReportRejected`. |
| `createdAt` | `Instant` | no | When `QuestionCreated` emitted. |
| `completedAt` | `Optional<Instant>` | yes | When the question reached `COMPLETED`. |

## Enums

`ResearchTaskStatus`: `OPEN`, `CLAIMED`, `IN_PROGRESS`, `FINDINGS_READY`, `DONE`, `BLOCKED`.
`QuestionStatus`: `PLANNING`, `READY`, `IN_PROGRESS`, `SYNTHESIZING`, `NEEDS_REVIEW`, `COMPLETED`.

## Events — `ResearchTaskEntity`

| Event | Payload | Transition |
|---|---|---|
| `TaskCreated` | `taskId, questionId, title, focusStatement, keywords, createdAt` | → OPEN |
| `TaskClaimed` | `researcherId, claimedAt` | OPEN → CLAIMED (emitted only when current status is OPEN) |
| `TaskStarted` | `startedAt` | CLAIMED → IN_PROGRESS |
| `FindingsSubmitted` | `findingsSummary, sourceCount, submittedAt` | IN_PROGRESS → FINDINGS_READY |
| `TaskCompleted` | `completedAt` | FINDINGS_READY → DONE |
| `TaskBlocked` | `reason, blockedAt` | (any active) → BLOCKED |
| `TaskReleased` | `releasedAt` | CLAIMED/IN_PROGRESS → OPEN, clears `claimedBy` |

## Events — `QuestionEntity`

| Event | Payload | Transition |
|---|---|---|
| `QuestionCreated` | `questionId, title, context, submittedBy, createdAt` | → PLANNING |
| `QuestionPlanned` | `taskIds, planSummary` | PLANNING → READY |
| `QuestionStarted` | `startedAt` | READY → IN_PROGRESS |
| `SynthesisStarted` | `startedAt` | IN_PROGRESS → SYNTHESIZING |
| `ReportReady` | `report` | SYNTHESIZING → COMPLETED (grounded) or SYNTHESIZING → NEEDS_REVIEW (un-grounded) |
| `ReportApproved` | `reviewedBy, note, approvedAt` | NEEDS_REVIEW → COMPLETED |
| `ReportRejected` | `reviewedBy, reason, rejectedAt` | NEEDS_REVIEW → IN_PROGRESS (re-opens for re-synthesis) |

## Events — `ResearcherMailbox`

| Event | Payload |
|---|---|
| `MessagePosted` | `CoordinationMessage` |
| `MessageReplied` | `messageId, reply, repliedAt` |

## Events — `IntakeQueue`

| Event | Payload |
|---|---|
| `QuestionSubmitted` | `questionId, title, context, submittedBy, submittedAt` |

## Key-value state — `OperatorControl`

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `halted` | `boolean` | no | Whether the team is frozen. |
| `haltedReason` | `Optional<String>` | yes | Operator-supplied reason. |
| `haltedBy` | `Optional<String>` | yes | Operator id. |
| `haltedAt` | `Optional<Instant>` | yes | When halted. |

## View row — `TaskRow`

`TaskRow` mirrors `ResearchTask` but omits source content — it keeps `findingsSummary` and `sourceCount` only, so the SSE stream stays small. Every nullable lifecycle field on the row is `Optional<T>` (Lesson 6).
