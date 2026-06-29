# Data model — employee-helpdesk-router

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingQuestion` | `questionId` | `String` | no | Server-assigned UUID. |
| | `employeeId` | `String` | no | Opaque employee identifier from the inbound channel; may be `EMP-\d+` formatted. |
| | `channel` | `String` | no | `"portal"` / `"chat"` / `"email"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedQuestion` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["employee-id","email","phone"]`. |
| `RoutingDecision` | `topic` | `QuestionTopic` | no | `HR` / `IT` / `POLICY` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `Answer` | `responseSubject` | `String` | no | `"Re: …"`. |
| | `responseBody` | `String` | no | 3–5 paragraphs. |
| | `action` | `AnswerAction` | no | `INFO_PROVIDED` / `POLICY_CITED` / `TICKET_OPENED` / `FOLLOW_UP_SCHEDULED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"hr"` / `"it"` / `"policy"`. |
| | `answeredAt` | `Instant` | no | When the specialist finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `RouteScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Question` (entity state) | `questionId` | `String` | no | — |
| | `incoming` | `IncomingQuestion` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedQuestion>` | yes | Populated after `QuestionSanitized`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `QuestionRouted`. |
| | `answer` | `Optional<Answer>` | yes | Populated after `AnswerDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `routeScore` | `Optional<RouteScore>` | yes | Populated after `RouteScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `QuestionEscalated`. |
| | `status` | `QuestionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuestionRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Question` record is reused as the `QuestionView` row type.

## Enums

`QuestionTopic`: `HR`, `IT`, `POLICY`, `UNCLEAR`.

`AnswerAction`: `INFO_PROVIDED`, `POLICY_CITED`, `TICKET_OPENED`, `FOLLOW_UP_SCHEDULED`, `ESCALATED`.

`QuestionStatus`: `RECEIVED`, `SANITIZED`, `ROUTED`, `ROUTED_HR`, `ROUTED_IT`, `ROUTED_POLICY`, `ANSWER_DRAFTED`, `BLOCKED`, `ANSWERED`, `ESCALATED`.

## Events (`QuestionEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `QuestionRegistered` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `QuestionSanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `QuestionRouted` | `routing` | `HelpWorkflow.routeStep` | → `ROUTED` |
| `RoutingBranched` | `topic` | `HelpWorkflow.branchStep` | → `ROUTED_HR` or `ROUTED_IT` or `ROUTED_POLICY` |
| `AnswerDrafted` | `answer` | `HelpWorkflow.hrStep` / `itStep` / `policyStep` | → `ANSWER_DRAFTED` |
| `GuardrailVerdictAttached` | `guardrail` | `HelpWorkflow.guardrailStep` | (no status change) |
| `AnswerPublished` | — | `HelpWorkflow.publishStep` (guardrail allowed) | → `ANSWERED` (terminal) |
| `AnswerBlocked` | `violations` | `HelpWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `QuestionEscalated` | `escalationReason` | `HelpWorkflow.escalateStep` (topic UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |
| `RouteScored` | `score, rationale, scoredAt` | `RouteEvalScorer` Consumer | (no status change; populates `routeScore`) |

A `BLOCKED → ANSWERED` transition is emitted via the `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `AnswerPublished` in the same command.

## Events (`QuestionQueue`)

| Event | Payload |
|---|---|
| `InboundQuestionReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`QuestionRow` is `Question` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS questions FROM question_view` with no `WHERE topic` or `WHERE status` filter — callers (`HelpdeskEndpoint`) apply those client-side.
