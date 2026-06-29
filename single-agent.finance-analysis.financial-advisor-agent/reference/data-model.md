# Data model — financial-advisor-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Holding` | `ticker` | `String` | no | Stock or fund ticker symbol. |
| | `assetClass` | `String` | no | e.g. `"equities"`, `"ETF"`, `"fixed-income"`. |
| | `currentWeight` | `double` | no | Fraction of portfolio (0.0–1.0). |
| | `marketValue` | `double` | no | Market value in the account currency. |
| `ClientProfile` | `clientId` | `String` | no | Client identifier (may contain regulated tokens). |
| | `accountType` | `AccountType` | no | `BROKERAGE`, `IRA`, or `ROTH_IRA`. |
| | `riskTolerance` | `RiskTolerance` | no | `CONSERVATIVE`, `MODERATE`, or `AGGRESSIVE`. |
| | `holdings` | `List<Holding>` | no | Current positions (1–N). |
| | `submittedBy` | `String` | no | Advisor or client user identifier. |
| `SanitizedProfile` | `redactedProfileJson` | `String` | no | Serialized ClientProfile with regulated identifiers replaced by redaction tokens. |
| | `identifierCategoriesFound` | `List<String>` | no | e.g. `["account-id","ssn","ein"]`. |
| `HoldingAdvice` | `ticker` | `String` | no | MUST match a ticker in the submitted portfolio. |
| | `assetClass` | `String` | no | MUST be within the authorized set for the `accountType`. |
| | `currentWeight` | `double` | no | Copied from the portfolio; unchanged. |
| | `suggestedWeight` | `double` | no | Agent's recommendation; in `[0.0, 1.0]`. |
| | `rationale` | `String` | no | 1–2 sentence directional explanation. |
| `AdvisoryResponse` | `riskRating` | `RiskRating` | no | Enum value. |
| | `recommendation` | `String` | no | 2–4 sentence top-level recommendation. |
| | `holdingAdvice` | `List<HoldingAdvice>` | no | One entry per holding in the submitted portfolio. |
| | `advisedAt` | `Instant` | no | When the agent returned. |
| `AdvisoryRequest` | `advisoryId` | `String` | no | UUID minted by `AdvisoryEndpoint`. |
| | `profile` | `ClientProfile` | no | Raw (pre-sanitization) profile. Audit-only field. |
| | `question` | `String` | no | Client's natural-language investment question. |
| | `requestedAt` | `Instant` | no | When the endpoint received the request. |
| `Advisory` (entity state) | `advisoryId` | `String` | no | — |
| | `request` | `Optional<AdvisoryRequest>` | yes | Populated after `AdvisoryRequested`. |
| | `sanitized` | `Optional<SanitizedProfile>` | yes | Populated after `ClientDataSanitized`. |
| | `response` | `Optional<AdvisoryResponse>` | yes | Populated after `ResponseRecorded`. |
| | `status` | `AdvisoryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `AdvisoryRequested` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Advisory` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AccountType`: `BROKERAGE`, `IRA`, `ROTH_IRA`.
`RiskTolerance`: `CONSERVATIVE`, `MODERATE`, `AGGRESSIVE`.
`RiskRating`: `CONSERVATIVE`, `MODERATE`, `AGGRESSIVE`.
`AdvisoryStatus`: `REQUESTED`, `SANITIZED`, `ADVISING`, `RESPONSE_RECORDED`, `AUDITED`, `FAILED`.

## Events (`AdvisoryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `AdvisoryRequested` | `request` | → REQUESTED |
| `ClientDataSanitized` | `sanitized` | → SANITIZED |
| `AdvisingStarted` | — | → ADVISING |
| `ResponseRecorded` | `response` | → RESPONSE_RECORDED |
| `AuditLogged` | `digestHex: String` | → AUDITED (terminal happy) |
| `AdvisoryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Advisory.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REQUESTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AdvisoryRow` mirrors `Advisory` minus `request.profile.clientId` and the raw holding market values — the entity retains those for compliance audit. The UI fetches the raw profile on demand via `GET /api/advisories/{id}` and reads `request.profile` from the JSON.

The view declares ONE query: `getAllAdvisories: SELECT * AS advisories FROM advisory_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AdvisoryTasks.java`)

```java
public final class AdvisoryTasks {
  public static final Task<AdvisoryResponse> ADVISE_CLIENT = Task
      .name("Advise client")
      .description("Read the attached sanitized portfolio and answer the client question with a structured AdvisoryResponse")
      .resultConformsTo(AdvisoryResponse.class);

  private AdvisoryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
