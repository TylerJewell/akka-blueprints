# Data model — blog-writer-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Reference` | `source` | `String` | no | Short name of the originating reference. |
| | `url` | `String` | no | Canonical URL of the reference. |
| | `keyInsight` | `String` | no | One-sentence takeaway retrieved by the RESEARCH phase. |
| | `capturedAt` | `Instant` | no | When the RESEARCH phase recorded the reference. |
| `ResearchNotes` | `references` | `List<Reference>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `topicSummary` | `String` | no | One-paragraph summary of the topic based on collected references. |
| | `researchedAt` | `Instant` | no | When the RESEARCH task returned. |
| `OutlineSection` | `sectionId` | `String` | no | Short slug. MUST be unique within an `Outline`. |
| | `heading` | `String` | no | Human-readable section heading. |
| | `keyPoints` | `List<String>` | no | 2-3 bullet strings drawn from matching reference insights. |
| `Outline` | `proposedTitle` | `String` | no | Draft title agreed at outline stage. |
| | `introduction` | `String` | no | 1-2 sentence framing paragraph. |
| | `sections` | `List<OutlineSection>` | no | Possibly empty. |
| | `outlinedAt` | `Instant` | no | When the OUTLINE task returned. |
| `PostSection` | `sectionId` | `String` | no | MUST equal an `OutlineSection.sectionId` from the input `Outline`. |
| | `heading` | `String` | no | Section heading (matches outline heading). |
| | `body` | `String` | no | ≥ 50 words (E1 rule 2). |
| `BlogPost` | `title` | `String` | no | Levenshtein distance from `Outline.proposedTitle` ≤ 20 (E1 rule 3). |
| | `introduction` | `String` | no | 2-4 sentence opening paragraph. |
| | `sections` | `List<PostSection>` | no | `sections.size() == outline.sections.size()` (E1 rule 4). |
| | `conclusion` | `String` | no | 1-paragraph conclusion with at least one CTA sentence (G1 rule 3). |
| | `style` | `String` | no | `Technical` / `Conversational` / `Thought leadership`. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned and the guardrail accepted. |
| `QualityResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `QualityScorer` finished. |
| `GuardrailRejection` | `rule` | `String` | no | `word-count` / `forbidden-phrase` / `cta-missing`. |
| | `detail` | `String` | no | Structured detail from `BrandGuardrail` naming the specific violation. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `BlogPostRecord` (entity state) | `postId` | `String` | no | — |
| | `topic` | `Optional<String>` | yes | Populated after `PostCreated`. |
| | `style` | `Optional<String>` | yes | Populated after `PostCreated`. |
| | `research` | `Optional<ResearchNotes>` | yes | Populated after `ResearchCollected`. |
| | `outline` | `Optional<Outline>` | yes | Populated after `OutlineProduced`. |
| | `post` | `Optional<BlogPost>` | yes | Populated after `DraftWritten`. |
| | `quality` | `Optional<QualityResult>` | yes | Populated after `QualityChecked`. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `GuardrailRejected` event; empty on the happy path. |
| | `status` | `BlogPostStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PostCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable lifecycle field on `BlogPostRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`BlogPostStatus`: `CREATED`, `RESEARCHING`, `RESEARCHED`, `OUTLINING`, `OUTLINED`, `DRAFTING`, `DRAFTED`, `QUALITY_CHECKED`, `FAILED`.

## Events (`BlogPostEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PostCreated` | `topic: String, style: String` | → CREATED |
| `ResearchStarted` | — | → RESEARCHING |
| `ResearchCollected` | `research: ResearchNotes` | → RESEARCHED |
| `OutlineStarted` | — | → OUTLINING |
| `OutlineProduced` | `outline: Outline` | → OUTLINED |
| `DraftStarted` | — | → DRAFTING |
| `DraftWritten` | `post: BlogPost` | → DRAFTED |
| `QualityChecked` | `quality: QualityResult` | → QUALITY_CHECKED (terminal happy) |
| `GuardrailRejected` | `rule, detail, rejectedAt` | no status change (audit-only) |
| `PostFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `BlogPostRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`BlogPostRow` mirrors `BlogPostRecord` exactly. The UI fetches the full row via `GET /api/posts/{id}` and streams updates via `GET /api/posts/sse`.

The view declares ONE query: `getAllPosts: SELECT * AS posts FROM blog_post_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`BlogTasks.java`)

```java
public final class BlogTasks {
  public static final Task<ResearchNotes> RESEARCH_TOPIC = Task
      .name("Research topic")
      .description("Gather references and key insights about a topic by calling searchTopicReferences and fetchKeyPoints")
      .resultConformsTo(ResearchNotes.class);

  public static final Task<Outline> OUTLINE_POST = Task
      .name("Outline post")
      .description("Generate section headings and assign key points from research notes into a structured Outline")
      .resultConformsTo(Outline.class);

  public static final Task<BlogPost> DRAFT_POST = Task
      .name("Draft post")
      .description("Write a full BlogPost whose PostSections mirror the Outline sections one-to-one")
      .resultConformsTo(BlogPost.class);

  private BlogTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Brand guardrail rules

`BrandGuardrail` checks the candidate `BlogPost` on every DRAFT response before `DraftWritten` is emitted. Three rules, applied in order:

1. **word-count** — combined word count of `introduction` + all `sections[].body` + `conclusion` must be ≥ 300.
2. **forbidden-phrase** — full post text (lowercased) must not contain any phrase from `src/main/resources/brand-rules/forbidden-phrases.txt`. First match reported.
3. **cta-missing** — `conclusion` must contain at least one sentence with a CTA marker (case-insensitive match against: `learn more`, `get started`, `try`, `contact us`, `sign up`, or a sentence ending with `?`).

On the first failing rule the guardrail rejects with a structured message naming the rule and the excerpt. On full pass the candidate is forwarded unchanged.

## Quality scorer dimensions

`QualityScorer` runs after `DraftWritten`. Four checks, one point per check satisfied, base of 1:

1. **section-coverage** — every `Outline.sections[i].sectionId` has a matching `BlogPost.sections[j].sectionId`.
2. **section-depth** — every `BlogPost.sections[j].body` contains ≥ 50 words.
3. **title-fidelity** — Levenshtein distance between `BlogPost.title` and `Outline.proposedTitle` is ≤ 20.
4. **section-parity** — `post.sections.size() == outline.sections.size()`.

Score range 1–5. Rationale names the largest gap.
