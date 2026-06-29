# Data model — medical-preauth

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MemberInfo` | `memberId` | `String` | no | Payer member identifier. PHI — sanitized to hashed token before any LLM call. |
| | `planId` | `String` | no | Payer plan identifier. |
| | `groupNumber` | `String` | no | Group policy number. |
| | `activeEligibility` | `boolean` | no | Whether the member has active coverage at request time. |
| `ProcedureRequest` | `procedureCode` | `String` | no | CPT code (e.g. `77080`). |
| | `diagnosisCode` | `String` | no | ICD-10 code (e.g. `M81.0`). |
| | `treatingPhysicianNpi` | `String` | no | NPI of the ordering physician. |
| | `requestingFacility` | `String` | no | Name of the requesting facility. |
| | `clinicalNotes` | `String` | no | Free-text clinical notes. PHI — sanitized before any LLM call; redacted in API responses. |
| `EligibilityCheck` | `eligible` | `boolean` | no | True if the member has active coverage. |
| | `planType` | `String` | no | `PPO` / `HMO` / `EPO` etc. |
| | `coverageTier` | `String` | no | Tier label from the plan configuration. |
| | `verifiedAt` | `Instant` | no | When `ValidateTools.checkMemberEligibility` returned. |
| `ProcedureCheck` | `codeRecognized` | `boolean` | no | True if the CPT code exists in the payer's code table. |
| | `codeDescription` | `String` | no | Human-readable description of the CPT code. |
| | `icd10Linked` | `boolean` | no | True if the ICD-10 code is a valid indication for the CPT code. |
| | `checkedAt` | `Instant` | no | When `ValidateTools.validateProcedureCode` returned. |
| `ValidationResult` | `eligibility` | `EligibilityCheck` | no | Output of the eligibility check. |
| | `procedureCheck` | `ProcedureCheck` | no | Output of the procedure/ICD-10 check. |
| | `validationPassed` | `boolean` | no | True if both checks passed. Gate for reviewStep. |
| | `validatedAt` | `Instant` | no | When the VALIDATE task returned. |
| `PolicyArticle` | `articleId` | `String` | no | Payer policy article identifier (e.g. `pol-dexa-001`). |
| | `title` | `String` | no | Article title. |
| | `criteriaText` | `String` | no | Criteria prose from the policy document. |
| | `policyVersion` | `String` | no | Policy version string (e.g. `2026-01`). |
| `CriterionResult` | `criterionId` | `String` | no | Short stable id within the article (e.g. `cr-age-risk`). |
| | `description` | `String` | no | Human-readable criterion description. |
| | `met` | `boolean` | no | True if the criterion is satisfied by the clinical evidence. |
| | `evidence` | `String` | no | The evidence text that determined met/unmet. |
| `CriteriaEvaluation` | `criteria` | `List<CriterionResult>` | no | One entry per criterion in the matched articles. |
| | `metCount` | `int` | no | Number of criteria where `met == true`. |
| | `totalCount` | `int` | no | Total criteria evaluated. |
| | `evaluatedAt` | `Instant` | no | When `ReviewTools.evaluateCriteria` returned. |
| `PolicyReview` | `matchedArticles` | `List<PolicyArticle>` | no | Articles whose scope matches the procedure/diagnosis pair. |
| | `criteriaEvaluation` | `CriteriaEvaluation` | no | Result of evaluating all criteria in matched articles. |
| | `reviewedAt` | `Instant` | no | When the REVIEW task returned. |
| `Determination` | `outcome` | `AuthOutcome` | no | `APPROVED` / `DENIED` / `PENDING_ADDITIONAL_INFO`. |
| | `rationale` | `String` | no | ≥ 40 words for DENIED, ≥ 15 for APPROVED (E1 rule 3). |
| | `citedArticleIds` | `List<String>` | no | MUST be a subset of `PolicyReview.matchedArticles[].articleId` (E1 rule 2). |
| | `coveredProcedureCodes` | `List<String>` | no | MUST match the validated procedure code (E1 rule 1). |
| | `determinedAt` | `Instant` | no | When the DETERMINE task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `ClinicalCompletenessScorer` finished. |
| `SanitizationRecord` | `field` | `String` | no | Name of the PHI field replaced (e.g. `memberId`). |
| | `hashedToken` | `String` | no | 12-character hex token substituted into the payload. |
| | `tool` | `String` | no | Name of the tool call that triggered sanitization. |
| | `sanitizedAt` | `Instant` | no | When the sanitizer replaced the field. |
| `PhaseRejection` | `phase` | `String` | no | `VALIDATE` / `REVIEW` / `DETERMINE`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `PhiSanitizer`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `PreAuthRecord` (entity state) | `requestId` | `String` | no | — |
| | `memberInfo` | `Optional<MemberInfo>` | yes | Populated after `RequestSubmitted`. |
| | `procedureRequest` | `Optional<ProcedureRequest>` | yes | Populated after `RequestSubmitted`. |
| | `validationResult` | `Optional<ValidationResult>` | yes | Populated after `RequestValidated`. |
| | `policyReview` | `Optional<PolicyReview>` | yes | Populated after `PolicyReviewed`. |
| | `determination` | `Optional<Determination>` | yes | Populated after `DeterminationWritten`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `PreAuthStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RequestSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `sanitizationLog` | `List<SanitizationRecord>` | no | Appended on every `SanitizationApplied` event; empty if no PHI fields were detected. |
| | `phaseRejections` | `List<PhaseRejection>` | no | Appended on every `PhaseGuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `PreAuthRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`PreAuthStatus`: `SUBMITTED`, `VALIDATING`, `VALIDATED`, `REVIEWING`, `POLICY_REVIEWED`, `DETERMINING`, `DETERMINED`, `PENDING_HUMAN_REVIEW`, `EVALUATED`, `ESCALATED`, `FAILED`.

