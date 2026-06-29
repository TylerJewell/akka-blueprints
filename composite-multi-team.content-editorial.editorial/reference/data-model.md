# Data model — editorial-desk

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `StoryBrief` | `storyId` | `String` | no | Id assigned at submission. |
| | `topic` | `String` | no | The story topic. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `EditorialBrief` | `angle` | `String` | no | One-sentence framing from the editor-in-chief. |
| | `keyQuestions` | `List<String>` | no | Questions the research must answer. |
| | `targetSections` | `List<String>` | no | Section titles the article should carry. |
| `ResearchPlan` | `subtopics` | `List<String>` | no | Subtopics, one per researcher instance. |
| `ResearchNote` | `subtopic` | `String` | no | The subtopic this note covers. |
| | `findings` | `String` | no | One short paragraph of findings. |
| | `sources` | `List<String>` | no | One to three sources. |
| `ResearchDigest` | `summary` | `String` | no | One-paragraph synthesis. |
| | `keyFacts` | `List<String>` | no | Four to six facts writers can rely on. |
| `SectionSpec` | `title` | `String` | no | Short section heading. |
| | `brief` | `String` | no | One or two sentences on what the section covers. |
| `SectionPlan` | `approach` | `String` | no | One sentence on the article structure. |
| | `sections` | `List<SectionSpec>` | no | Three to five section specs. |
| `SectionDraft` | `sectionId` | `String` | no | The section this content fills. |
| | `title` | `String` | no | Section heading. |
| | `content` | `String` | no | One to three paragraphs (non-empty). |
| | `wordCount` | `int` | no | Word count of the content. |
| `Draft` | `headline` | `String` | no | Working headline for the assembled draft. |
| | `sections` | `List<SectionDraft>` | no | The written sections, in order. |
| | `wordCount` | `int` | no | Total word count. |
| `ReviewNote` | `axis` | `String` | no | The axis reviewed (`factcheck`/`style`/`legal`). |
| | `outcome` | `ReviewOutcome` | no | `PASS` or `REVISE`. |
| | `comments` | `String` | no | For a `REVISE`, names the section needing work. |
| `ReviewVerdict` | `outcome` | `ReviewOutcome` | no | Aggregated `PASS` only if every note passes. |
| | `notes` | `List<ReviewNote>` | no | The panel's notes. |
| | `mustRevise` | `List<String>` | no | Section titles to redo (empty on `PASS`). |
| `Article` | `headline` | `String` | no | Published headline. |
| | `body` | `String` | no | The assembled article body. |
| | `byline` | `String` | no | Attribution line (non-empty; gated by G1). |
| | `assembledAt` | `Instant` | no | When the editor-in-chief assembled it. |
| `StageEval` | `stage` | `String` | no | `research` / `draft` / `review`. |
| | `score` | `int` | no | Quality score 0-100 from `StageEvaluator`. |
| | `flags` | `List<String>` | no | Issues found (may be empty). |
| | `evaluatedAt` | `Instant` | no | When the eval ran. |
| `ComplianceReview` | `reviewedBy` | `String` | no | Compliance officer id. |
| | `outcome` | `ComplianceOutcome` | no | `CLEARED` or `FLAGGED`. |
| | `comments` | `String` | no | Post-publication review notes. |
| | `reviewedAt` | `Instant` | no | When recorded. |

## Entity state — `Document` (`DocumentEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `storyId` | `String` | no | Unique id; also the `EditorialWorkflow` id. |
| `topic` | `String` | no | Submitted topic. |
| `requestedBy` | `String` | no | Submitter id. |
| `status` | `StoryStatus` | no | See enum. |
| `brief` | `Optional<EditorialBrief>` | yes | Populated on `BriefAssigned`. |
| `researchDigest` | `Optional<ResearchDigest>` | yes | Populated on `ResearchSynthesized`. |
| `sectionIds` | `List<String>` | no | Section ids created from the plan (may be empty early). |
| `draft` | `Optional<Draft>` | yes | Populated on `DraftAssembled`. |
| `reviewVerdict` | `Optional<ReviewVerdict>` | yes | Populated on `ReviewCompleted`. |
| `article` | `Optional<Article>` | yes | Populated on `ArticlePublished`. |
| `publishedUrl` | `Optional<String>` | yes | Generated URL on publish. |
| `publishedAt` | `Optional<Instant>` | yes | When published. |
| `stageEvals` | `List<StageEval>` | no | Eval results recorded by `StageEvalConsumer` (may be empty). |
| `complianceReview` | `Optional<ComplianceReview>` | yes | Populated on `ComplianceReviewRecorded`. |
| `revisionCount` | `int` | no | Number of revision rounds taken (0 or 1). |
| `createdAt` | `Instant` | no | When `DocumentCreated` emitted. |

