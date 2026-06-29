# Data model — hrsd-inquiry

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PolicyRef` | `policyId` | `String` | no | Policy identifier (e.g. `"BEN-101"`). |
| | `title` | `String` | no | Policy title from the catalog. |
| | `section` | `String` | no | Specific section cited; `""` if whole-policy reference. |
| `InquiryRequest` | `inquiryId` | `String` | no | UUID minted by `InquiryEndpoint`. |
| | `employeeId` | `String` | no | Employee identifier. |
| | `topic` | `HrTopic` | no | Enum value. |
| | `rawMessage` | `String` | no | Pre-screening message body. Audit-only. |
| | `submitRequestIfApplicable` | `boolean` | no | Employee opt-in for service request submission. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ScreenedMessage` | `redactedMessage` | `String` | no | Special-category data redacted; this is what the agent sees. |
| | `specialCategoriesFound` | `List<String>` | no | e.g. `["health-condition","union-membership"]`. |
| `HrServiceRequest` | `requestType` | `String` | no | One of `LEAVE_REQUEST`, `BENEFITS_UPDATE`, `PAYROLL_CORRECTION`, `ONBOARDING_TASK`, `GENERAL_REQUEST`. |
| | `description` | `String` | no | What the employee needs. |
| | `fields` | `Map<String, String>` | no | Request-type-specific key-value pairs. |
| `InquiryResponse` | `answer` | `String` | no | 1–4-sentence policy-grounded answer. |
| | `citedPolicies` | `List<PolicyRef>` | no | Non-empty; every policy drawn on. |
| | `serviceRequest` | `HrServiceRequest` | yes | Present only for actionable inquiries. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `Inquiry` (entity state) | `inquiryId` | `String` | no | — |
| | `request` | `Optional<InquiryRequest>` | yes | Populated after `InquirySubmitted`. |
| | `screened` | `Optional<ScreenedMessage>` | yes | Populated after `InquiryScreened`. |
| | `response` | `Optional<InquiryResponse>` | yes | Populated after `InquiryAnswered`. |
| | `serviceRequestRef` | `Optional<String>` | yes | Populated after `ServiceRequestSubmitted`. |
| | `status` | `InquiryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `InquirySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Inquiry` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`HrTopic`: `BENEFITS`, `LEAVE_AND_ABSENCE`, `ONBOARDING`, `PAYROLL`, `GENERAL_HR`.
`InquiryStatus`: `SUBMITTED`, `SCREENED`, `ANSWERING`, `ANSWERED`, `REQUEST_SUBMITTED`, `FAILED`.

## Events (`InquiryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `InquirySubmitted` | `request` | → SUBMITTED |
| `InquiryScreened` | `screened` | → SCREENED |
| `AnswerStarted` | — | → ANSWERING |
| `InquiryAnswered` | `response` | → ANSWERED |
| `ServiceRequestSubmitted` | `ref: String` | → REQUEST_SUBMITTED (terminal happy with request) |
| `InquiryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Inquiry.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

Note: `ANSWERED` is also a terminal happy state for inquiries where no service request was produced or the employee did not opt in.

## View row

`InquiryRow` mirrors `Inquiry` minus `request.rawMessage` (the audit log keeps that). The UI fetches the raw message on demand via `GET /api/inquiries/{id}` and reads `request.rawMessage` from the JSON.

The view declares ONE query: `getAllInquiries: SELECT * AS inquiries FROM inquiry_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`InquiryTasks.java`)

```java
public final class InquiryTasks {
  public static final Task<InquiryResponse> ANSWER_INQUIRY = Task
      .name("Answer HR inquiry")
      .description("Read the attached policy catalog and answer the employee inquiry with citations")
      .resultConformsTo(InquiryResponse.class);

  private InquiryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
