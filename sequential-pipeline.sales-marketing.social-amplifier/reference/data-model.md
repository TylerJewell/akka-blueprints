# Data model — social-amplifier

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `KeyMessage` | `messageId` | `String` | no | Short stable id (`km-<3 digits>`). |
| | `text` | `String` | no | The extracted message, written for general audiences. |
| | `platformHint` | `String` | no | One of `LINKEDIN`, `X`, `BLUESKY`, or `ALL`. |
| `ParsedArticle` | `headline` | `String` | no | Article headline extracted by the PARSE phase. |
| | `sourceUrl` | `String` | no | The URL the article was fetched from. |
| | `keyMessages` | `List<KeyMessage>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `parsedAt` | `Instant` | no | When the PARSE task returned. |
| `Draft` | `draftId` | `String` | no | Short stable id (`d-<6 hex>`). |
| | `platform` | `Platform` | no | `LINKEDIN`, `X`, or `BLUESKY`. |
| | `text` | `String` | no | The drafted post text including inline hashtags. |
| | `hashtags` | `List<String>` | no | Extracted hashtag list (each entry starts with `#`). |
| | `characterCount` | `int` | no | MUST equal `text.length()`. Checked by G1 rule 1. |
| | `brandCheckAttempts` | `int` | no | 0 on first attempt; incremented on each G1 rejection. |
| `DraftSet` | `drafts` | `List<Draft>` | no | One entry per `targetPlatform`; possibly empty. |
| | `draftedAt` | `Instant` | no | When the DRAFT task returned a passing `DraftSet`. |
| `PublicationReceipt` | `receiptId` | `String` | no | Short stable id (`rcpt-<3 digits>`). |
| | `platform` | `Platform` | no | The platform the draft was published to. |
| | `postUrlStub` | `String` | no | `https://stub.example.com/<platform>/<receiptId>`. |
| | `publishedAt` | `Instant` | no | When the publish tool returned. |
| `PublishedSet` | `receipts` | `List<PublicationReceipt>` | no | One entry per `Draft` in the `DraftSet`. |
| | `publishedAt` | `Instant` | no | When the PUBLISH task returned. |
| `BrandAuditResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `auditedAt` | `Instant` | no | When `BrandPolicyScorer` finished. |
| `BrandCheckRejection` | `draftId` | `String` | no | The `Draft.draftId` that failed. |
| | `platform` | `Platform` | no | `LINKEDIN`, `X`, or `BLUESKY`. |
| | `rule` | `String` | no | `character-limit`, `forbidden-word`, `missing-hashtag`, or `informal-tone`. |
| | `reason` | `String` | no | Structured reason from `BrandPolicyGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the response-hook guardrail fired. |
| `AmplificationRecord` (entity state) | `amplificationId` | `String` | no | — |
| | `sourceUrl` | `Optional<String>` | yes | Populated after `AmplificationCreated`. |
| | `targetPlatforms` | `Optional<List<Platform>>` | yes | Populated after `AmplificationCreated`. |
| | `parsedArticle` | `Optional<ParsedArticle>` | yes | Populated after `ArticleParsed`. |
| | `draftSet` | `Optional<DraftSet>` | yes | Populated after `DraftsProduced`. |
| | `publishedSet` | `Optional<PublishedSet>` | yes | Populated after `PostsPublished`. |
| | `brandAudit` | `Optional<BrandAuditResult>` | yes | Populated after `BrandAuditCompleted`. |
| | `status` | `AmplificationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `AmplificationCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `brandCheckRejections` | `List<BrandCheckRejection>` | no | Appended on every `BrandCheckFailed` event; empty on the happy path. |

Every nullable lifecycle field on `AmplificationRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AmplificationStatus`: `CREATED`, `PARSING`, `PARSED`, `DRAFTING`, `DRAFTED`, `PUBLISHING`, `PUBLISHED`, `FAILED`.

`Platform`: `LINKEDIN`, `X`, `BLUESKY`.

`AmplifierPhase` (used by `BrandPolicyGuardrail` and tool dispatch): `PARSE`, `DRAFT`, `PUBLISH`.

## Events (`AmplificationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `AmplificationCreated` | `sourceUrl: String, targetPlatforms: List<Platform>` | → CREATED |
| `ParseStarted` | — | → PARSING |
| `ArticleParsed` | `parsedArticle: ParsedArticle` | → PARSED |
| `DraftStarted` | — | → DRAFTING |
| `DraftsProduced` | `draftSet: DraftSet` | → DRAFTED |
| `PublishStarted` | — | → PUBLISHING |
| `PostsPublished` | `publishedSet: PublishedSet` | → PUBLISHED |
| `BrandAuditCompleted` | `brandAudit: BrandAuditResult` | no status change (appended) |
| `BrandCheckFailed` | `draftId, platform, rule, reason, rejectedAt` | no status change (audit-only) |
| `PublishToolBlocked` | `platform, tool, reason, rejectedAt` | no status change (audit-only) |
| `AmplificationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `AmplificationRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `brandCheckRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AmplificationRow` mirrors `AmplificationRecord` exactly. The UI fetches the full row via `GET /api/amplifications/{id}` and streams updates via `GET /api/amplifications/sse`.

The view declares ONE query: `getAllAmplifications: SELECT * AS amplifications FROM amplification_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`AmplifierTasks.java`)

```java
public final class AmplifierTasks {
  public static final Task<ParsedArticle> PARSE_ARTICLE = Task
      .name("Parse article")
      .description("Fetch the article and extract key messages with platform hints")
      .resultConformsTo(ParsedArticle.class);

  public static final Task<DraftSet> DRAFT_POSTS = Task
      .name("Draft posts")
      .description("Draft one platform-tailored post per target platform")
      .resultConformsTo(DraftSet.class);

  public static final Task<PublishedSet> PUBLISH_POSTS = Task
      .name("Publish posts")
      .description("Publish each approved draft to its target platform and return receipts")
      .resultConformsTo(PublishedSet.class);

  private AmplifierTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Brand-policy rules (checked by `BrandPolicyGuardrail`)

Each rule maps to a score point in `BrandPolicyScorer`:

| Rule id | Check | Source |
|---|---|---|
| `character-limit` | `draft.characterCount <= platformCharLimit(platform)` | `brand-policy/platform-char-limits.json` |
| `forbidden-word` | No token in `draft.text` is in the forbidden list | `brand-policy/forbidden-words.txt` |
| `missing-hashtag` | At least one `draft.hashtags` entry matches a required hashtag | `brand-policy/required-hashtags.txt` |
| `informal-tone` | LINKEDIN and BLUESKY drafts contain no informal-tone markers | hard-coded marker list in `BrandPolicyGuardrail` |

The guardrail checks all four rules per draft in the `DraftSet`. The first violation short-circuits and returns a structured rejection naming the offending draft and rule.
