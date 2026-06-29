# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## MortgageApplication (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `applicationId` | `String` | no | UUID; also the workflow id |
| `applicantRef` | `String` | no | Opaque reference; PII sanitiser holds the mapping to real identity |
| `loanProduct` | `String` | no | Product code, e.g. `"30-year-fixed"` |
| `requestedAmountGBP` | `int` | no | Loan amount in GBP |
| `status` | `ApplicationStatus` | no | Lifecycle state |
| `packet` | `Optional<RecommendationPacket>` | yes | Supervisor's merged assessment; null until `PacketAssembled` |
| `complianceHoldReason` | `Optional<String>` | yes | Reason text; set on `ComplianceHoldApplied` |
| `decisionBy` | `Optional<String>` | yes | Underwriter identifier; set on `DecisionRecorded` |
| `decisionNotes` | `Optional<String>` | yes | Underwriter notes; set on `DecisionRecorded` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `DecisionEvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `receivedAt` | `Instant` | no | Application receipt time |
| `decidedAt` | `Optional<Instant>` | yes | Set on `DecisionRecorded` |

The `ApplicationView` row type (`ApplicationRow`) mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record ApplicationRequest(
    String applicantRef,
    String loanProduct,
    int    requestedAmountGBP,
    int    propertyValueGBP,
    String employmentStatus,
    int    annualIncomeGBP,
    int    creditScore,
    List<String> documentIds
) {}

record WorkPlan(
    String eligibilityQuery,
    String documentationQuery,
    String underwritingQuery
) {}

record EligibilityAssessment(
    boolean eligible,
    double  ltv,
    double  dti,
    String  verdict,
    Instant assessedAt
) {}

record DocumentationAssessment(
    boolean complete,
    List<String> missingDocuments,
    List<String> authenticityFlags,
    Instant reviewedAt
) {}

record TermStructure(
    double annualRatePercent,
    int    amortisationMonths,
    String conditions,
    Instant proposedAt
) {}

record UnderwritingAssessment(
    String       riskBand,
    TermStructure proposedTerms,
    String        rationale,
    Instant       assessedAt
) {}

record NotificationDraft(
    String subject,
    String bodyText,
    String channel,
    Instant draftedAt
) {}

record RecommendationPacket(
    EligibilityAssessment   eligibilityVerdict,
    DocumentationAssessment documentationVerdict,
    UnderwritingAssessment  underwritingAssessment,
    String                  complianceStatus,
    String                  sanitisedSummary,
    Instant                 packetAt
) {}
```

## Status enum

```java
enum ApplicationStatus {
    RECEIVED, UNDER_REVIEW, PENDING_SIGNOFF,
    APPROVED, DECLINED, COMPLIANCE_HOLD, REVIEW_REQUIRED
}
```

## Events

### ApplicationEntity

| Event | Trigger |
|---|---|
| `ApplicationReceived` | Workflow creates the application (`receiveApplication`) |
| `ReviewStarted` | Workers have been dispatched (`startReview`) |
| `PacketAssembled` | Supervisor returns a `RecommendationPacket` (`assemblePacket`) |
| `ComplianceHoldApplied` | complianceStep detects a product-rule violation |
| `SignOffRequested` | Compliance passed; workflow pauses for the human gate |
| `DecisionRecorded` | Human underwriter calls `POST /api/applications/{id}/decision` |
| `WorkerTimedOut` | Any of the three parallel workers exceeded its 90s step timeout |
| `DecisionEvalScored` | `DecisionEvalSampler` recorded a 1–5 score |

### ApplicationQueue

| Event | Trigger |
|---|---|
| `ApplicationSubmitted` | `enqueueApplication(...)` from endpoint or simulator |

Fields: `{ applicationId, applicantRef, loanProduct, requestedAmountGBP, submittedAt }`.
