# Data model — deterministic-multi-stage-agent-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Beat` | `beatId` | `String` | no | Unique slug within an `Outline` (e.g. `beat-01`). |
| | `text` | `String` | no | One sentence describing the plot beat. Non-empty. |
| | `position` | `int` | no | 0-indexed position in the outline sequence. |
| `Outline` | `title` | `String` | no | Story title. |
| | `genre` | `String` | no | Classified genre (e.g. `literary-fiction`). Non-empty after guardrail acceptance. |
| | `beats` | `List<Beat>` | no | At least 2 entries (G1 rule). |
| | `outlinedAt` | `Instant` | no | When the OUTLINE task returned. |
| `Paragraph` | `paragraphId` | `String` | no | Unique stable id (e.g. `p-<8 hex>`). |
| | `beatId` | `String` | no | MUST equal a `Beat.beatId` from the upstream `Outline` (G1 rule). |
| | `text` | `String` | no | 2–4 sentences expanding the beat. Non-empty. |
| `Body` | `paragraphs` | `List<Paragraph>` | no | At least `outline.beats.size()` entries after guardrail acceptance. |
| | `writtenAt` | `Instant` | no | When the WRITE_BODY task returned. |
| `Arc` | `arcId` | `String` | no | Short slug identifying the narrative arc. |
| | `label` | `String` | no | Human-readable arc name. |
| | `paragraphIds` | `List<String>` | no | Each entry MUST appear in `Body.paragraphs[].paragraphId` (G1 rule). |
| `Ending` | `closingText` | `String` | no | Non-empty closing paragraph. |
| | `arcs` | `List<Arc>` | no | At least 1 entry after guardrail acceptance. |
| | `endingWrittenAt` | `Instant` | no | When the WRITE_ENDING task returned. |
| `ValidationResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `validatedAt` | `Instant` | no | When `StoryStructureValidator` finished. |
| `GuardrailRejection` | `stage` | `String` | no | `OUTLINE` / `WRITE_BODY` / `WRITE_ENDING`. |
| | `field` | `String` | no | Name of the field that violated the constraint. |
| | `reason` | `String` | no | Structured reason from `StageOutputGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `StoryRecord` (entity state) | `storyId` | `String` | no | — |
| | `prompt` | `Optional<String>` | yes | Populated after `StoryCreated`. |
| | `outline` | `Optional<Outline>` | yes | Populated after `OutlineProduced`. |
| | `body` | `Optional<Body>` | yes | Populated after `BodyWritten`. |
| | `ending` | `Optional<Ending>` | yes | Populated after `EndingWritten`. |
| | `validation` | `Optional<ValidationResult>` | yes | Populated after `StoryValidated`. |
| | `status` | `StoryStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `StoryCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `StoryRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`StoryStatus`: `CREATED`, `OUTLINING`, `OUTLINED`, `BODY_WRITING`, `BODY_WRITTEN`, `ENDING_WRITING`, `ENDING_WRITTEN`, `VALIDATED`, `FAILED`.

`Stage` (used by `@FunctionTool` annotations and `StageOutputGuardrail`): `OUTLINE`, `WRITE_BODY`, `WRITE_ENDING`.

## Events (`StoryEntity`)

| Event | Payload | Transition |
|---|---|---|
| `StoryCreated` | `prompt: String` | → CREATED |
| `OutlineStarted` | — | → OUTLINING |
| `OutlineProduced` | `outline: Outline` | → OUTLINED |
| `BodyStarted` | — | → BODY_WRITING |
| `BodyWritten` | `body: Body` | → BODY_WRITTEN |
| `EndingStarted` | — | → ENDING_WRITING |
| `EndingWritten` | `ending: Ending` | → ENDING_WRITTEN |
| `StoryValidated` | `validation: ValidationResult` | → VALIDATED (terminal happy) |
| `GuardrailRejected` | `stage, field, reason, rejectedAt` | no status change (audit-only) |
| `StoryFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `StoryRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`StoryRow` mirrors `StoryRecord` exactly. The UI fetches the full row via `GET /api/stories/{id}` and streams updates via `GET /api/stories/sse`.

The view declares ONE query: `getAllStories: SELECT * AS stories FROM story_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`StoryTasks.java`)

```java
public final class StoryTasks {
  public static final Task<Outline> OUTLINE_STORY = Task
      .name("Outline story")
      .description("Build a structured beat outline for the prompt by calling generateBeats and classifyGenre")
      .resultConformsTo(Outline.class);

  public static final Task<Body> WRITE_BODY = Task
      .name("Write body")
      .description("Expand each outline beat into a paragraph by calling expandBeat and linkParagraphs")
      .resultConformsTo(Body.class);

  public static final Task<Ending> WRITE_ENDING = Task
      .name("Write ending")
      .description("Resolve narrative arcs and compose a closing paragraph by calling resolveArcs and composeClosure")
      .resultConformsTo(Ending.class);

  private StoryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Stage-tagged tools

Each `@FunctionTool` method on `OutlineTools`, `BodyTools`, and `EndingTools` carries a `Stage` constant. `StageOutputGuardrail` runs after the agent task returns its typed result and validates structural invariants before the workflow advances. The tool registry is built once at startup; the guardrail reads stage metadata from the `TaskDef` for every validation call.
