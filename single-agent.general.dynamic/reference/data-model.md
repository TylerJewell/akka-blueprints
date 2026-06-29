# Data model — Dynamic Route Agent

Authoritative field list. `/akka:implement` writes these records exactly. Every nullable lifecycle field is `Optional<T>` (Lesson 6); Akka's Jackson config serializes `Optional<T>` as the raw value or `null`.

## Enums

`Route`:

| Value | Meaning |
|---|---|
| `SUMMARIZE` | Condense the input into a 2-3 sentence summary. |
| `TRANSLATE` | Translate the input into the named target language. |

`RequestStatus`:

| Value | Meaning |
|---|---|
| `SUBMITTED` | Recorded; passed G1; agent run pending or in flight. |
| `COMPLETED` | Agent output passed G2 and is the result. |
| `BLOCKED` | Rejected by G1 (before run) or G2 (after run). |

## `RequestRecord` (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Request UUID. |
| `route` | `Route` | no | Selected route. |
| `inputText` | `String` | no | User-supplied input. |
| `status` | `RequestStatus` | no | Lifecycle state. |
| `submittedAt` | `Instant` | no | Submit time. |
| `targetLanguage` | `Optional<String>` | yes | Set for TRANSLATE only. |
| `output` | `Optional<String>` | yes | Agent result; set on COMPLETED. |
| `completedAt` | `Optional<Instant>` | yes | Set on COMPLETED. |
| `blockedReason` | `Optional<String>` | yes | Set on BLOCKED. |
| `blockedAt` | `Optional<Instant>` | yes | Set on BLOCKED. |

`emptyState()` returns `RequestRecord.initial("")` with placeholder values and `Optional.empty()` lifecycle fields — no `commandContext()` reference (Lesson 3).

## Agent records

| Record | Fields | Meaning |
|---|---|---|
| `AgentRequest` | `Route route, String text, Optional<String> targetLanguage` | The per-request input passed to `DynamicAgent.run`. |
| `AgentResult` | `String output` | The typed agent return. |
| `SubmitRequest` | `String route, String text, String targetLanguage` | The inbound JSON `POST /api/requests` body. |

## Events

| Event | Trigger | Fields |
|---|---|---|
| `RequestSubmitted` | Submit passes G1 | `id, route, inputText, targetLanguage, submittedAt` |
| `RequestCompleted` | Agent output passes G2 | `id, output, completedAt` |
| `RequestBlocked` | G1 or G2 rejects | `id, blockedReason, blockedAt` |

## View

`RequestsView` row type is `RequestRecord`. ONE query: `getAllRequests` -> `SELECT * AS requests FROM requests_view`. No `WHERE status` filter — Akka cannot auto-index enum columns (Lesson 2); callers filter client-side.
