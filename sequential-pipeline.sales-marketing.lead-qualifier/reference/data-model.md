# Data model — lead-qualifier

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ContactFields` | `nameInitial` | `String` | no | Initials only; safe for downstream use. |
| | `rawEmail` | `String` | no | Full email address; stored on `LeadEntity`, never projected. |
| | `rawPhone` | `String` | no | Full phone number; stored on `LeadEntity`, never projected. |
| | `company` | `String` | no | Company name as extracted from the inquiry text. |
| | `channel` | `String` | no | One of `web-form \| email \| referral \| event`. |
| `IntentSignals` | `productInterest` | `String` | no | Product or feature area mentioned in the inquiry. |
| | `useCaseSummary` | `String` | no | Short summary of the described use case. |
| | `budgetIndicator` | `String` | no | One of `unknown \| <10k \| 10k-50k \| 50k+`. |
| `InquiryForm` | `contact` | `ContactFields` | no | Extracted contact block. |
| | `intent` | `IntentSignals` | no | Extracted intent block. |
| | `rawText` | `String` | no | The original inquiry string. |
| | `capturedAt` | `Instant` | no | When the CAPTURE task returned. |
| `FitScore` | `score` | `int` | no | 0–100 composite fit score. |
| | `reasoning` | `String` | no | One-sentence explanation of the score. |
| `LeadScore` | `fit` | `FitScore` | no | Composite fit score. |
| | `urgency` | `UrgencyTier` | no | `HIGH \| MEDIUM \| LOW \| DISQUALIFIED`. |
| | `recommendedStage` | `String` | no | One of `New \| Qualified \| MQL \| SQL \| Disqualified`. |
| | `disqualificationReason` | `Optional<String>` | yes | Non-null only when `urgency = DISQUALIFIED`. |
| | `scoredAt` | `Instant` | no | When the QUALIFY task returned. |
| `CrmStageResult` | `leadId` | `String` | no | Echoed from the in-flight lead. |
| | `stage` | `String` | no | The stage value written. |
| | `written` | `boolean` | no | `true` if the simulated CRM write succeeded. |
| `OwnerAssignment` | `ownerId` | `String` | no | Internal owner ID. |
| | `ownerName` | `String` | no | Display name. |
| | `rationale` | `String` | no | Why this owner was selected. |
| `CrmEntry` | `leadId` | `String` | no | — |
| | `stage` | `String` | no | MUST be one of `New \| Qualified \| MQL \| SQL \| Disqualified`. |
| | `ownerCandidate` | `String` | no | Display name of the assigned owner. |
| | `notes` | `String` | no | Free-text summary for the SDR. |
| | `maskedEmail` | `String` | no | SHA-256(rawEmail)[0:8] + `@masked`. |
| | `maskedPhone` | `String` | no | `[REDACTED]`. |
| | `nameInitial` | `String` | no | Initials from `ContactFields.nameInitial`. |
| | `enrichedAt` | `Instant` | no | When the ENRICH task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `QualityScorer` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `CAPTURE \| QUALIFY \| ENRICH`. |
| | `tool` | `String` | no | Name of the rejected tool. |
| | `reason` | `String` | no | Structured reason from `CrmWriteGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `LeadRecord` (entity state) | `leadId` | `String` | no | — |
| | `rawInquiry` | `Optional<String>` | yes | Populated after `LeadCreated`. |
| | `form` | `Optional<InquiryForm>` | yes | Populated after `InquiryCaptured`. |
| | `score` | `Optional<LeadScore>` | yes | Populated after `QualificationScored`. |
| | `crmEntry` | `Optional<CrmEntry>` | yes | Populated after `CrmEntryWritten`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `LeadStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `LeadCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event. |

Every nullable lifecycle field on `LeadRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

`LeadRow` mirrors `LeadRecord` exactly, **except** that `form.contact` in the projected row omits `rawEmail` and `rawPhone` — those fields are never serialised into the view or any API response.

## Enums

`LeadStatus`: `CREATED`, `CAPTURING`, `CAPTURED`, `QUALIFYING`, `QUALIFIED`, `ENRICHING`, `ENRICHED`, `EVALUATED`, `FAILED`.

`UrgencyTier`: `HIGH`, `MEDIUM`, `LOW`, `DISQUALIFIED`.

`Phase` (used by `@FunctionTool` annotations and `CrmWriteGuardrail`): `CAPTURE`, `QUALIFY`, `ENRICH`.

## Events (`LeadEntity`)

| Event | Payload | Transition |
|---|---|---|
| `LeadCreated` | `rawInquiry: String` | → CREATED |
| `CaptureStarted` | — | → CAPTURING |
| `InquiryCaptured` | `form: InquiryForm` | → CAPTURED |
| `QualifyStarted` | — | → QUALIFYING |
| `QualificationScored` | `score: LeadScore` | → QUALIFIED |
| `EnrichStarted` | — | → ENRICHING |
| `CrmEntryWritten` | `crmEntry: CrmEntry` | → ENRICHED |
| `EvaluationScored` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `LeadFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `LeadRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`LeadRow` mirrors `LeadRecord` with the PII exclusion noted above. The UI fetches the full row via `GET /api/leads/{id}` and streams updates via `GET /api/leads/sse`.

The view declares ONE query: `getAllLeads: SELECT * AS leads FROM lead_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`LeadTasks.java`)

```java
public final class LeadTasks {
  public static final Task<InquiryForm> CAPTURE_INQUIRY = Task
      .name("Capture inquiry")
      .description("Extract contact fields and intent signals from the raw inquiry text")
      .resultConformsTo(InquiryForm.class);

  public static final Task<LeadScore> QUALIFY_LEAD = Task
      .name("Qualify lead")
      .description("Score the captured inquiry for fit and urgency; assign a recommended CRM stage")
      .resultConformsTo(LeadScore.class);

  public static final Task<CrmEntry> ENRICH_CRM = Task
      .name("Enrich CRM")
      .description("Write the qualified lead into CRM stages and assign an owner candidate")
      .resultConformsTo(CrmEntry.class);

  private LeadTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `CaptureTools`, `QualifyTools`, and `EnrichTools` carries a `Phase` constant. `CrmWriteGuardrail` reads this constant before the tool body runs and rejects calls whose phase does not match the per-status accept matrix (see eval-matrix.yaml G1). For ENRICH-phase tools, the guardrail performs the additional schema and scope checks described in Section 8 of SPEC.md, then delegates to `PiiSanitizer` before passing the payload to the tool body.

## QualityScorer checks

`QualityScorer` runs four checks on `(CrmEntry, LeadScore, InquiryForm)`, one point per check satisfied, on a base of 1:

1. **Stage validity** — `crmEntry.stage` is in `{New, Qualified, MQL, SQL, Disqualified}`.
2. **Owner assignment** — `crmEntry.ownerCandidate` is non-empty.
3. **Notes present** — `crmEntry.notes.length() > 0`.
4. **Fit-score consistency** — `crmEntry.stage` matches the expected stage bracket for `score.urgency` (e.g., `HIGH` → `SQL` or `MQL`; `DISQUALIFIED` → `Disqualified`).

Score range 1–5. Rationale names the largest gap or reads "Stage valid, owner assigned, notes present, fit-score consistent with stage" when all four pass.
