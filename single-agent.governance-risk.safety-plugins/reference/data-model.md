# Data model — safety-plugins

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SafetyRule` | `ruleId` | `String` | no | Stable id supplied by the caller. |
| | `category` | `String` | no | Safety category (e.g., `HATE_SPEECH`, `PROMPT_INJECTION`). |
| | `description` | `String` | no | What constitutes a violation under this rule. |
| | `actionFloor` | `String` | no | Minimum action if the rule fires (`ALLOW`/`REDACT`/`BLOCK`). |
| `ScreeningRequest` | `screeningId` | `String` | no | UUID minted by `SafetyEndpoint`. |
| | `payloadTitle` | `String` | no | User-supplied label. |
| | `rawPayload` | `String` | no | Pre-sanitization payload body. Audit-only. |
| | `payloadDirection` | `String` | no | `"user-to-model"` or `"model-to-user"`. |
| | `rules` | `List<SafetyRule>` | no | Submitted rule list (1–N). |
| | `submittedBy` | `String` | no | Caller identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedPayload` | `redactedPayload` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","ssn","phone","payment-card-number","address","person-name"]`. |
| `CategoryFinding` | `ruleId` | `String` | no | MUST equal a submitted `ruleId`. |
| | `category` | `String` | no | Category from the matched rule. |
| | `action` | `RuleAction` | no | Enum value. |
| | `evidenceQuote` | `String` | no | Short verbatim passage; empty string when action is ALLOW. |
| | `rationale` | `String` | no | One sentence. |
| `SafetyDecision` | `overallAction` | `OverallAction` | no | Enum value. |
| | `summary` | `String` | no | 1–3 sentences. |
| | `findings` | `List<CategoryFinding>` | no | One entry per submitted rule. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `DecisionEvaluator` finished. |
| `Screening` (entity state) | `screeningId` | `String` | no | — |
| | `request` | `Optional<ScreeningRequest>` | yes | Populated after `PayloadSubmitted`. |
| | `sanitized` | `Optional<SanitizedPayload>` | yes | Populated after `PayloadSanitized`. |
| | `decision` | `Optional<SafetyDecision>` | yes | Populated after `DecisionRecorded`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `ScreeningStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PayloadSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Screening` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`RuleAction`: `ALLOW`, `REDACT`, `BLOCK`.
`OverallAction`: `ALLOW`, `REDACT`, `BLOCK`.
`ScreeningStatus`: `SUBMITTED`, `SANITIZED`, `SCREENING`, `DECISION_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`SafetyEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PayloadSubmitted` | `request` | → SUBMITTED |
| `PayloadSanitized` | `sanitized` | → SANITIZED |
| `ScreeningStarted` | — | → SCREENING |
| `DecisionRecorded` | `decision` | → DECISION_RECORDED |
| `EvaluationScored` | `eval` | → EVALUATED (terminal happy) |
| `ScreeningFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Screening.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ScreeningRow` mirrors `Screening` minus `request.rawPayload` (the audit log keeps that). The UI fetches the raw payload on demand via `GET /api/screenings/{id}` and reads `request.rawPayload` from the JSON.

The view declares ONE query: `getAllScreenings: SELECT * AS screenings FROM safety_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SafetyTasks.java`)

```java
public final class SafetyTasks {
  public static final Task<SafetyDecision> SCREEN_PAYLOAD = Task
      .name("Screen payload")
      .description("Read the attached sanitized payload and produce a SafetyDecision per submitted safety rule")
      .resultConformsTo(SafetyDecision.class);

  private SafetyTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Safety categories (reference)

The following `category` values appear in seeded safety profiles. Deployers may add custom values.

| Category | Default action floor | Description |
|---|---|---|
| `HATE_SPEECH` | BLOCK | Language targeting groups by protected characteristics. |
| `SELF_HARM` | BLOCK | Content encouraging or describing self-harm. |
| `SEXUAL_CONTENT` | BLOCK | Explicit sexual content outside adult platforms. |
| `VIOLENCE` | REDACT | Graphic depictions of violence. |
| `PROMPT_INJECTION` | BLOCK | Directives attempting to override system instructions. |
| `DATA_EXFILTRATION` | REDACT | Requests to transmit credentials, PII, or sensitive data. |
| `HARASSMENT` | REDACT | Targeted abuse of an individual. |
| `ILLEGAL_GOODS` | BLOCK | Facilitation of purchase or distribution of illegal items. |
| `MISINFORMATION` | REDACT | Demonstrably false claims presented as fact. |
| `OTHER` | ALLOW | Catch-all for platform-specific custom rules. |
