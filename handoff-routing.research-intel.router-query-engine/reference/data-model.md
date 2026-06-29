# Data model — router-query-engine

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ResearchQuestion` | `queryId` | `String` | no | Server-assigned uuid. |
| | `questionText` | `String` | no | The raw research question text. |
| | `source` | `String` | no | `"simulator"` / `"manual"`. |
| | `receivedAt` | `Instant` | no | When the question was received. |
| `RoutingDecision` | `engineType` | `EngineType` | no | `STRUCTURED` / `SEMANTIC` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `Answer` | `answerText` | `String` | no | The full answer text. |
| | `action` | `AnswerAction` | no | `DIRECT_LOOKUP` / `AGGREGATION_RESULT` / `CONCEPT_SUMMARY` / `SYNTHESIS` / `ESCALATED`. |
| | `engineTag` | `String` | no | `"structured"` / `"semantic"`. |
| | `sourceRefs` | `List<String>` | no | Table names or document ids cited. Empty list when `action=ESCALATED`. |
| | `answeredAt` | `Instant` | no | When the engine finished. |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Query` (entity state) | `queryId` | `String` | no | — |
| | `question` | `ResearchQuestion` | no | Captured once at create. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `RoutingDecided`. |
| | `answer` | `Optional<Answer>` | yes | Populated after `AnswerDrafted`. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `QueryEscalated`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QueryRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Query` record is reused as the `QueryView` row type.

## Enums

`EngineType`: `STRUCTURED`, `SEMANTIC`, `UNCLEAR`.

`AnswerAction`: `DIRECT_LOOKUP`, `AGGREGATION_RESULT`, `CONCEPT_SUMMARY`, `SYNTHESIS`, `ESCALATED`.

`QueryStatus`: `RECEIVED`, `ROUTING`, `ROUTED_STRUCTURED`, `ROUTED_SEMANTIC`, `ANSWERING`, `ANSWERED`, `ESCALATED`.

## Events (`QueryEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `QueryRegistered` | `question` | `QuestionConsumer.registerQuestion` | → `RECEIVED` |
| `RoutingDecided` | `routing` | `QueryWorkflow.routeStep` | → `ROUTING` |
| `QueryRouted` | `engineType` | `QueryWorkflow.branchStep` | → `ROUTED_STRUCTURED` or `ROUTED_SEMANTIC` |
| `AnswerDrafted` | `answer` | `QueryWorkflow.structuredStep` / `semanticStep` | → `ANSWERING` |
| `AnswerPublished` | — | `QueryWorkflow.publishStep` | → `ANSWERED` (terminal) |
| `QueryEscalated` | `escalationReason` | `QueryWorkflow.escalateStep` (engineType UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RoutingEvalScorer` Consumer | (no status change; populates `routingScore`) |

## Events (`QuestionQueue`)

| Event | Payload |
|---|---|
| `InboundQuestionReceived` | `question` (the raw question — used as the audit log) |

## View row

`QueryRow` is `Query` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without an `AK-00111` error. The view's single query is `SELECT * AS queries FROM query_view` with no `WHERE engineType` or `WHERE status` filter — callers (`QueryEndpoint`) apply those client-side.
