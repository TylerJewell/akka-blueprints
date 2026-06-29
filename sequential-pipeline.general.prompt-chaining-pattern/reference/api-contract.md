# API contract — prompt-chaining-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/drafts` | `SubmitDraftRequest` | `201 { draftId }` | `DraftEndpoint` → `DraftEntity` |
| `GET` | `/api/drafts` | — | `200 [ DraftRecord... ]` (newest-first) | `DraftEndpoint` ← `DraftView` |
| `GET` | `/api/drafts/{id}` | — | `200 DraftRecord` / `404` | `DraftEndpoint` ← `DraftView` |
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
  "prompt": "Introduction to vector databases"
}
```

### DraftRecord (response body)

```json
{
  "draftId": "d-7bc3a1...",
  "prompt": "Introduction to vector databases",
  "outline": {
    "sections": [
      { "sectionId": "what-are-vectors", "heading": "What are vector embeddings", "description": "Defines high-dimensional vectors and their role in semantic similarity search." },
      { "sectionId": "indexing-strategies", "heading": "Indexing strategies for approximate nearest-neighbour search", "description": "Covers HNSW, IVF, and product quantisation approaches." }
    ],
    "documentTitle": "Introduction to vector databases",
    "outlinedAt": "2026-06-28T10:00:00Z"
  },
  "draft": {
    "sectionDrafts": [
      {
        "sectionId": "what-are-vectors",
        "heading": "What are vector embeddings",
        "body": "A vector embedding maps a piece of data — text, image, or audio — to a point in a high-dimensional numeric space...",
        "citationIds": ["cit-3a9f22b1"]
      }
    ],
    "citations": [
      { "citationId": "cit-3a9f22b1", "label": "VectorDB Primer", "url": "https://example.org/vectordb-primer", "snippet": "Embeddings map inputs to a continuous vector space..." }
    ],
    "draftedAt": "2026-06-28T10:00:15Z"
  },
  "refinedDocument": {
    "title": "Introduction to vector databases",
    "abstract_": "Vector databases store data as high-dimensional embeddings, enabling fast semantic search through approximate nearest-neighbour indexing. This document covers the fundamentals of embeddings and practical indexing strategies.",
    "sections": [
      {
        "sectionId": "what-are-vectors",
        "heading": "What are vector embeddings",
        "body": "A vector embedding maps a piece of data to a point in a high-dimensional numeric space where proximity reflects semantic similarity. Models such as BERT and CLIP generate these embeddings as a by-product of their training objective.",
        "citationMarkers": ["VectorDB Primer"]
      }
    ],
    "bibliography": [
      { "citationId": "cit-3a9f22b1", "label": "VectorDB Primer", "url": "https://example.org/vectordb-primer", "snippet": "Embeddings map inputs to a continuous vector space..." }
    ],
    "refinedAt": "2026-06-28T10:00:30Z"
  },
  "quality": {
    "score": 5,
    "rationale": "Outline coverage, draft coverage, citation provenance, and structural parity all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:31Z"
  },
  "status": "EVALUATED",
  "guardrailRejections": [],
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:31Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when the response guardrail caught a structurally incomplete document.

### SSE event format

```
event: draft-update
data: { "draftId": "d-7bc3a1...", "status": "DRAFTED", "draft": { ... }, ... }
```

One event per state transition (`CREATED`, `OUTLINING`, `OUTLINED`, `DRAFTING`, `DRAFTED`, `REFINING`, `REFINED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: draft-rejection
data: { "draftId": "d-7bc3a1...", "check": "section-count", "reason": "quality-violation: sections.size() = 1, minimum is 2", "rejectedAt": "..." }
```

Clients reconcile by `draftId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitDraftRequest` record and the `DraftCreated` event to carry it.
