# Data model — content-pipeline

Authoritative record, event, enum, and view-row definitions. `/akka:implement` writes these exactly. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

## `Article` (entity state and View row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Article id (workflow id). |
| `topic` | `Optional<String>` | yes | Submitted topic. |
| `status` | `ArticleStatus` | no | Lifecycle stage. |
| `researchedAt` | `Optional<Instant>` | yes | When research completed. |
| `researchSummary` | `Optional<String>` | yes | Research brief summary. |
| `researchSources` | `Optional<String>` | yes | Comma-separated allowed sources. |
| `draftedAt` | `Optional<Instant>` | yes | When the draft was written. |
| `title` | `Optional<String>` | yes | Draft headline. |
| `body` | `Optional<String>` | yes | Draft article body. |
| `critiquedAt` | `Optional<Instant>` | yes | When critique completed. |
| `critiqueScore` | `Optional<Double>` | yes | Self-critique score 0.0–1.0. |
| `critiqueNotes` | `Optional<String>` | yes | Critique weaknesses. |
| `publishedAt` | `Optional<Instant>` | yes | When published. |
| `publishedUrl` | `Optional<String>` | yes | Synthesized published URL. |
| `failedAt` | `Optional<Instant>` | yes | When the pipeline failed. |
| `failureReason` | `Optional<String>` | yes | Why it failed (guard block or step error). |

`Article.initial(String id)` returns the record with `status = RESEARCHING` and every Optional empty (no `commandContext()` reference — Lesson 3).

## `ArticleStatus` (enum)

`RESEARCHING · WRITING · CRITIQUING · PUBLISHED · FAILED`

## `ArticleEvent` (sealed interface, 5 variants)

| Event | Trigger | Carries |
|---|---|---|
| `ResearchCompleted` | researchStep result recorded | `articleId, summary, sources, timestamp` |
| `ArticleDrafted` | writeStep result recorded | `articleId, title, body, timestamp` |
| `ArticleCritiqued` | critiqueStep result recorded | `articleId, score, notes, timestamp` |
| `ArticlePublished` | publishStep records the URL | `articleId, url, timestamp` |
| `ArticleFailed` | a step fails over to the error step | `articleId, reason, timestamp` |

Each variant exposes `Instant timestamp()`.

## `TopicEvent` (InboundTopicQueue)

| Event | Trigger | Carries |
|---|---|---|
| `TopicQueued` | `enqueueTopic(topic)` command | `topic, timestamp` |

## Typed agent results (in `application/`)

```java
record ResearchBrief(String summary, String sources) {}
record ArticleDraft(String title, String body) {}
record CritiqueResult(double score, String notes) {}
```

## View row

`ArticlesView` uses `Article` as its row type. One query: `getAllArticles` → `SELECT * AS articles FROM articles_view`. No `WHERE status` filter — the enum column cannot be auto-indexed (Lesson 2); callers filter client-side. `streamAllArticles` backs the SSE endpoint.

## Tasks companion (Lesson 7)

`ContentPipelineTasks.java` declares two `Task<R>` constants for the AutonomousAgents:

- `RESEARCH` — `resultConformsTo(ResearchBrief.class)`
- `WRITE` — `resultConformsTo(ArticleDraft.class)`

`CritiqueAgent` is a request/response `Agent` (method `critique`), so it needs no task constant.
