# Data model — chain-workflow

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Fact` | `factId` | `String` | no | Short stable id (`f-<8 hex>`). |
| | `text` | `String` | no | The extracted fact, as stated in the raw input. |
| | `category` | `String` | no | One of: `feature`, `limitation`, `requirement`, `context`. |
| | `sourceSpan` | `String` | no | Character-offset range in the raw input, e.g. `"0-52"`. |
| `ExtractedStructure` | `facts` | `List<Fact>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `extractedAt` | `Instant` | no | When the EXTRACT task returned. |
| `RefinedFact` | `refinedId` | `String` | no | Short stable id (`r-<8 hex>`). |
| | `text` | `String` | no | Polished restatement of the fact. |
| | `sourceFactId` | `String` | no | MUST equal a `Fact.factId` from the upstream `ExtractedStructure`. |
| | `polishNote` | `String` | no | One-sentence note describing the editorial change. |
| `RefinedContent` | `refinedFacts` | `List<RefinedFact>` | no | Possibly empty. |
| | `refinedAt` | `Instant` | no | When the REFINE task returned. |
| `CitedFact` | `refinedId` | `String` | no | MUST equal a `RefinedFact.refinedId` from the upstream `RefinedContent`. |
| | `text` | `String` | no | Display text of the cited refined fact. |
| `Section` | `sectionId` | `String` | no | Slug derived from the heading. |
| | `heading` | `String` | no | Section heading. |
| | `body` | `String` | no | 2–4 sentences paraphrasing the grouped facts. |
| | `citations` | `List<CitedFact>` | no | Non-empty (E1 rule 2). |
| `Document` | `title` | `String` | no | 1-line title. Non-empty (G1 FORMAT check). |
| | `summary` | `String` | no | 2–3 sentence summary. |
| | `sections` | `List<Section>` | no | At least one section per logical content group. |
| | `formattedAt` | `Instant` | no | When the FORMAT task returned. |
| `QualityResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `QualityScorer` finished. |
| `GuardrailRejection` | `stage` | `String` | no | `EXTRACT` / `REFINE` / `FORMAT`. |
| | `failedField` | `String` | no | Name of the field that failed validation. |
| | `reason` | `String` | no | Structured reason from `OutputGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `DocumentRecord` (entity state) | `documentId` | `String` | no | — |
| | `rawInput` | `Optional<String>` | yes | Populated after `DocumentCreated`. |
| | `extracted` | `Optional<ExtractedStructure>` | yes | Populated after `StructureExtracted`. |
| | `refined` | `Optional<RefinedContent>` | yes | Populated after `ContentRefined`. |
| | `document` | `Optional<Document>` | yes | Populated after `DocumentFormatted`. |
| | `quality` | `Optional<QualityResult>` | yes | Populated after `QualityScored`. |
| | `status` | `DocumentStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `DocumentCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `DocumentRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`DocumentStatus`: `CREATED`, `EXTRACTING`, `EXTRACTED`, `REFINING`, `REFINED`, `FORMATTING`, `FORMATTED`, `EVALUATED`, `FAILED`.

`Stage` (used by `OutputGuardrail` and `TaskDef.metadata`): `EXTRACT`, `REFINE`, `FORMAT`.

## Events (`DocumentEntity`)

| Event | Payload | Transition |
|---|---|---|
| `DocumentCreated` | `rawInput: String` | → CREATED |
| `ExtractStarted` | — | → EXTRACTING |
| `StructureExtracted` | `extracted: ExtractedStructure` | → EXTRACTED |
| `RefineStarted` | — | → REFINING |
| `ContentRefined` | `refined: RefinedContent` | → REFINED |
| `FormatStarted` | — | → FORMATTING |
| `DocumentFormatted` | `document: Document` | → FORMATTED |
| `QualityScored` | `quality: QualityResult` | → EVALUATED (terminal happy) |
| `GuardrailRejected` | `stage, failedField, reason, rejectedAt` | no status change (audit-only) |
| `DocumentFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `DocumentRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`DocumentRow` mirrors `DocumentRecord` exactly. The UI fetches the full row via `GET /api/documents/{id}` and streams updates via `GET /api/documents/sse`.

The view declares ONE query: `getAllDocuments: SELECT * AS documents FROM document_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`TransformTasks.java`)

```java
public final class TransformTasks {
  public static final Task<ExtractedStructure> EXTRACT_STRUCTURE = Task
      .name("Extract structure")
      .description("Parse raw input text into typed Facts using parseInput and classifyFacts")
      .resultConformsTo(ExtractedStructure.class);

  public static final Task<RefinedContent> REFINE_CONTENT = Task
      .name("Refine content")
      .description("Polish and deduplicate facts into RefinedFacts using polishFact and mergeDuplicates")
      .resultConformsTo(RefinedContent.class);

  public static final Task<Document> FORMAT_DOCUMENT = Task
      .name("Format document")
      .description("Compose a Document whose Sections group the refined facts using buildSection and composeSummary")
      .resultConformsTo(Document.class);

  private TransformTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Output validation matrix (`OutputGuardrail`)

Each stage's accept rules, applied after the LLM returns but before the workflow records the event:

| Stage | Field | Rule |
|---|---|---|
| EXTRACT | `facts` | `size() >= 1` |
| EXTRACT | `facts[i].factId` | non-empty |
| REFINE | `refinedFacts[i].sourceFactId` | must equal a `Fact.factId` in the recorded `ExtractedStructure` |
| FORMAT | `sections[i].citations` | non-empty |
| FORMAT | `document.title` | non-empty |

On any rule violation the guardrail returns a structured `output-validation-failure` rejection and records `GuardrailRejected{stage, failedField, reason}` on the entity. The agent loop retries within its 4-iteration budget.
