# Data model — global-kyc-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Document` | `documentId` | `String` | no | Stable ID minted by `CollectTools` (`<applicantId>-<docType>`). |
| | `applicantId` | `String` | no | Applicant this document belongs to. |
| | `docType` | `DocumentType` | no | `PASSPORT`, `NATIONAL_ID`, `DRIVERS_LICENSE`, or `RESIDENCE_PERMIT`. |
| | `issuingJurisdiction` | `String` | no | ISO 3166-1 alpha-2 or sub-division code (e.g., `GB`, `US-NY`). |
| | `fullName` | `String` | no | Raw PII — sanitized to `NAME-<hash8>` before view projection. |
| | `dateOfBirth` | `String` | no | Raw PII — `YYYY-MM-DD`; sanitized to `DOB-<hash8>` before view projection. |
| | `documentNumber` | `String` | no | Raw PII — sanitized to `DOC-****-<last4>` before view projection. |
| | `fetchedAt` | `Instant` | no | When `CollectTools.fetchDocument` returned this entry. |
| `DocumentSet` | `applicantId` | `String` | no | Carries the same applicant ID as all its `Document` entries. |
| | `documents` | `List<Document>` | no | Possibly empty (J5 demonstrates the empty-corpus path). |
| | `applicableRules` | `List<JurisdictionRule>` | no | Possibly empty if jurisdiction is unrecognised. |
| | `collectedAt` | `Instant` | no | When the COLLECT task returned. |
| `JurisdictionRule` | `ruleId` | `String` | no | Stable slug (e.g., `GB-AML-DOC-001`). Unique within a `DocumentSet`. |
| | `jurisdiction` | `String` | no | Matches the case's declared jurisdiction. |
| | `description` | `String` | no | Human-readable rule text for the UI and rationale. |
| | `requiredDocumentType` | `String` | yes | `null` when the rule applies to all document types. |
| `DocumentVerification` | `documentId` | `String` | no | MUST equal a `Document.documentId` from the upstream `DocumentSet`. |
| | `status` | `DocumentStatus` | no | `VERIFIED`, `EXPIRED`, `TAMPERED`, `UNREADABLE`, or `MISSING`. |
| | `statusReason` | `String` | no | Brief phrase (e.g., "Expiry date 2023-01-01 is in the past"). |
| `RuleCheckResult` | `ruleId` | `String` | no | MUST equal a `JurisdictionRule.ruleId` from the upstream `DocumentSet`. |
| | `passed` | `boolean` | no | `true` when the rule is satisfied; `false` when the rule failed. |
| | `failureReason` | `String` | yes | `null` when `passed == true`; one sentence when `passed == false`. |
| `VerificationResult` | `documentVerifications` | `List<DocumentVerification>` | no | One entry per `Document` in the `DocumentSet`. |
| | `ruleResults` | `List<RuleCheckResult>` | no | One entry per `JurisdictionRule` in the `DocumentSet`. |
| | `verifiedAt` | `Instant` | no | When the VERIFY task returned. |
| `KycDecision` | `outcome` | `KycOutcome` | no | `PASS`, `DECLINE`, `REFER`, or `PENDING_DOCUMENTS`. |
| | `jurisdiction` | `String` | no | Matches the case's declared jurisdiction. |
| | `citedRuleIds` | `List<String>` | no | ≥ 2 entries; each MUST exist in `DocumentSet.applicableRules[].ruleId`. |
| | `rationale` | `String` | no | One sentence naming the decisive rule or document status. |
| | `decidedAt` | `Instant` | no | When the DECIDE task returned. |
| `ScorerResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `DecisionScorer` finished. |
| `ReviewResolution` | `reviewerId` | `String` | no | Identity of the compliance officer who resolved the review. |
| | `resolution` | `ReviewOutcome` | no | `APPROVED_AS_SUBMITTED` or `OVERRIDDEN_TO_PASS`. |
| | `notes` | `String` | no | Free-text notes entered by the reviewer. May be empty string but not null. |
| | `resolvedAt` | `Instant` | no | When the POST to `/api/cases/{id}/review` was processed. |
| `KycCaseRecord` (entity state) | `caseId` | `String` | no | — |
| | `applicantId` | `Optional<String>` | yes | Populated after `CaseCreated`. |
| | `documents` | `Optional<DocumentSet>` | yes | Populated after `DocumentsCollected`. Contains raw PII in entity log. |
| | `verification` | `Optional<VerificationResult>` | yes | Populated after `IdentityVerified`. |
| | `decision` | `Optional<KycDecision>` | yes | Populated after `DecisionRendered`. |
| | `eval` | `Optional<ScorerResult>` | yes | Populated after `DecisionEvaluated`. |
| | `review` | `Optional<ReviewResolution>` | yes | Populated after `ReviewResolved` (null for PASS-outcome cases). |
| | `status` | `KycCaseStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CaseCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp (EVALUATED or FAILED). |

