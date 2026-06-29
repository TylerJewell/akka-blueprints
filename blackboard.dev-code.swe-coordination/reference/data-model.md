# Data model — blackboard-swe-coordination

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TicketBrief` | `ticketId` | `String` | no | Id assigned at submission. |
| | `title` | `String` | no | Short ticket name. |
| | `description` | `String` | no | Free-text brief. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `TaskItem` | `title` | `String` | no | Short imperative task title. |
| | `description` | `String` | no | One or two sentences a specialist can act on. |
| | `layer` | `String` | no | One of: `backend`, `frontend`, `shared`, `infra`. |
| `TaskBreakdown` | `summary` | `String` | no | One-sentence approach from the Planner. |
| | `tasks` | `List<TaskItem>` | no | Three to six task items. |
| `ArchDecision` | `approach` | `String` | no | One-paragraph design strategy. |
| | `components` | `List<String>` | no | Named components with one-sentence purpose each. |
| | `constraints` | `List<String>` | no | Design constraints developers must respect. |
| `CodeFile` | `path` | `String` | no | Workspace-relative file path. |
| | `content` | `String` | no | File contents (no placeholder stubs). |
| `CodeArtifact` | `layer` | `String` | no | `"backend"` or `"frontend"`. |
| | `files` | `List<CodeFile>` | no | One to three files. |
| | `summary` | `String` | no | One-sentence description of the work. |
| `ReviewFinding` | `file` | `String` | no | Path of the file under review. |
| | `severity` | `String` | no | One of: `info`, `warning`, `blocking`. |
| | `comment` | `String` | no | One-sentence description of the finding. |
| `ReviewFindings` | `findings` | `List<ReviewFinding>` | no | All findings; may be empty. |
| | `approvedForTesting` | `boolean` | no | `true` when no blocking findings. |
| `SecurityFinding` | `category` | `String` | no | One of: `injection`, `auth`, `data-exposure`, `dependency`, `config`, `logic`, `other`. |
| | `severity` | `String` | no | One of: `low`, `medium`, `high`, `critical`. |
| | `description` | `String` | no | Risk and recommended fix. |
| `SecurityFindings` | `findings` | `List<SecurityFinding>` | no | All findings; may be empty. |
| | `clearToMerge` | `boolean` | no | `true` when no `critical` or `high` findings. |
| `TestCase` | `name` | `String` | no | Given/when/then test name. |
| | `passed` | `boolean` | no | Whether the case passes. |
| | `failureReason` | `String` | no | Empty string on pass; reason on failure. |
| `TestResults` | `cases` | `List<TestCase>` | no | Three to eight cases. |
| | `allPassed` | `boolean` | no | `true` when every case passed. |
| `DocArtifact` | `overview` | `String` | no | Two to four sentence overview. |
| | `apiSections` | `List<String>` | no | One entry per public API surface. |
| | `deploymentNotes` | `String` | no | Configuration and deployment instructions. |
| `IntegrationPlan` | `approach` | `String` | no | One-sentence integration strategy. |
| | `steps` | `List<String>` | no | Four to eight ordered integration steps. |
| | `rollbackSteps` | `List<String>` | no | Corresponding rollback steps in reverse order. |
| `SignoffRequest` | `ticketId` | `String` | no | The ticket awaiting signoff. |
| | `requestedBy` | `String` | no | Who triggered the signoff request. |
| | `requestedAt` | `Instant` | no | When the request was created. |
| `SignoffDecision` | `ticketId` | `String` | no | The ticket this decision concerns. |
| | `by` | `String` | no | Lead engineer id. |
| | `approved` | `boolean` | no | Whether the merge is approved. |
| | `notes` | `String` | no | Optional review notes (empty string when absent). |
| | `decidedAt` | `Instant` | no | When the decision was recorded. |

## Entity state — `Blackboard` (`BlackboardEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `ticketId` | `String` | no | Unique ticket id. |
| `title` | `String` | no | Ticket name. |
| `description` | `String` | no | Original brief. |
| `stage` | `BlackboardStage` | no | See enum. |
| `taskBreakdown` | `Optional<TaskBreakdown>` | yes | Written in `PLANNED` stage. |
| `archDecision` | `Optional<ArchDecision>` | yes | Written in `ARCHITECTED` stage. |
| `backendArtifact` | `Optional<CodeArtifact>` | yes | Written when backend code lands. |
| `frontendArtifact` | `Optional<CodeArtifact>` | yes | Written when frontend code lands. |
| `reviewFindings` | `Optional<ReviewFindings>` | yes | Written in `REVIEWED` stage. |
| `securityFindings` | `Optional<SecurityFindings>` | yes | Written in `REVIEWED` stage. |
| `testResults` | `Optional<TestResults>` | yes | Written in `VERIFIED` stage. |
| `docArtifact` | `Optional<DocArtifact>` | yes | Written in `VERIFIED` stage. |
| `integrationPlan` | `Optional<IntegrationPlan>` | yes | Written in `INTEGRATION_PLANNED` stage. |
| `signoffDecision` | `Optional<SignoffDecision>` | yes | Populated on `SignoffApproved` or `SignoffRejected`. |
| `stuckAlert` | `Optional<String>` | yes | Populated by `StuckBoardMonitor`. |
| `createdAt` | `Instant` | no | When `TicketReceived` emitted. |
| `mergedAt` | `Optional<Instant>` | yes | When `SignoffApproved` emitted. |