`AuthOutcome`: `APPROVED`, `DENIED`, `PENDING_ADDITIONAL_INFO`.

`Phase` (used by `@FunctionTool` annotations and `PhiSanitizer`): `VALIDATE`, `REVIEW`, `DETERMINE`.

## Events (`PreAuthEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RequestSubmitted` | `memberId, procedureCode, diagnosisCode, treatingPhysicianNpi, clinicalNotes` | → SUBMITTED |
| `ValidationStarted` | — | → VALIDATING |
| `RequestValidated` | `validationResult: ValidationResult` | → VALIDATED |
| `ReviewStarted` | — | → REVIEWING |
| `PolicyReviewed` | `policyReview: PolicyReview` | → POLICY_REVIEWED |
| `DetermineStarted` | — | → DETERMINING |
| `DeterminationWritten` | `determination: Determination` | → DETERMINED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (if outcome ≠ DENIED) |
| `HumanApprovalRequested` | `requestedAt: Instant` | → PENDING_HUMAN_REVIEW |
| `HumanApprovalReceived` | `reviewerNotes: String, receivedAt: Instant` | → EVALUATED |
| `EscalationRequired` | `reason: String` | → ESCALATED (terminal) |
| `SanitizationApplied` | `field, hashedToken, tool, sanitizedAt` | no status change (audit-only) |
| `PhaseGuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `RequestFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `PreAuthRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `sanitizationLog = List.of()`, `phaseRejections = List.of()`, and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`PreAuthRow` mirrors `PreAuthRecord` exactly. The UI fetches the full row via `GET /api/auth-requests/{id}` and streams updates via `GET /api/auth-requests/sse`.

The view declares ONE query: `getAllRequests: SELECT * AS requests FROM preauth_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`PreAuthTasks.java`)

```java
public final class PreAuthTasks {
  public static final Task<ValidationResult> VALIDATE_REQUEST = Task
      .name("Validate request")
      .description("Check member eligibility and validate the procedure code / ICD-10 linkage")
      .resultConformsTo(ValidationResult.class);

  public static final Task<PolicyReview> REVIEW_POLICY = Task
      .name("Review policy")
      .description("Match payer policy articles and evaluate each criterion against the submitted clinical evidence")
      .resultConformsTo(PolicyReview.class);

  public static final Task<Determination> DETERMINE_OUTCOME = Task
      .name("Determine outcome")
      .description("Build a rationale and classify the authorization outcome based on the policy review")
      .resultConformsTo(Determination.class);

  private PreAuthTasks() {}
}
```

The companion Tasks class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ValidateTools`, `ReviewTools`, and `DetermineTools` carries a `Phase` constant. `PhiSanitizer` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml S1). Additionally, `PhiSanitizer` strips PHI fields from the payload before forwarding the call — regardless of whether the phase check passes or fails. The tool registry is built once at startup; the guardrail reads it for every call.
