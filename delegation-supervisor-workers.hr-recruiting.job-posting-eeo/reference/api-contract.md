# API contract — job-posting-eeo

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/postings` | `{ "company": String, "roleTitle": String, "requestedBy"?: String }` | `202 { "postingId": String }` | `JobPostingEndpoint` → `RequestQueue` |
| `GET` | `/api/postings` | — | `200 [ JobPosting... ]` (optional `?status=`) | `JobPostingEndpoint` ← `JobPostingView` |
| `GET` | `/api/postings/{id}` | — | `200 JobPosting` / `404` | `JobPostingEndpoint` ← `JobPostingView` |
| `GET` | `/api/postings/sse` | — | `text/event-stream` (one event per posting change) | `JobPostingEndpoint` ← `JobPostingView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### JobPosting

```json
{
  "postingId": "jp-7f2…",
  "company": "Northwind Logistics",
  "roleTitle": "Warehouse Operations Lead",
  "status": "CLEARED",
  "culture": { "values": ["safety-first","reliability"], "tone": "direct", "summary": "...", "analyzedAt": "2026-06-28T07:33:01Z" },
  "role": { "responsibilities": ["..."], "qualifications": ["..."], "marketSalaryRange": "$72k–$95k", "analyzedAt": "2026-06-28T07:33:05Z" },
  "draft": { "title": "Warehouse Operations Lead", "body": "...", "eeoStatement": "...", "guardrailVerdict": "ok", "draftedAt": "2026-06-28T07:33:12Z" },
  "sanitized": { "title": "Warehouse Operations Lead", "body": "...", "removedTerms": [], "sanitizedAt": "2026-06-28T07:33:13Z" },
  "failureReason": null,
  "createdAt": "2026-06-28T07:32:55Z",
  "finishedAt": "2026-06-28T07:33:14Z"
}
```

Lifecycle fields (`culture`, `role`, `draft`, `sanitized`, `failureReason`, `finishedAt`) are `Optional<T>` in Java and serialize as the raw value or `null` on the wire (Lesson 6). The view row truncates `draft.body` and the nested payloads; the UI fetches the full posting by id on click.

### SSE event format

```
event: posting-update
data: { "postingId": "jp-7f2…", "status": "SANITIZED", ... }
```

One event per state transition. Clients reconcile by `postingId`.
