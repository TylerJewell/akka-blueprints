# Data model — conditional-recruiting-router

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundEmail` | `applicationId` | `String` | no | Server-assigned UUID. |
| | `candidateId` | `String` | no | Opaque candidate identifier from the inbound channel. |
| | `channel` | `String` | no | `"email"` / `"careers-portal"` / `"recruiter-forward"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the email arrived. |
| `SanitizedEmail` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","address"]`. |
| `RoutingDecision` | `route` | `ApplicationRoute` | no | `INFO_REQUEST` / `INTERVIEW_REQUEST` / `UNROUTABLE`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `OutboundReply` | `replySubject` | `String` | no | `"Re: …"`. |
| | `replyBody` | `String` | no | 2–4 paragraphs. |
| | `action` | `ReplyAction` | no | `QUESTION_ANSWERED` / `ARTICLE_LINKED` / `FOLLOW_UP_PROMISED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"info-requester"`. |
| | `repliedAt` | `Instant` | no | When the specialist finished. |
| `CalendarConfirmation` | `interviewFormat` | `String` | no | `"phone-screen"` / `"technical"` / `"panel"` / `"executive"`. |
| | `interviewerId` | `String` | no | Interviewer pool ID. |
| | `proposedSlot` | `String` | no | ISO-8601 datetime. |
| | `outcome` | `ScheduleOutcome` | no | `SLOT_BOOKED` / `CANDIDATE_DEFERRED` / `ESCALATED_TO_RECRUITER`. |
| | `specialistTag` | `String` | no | `"interview-organizer"`. |
| | `confirmedAt` | `Instant` | no | When the specialist finished. |
| `ToolCallVerdict` | `allowed` | `boolean` | no | `true` to proceed, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Application` (entity state) | `applicationId` | `String` | no | — |
| | `incoming` | `InboundEmail` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedEmail>` | yes | Populated after `ApplicationSanitized`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `RoutingDecided`. |
| | `reply` | `Optional<OutboundReply>` | yes | Populated after `ReplyDrafted`. |
| | `calendar` | `Optional<CalendarConfirmation>` | yes | Populated after `SlotProposed`. |
| | `toolGuardrail` | `Optional<ToolCallVerdict>` | yes | Populated after `ToolCallBlocked` or successful tool run. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `closureReason` | `Optional<String>` | yes | Populated after `ApplicationClosed`. |
| | `status` | `ApplicationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ApplicationRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Application` record is reused as the `ApplicationView` row type.

## Enums

`ApplicationRoute`: `INFO_REQUEST`, `INTERVIEW_REQUEST`, `UNROUTABLE`.

`ReplyAction`: `QUESTION_ANSWERED`, `ARTICLE_LINKED`, `FOLLOW_UP_PROMISED`, `ESCALATED`.

`ScheduleOutcome`: `SLOT_BOOKED`, `CANDIDATE_DEFERRED`, `ESCALATED_TO_RECRUITER`.

`ApplicationStatus`: `RECEIVED`, `SANITIZED`, `CLASSIFIED`, `ROUTED_INFO`, `ROUTED_INTERVIEW`, `REPLY_DRAFTED`, `SLOT_PROPOSED`, `TOOL_BLOCKED`, `COMPLETED`, `UNROUTABLE_CLOSED`.

## Events (`ApplicationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `ApplicationRegistered` | `incoming` | `CandidateSanitizer.registerIncoming` | → `RECEIVED` |
| `ApplicationSanitized` | `sanitized` | `CandidateSanitizer.attachSanitized` | → `SANITIZED` |
| `RoutingDecided` | `routing` | `RecruitingWorkflow.classifyStep` | → `CLASSIFIED` |
| `ApplicationRouted` | `route` | `RecruitingWorkflow.routeStep` | → `ROUTED_INFO` or `ROUTED_INTERVIEW` |
| `ReplyDrafted` | `reply` | `RecruitingWorkflow.infoStep` | → `REPLY_DRAFTED` |
| `SlotProposed` | `calendar` | `RecruitingWorkflow.scheduleStep` | → `SLOT_PROPOSED` |
| `ToolCallBlocked` | `toolGuardrail` | `RecruitingWorkflow.toolGuardrailStep` | → `TOOL_BLOCKED` (terminal until unblock) |
| `ApplicationCompleted` | `finishedAt` | `RecruitingWorkflow.completeStep` | → `COMPLETED` (terminal) |
| `ApplicationClosed` | `closureReason` | `RecruitingWorkflow.closeStep` (route UNROUTABLE or error) | → `UNROUTABLE_CLOSED` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RoutingEvalScorer` Consumer | (no status change; populates `routingScore`) |

A `TOOL_BLOCKED → COMPLETED` transition is emitted via `unblock` command, which writes `ApplicationCompleted` with an override marker preserved in the `toolGuardrail.rubricVersion` field.

## Events (`InboxQueue`)

| Event | Payload |
|---|---|
| `InboundEmailReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`ApplicationRow` is `Application` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without error. The view's single query is `SELECT * AS applications FROM application_view` with no `WHERE route` or `WHERE status` filter — callers (`RecruitingEndpoint`) apply those client-side.
