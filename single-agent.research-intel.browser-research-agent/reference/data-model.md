# Data model — reddit-search

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ResearchTopic` | `topic` | `String` | no | The research question or subject supplied by the user. |
| | `subredditScope` | `List<String>` | no | Subreddit names to restrict search to; empty list = all Reddit. |
| | `maxPages` | `int` | no | Page-visit ceiling (1–20). |
| `PostSummary` | `postId` | `String` | no | Reddit post id (e.g., `t3_abc123`). |
| | `title` | `String` | no | Post title verbatim. |
| | `subreddit` | `String` | no | e.g., `r/scala`. |
| | `upvotes` | `int` | no | Post score at time of visit. |
| | `summaryLine` | `String` | no | One sentence summarising the post content. |
| | `url` | `String` | no | Full HTTPS Reddit URL. |
| `Theme` | `phrase` | `String` | no | Recurring noun-phrase extracted across posts. |
| | `occurrences` | `int` | no | Count of posts mentioning the phrase. |
| `SentimentCounts` | `positive` | `int` | no | Posts classified as positive tone. |
| | `neutral` | `int` | no | Posts classified as neutral tone. |
| | `negative` | `int` | no | Posts classified as negative tone. |
| `ResearchReport` | `posts` | `List<PostSummary>` | no | Ranked by relevance descending, max 20. |
| | `themes` | `List<Theme>` | no | Top extracted themes. |
| | `sentiment` | `SentimentCounts` | no | Aggregate counts across all posts. |
| | `pagesVisited` | `int` | no | Number of pages the agent actually visited. |
| | `noResultsReason` | `String` | yes | Non-null only when `posts` is empty. |
| | `completedAt` | `Instant` | no | When the agent task finished or the budget fired. |
| `ResearchJob` | `jobId` | `String` | no | UUID minted by `ResearchEndpoint`. |
| (entity state) | `topic` | `Optional<ResearchTopic>` | yes | Populated after `JobQueued`. |
| | `report` | `Optional<ResearchReport>` | yes | Populated after `ReportRecorded` or `BudgetExhausted`. |
| | `status` | `JobStatus` | no | See enum. |
| | `pagesVisited` | `int` | no | Live counter; incremented by each `PageVisited` event. |
| | `createdAt` | `Instant` | no | When `JobQueued` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `ResearchJob` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`JobStatus`: `QUEUED`, `BROWSING`, `REPORT_READY`, `BUDGET_EXHAUSTED`, `FAILED`.

## Events (`ResearchJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobQueued` | `topic: ResearchTopic` | → QUEUED |
| `BrowsingStarted` | — | → BROWSING |
| `PageVisited` | `url: String` | pagesVisited++ (no status change) |
| `ReportRecorded` | `report: ResearchReport` | → REPORT_READY (terminal happy) |
| `BudgetExhausted` | `pagesVisited: int`, `partialReport: ResearchReport` | → BUDGET_EXHAUSTED (terminal) |
| `JobFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `ResearchJob.initial("")` with all `Optional` fields as `Optional.empty()`, `pagesVisited = 0`, and `status = QUEUED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ResearchJobRow` mirrors `ResearchJob`. The view declares ONE query: `getAllJobs: SELECT * AS jobs FROM research_job_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ResearchTasks.java`)

```java
public final class ResearchTasks {
  public static final Task<ResearchReport> RESEARCH_TOPIC = Task
      .name("Research topic")
      .description("Browse Reddit and extract structured post summaries, themes, and sentiment for the given topic")
      .resultConformsTo(ResearchReport.class);

  private ResearchTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Supporting class signatures

```java
// BudgetEnforcer.java
public final class BudgetEnforcer {
  public BudgetSignal checkBudget(int pagesVisited, int maxPages) { ... }
  public enum BudgetSignal { OK, EXCEEDED }
}

// RelevanceScorer.java
public final class RelevanceScorer {
  /**
   * Re-ranks posts by composite(upvotes=0.5, keywordDensity=0.3, recency=0.2).
   * Caps result at 20 entries.
   */
  public ResearchReport score(ResearchReport raw, String topic) { ... }
}
```
