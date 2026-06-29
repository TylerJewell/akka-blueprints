# Data model — core-semantic-router

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingQuery` | `queryId` | `String` | no | Server-assigned UUID. |
| | `requesterId` | `String` | no | Opaque employee identifier from the inbound channel. |
| | `channel` | `String` | no | `"portal"` / `"slack"` / `"email"`. |
| | `subject` | `String` | no | Raw subject; *not yet sanitized*. |
| | `body` | `String` | no | Raw body. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedQuery` | `redactedSubject` | `String` | no | PII redacted. |
| | `redactedBody` | `String` | no | PII redacted. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["employee-id","ssn","salary","account-id","benefit-plan"]`. |
| `RoutingDecision` | `domain` | `QueryDomain` | no | `HR` / `FINANCE` / `AMBIGUOUS`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `GuardrailVerdict` | `authorized` | `boolean` | no | `true` to proceed, `false` to block. |
| | `rejections` | `List<String>` | no | Empty when `authorized=true`. Rubric-token list otherwise. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `QueryAnswer` | `answerBody` | `String` | no | 2–4 paragraphs in policy terms. |
| | `action` | `AnswerAction` | no | `POLICY_CITED` / `PROCESS_EXPLAINED` / `REFERRED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"hr"` / `"finance"`. |
| | `answeredAt` | `Instant` | no | When the specialist finished. |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `incoming` | `IncomingQuery` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedQuery>` | yes | Populated after `QuerySanitized`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `IntentClassified`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `answer` | `Optional<QueryAnswer>` | yes | Populated after `QueryAnswered`. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `QueryEscalated`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QueryRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Query` record is reused as the `QueryView` row type.

## Enums

`QueryDomain`: `HR`, `FINANCE`, `AMBIGUOUS`.

`AnswerAction`: `POLICY_CITED`, `PROCESS_EXPLAINED`, `REFERRED`, `ESCALATED`.

`QueryStatus`: `RECEIVED`, `SANITIZED`, `CLASSIFIED`, `AUTHORIZED`, `ROUTED_HR`, `ROUTED_FINANCE`, `ANSWERED`, `BLOCKED`, `ESCALATED`.

## Events (`QueryEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `QueryRegistered` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `QuerySanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `IntentClassified` | `routing` | `RouterWorkflow.classifyStep` | → `CLASSIFIED` |
| `GuardrailVerdictAttached` | `guardrail` | `RouterWorkflow.guardrailStep` | (no status change) |
| `IntentRouted` | `domain` | `RouterWorkflow.guardrailStep` (authorized) | → `AUTHORIZED` → `ROUTED_HR` or `ROUTED_FINANCE` |
| `QueryAnswered` | `answer` | `RouterWorkflow.hrStep` / `financeStep` | → `ANSWERED` (terminal) |
| `QueryBlocked` | `rejections` | `RouterWorkflow.guardrailStep` (rejected) | → `BLOCKED` (terminal) |
| `QueryEscalated` | `escalationReason` | `RouterWorkflow.escalateStep` (AMBIGUOUS or unrecoverable error) | → `ESCALATED` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RoutingEvalScorer` Consumer (on `IntentRouted`) | (no status change; populates `routingScore`) |

`RoutingScored` is only emitted when `IntentRouted` fires — queries that terminate in `BLOCKED` (guardrail rejected the classification) or `ESCALATED` (domain was `AMBIGUOUS`) before routing is confirmed do not receive a `RoutingScored` event.

## Events (`QueryQueue`)

| Event | Payload |
|---|---|
| `InboundQueryReceived` | `incoming` (the raw, pre-sanitization payload — audit log) |

## View row

`QueryRow` is `Query` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS queries FROM query_view` with no `WHERE domain` or `WHERE status` filter — callers (`RouterEndpoint`) apply those client-side.
