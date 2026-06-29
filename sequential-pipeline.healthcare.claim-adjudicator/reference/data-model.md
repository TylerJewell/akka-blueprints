# Data model — claim-adjudication-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MemberInfo` | `tokenisedMemberId` | `String` | no | Masked member identifier (`mbr-<sha1-prefix>`). Raw member ID never stored. |
| | `planId` | `String` | no | Payer plan identifier (e.g., `PPO-100`). |
| | `groupId` | `String` | no | Employer group identifier. |
| `ProcedureCode` | `code` | `String` | no | Procedure code value (e.g., `80053`). |
| | `system` | `String` | no | Coding system (e.g., `CPT`, `ICD-10`). |
| | `description` | `String` | no | Human-readable description. |
| `EligibilityResult` | `eligible` | `boolean` | no | True when the member's plan is active and covers the service date. |
| | `planName` | `String` | no | Full plan name from the member record. |
| | `coverageType` | `String` | no | `PPO` / `HMO` / `HDHP`. |
| | `priorAuthRequired` | `List<String>` | no | Procedure codes requiring prior authorisation; empty when none required. |
| | `checkedAt` | `Instant` | no | When `checkEligibility` returned. |
| `CodeVerification` | `code` | `String` | no | The procedure code checked. |
| | `recognised` | `boolean` | no | True when the code appears in the recognised code registry. |
| | `procedureFamily` | `String` | no | `lab` / `surgery` / `mental-health` / `dme` / `unknown`. |
| `ValidationResult` | `eligibility` | `EligibilityResult` | no | Populated by `checkEligibility`. |
| | `codeVerifications` | `List<CodeVerification>` | no | One entry per submitted procedure code. |
| | `validationWarnings` | `List<String>` | no | Non-fatal findings (e.g., `"prior-auth-required"`). Empty on clean validation. |
| | `validatedAt` | `Instant` | no | When the VALIDATE task returned. |
| `PolicyRule` | `ruleId` | `String` | no | Stable rule identifier (e.g., `LAB-R1`). |
| | `description` | `String` | no | Plain-language rule description. |
| | `applicability` | `String` | no | Procedure family this rule governs. |
| | `exclusion` | `boolean` | no | True when this rule limits or excludes coverage. |
| `CoverageAmount` | `currency` | `String` | no | ISO 4217 currency code. |
| | `coveredUnits` | `int` | no | Number of covered units (services). |
| | `coveredPercent` | `double` | no | Fraction of allowed amount covered by the plan (0.0–1.0). |
| | `memberResponsibility` | `double` | no | Member's out-of-pocket responsibility in currency units. |
| `CoverageEvaluation` | `applicableRules` | `List<PolicyRule>` | no | Rules supporting coverage. Possibly empty for ineligible members. |
| | `exclusionRules` | `List<PolicyRule>` | no | Rules that limit or exclude coverage. Possibly empty. |
| | `coverageAmount` | `CoverageAmount` | no | Computed covered amount. |
| | `coverageSummary` | `String` | no | 1–2 sentence plain-language summary for the reviewer. |
| | `evaluatedAt` | `Instant` | no | When the EVALUATE task returned. |
| `DecisionRationale` | `narrative` | `String` | no | Decision explanation citing policy rules. |
| | `citedRuleIds` | `List<String>` | no | MUST have ≥ 2 entries; each MUST match a `PolicyRule.ruleId` from the evaluation. |
| `AdjudicationDecision` | `outcome` | `ClaimOutcome` | no | `APPROVED`, `DENIED`, or `PENDING_REVIEW`. |
| | `rationale` | `DecisionRationale` | no | Decision rationale with cited rule IDs. |
| | `approvedAmount` | `CoverageAmount` | no | Mirrors `coverage.coverageAmount` for approvals; zero-value for denials. |
| | `denialCode` | `String` | yes | Non-null only when `outcome == DENIED`. |
| | `decidedAt` | `Instant` | no | When the ADJUDICATE task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the first failing check (or "All checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `AdjudicationScorer` finished. |
| `PhiRedaction` | `field` | `String` | no | Field name that was redacted (`memberId`, `dob`, `name`, `npi`). |
| | `maskedValue` | `String` | no | The masked token written in place of the raw value. |
| | `redactedAt` | `Instant` | no | When the sanitiser fired. |
| `GuardrailRejection` | `phase` | `String` | no | `VALIDATE` / `EVALUATE` / `ADJUDICATE`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `PhiSanitizerGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `ClaimRecord` (entity state) | `claimId` | `String` | no | — |
| | `memberInfo` | `MemberInfo` | no | Pre-tokenised at submission time. |
| | `procedureCodes` | `List<ProcedureCode>` | no | As submitted. |
| | `serviceDate` | `String` | no | ISO 8601 date of service. |
| | `providerId` | `String` | no | Sanitised provider NPI token. |
| | `validation` | `Optional<ValidationResult>` | yes | Populated after `ClaimValidated`. |
| | `coverage` | `Optional<CoverageEvaluation>` | yes | Populated after `CoverageEvaluated`. |
| | `decision` | `Optional<AdjudicationDecision>` | yes | Populated after `DecisionReached`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `ClaimStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ClaimCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `phiRedactions` | `List<PhiRedaction>` | no | Appended on every `PhiRedacted` event. Empty when no PHI was processed. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event. Empty on the happy path. |

Every nullable lifecycle field on `ClaimRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ClaimStatus`: `CREATED`, `VALIDATING`, `VALIDATED`, `EVALUATING`, `EVALUATED_COVERAGE`, `DECIDING`, `DECIDED`, `PENDING_REVIEW`, `APPROVED`, `DENIED`, `ESCALATED`, `ADJUDICATION_EVALUATED`, `FAILED`.

`ClaimOutcome`: `APPROVED`, `DENIED`, `PENDING_REVIEW`.

`AdjudicationPhase` (used by `@FunctionTool` annotations and `PhiSanitizerGuardrail`): `VALIDATE`, `EVALUATE`, `ADJUDICATE`.

## Events (`ClaimEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ClaimCreated` | `memberInfo, procedureCodes, serviceDate, providerId` | → CREATED |
| `ValidationStarted` | — | → VALIDATING |
| `ClaimValidated` | `validation: ValidationResult` | → VALIDATED |
| `EvaluationStarted` | — | → EVALUATING |
| `CoverageEvaluated` | `coverage: CoverageEvaluation` | → EVALUATED_COVERAGE |
| `AdjudicationStarted` | — | → DECIDING |
| `DecisionReached` | `decision: AdjudicationDecision` | → DECIDED |
| `HumanReviewRequested` | `decisionSummary, denialCode, reviewDeadline` | → PENDING_REVIEW |
| `DenialApproved` | `reviewerId, notes, approvedAt` | → DENIED |
| `DenialOverridden` | `reviewerId, notes, overriddenAt` | → APPROVED |
| `ClaimEscalated` | `reason: "review-timeout"` | → ESCALATED (terminal) |
| `EvaluationScored` | `eval: EvalResult` | → ADJUDICATION_EVALUATED (terminal happy) |
| `PhiRedacted` | `field, maskedValue, redactedAt` | no status change (audit-only) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `ClaimFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ClaimRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `phiRedactions = List.of()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ClaimRow` mirrors `ClaimRecord` exactly. The UI fetches the full row via `GET /api/claims/{id}` and streams updates via `GET /api/claims/sse`.

The view declares ONE query: `getAllClaims: SELECT * AS claims FROM claim_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`AdjudicationTasks.java`)

