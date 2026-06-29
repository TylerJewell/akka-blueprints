# Data model — sba-loan-processor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `BusinessProfile` | `businessEin` | `String` | no | Employer Identification Number — stripped by `PiiSanitizer` before agent context; serialised as `"[SANITIZED]"` in API responses. |
| | `businessName` | `String` | no | Legal business name. |
| | `industry` | `String` | no | Industry classification (e.g. `Food & Beverage`). |
| | `yearsInOperation` | `int` | no | Years since business formation. |
| | `legalStructure` | `String` | no | `LLC / Corp / Sole Proprietor / Partnership`. |
| `CreditScore` | `tier` | `String` | no | `EXCELLENT / GOOD / FAIR / POOR`. |
| | `score` | `int` | no | Numeric score (300–850 range). |
| | `fetchedAt` | `Instant` | no | When `IntakeTools.lookupCreditScore` returned. |
| `CreditProfile` | `creditScore` | `CreditScore` | no | — |
| | `businessProfile` | `BusinessProfile` | no | — |
| | `profiledAt` | `Instant` | no | When the INTAKE task returned. |
| `CashFlowSummary` | `annualRevenue` | `BigDecimal` | no | USD. From `UnderwriteTools.analyzeCashFlow`. |
| | `operatingExpenses` | `BigDecimal` | no | USD. |
| | `netOperatingIncome` | `BigDecimal` | no | `annualRevenue - operatingExpenses`. |
| | `analyzedAt` | `Instant` | no | When the calculation completed. |
| `CollateralValue` | `collateralType` | `String` | no | e.g. `commercial-equipment`, `real-estate`. |
| | `estimatedValue` | `BigDecimal` | no | USD. |
| | `ltvRatio` | `BigDecimal` | no | `loanAmount / estimatedValue`. |
| `UnderwritingAnalysis` | `cashFlow` | `CashFlowSummary` | no | — |
| | `collateral` | `CollateralValue` | no | — |
| | `keyRiskFactors` | `List<String>` | no | Possibly empty. |
| | `mitigants` | `List<String>` | no | Possibly empty. |
| | `analyzedAt` | `Instant` | no | When the UNDERWRITE task returned. |
| `DscrResult` | `dscr` | `BigDecimal` | no | `netOperatingIncome / annualDebtService`. `FairLendingScorer` verifies this within ±0.01. |
| | `annualDebtService` | `BigDecimal` | no | USD. Taken from task instructions (requested loan terms). |
| | `meetsThreshold` | `boolean` | no | `dscr >= 1.25`. |
| `LoanCondition` | `conditionId` | `String` | no | Short slug (e.g. `cond-personal-guarantee`). |
| | `description` | `String` | no | Human-readable condition text. |
| | `conditionType` | `String` | no | `pre-disbursement / ongoing / informational`. |
| `LoanDecision` | `recommendation` | `RecommendationType` | no | `APPROVE / APPROVE_WITH_CONDITIONS / DENY`. |
| | `proposedAmount` | `BigDecimal` | no | USD. May be less than requested. |
| | `interestRateTier` | `String` | no | `TIER_1 / TIER_2 / TIER_3 / TIER_4`. |
| | `rationale` | `String` | no | Must cite ≥ 1 financial criterion for DENY or APPROVE_WITH_CONDITIONS (E1 rule 2). Must not contain `PROXY_TERMS` (E1 rule 3). |
| | `conditions` | `List<LoanCondition>` | no | Empty for APPROVE; non-empty for APPROVE_WITH_CONDITIONS. |
| | `dscrResult` | `DscrResult` | no | Verified by `FairLendingScorer` (E1 rule 4). |
| | `riskTier` | `RiskTier` | no | `TIER_1 / TIER_2 / TIER_3 / TIER_4`. |
| | `decidedAt` | `Instant` | no | When the DECISION task returned. |
| `RationaleText` | `rationale` | `String` | no | Formatted rationale text from `ReportTools.formatDecisionRationale`. |
| `UnderwritingMemo` | `applicationId` | `String` | no | — |
| | `creditProfile` | `CreditProfile` | no | — |
| | `analysis` | `UnderwritingAnalysis` | no | — |
| | `decision` | `LoanDecision` | no | — |
| | `fairnessEval` | `FairnessEvalResult` | no | — |
| | `generatedAt` | `Instant` | no | When the REPORT task returned. |
| `FairnessEvalResult` | `score` | `int` | no | 1–5. |
| | `flags` | `List<String>` | no | Describes each failed check. Empty on score 5. |
| | `rationale` | `String` | no | One sentence naming the largest gap, or "All four fair-lending checks passed." |
| | `evaluatedAt` | `Instant` | no | When `FairLendingScorer` finished. |
| `OfficerReview` | `officerId` | `String` | no | Authenticated officer identifier. |
| | `reviewOutcome` | `String` | no | `APPROVED / DENIED`. |
| | `officerNotes` | `String` | no | Optional freetext; may be empty. |
| | `reviewedAt` | `Instant` | no | When the officer submitted. |
| `GuardrailRejection` | `phase` | `String` | no | `INTAKE / UNDERWRITE / DECISION / REPORT`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `PhaseGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `LoanApplicationRecord` (entity state) | `applicationId` | `String` | no | — |
| | `applicantRef` | `Optional<String>` | yes | Populated after `ApplicationSubmitted`. |
| | `creditProfile` | `Optional<CreditProfile>` | yes | Populated after `IntakeCompleted`. |
| | `analysis` | `Optional<UnderwritingAnalysis>` | yes | Populated after `UnderwritingCompleted`. |
| | `decision` | `Optional<LoanDecision>` | yes | Populated after `DecisionMade`. |
| | `fairnessEval` | `Optional<FairnessEvalResult>` | yes | Populated after `FairnessEvaluated`. |
| | `memo` | `Optional<UnderwritingMemo>` | yes | Populated after `MemoGenerated` (officer-approved path only). |
| | `officerReview` | `Optional<OfficerReview>` | yes | Populated after `OfficerApproved` or `OfficerDenied`. |
| | `status` | `LoanApplicationStatus` | no | See enum. |
| | `submittedAt` | `Instant` | no | When `ApplicationSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `LoanApplicationRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`LoanApplicationStatus`: `SUBMITTED`, `INTAKE_IN_PROGRESS`, `INTAKE_COMPLETE`, `UNDERWRITING`, `UNDERWRITING_COMPLETE`, `DECISION_IN_PROGRESS`, `DECISION_MADE`, `PENDING_REVIEW`, `OFFICER_APPROVED`, `OFFICER_DENIED`, `FAILED`.

`RecommendationType`: `APPROVE`, `APPROVE_WITH_CONDITIONS`, `DENY`.

`RiskTier`: `TIER_1`, `TIER_2`, `TIER_3`, `TIER_4`.

`LoanPhase` (used by `@FunctionTool` annotations and `PhaseGuardrail`): `INTAKE`, `UNDERWRITE`, `DECISION`, `REPORT`.

## Events (`LoanApplicationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ApplicationSubmitted` | `applicantRef, loanAmount, loanPurpose` | → SUBMITTED |
| `IntakeStarted` | — | → INTAKE_IN_PROGRESS |
| `IntakeCompleted` | `creditProfile: CreditProfile` | → INTAKE_COMPLETE |
| `UnderwriteStarted` | — | → UNDERWRITING |
| `UnderwritingCompleted` | `analysis: UnderwritingAnalysis` | → UNDERWRITING_COMPLETE |
| `DecisionStarted` | — | → DECISION_IN_PROGRESS |
| `DecisionMade` | `decision: LoanDecision` | → DECISION_MADE |
| `FairnessEvaluated` | `fairnessEval: FairnessEvalResult` | no status change (audit side-event) |
| `OfficerReviewRequested` | — | → PENDING_REVIEW |
| `OfficerApproved` | `officerReview: OfficerReview` | → OFFICER_APPROVED (terminal happy) |
| `OfficerDenied` | `officerReview: OfficerReview` | → OFFICER_DENIED (terminal denial) |
| `MemoGenerated` | `memo: UnderwritingMemo` | no status change (appended after OfficerApproved) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `ApplicationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `LoanApplicationRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`LoanApplicationRow` mirrors `LoanApplicationRecord` exactly. The UI fetches the full row via `GET /api/loans/{id}` and streams updates via `GET /api/loans/sse`. The view declares ONE query: `getAllApplications: SELECT * AS applications FROM loan_application_view`. No `WHERE status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`LoanTasks.java`)

