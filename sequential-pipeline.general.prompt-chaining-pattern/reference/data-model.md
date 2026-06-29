# Data model — prompt-chaining-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Section` | `sectionId` | `String` | no | Stable slug, unique within an `Outline`. |
| | `heading` | `String` | no | 3-8-word section title. |
| | `description` | `String` | no | One-sentence description of the section's content. |
| `Outline` | `sections` | `List<Section>` | no | 2-5 sections. |
| | `documentTitle` | `String` | no | Proposed document title derived from the prompt. |
| | `outlinedAt` | `Instant` | no | When the OUTLINE task returned. |
| `Citation` | `citationId` | `String` | no | Stable short id (`cit-<8 hex>`). |
| | `label` | `String` | no | Display name for the source. |
| | `url` | `String` | no | Canonical URL of the source entry. |
| | `snippet` | `String` | no | Quoted passage from the source. |
| `SectionDraft` | `sectionId` | `String` | no | MUST equal a `Section.sectionId` from the upstream `Outline`. |
| | `heading` | `String` | no | Section heading (mirrors the outline heading). |
| | `body` | `String` | no | 3-5 sentences of body content. |
| | `citationIds` | `List<String>` | no | Each entry MUST equal a `Citation.citationId` from the same `Draft`. |
| `Draft` | `sectionDrafts` | `List<SectionDraft>` | no | One per `Outline.section`. |
| | `citations` | `List<Citation>` | no | All citations gathered during the DRAFT step. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned. |
| `RefinedSection` | `sectionId` | `String` | no | MUST equal a `SectionDraft.sectionId` from the upstream `Draft`. |
| | `heading` | `String` | no | Polished section heading. |
| | `body` | `String` | no | Polished 3-5 sentence body text with inline citation markers. |
| | `citationMarkers` | `List<String>` | no | Citation labels referenced inline in body. |
| `RefinedDocument` | `title` | `String` | no | Final document title. |
| | `abstract_` | `String` | no | 1-2 sentence abstract. |
| | `sections` | `List<RefinedSection>` | no | `sections.size() == outline.sections.size()` (E1 rule 4). |
| | `bibliography` | `List<Citation>` | no | Every entry MUST appear in `Draft.citations` (E1 rule 3). Non-empty (G1 check 3). |
| | `refinedAt` | `Instant` | no | When the REFINE task returned and the guardrail accepted. |
| `QualityResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `QualityScorer` finished. |
| `GuardrailRejection` | `check` | `String` | no | `section-count` / `word-count` / `citation-presence`. |
| | `reason` | `String` | no | Structured reason from `OutputGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `DraftRecord` (entity state) | `draftId` | `String` | no | — |
| | `prompt` | `Optional<String>` | yes | Populated after `DraftCreated`. |
| | `outline` | `Optional<Outline>` | yes | Populated after `OutlineProduced`. |
| | `draft` | `Optional<Draft>` | yes | Populated after `DraftWritten`. |
| | `refinedDocument` | `Optional<RefinedDocument>` | yes | Populated after `RefinementApplied`. |
| | `quality` | `Optional<QualityResult>` | yes | Populated after `QualityScored`. |
| | `status` | `DraftStatus` | no | See enum. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |
| | `createdAt` | `Instant` | no | When `DraftCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `DraftRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`DraftStatus`: `CREATED`, `OUTLINING`, `OUTLINED`, `DRAFTING`, `DRAFTED`, `REFINING`, `REFINED`, `EVALUATED`, `FAILED`.

## Events (`DraftEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DraftCreated` | `prompt: String` | → CREATED |
| `OutlineStarted` | — | → OUTLINING |
| `OutlineProduced` | `outline: Outline` | → OUTLINED |
| `DraftStarted` | — | → DRAFTING |
| `DraftWritten` | `draft: Draft` | → DRAFTED |
| `RefineStarted` | — | → REFINING |
| `RefinementApplied` | `refinedDocument: RefinedDocument` | → REFINED |
| `QualityScored` | `quality: QualityResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `check, reason, rejectedAt` | no status change (audit-only) |
| `DraftFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `DraftRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`DraftRow` mirrors `DraftRecord` exactly. The UI fetches the full row via `GET /api/drafts/{id}` and streams updates via `GET /api/drafts/sse`.

The view declares ONE query: `getAllDrafts: SELECT * AS drafts FROM draft_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`DraftingTasks.java`)

```java
public final class DraftingTasks {
  public static final Task<Outline> OUTLINE_DOCUMENT = Task
      .name("Outline document")
      .description("Structure the prompt into 2-5 logical sections with headings and brief descriptions")
      .resultConformsTo(Outline.class);

  public static final Task<Draft> DRAFT_DOCUMENT = Task
      .name("Draft document")
      .description("Write body paragraphs for each outline section and gather supporting citations")
      .resultConformsTo(Draft.class);

  public static final Task<RefinedDocument> REFINE_DOCUMENT = Task
      .name("Refine document")
      .description("Polish each section for clarity, add citation markers, and produce the final RefinedDocument")
      .resultConformsTo(RefinedDocument.class);

  private DraftingTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Output guardrail checks

`OutputGuardrail` runs after the REFINE_DOCUMENT task returns. Three checks, any failure rejects the response:

1. `section-count` — `refinedDocument.sections.size() >= 2`.
2. `word-count` — sum of word counts across all `RefinedSection.body` fields >= 150.
3. `citation-presence` — `refinedDocument.bibliography.size() >= 1`.

On rejection, the guardrail calls `DraftEntity.recordGuardrailRejection(check, reason)` and returns the structured rejection to `DraftingWorkflow.refineStep`. The workflow retries within a 2-retry budget, appending the rejection reason to the next task's instruction context.

## QualityScorer checks

Four checks, one point per check satisfied, starting from a base of 1:

1. Outline coverage — every `Outline.section.sectionId` has a matching `SectionDraft.sectionId` in `Draft.sectionDrafts`.
2. Draft coverage — every `SectionDraft.sectionId` appears as a `RefinedSection.sectionId` in `RefinedDocument.sections`.
3. Citation provenance — every `Citation` in `RefinedDocument.bibliography` has a matching `citationId` in `Draft.citations`.
4. Structural parity — `refinedDocument.sections.size() == outline.sections.size()`.