Every nullable lifecycle field on `KycCaseRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6). The `documents` field in the view row carries only tokenised PII; the raw `DocumentSet` is retained only in the entity event log.

## Enums

`DocumentType`: `PASSPORT`, `NATIONAL_ID`, `DRIVERS_LICENSE`, `RESIDENCE_PERMIT`.

`DocumentStatus`: `VERIFIED`, `EXPIRED`, `TAMPERED`, `UNREADABLE`, `MISSING`.

`KycOutcome`: `PASS`, `DECLINE`, `REFER`, `PENDING_DOCUMENTS`.

`ReviewOutcome`: `APPROVED_AS_SUBMITTED`, `OVERRIDDEN_TO_PASS`.

`KycCaseStatus`: `CREATED`, `COLLECTING`, `COLLECTED`, `VERIFYING`, `VERIFIED`, `DECIDING`, `DECIDED`, `PENDING_REVIEW`, `REVIEWED`, `EVALUATED`, `FAILED`.

`KycPhase` (used by `@FunctionTool` annotations): `COLLECT`, `VERIFY`, `DECIDE`.

## Events (`KycCaseEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CaseCreated` | `applicantId: String, jurisdiction: String` | → CREATED |
| `CollectStarted` | — | → COLLECTING |
| `DocumentsCollected` | `documents: DocumentSet` (contains raw PII) | → COLLECTED |
| `VerifyStarted` | — | → VERIFYING |
| `IdentityVerified` | `verification: VerificationResult` | → VERIFIED |
| `DecideStarted` | — | → DECIDING |
| `DecisionRendered` | `decision: KycDecision` | → DECIDED (PASS) |
| `ReviewRequested` | `caseId: String` | → PENDING_REVIEW (non-PASS outcomes) |
| `ReviewResolved` | `review: ReviewResolution` | → REVIEWED |
| `DecisionEvaluated` | `eval: ScorerResult` | → EVALUATED (terminal happy) |
| `SanitizerApplied` | `fieldsRedacted: List<String>` | no status change (audit-only) |
| `CaseFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `KycCaseRecord.initial("")` with all `Optional` fields as `Optional.empty()` and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`KycViewRow` mirrors `KycCaseRecord`. The `documents` field in the view row is a sanitised copy where `fullName`, `dateOfBirth`, and `documentNumber` have been replaced by stable tokens. The UI fetches the full row via `GET /api/cases/{id}` and streams updates via `GET /api/cases/sse`. Both paths serve only tokenised PII.

The view declares ONE query: `getAllCases: SELECT * AS cases FROM kyc_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`KycTasks.java`)

```java
public final class KycTasks {
  public static final Task<DocumentSet> COLLECT_DOCUMENTS = Task
      .name("Collect documents")
      .description("Fetch applicant identity documents and the jurisdiction's applicable rule set")
      .resultConformsTo(DocumentSet.class);

  public static final Task<VerificationResult> VERIFY_IDENTITY = Task
      .name("Verify identity")
      .description("Check each document for authenticity and evaluate every applicable jurisdiction rule")
      .resultConformsTo(VerificationResult.class);

  public static final Task<KycDecision> RENDER_DECISION = Task
      .name("Render decision")
      .description("Compile all verification results and rule checks into a KycDecision with outcome and cited rule IDs")
      .resultConformsTo(KycDecision.class);

  private KycTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `CollectTools`, `VerifyTools`, and `DecideTools` carries a `KycPhase` constant. The workflow step metadata carries a `phase` tag matching the task's phase. The agent's instruction context names the expected tools for each phase; the system prompt reinforces phase discipline. See `prompts/kyc-agent.md` for the full per-phase tool listing.

## PII tokenization contract

`PiiSanitizer` produces stable deterministic tokens:

| Field | Raw value | Token format | Example |
|---|---|---|---|
| `fullName` | `Jane Thornton` | `NAME-` + sha1(value)[0:8] | `NAME-7f3a1b22` |
| `dateOfBirth` | `1982-04-15` | `DOB-` + sha1(value)[0:8] | `DOB-9c0e44d1` |
| `documentNumber` | `GB1234567` | `DOC-****-` + last4(value) | `DOC-****-4567` |

The same raw value always produces the same token (deterministic hash). Cross-case deduplication of applicants is possible using tokens without ever exposing raw PII to the view layer.
