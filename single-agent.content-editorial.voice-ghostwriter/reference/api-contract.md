# API contract — ghostwriter

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/drafts` | `SubmitDraftRequest` | `201 { draftId }` | `DraftEndpoint` → `DraftEntity` |
| `GET` | `/api/drafts` | — | `200 [ Draft... ]` (newest-first) | `DraftEndpoint` ← `DraftView` |
| `GET` | `/api/drafts/{id}` | — | `200 Draft` / `404` | `DraftEndpoint` ← `DraftView` |
| `GET` | `/api/drafts/sse` | — | `text/event-stream` | `DraftEndpoint` ← `DraftView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitDraftRequest (request body)

```json
{
  "voiceOwnerId": "alex-chen",
  "topic": "Why async observability should be a first-class product feature",
  "format": "blog-post",
  "targetWordCount": 400,
  "samples": [
    {
      "sampleId": "alex-chen-001",
      "title": "On building developer tools that earn trust",
      "rawBody": "When I joined the platform team three years ago...",
      "format": "blog-post"
    }
  ],
  "requestedBy": "content-team-42"
}
```

### Draft (response body)

```json
{
  "draftId": "d-9fa...",
  "brief": {
    "draftId": "d-9fa...",
    "voiceOwnerId": "alex-chen",
    "topic": "Why async observability should be a first-class product feature",
    "format": "blog-post",
    "targetWordCount": 400,
    "samples": [
      {
        "sampleId": "alex-chen-001",
        "title": "On building developer tools that earn trust",
        "rawBody": "(raw body preserved for audit)",
        "format": "blog-post"
      }
    ],
    "requestedBy": "content-team-42",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "sanitizedCorpus": {
    "samples": [
      {
        "sampleId": "alex-chen-001",
        "redactedBody": "When I joined the platform team three years ago, [REDACTED-NAME] and I debated..."
      }
    ],
    "piiCategoriesFound": ["person-name", "email"]
  },
  "result": {
    "draftBody": "Async systems have a fundamental accountability gap. When a job fails at 2am ...",
    "fidelityScore": 78,
    "markers": [
      {
        "token": "first-person singular opener",
        "evidence": "When I joined the platform team three years ago..."
      },
      {
        "token": "short declarative sentences under 20 words",
        "evidence": "The fix is not more dashboards. It is better defaults."
      }
    ],
    "guardrailRejections": 0,
    "generatedAt": "2026-06-28T09:00:22Z"
  },
  "status": "DRAFT_READY",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:22Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: draft-update
data: { "draftId": "d-9fa...", "status": "DRAFT_READY", "result": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `DRAFTING`, `DRAFT_READY`, `FAILED`). Clients reconcile by `draftId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body. Voice-owner consent checks (see `risk-survey.yaml` field `voice-owner-consent-to-ai-style-training`) are deployer responsibility.
