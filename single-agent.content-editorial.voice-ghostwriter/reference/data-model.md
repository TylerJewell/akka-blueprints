# Data model — ghostwriter

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `VoiceSample` | `sampleId` | `String` | no | Stable id supplied by the user. |
| | `title` | `String` | no | Human-readable label for the sample. |
| | `rawBody` | `String` | no | Pre-sanitization sample text. Audit-only. |
| | `format` | `String` | no | Content format: `blog-post`, `memo`, `release-note`, `social`. |
| `WritingBrief` | `draftId` | `String` | no | UUID minted by `DraftEndpoint`. |
| | `voiceOwnerId` | `String` | no | Identifier of the person whose voice is being replicated. |
| | `topic` | `String` | no | The subject the draft should cover. |
| | `format` | `String` | no | Content format for the output draft. |
| | `targetWordCount` | `int` | no | Desired draft length (±15% acceptable). |
| | `samples` | `List<VoiceSample>` | no | Submitted writing samples (1–N). |
| | `requestedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedCorpus` | `samples` | `List<SanitizedSample>` | no | One per submitted `VoiceSample`. |
| | `piiCategoriesFound` | `List<String>` | no | Union of categories found across all samples, e.g. `["email","person-name","ein"]`. |
| `SanitizedSample` | `sampleId` | `String` | no | Matches the source `VoiceSample.sampleId`. |
| | `redactedBody` | `String` | no | PII-redacted body; this is what the agent sees. |
| `StyleMarker` | `token` | `String` | no | Name of the style pattern, e.g. `"em-dash mid-sentence"`. |
| | `evidence` | `String` | no | Short quote from a source sample where the pattern was observed. |
| `DraftResult` | `draftBody` | `String` | no | The complete generated draft text. |
| | `fidelityScore` | `int` | no | 0–100; agent's self-assessment of voice fidelity. |
| | `markers` | `List<StyleMarker>` | no | 2–6 style patterns identified and replicated. |
| | `guardrailRejections` | `int` | no | Count of iterations rejected by `DraftOutputGuardrail`. |
| | `generatedAt` | `Instant` | no | When the agent returned. |
| `Draft` (entity state) | `draftId` | `String` | no | — |
| | `brief` | `Optional<WritingBrief>` | yes | Populated after `BriefSubmitted`. |
| | `sanitizedCorpus` | `Optional<SanitizedCorpus>` | yes | Populated after `CorpusSanitized`. |
| | `result` | `Optional<DraftResult>` | yes | Populated after `DraftReady`. |
| | `status` | `DraftStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `BriefSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Draft` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`DraftStatus`: `SUBMITTED`, `SANITIZED`, `DRAFTING`, `DRAFT_READY`, `FAILED`.

## Events (`DraftEntity`)

| Event | Payload | Transition |
|---|---|---|
| `BriefSubmitted` | `brief` | → SUBMITTED |
| `CorpusSanitized` | `sanitizedCorpus` | → SANITIZED |
| `DraftingStarted` | — | → DRAFTING |
| `DraftReady` | `result` | → DRAFT_READY (terminal happy) |
| `DraftFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Draft.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`DraftRow` mirrors `Draft` minus `brief.samples[].rawBody` (the audit log keeps that). The UI fetches the raw sample on demand via `GET /api/drafts/{id}` and reads `brief.samples[].rawBody` from the JSON.

The view declares ONE query: `getAllDrafts: SELECT * AS drafts FROM draft_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`DraftTasks.java`)

```java
public final class DraftTasks {
  public static final Task<DraftResult> GENERATE_DRAFT = Task
      .name("Generate draft")
      .description("Read the attached writing samples and produce a DraftResult matching "
          + "the voice owner's style for the given brief")
      .resultConformsTo(DraftResult.class);

  private DraftTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