## Entity state — `Ticket` (`TicketEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `ticketId` | `String` | no | Unique id. |
| `title` | `String` | no | Ticket name. |
| `description` | `String` | no | Original brief. |
| `submittedBy` | `String` | no | Submitter id. |
| `status` | `TicketStatus` | no | See enum. |
| `createdAt` | `Instant` | no | When `TicketCreated` emitted. |
| `completedAt` | `Optional<Instant>` | yes | When the ticket reached `DONE`. |

## Enums

`BlackboardStage`: `INTAKE`, `PLANNED`, `ARCHITECTED`, `CODED`, `REVIEWED`, `VERIFIED`, `INTEGRATION_PLANNED`, `AWAITING_SIGNOFF`, `IN_REVIEW`, `MERGED`.
`TicketStatus`: `OPEN`, `IN_PROGRESS`, `DONE`, `REJECTED`.

## Events — `BlackboardEntity`

| Event | Payload | Transition |
|---|---|---|
| `TicketReceived` | `ticketId, title, description, createdAt` | → INTAKE |
| `PlanWritten` | `taskBreakdown` | INTAKE → PLANNED |
| `ArchWritten` | `archDecision` | PLANNED → ARCHITECTED |
| `BackendCodeWritten` | `backendArtifact` | ARCHITECTED → (pending frontend) |
| `FrontendCodeWritten` | `frontendArtifact` | ARCHITECTED / pending-backend → CODED (when both present) |
| `ReviewWritten` | `reviewFindings` | CODED → (pending security) |
| `SecurityWritten` | `securityFindings` | CODED / pending-review → REVIEWED (when both present) |
| `TestsWritten` | `testResults` | REVIEWED → (pending docs) |
| `DocsWritten` | `docArtifact` | REVIEWED / pending-tests → VERIFIED (when both present) |
| `IntegrationPlanWritten` | `integrationPlan` | VERIFIED → INTEGRATION_PLANNED |
| `SignoffRequested` | `signoffRequest` | INTEGRATION_PLANNED → AWAITING_SIGNOFF |
| `SignoffApproved` | `signoffDecision, mergedAt` | AWAITING_SIGNOFF → MERGED |
| `SignoffRejected` | `signoffDecision` | AWAITING_SIGNOFF → IN_REVIEW |
| `BoardStuckAlerted` | `alert, alertedAt` | any non-terminal → (stuckAlert set, stage unchanged) |

## Events — `TicketEntity`

| Event | Payload | Transition |
|---|---|---|
| `TicketCreated` | `ticketId, title, description, submittedBy, createdAt` | → OPEN |
| `TicketStarted` | `startedAt` | OPEN → IN_PROGRESS |
| `TicketCompleted` | `completedAt` | IN_PROGRESS → DONE |
| `TicketRejected` | `rejectedAt` | IN_PROGRESS → REJECTED |

## Events — `SignoffEntity`

| Event | Payload |
|---|---|
| `SignoffRequestCreated` | `SignoffRequest` |
| `SignoffDecisionRecorded` | `SignoffDecision` |

## Events — `TicketQueue`

| Event | Payload |
|---|---|
| `TicketSubmitted` | `ticketId, title, description, submittedBy, submittedAt` |

## View row — `BoardRow`

`BoardRow` mirrors `Blackboard` but drops the heavy `CodeFile` contents — it keeps artifact summaries and file counts only. Every nullable lifecycle field on the row is `Optional<T>` (Lesson 6). The view does not expose any artifact file contents; the UI fetches nothing heavier.
