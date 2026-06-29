# Data model — web-scraper-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ScrapeRequest` | `scrapeId` | `String` | no | UUID minted by `ScrapeEndpoint`. |
| | `targetUrl` | `String` | no | URL submitted by the user. |
| | `extractionSchema` | `String` | no | `"news-article"`, `"product-doc"`, `"tech-blog"`, or free-text goal. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `DataPoint` | `fieldName` | `String` | no | Short name matching the extraction schema (e.g., `"author"`, `"version"`). |
| | `value` | `String` | no | Extracted value; `"(not found)"` if absent. |
| | `sourceQuote` | `String` | no | Verbatim passage from the page supporting the value; `"(no passage)"` if none. |
| `ScrapeResult` | `title` | `String` | no | Page title or best candidate heading. |
| | `summary` | `String` | no | 2-4 sentences describing the page content. |
| | `dataPoints` | `List<DataPoint>` | no | Extracted fields; may be empty if the page was unreadable. |
| | `fetchedUrl` | `String` | no | Resolved URL after any redirect. |
| | `httpStatus` | `int` | no | HTTP response code; 0 if the tool call was blocked. |
| | `extractedAt` | `Instant` | no | When the agent completed extraction. |
| `SanitizedResult` | `sanitizedSummary` | `String` | no | Summary with PII tokens replaced. |
| | `sanitizedDataPoints` | `List<DataPoint>` | no | Data points with PII tokens replaced in `value` and `sourceQuote`. |
| | `piiCategoriesFound` | `List<String>` | no | e.g., `["email","phone","person-name","address"]`. |
| `Scrape` (entity state) | `scrapeId` | `String` | no | — |
| | `request` | `Optional<ScrapeRequest>` | yes | Populated after `ScrapeSubmitted`. |
| | `result` | `Optional<ScrapeResult>` | yes | Populated after `RawContentExtracted`. |
| | `sanitized` | `Optional<SanitizedResult>` | yes | Populated after `ContentSanitized`. |
| | `status` | `ScrapeStatus` | no | See enum. |
| | `blockedReason` | `Optional<String>` | yes | Populated when `ScrapeBlocked` fires. |
| | `createdAt` | `Instant` | no | When `ScrapeSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Scrape` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ScrapeStatus`: `SUBMITTED`, `FETCHING`, `EXTRACTED`, `SANITIZED`, `BLOCKED`, `FAILED`.

## Events (`ScrapeEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ScrapeSubmitted` | `request` | → SUBMITTED |
| `ScrapeFetching` | — | → FETCHING |
| `RawContentExtracted` | `result` | → EXTRACTED |
| `ContentSanitized` | `sanitized` | → SANITIZED (terminal happy) |
| `ScrapeBlocked` | `reason: String` | → BLOCKED (terminal) |
| `ScrapeFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Scrape.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ScrapeRow` mirrors `Scrape`. The view does not strip any fields — the raw `result` is included because the UI uses a dedicated API link (`GET /api/scrapes/{id}`) to show the unredacted audit copy, and the view row is what the list and SSE stream carry. The list panel shows only the sanitized form for display; the `result` field is present for the detail link.

The view declares ONE query: `getAllScrapes: SELECT * AS scrapes FROM scrape_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ScrapeTasks.java`)

```java
public final class ScrapeTasks {
  public static final Task<ScrapeResult> SCRAPE_URL = Task
      .name("Scrape URL")
      .description("Fetch the page at the given URL and extract structured data per the requested schema")
      .resultConformsTo(ScrapeResult.class);

  private ScrapeTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