```java
public final class AdjudicationTasks {
  public static final Task<ValidationResult> VALIDATE_CLAIM = Task
      .name("Validate claim")
      .description("Check member eligibility and verify procedure codes against the payer's recognised code system")
      .resultConformsTo(ValidationResult.class);

  public static final Task<CoverageEvaluation> EVALUATE_COVERAGE = Task
      .name("Evaluate coverage")
      .description("Look up applicable policy rules and compute the covered amount for the validated claim")
      .resultConformsTo(CoverageEvaluation.class);

  public static final Task<AdjudicationDecision> ADJUDICATE_CLAIM = Task
      .name("Adjudicate claim")
      .description("Produce a decision with outcome (APPROVED/DENIED/PENDING_REVIEW), rationale citing policy rules, and approved amount")
      .resultConformsTo(AdjudicationDecision.class);

  private AdjudicationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools and sanitiser

Each `@FunctionTool` method on `ValidateTools`, `EvaluateTools`, and `AdjudicateTools` carries an `AdjudicationPhase` constant. `PhiSanitizerGuardrail` reads this constant and the current `ClaimEntity.status` before the tool body runs. It first sanitises PHI in the tool arguments (emitting `PhiRedacted` events), then checks phase order and rejects out-of-phase calls (emitting `GuardrailRejected` events). The sanitiser always passes a sanitised call through on phase match; it only rejects on phase mismatch. PHI sanitisation and phase gating are combined in one guardrail to ensure that even a rejected call's arguments never contain raw PHI in the rejection log.