```java
public final class LoanTasks {
  public static final Task<CreditProfile> INTAKE_APPLICATION = Task
      .name("Intake application")
      .description("Gather the business credit profile and verify the applicant identity reference")
      .resultConformsTo(CreditProfile.class);

  public static final Task<UnderwritingAnalysis> UNDERWRITE_APPLICATION = Task
      .name("Underwrite application")
      .description("Analyze cash flow, value collateral, and identify key risk factors")
      .resultConformsTo(UnderwritingAnalysis.class);

  public static final Task<LoanDecision> MAKE_DECISION = Task
      .name("Make decision")
      .description("Compute DSCR, classify risk tier, and produce a structured loan recommendation")
      .resultConformsTo(LoanDecision.class);

  public static final Task<UnderwritingMemo> GENERATE_REPORT = Task
      .name("Generate report")
      .description("Compile the underwriting memo from the credit profile, analysis, and decision")
      .resultConformsTo(UnderwritingMemo.class);

  private LoanTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools and sanitizer hooks

Each `@FunctionTool` method on `IntakeTools`, `UnderwriteTools`, `DecisionTools`, and `ReportTools` carries a `LoanPhase` constant. `PhaseGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml H1 and the guardrail accept matrix in SPEC.md §8).

`PiiSanitizer` and `ProtectedAttributeSanitizer` are `before-call` hooks on the agent. They receive the full task-context assembly map and return a sanitized version before the map is serialised into the LLM instruction string. Both sanitizers run synchronously on the Akka dispatcher thread; they contain no I/O.
