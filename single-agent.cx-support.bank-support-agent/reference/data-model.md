# Data model — bank-support-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CustomerEnquiry` | `enquiryId` | `String` | no | UUID minted by `EnquiryEndpoint`. |
| | `customerId` | `String` | no | Customer identifier supplied by caller. |
| | `enquiryText` | `String` | no | Free-text question or concern. |
| | `category` | `EnquiryCategory` | no | Enum value. |
| | `submittedBy` | `String` | no | Support agent identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `AccountSummary` | `customerId` | `String` | no | Matches the requesting `customerId`. |
| | `maskedAccountNumber` | `String` | no | e.g. `"****4821"`. |
| | `accountType` | `String` | no | `"CHECKING"`, `"SAVINGS"`, or `"CREDIT"`. |
| | `availableBalance` | `BigDecimal` | no | Current available balance. |
| | `currencyCode` | `String` | no | ISO-4217 code, e.g. `"GBP"`. |
| | `cardActive` | `boolean` | no | Whether the card is currently active. |
| `SupportResponse` | `answer` | `String` | no | 2–5-sentence response to the customer. |
| | `riskScore` | `int` | no | 1–10; 10 = highest risk. |
| | `blockCard` | `boolean` | no | True only when `riskScore >= 7`. |
| | `tone` | `ResponseTone` | no | Enum value. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `SanitizedLog` | `redactedAnswer` | `String` | no | PII-redacted version of `SupportResponse.answer`. |
| | `piiCategoriesRedacted` | `List<String>` | no | e.g. `["account-number","name","email"]`. |
| `Enquiry` (entity state) | `enquiryId` | `String` | no | — |
| | `enquiry` | `Optional<CustomerEnquiry>` | yes | Populated after `EnquirySubmitted`. |
| | `account` | `Optional<AccountSummary>` | yes | Populated after `AccountLoaded`. |
| | `response` | `Optional<SupportResponse>` | yes | Populated after `ResponseRecorded`. |
| | `sanitizedLog` | `Optional<SanitizedLog>` | yes | Populated after `LogSanitized`. |
| | `status` | `EnquiryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `EnquirySubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Enquiry` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`EnquiryCategory`: `BALANCE_QUERY`, `DISPUTED_TRANSACTION`, `LOST_CARD`, `GENERAL`.
`ResponseTone`: `REASSURING`, `NEUTRAL`, `URGENT`.
`EnquiryStatus`: `SUBMITTED`, `ACCOUNT_LOADED`, `RESPONDING`, `RESPONSE_RECORDED`, `LOGGED`, `FAILED`.

## Events (`EnquiryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `EnquirySubmitted` | `enquiry: CustomerEnquiry` | → SUBMITTED |
| `AccountLoaded` | `account: AccountSummary` | → ACCOUNT_LOADED |
| `RespondingStarted` | — | → RESPONDING |
| `ResponseRecorded` | `response: SupportResponse` | → RESPONSE_RECORDED |
| `LogSanitized` | `sanitizedLog: SanitizedLog` | → LOGGED (terminal happy) |
| `EnquiryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Enquiry.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`EnquiryRow` mirrors `Enquiry` but replaces `response.answer` with `sanitizedLog.redactedAnswer` and `sanitizedLog.piiCategoriesRedacted`. The raw `response.answer` is accessible only via `GET /api/enquiries/{id}` which returns the full `Enquiry` entity state.

The view declares ONE query: `getAllEnquiries: SELECT * AS enquiries FROM enquiry_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`EnquiryTasks.java`)

```java
public final class EnquiryTasks {
  public static final Task<SupportResponse> HANDLE_ENQUIRY = Task
      .name("Handle enquiry")
      .description("Review the customer's enquiry and account data, then produce a SupportResponse with a risk score and card-blocking recommendation")
      .resultConformsTo(SupportResponse.class);

  private EnquiryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## AccountLookupTool operations

| Operation | Parameters | Returns | Side effects |
|---|---|---|---|
| `lookup(customerId)` | `customerId: String` | `AccountSummary` | None — read-only fixture lookup. |
| `blockCardRequest(customerId)` | `customerId: String` | `AccountSummary` (updated, `cardActive=false`) | Records card-block intent in-process; does not call any external system in the sample. |

`ToolCallGuardrail` intercepts calls where `blockCard=true` appears in the tool invocation parameters before `blockCardRequest` executes.
