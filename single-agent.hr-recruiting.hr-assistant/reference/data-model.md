# Data model — hr-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PolicyCitation` | `policyId` | `String` | no | Stable id from the seeded policy corpus. MUST match an entry in `policies.jsonl`. |
| | `sectionReference` | `String` | no | Human-readable section label, e.g. `"Section 3.2"`, `"§4"`. |
| | `quotedPassage` | `String` | no | Verbatim or near-verbatim excerpt from the policy section. |
| `PolicyAnswer` | `applicabilityVerdict` | `ApplicabilityVerdict` | no | Enum value. |
| | `answerText` | `String` | no | Direct answer, 2–5 sentences. Non-empty enforced by G1 guardrail. |
| | `citations` | `List<PolicyCitation>` | no | One entry per supporting policy section. May be empty for NOT_APPLICABLE verdicts. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `EmployeeProfile` | `employeeId` | `String` | no | Stable identifier, e.g. `"EM-001"`. |
| | `department` | `String` | no | Organisational department. |
| | `jobTitle` | `String` | no | Job title. |
| | `hireDate` | `Instant` | no | Date employment began. |
| | `workLocation` | `String` | no | Office or remote location string. |
| `SanitizedQueryContext` | `redactedQueryText` | `String` | no | PII- and special-category-redacted query. This is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","employee-id","phone"]`. Empty list if none found. |
| | `specialCategoriesFound` | `List<String>` | no | e.g. `["health-condition","disability-status"]`. Empty list if none found. |
| | `employeeProfile` | `Optional<EmployeeProfile>` | yes | Present only when `employeeId` was supplied and resolved. |
| `QueryRequest` | `queryId` | `String` | no | UUID minted by `QueryEndpoint`. |
| | `rawQueryText` | `String` | no | Pre-sanitization query body. Audit-only; never shown in the UI. |
| | `employeeId` | `Optional<String>` | yes | Supplied or absent. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `HrQuery` (entity state) | `queryId` | `String` | no | — |
| | `request` | `Optional<QueryRequest>` | yes | Populated after `QuerySubmitted`. |
| | `sanitized` | `Optional<SanitizedQueryContext>` | yes | Populated after `QuerySanitized`. |
| | `answer` | `Optional<PolicyAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `status` | `QueryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `QuerySubmitted` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `HrQuery` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ApplicabilityVerdict`: `APPLICABLE`, `NOT_APPLICABLE`, `PARTIALLY_APPLICABLE`.

`QueryStatus`: `SUBMITTED`, `SANITIZED`, `ANSWERING`, `ANSWER_RECORDED`, `FAILED`.

## Events (`QueryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `QuerySubmitted` | `request` | → SUBMITTED |
| `QuerySanitized` | `sanitized` | → SANITIZED |
| `AnsweringStarted` | — | → ANSWERING |
| `AnswerRecorded` | `answer` | → ANSWER_RECORDED (terminal happy) |
| `QueryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `HrQuery.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`QueryRow` mirrors `HrQuery` minus `request.rawQueryText` (the audit log keeps that). The UI fetches the raw query on demand via `GET /api/queries/{id}` and reads `request.rawQueryText` from the JSON.

The view declares ONE query: `getAllQueries: SELECT * AS queries FROM query_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`HrQueryTasks.java`)

```java
public final class HrQueryTasks {
  public static final Task<PolicyAnswer> ANSWER_HR_QUERY = Task
      .name("Answer HR query")
      .description("Read the attached policy corpus and employee profile context and produce a PolicyAnswer")
      .resultConformsTo(PolicyAnswer.class);

  private HrQueryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
