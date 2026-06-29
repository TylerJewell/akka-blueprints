# Data model — cold-outreach

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IntentSignal` | `signalType` | `String` | no | Short slug identifying the type of signal (e.g. `hiring-engineering`, `recent-funding`). |
| | `description` | `String` | no | Human-readable description of the signal observation. |
| | `observedAt` | `Instant` | no | When the signal was observed or published. |
| `FirmographicData` | `companyName` | `String` | no | Legal or trading name of the prospect's company. |
| | `domain` | `String` | no | Primary web domain. |
| | `industry` | `String` | no | Industry slug (e.g. `developer-tooling`, `fintech`). |
| | `employeeCount` | `int` | no | Approximate headcount; 0 if unknown. |
| | `hqCountry` | `String` | no | ISO-3166-1 alpha-2 country code. Used by `ComplianceGuardrail` to check the blocked-regions list. |
| | `intentSignals` | `List<IntentSignal>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `researchedAt` | `Instant` | no | When the RESEARCH task completed firmographic lookup. |
| `ProspectProfile` | `prospectId` | `String` | no | — |
| | `contactEmail` | `String` | no | The prospect's email address (PII). |
| | `firmographics` | `FirmographicData` | no | Full firmographic record. |
| | `personalizationHooks` | `List<String>` | no | Short strings derived from `intentSignals` for use as email openers. Possibly empty. |
| | `profiledAt` | `Instant` | no | When the RESEARCH task returned. |
| `EmailDraft` | `subject` | `String` | no | Email subject line. |
| | `body` | `String` | no | Full email body. MUST contain `[unsubscribe]` and `[sender-address]` sentinels (H2 rule). |
| | `personalizationFields` | `List<String>` | no | Names of the template variables substituted in this draft. |
| | `templateId` | `String` | no | Id of the email template used (from `email-templates.json`). |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned. |
| `ComplianceCheckResult` | `passed` | `boolean` | no | `true` if all three compliance rules passed. |
| | `failedRules` | `List<String>` | no | Rule ids that failed; empty when `passed == true`. |
| | `checkedAt` | `Instant` | no | When `ComplianceGuardrail` ran. |
| `ReviewDecision` | `approved` | `boolean` | no | Reviewer's decision. |
| | `reviewerNote` | `String` | no | Free-text note from the reviewer (may be empty string). |
| | `decidedAt` | `Instant` | no | When the reviewer submitted the decision. |
| `SendReceipt` | `messageId` | `String` | no | Stable id minted as `"msg-" + sha1(recipient + subject).substring(0,8)`. |
| | `recipient` | `String` | no | The prospect's contact email. |
| | `sentAt` | `Instant` | no | When `SendTools.sendEmail` completed. |
| `OutreachEmail` | `subject` | `String` | no | Subject line from the approved draft. |
| | `body` | `String` | no | Body from the approved draft (with sentinels intact). |
| | `recipient` | `String` | no | Contact email. |
| | `compliance` | `ComplianceCheckResult` | no | The check result recorded during DRAFT. |
| | `receipt` | `SendReceipt` | no | Receipt from `SendTools.sendEmail`. |
| | `finishedAt` | `Instant` | no | When `sendStep` completed. |
| `GuardrailRejection` | `phase` | `String` | no | `RESEARCH` / `DRAFT` / `SEND`. |
| | `tool` | `String` | no | Name of the rejected tool call (or `agent-response` for compliance rejections). |
| | `reason` | `String` | no | Structured reason from the guardrail. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `ProspectRecord` (entity state) | `prospectId` | `String` | no | — |
| | `contactEmail` | `Optional<String>` | yes | Populated after `ProspectCreated`. |
| | `profile` | `Optional<ProspectProfile>` | yes | Populated after `ProspectResearched`. |
| | `draft` | `Optional<EmailDraft>` | yes | Populated after `EmailDrafted`. |
| | `compliance` | `Optional<ComplianceCheckResult>` | yes | Populated after `ComplianceChecked`. |
| | `reviewDecision` | `Optional<ReviewDecision>` | yes | Populated after `ReviewDecided`. |
| | `outreachEmail` | `Optional<OutreachEmail>` | yes | Populated after `EmailSent`. |
| | `status` | `OutreachStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ProspectCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `ProspectRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`OutreachStatus`: `CREATED`, `RESEARCHING`, `RESEARCHED`, `DRAFTING`, `DRAFTED`, `AWAITING_REVIEW`, `SENDING`, `SENT`, `REVIEW_REJECTED`, `FAILED`.

`OutreachPhase` (used by `@FunctionTool` annotations and guardrails): `RESEARCH`, `DRAFT`, `SEND`.

## Events (`ProspectEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ProspectCreated` | `contactEmail: String, companyName: String` | → CREATED |
| `ResearchStarted` | — | → RESEARCHING |
| `ProspectResearched` | `profile: ProspectProfile` | → RESEARCHED |
| `DraftStarted` | — | → DRAFTING |
| `EmailDrafted` | `draft: EmailDraft` | → DRAFTED |
| `ComplianceChecked` | `result: ComplianceCheckResult` | no status change (audit-only) |
| `ReviewRequested` | — | → AWAITING_REVIEW |
| `ReviewDecided` | `decision: ReviewDecision` | → SENDING (approved) / REVIEW_REJECTED (rejected) |
| `SendStarted` | — | → SENDING |
| `EmailSent` | `outreachEmail: OutreachEmail` | → SENT (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `OutreachFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ProspectRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ProspectRow` mirrors `ProspectRecord` exactly. The UI fetches the full row via `GET /api/outreach/{id}` and streams updates via `GET /api/outreach/sse`.

The view declares ONE query: `getAllProspects: SELECT * AS prospects FROM prospect_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`OutreachTasks.java`)

```java
public final class OutreachTasks {
  public static final Task<ProspectProfile> RESEARCH_PROSPECT = Task
      .name("Research prospect")
      .description("Gather firmographic data and intent signals for a prospect by calling "
          + "lookupFirmographics and fetchIntentSignals")
      .resultConformsTo(ProspectProfile.class);

  public static final Task<EmailDraft> DRAFT_EMAIL = Task
      .name("Draft email")
      .description("Compose a personalized email body using personalizeLine and renderTemplate; "
          + "body MUST include [unsubscribe] and [sender-address] sentinels")
      .resultConformsTo(EmailDraft.class);

  public static final Task<OutreachEmail> SEND_EMAIL = Task
      .name("Send email")
      .description("Send the approved draft via sendEmail and return a SendReceipt")
      .resultConformsTo(OutreachEmail.class);

  private OutreachTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `ResearchTools`, `DraftTools`, and `SendTools` carries an `OutreachPhase` constant. `SendGuardrail` checks the tool name (`sendEmail`) and the entity state before the `sendEmail` tool body runs. `ComplianceGuardrail` checks the phase metadata (`"DRAFT"`) and the proposed response body before the agent's DRAFT task response is committed. Both guardrails are built once at startup and registered on `OutreachAgent`.
