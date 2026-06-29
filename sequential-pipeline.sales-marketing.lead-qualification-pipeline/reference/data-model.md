# Data model — inbound-lead-qualification

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `LeadFormData` | `leadId` | `String` | no | Minted by `LeadEndpoint` on POST. |
| | `firstName` | `String` | no | Submitter first name. PII — sanitized in agent context. |
| | `lastName` | `String` | no | Submitter last name. PII — sanitized in agent context. |
| | `email` | `String` | no | Submitter email. PII — sanitized in agent context. |
| | `companyName` | `String` | no | Company display name. |
| | `companySize` | `CompanySize` | no | XS / S / M / L / XL. |
| | `message` | `Optional<String>` | yes | Optional free-text from the form. |
| | `submittedAt` | `Instant` | no | When the POST was received. |
| `TechStackSignal` | `technology` | `String` | no | Technology name (e.g. "Kubernetes"). |
| | `evidence` | `String` | no | Short evidence string (e.g. "Listed in job postings"). |
| `FirmographicProfile` | `domain` | `String` | no | Company domain. |
| | `industry` | `String` | no | Industry label. `"(unknown)"` if enrichment data absent. |
| | `estimatedArrBand` | `String` | no | ARR range string. `"(unknown)"` if absent. |
| | `employeeBand` | `String` | no | Employee band string. `"(unknown)"` if absent. |
| | `techStack` | `List<TechStackSignal>` | no | Possibly empty. |
| | `linkedinUrl` | `Optional<String>` | yes | Company LinkedIn URL. |
| | `enrichedAt` | `Instant` | no | When the ENRICH task returned. |
| `QualificationScore` | `leadId` | `String` | no | From the originating `LeadFormData`. |
| | `tier` | `LeadTier` | no | HOT / WARM / COLD. |
| | `score` | `int` | no | 0–100. |
| | `rationale` | `String` | no | 1–2 sentences naming primary scoring signals. |
| | `assignedRepName` | `String` | no | Full name of the assigned rep. |
| | `assignedRepSlackId` | `String` | no | Slack user ID. Non-blank (E1 rule 2). |
| | `qualifiedAt` | `Instant` | no | When the QUALIFY task returned. |
| `SlackBlock` | `type` | `String` | no | `"header"` or `"section"`. |
| | `text` | `String` | no | Block content. |
| `SlackPayload` | `channel` | `String` | no | Slack channel ID. |
| | `text` | `String` | no | Fallback notification text. |
| | `blocks` | `List<SlackBlock>` | no | At least 2 blocks. |
| | `sentAt` | `Instant` | no | When `postLeadToSlack` succeeded. |
| `EvalRecord` | `confidence` | `EvalConfidence` | no | HIGH / MED / LOW. |
| | `rationale` | `String` | no | One sentence naming the failing check, or "All checks passed." |
| | `evaluatedAt` | `Instant` | no | When `QualificationEvaluator` finished. |
| `GuardrailRejection` | `phase` | `String` | no | `ENRICH` / `QUALIFY` / `NOTIFY`. |
| | `tool` | `String` | no | Name of the blocked tool (typically `"postLeadToSlack"`). |
| | `reason` | `String` | no | Structured reason from `SlackGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `LeadRecord` (entity state) | `leadId` | `String` | no | — |
| | `formData` | `Optional<LeadFormData>` | yes | Populated after `LeadSubmitted`. |
| | `profile` | `Optional<FirmographicProfile>` | yes | Populated after `EnrichmentCompleted`. |
| | `score` | `Optional<QualificationScore>` | yes | Populated after `QualificationScored`. |
| | `notification` | `Optional<SlackPayload>` | yes | Populated after `NotificationSent`. |
| | `eval` | `Optional<EvalRecord>` | yes | Populated after `EvaluationRecorded`. |
| | `status` | `LeadStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `LeadSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `LeadRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`LeadStatus`: `SUBMITTED`, `ENRICHING`, `ENRICHED`, `QUALIFYING`, `QUALIFIED`, `NOTIFYING`, `NOTIFIED`, `EVALUATED`, `FAILED`.

`CompanySize`: `XS` (1–10), `S` (11–50), `M` (51–200), `L` (201–1 000), `XL` (1 001+).

`LeadTier`: `HOT`, `WARM`, `COLD`.

`EvalConfidence`: `HIGH`, `MED`, `LOW`.

`LeadPhase` (used by `@FunctionTool` annotations and `SlackGuardrail`): `ENRICH`, `QUALIFY`, `NOTIFY`.

## Events (`LeadEntity`)

| Event | Payload | Transition |
|---|---|---|
| `LeadSubmitted` | `formData: LeadFormData` | → SUBMITTED |
| `EnrichStarted` | — | → ENRICHING |
| `EnrichmentCompleted` | `profile: FirmographicProfile` | → ENRICHED |
| `QualifyStarted` | — | → QUALIFYING |
| `QualificationScored` | `score: QualificationScore` | → QUALIFIED |
| `NotifyStarted` | — | → NOTIFYING |
| `NotificationSent` | `notification: SlackPayload` | → NOTIFIED |
| `EvaluationRecorded` | `eval: EvalRecord` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `phase, tool, reason, rejectedAt` | no status change (audit-only) |
| `LeadFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `LeadRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`LeadRow` mirrors `LeadRecord` exactly. The UI fetches the full row via `GET /api/leads/{id}` and streams updates via `GET /api/leads/sse`.

The view declares ONE query: `getAllLeads: SELECT * AS leads FROM lead_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`LeadTasks.java`)

```java
public final class LeadTasks {
  public static final Task<FirmographicProfile> ENRICH_LEAD = Task
      .name("Enrich lead")
      .description("Look up firmographic data and tech-stack signals for the submitted company domain")
      .resultConformsTo(FirmographicProfile.class);

  public static final Task<QualificationScore> QUALIFY_LEAD = Task
      .name("Qualify lead")
      .description("Score the enriched profile against the qualification rubric and assign a sales rep")
      .resultConformsTo(QualificationScore.class);

  public static final Task<SlackPayload> NOTIFY_SALES = Task
      .name("Notify sales")
      .description("Build a Slack message block and post it to the assigned rep's channel")
      .resultConformsTo(SlackPayload.class);

  private LeadTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Phase-tagged tools

Each `@FunctionTool` method on `EnrichTools`, `QualifyTools`, and `NotifyTools` carries a `LeadPhase` constant. `SlackGuardrail` specifically intercepts calls where the tool name is `postLeadToSlack` and validates the `LeadEntity` state. All other tools pass through unconditionally; phase isolation is enforced by the agent's task instructions, not by additional guardrail conditions on non-Slack tools.

## PII sanitizer tokens

`PiiSanitizer` generates stable pseudonymous tokens from `sha1(leadId + sessionSalt)` truncated to 8 hex chars:

| Original field | Token form |
|---|---|
| `firstName` | `[NAME-1]` |
| `lastName` | `[NAME-2]` |
| `email` | `[EMAIL-1]` |
| Personal company contact | `[CONTACT-1]` |

The mapping is held in memory per session; it is not written to disk and is not logged.
