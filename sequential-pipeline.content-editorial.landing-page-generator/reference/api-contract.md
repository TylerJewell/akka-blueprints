# API contract — Landing Page Generator

All endpoints are JSON unless noted. ACL is open to the internet (local-dev only). Base path `/api` is served by `LandingPageEndpoint`; `/` and `/app/*` by `AppEndpoint`.

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/concepts` | `{ "concept": "string" }` | `{ "pageId": "uuid" }` | `LandingPageEndpoint` → `ConceptQueue.enqueueConcept` |
| GET | `/api/pages` | — | `{ "pages": [LandingPage, ...] }` | `LandingPageEndpoint` → `LandingPagesView.getAllPages` |
| GET | `/api/pages/{pageId}` | — | `LandingPage` or 404 | `LandingPageEndpoint` → `LandingPageEntity.getPage` |
| GET | `/api/pages/sse` | — | SSE stream of `LandingPage` | `LandingPageEndpoint` → `LandingPagesView` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `LandingPageEndpoint` (classpath `metadata/eval-matrix.yaml`) |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `LandingPageEndpoint` (classpath `metadata/risk-survey.yaml`) |
| GET | `/api/metadata/readme` | — | `text/markdown` | `LandingPageEndpoint` (classpath `metadata/README.md`) |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static file | `AppEndpoint` |

## Payload shapes

```json
// POST /api/concepts request
{ "concept": "a budgeting app for freelancers" }

// POST /api/concepts response
{ "pageId": "f1c2…" }

// LandingPage (lifecycle fields are null until their event fires)
{
  "id": "uuid",
  "concept": "string or null",
  "status": "DRAFTING_COPY | STRUCTURING | WRITING_CTA | REVIEWING | READY | FLAGGED",
  "headline": "string or null",
  "subhead": "string or null",
  "bodyCopy": "string or null",
  "sections": ["string", "..."],
  "ctaPrimary": "string or null",
  "ctaSecondary": "string or null",
  "reviewScore": "int or null",
  "flagReason": "string or null",
  "copyDraftedAt": "ISO-8601 or null",
  "structuredAt": "ISO-8601 or null",
  "ctaWrittenAt": "ISO-8601 or null",
  "reviewedAt": "ISO-8601 or null"
}
```

`Optional<T>` fields serialize as the raw value or `null` (Akka's Jackson config, Lesson 6) — never an `{ "value": … }` wrapper.

## SSE event format

`GET /api/pages/sse` streams one event per `LandingPage` change:

```
event: page
data: { …LandingPage JSON… }

```

The UI's App UI tab keeps a client-side map keyed by `id`, replacing each entry as events arrive.
