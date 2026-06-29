# API contract — book-writer-fanout

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Books

### POST /api/books
- Component: BookEndpoint → BookRequestQueue.enqueueRequest
- Body: `{ "topic": "string" }`
- Response: `200 { "bookId": "uuid" }`

### GET /api/books?status=...
- Component: BookEndpoint → BooksView.getAllBooks (status filtered client-side)
- Response: `200 { "books": [Book, ...] }`

### GET /api/books/{bookId}
- Component: BookEndpoint → BooksView
- Response: `200 Book` | `404`

### GET /api/books/sse
- Component: BookEndpoint → BooksView.streamAllBooks
- Response: `text/event-stream`; each event `data:` is a `Book` JSON object.

## Web search (in-process, guarded — control G1)

### POST /api/search
- Component: WebSearchEndpoint
- Body: `{ "bookId": "uuid", "query": "string" }`
- Behavior: loads the book topic from BooksView; runs the before-tool-call guard. Refuses
  when the query shares no salient term with the topic or matches the injection denylist
  (`ignore previous`, `system prompt`, URL-exfiltration patterns).
- Response: `200 { "results": [SearchResult, ...] }` | `403 { "refused": true, "reason": "string" }`

## Metadata

```
GET /api/metadata/eval-matrix   -> text/yaml      (BookEndpoint, from classpath metadata/)
GET /api/metadata/risk-survey   -> text/yaml
GET /api/metadata/readme        -> text/markdown
```

## App

```
GET /                 -> 302 /app/index.html   (AppEndpoint)
GET /app/{*path}      -> static-resources/{*path}
```

## Payload shapes

```
Book {
  "id": "uuid",
  "topic": "string or null",
  "status": "REQUESTED | OUTLINED | WRITING | CONSOLIDATING | COMPLETED | FAILED",
  "title": "string or null",
  "chapters": [ ChapterSummary, ... ],
  "chapterCount": "int or null",
  "chaptersDrafted": "int or null",
  "manuscript": "string or null",
  "outlinedAt": "ISO-8601 or null",
  "consolidatedAt": "ISO-8601 or null",
  "completedAt": "ISO-8601 or null",
  "failedAt": "ISO-8601 or null",
  "failureReason": "string or null"
}

ChapterSummary {
  "chapterId": "string",
  "index": "int",
  "title": "string",
  "status": "PENDING | DRAFTING | DRAFTED | APPROVED | REJECTED",
  "qualityScore": "double or null"
}

SearchResult {
  "title": "string",
  "snippet": "string",
  "url": "string"
}
```

Lifecycle fields are nullable on the wire (`Optional<T>` in Java; Jackson serializes the raw
value or `null`, never an `{ "value": ... }` wrapper — Lesson 6).

## SSE event format

`GET /api/books/sse` emits one event per book change:

```
data: {"id":"...","status":"WRITING","chaptersDrafted":2,"chapterCount":4, ...}

```

Each `data:` line is a complete `Book` JSON object; the client replaces its row for that `id`.