## Entity state — `Section` (`SectionEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `sectionId` | `String` | no | Deterministic id `storyId + "-s" + index`. |
| `storyId` | `String` | no | Owning story. |
| `title` | `String` | no | From the `SectionSpec`. |
| `brief` | `String` | no | From the `SectionSpec`. |
| `status` | `SectionStatus` | no | See enum. |
| `claimedBy` | `Optional<String>` | yes | Writer that won the claim. |
| `claimedAt` | `Optional<Instant>` | yes | When claimed. |
| `content` | `Optional<String>` | yes | Written section content. |
| `wordCount` | `Optional<Integer>` | yes | Word count of the content. |
| `writtenAt` | `Optional<Instant>` | yes | When marked written. |
| `createdAt` | `Instant` | no | When `SectionCreated` emitted. |

## Enums

`StoryStatus`: `SUBMITTED`, `ASSIGNED`, `RESEARCHING`, `RESEARCHED`, `WRITING`, `DRAFTED`, `REVIEWING`, `APPROVED`, `PUBLISHED`.
`SectionStatus`: `OPEN`, `CLAIMED`, `WRITTEN`.
`ReviewOutcome`: `PASS`, `REVISE`.
`ComplianceOutcome`: `CLEARED`, `FLAGGED`.

## Events — `DocumentEntity`

| Event | Payload | Transition |
|---|---|---|
| `DocumentCreated` | `storyId, topic, requestedBy, createdAt` | → SUBMITTED |
| `BriefAssigned` | `brief` | SUBMITTED → ASSIGNED |
| `ResearchSynthesized` | `digest` | RESEARCHING → RESEARCHED (also marks ASSIGNED → RESEARCHING when research begins) |
| `SectionsPlanned` | `sectionIds` | RESEARCHED → WRITING |
| `DraftAssembled` | `draft` | WRITING → DRAFTED |
| `ReviewCompleted` | `verdict` | DRAFTED/REVIEWING → APPROVED (on PASS) |
| `RevisionRequested` | `mustRevise, revisionCount` | REVIEWING → WRITING (one bounded round) |
| `ArticlePublished` | `article, publishedUrl, publishedAt` | APPROVED → PUBLISHED |
| `StageEvaluated` | `stageEval` | (no status change; appends to `stageEvals`) |
| `ComplianceReviewRecorded` | `complianceReview` | (no status change; PUBLISHED only) |

Note: the implementation may emit an explicit `ResearchStarted` for the `ASSIGNED → RESEARCHING` edge or fold it into the first research write — prefer whichever keeps the projection and the App UI status consistent. A publish blocked by the G1 guardrail records its reason via a `recordPublishBlock` command (no status change) and leaves the document `APPROVED`.

## Events — `SectionEntity`

| Event | Payload | Transition |
|---|---|---|
| `SectionCreated` | `sectionId, storyId, title, brief, createdAt` | → OPEN |
| `SectionClaimed` | `writerId, claimedAt` | OPEN → CLAIMED (emitted only when current status is OPEN) |
| `SectionWritten` | `content, wordCount, writtenAt` | CLAIMED → WRITTEN |
| `SectionReleased` | `releasedAt` | CLAIMED → OPEN, clears `claimedBy` |

## Events — `StoryQueue`

| Event | Payload |
|---|---|
| `StorySubmitted` | `storyId, topic, requestedBy, submittedAt` |

## View rows

`DocumentRow` (row of `DocumentBoardView`) mirrors `Document` but drops the heavy `Article.body` and the `ResearchNote` contents — it keeps the brief angle, the digest summary, `sectionIds`, the draft headline and word count, the verdict outcome, the `stageEvals`, the `publishedUrl`, and the `complianceReview`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`SectionRow` (row of `SectionBoardView`) mirrors `Section` but drops the heavy `content` — it keeps `wordCount` and a `contentPresent` boolean so the writers' poll and the board stay small. Every nullable lifecycle field is `Optional<T>`.

Both views expose one unfiltered query (`getAllDocuments` / `getAllSections`) plus a streaming query for the SSE endpoints. Status filtering happens client-side because the views cannot auto-index the enum status column (Lesson 2).
