# API contract — long-rag-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
| `GET` | `/api/queries` | — | `200 [ QueryRecord... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 QueryRecord` / `404` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "queryText": "EU AI Act liability provisions"
}
```

### QueryRecord (response body)

```json
{
  "queryId": "q-7c4fa1...",
  "queryText": "EU AI Act liability provisions",
  "chunkWindow": {
    "chunks": [
      {
        "chunkId": "c-eu-ai-001",
        "docId": "eu-ai-act-2024",
        "docTitle": "EU Artificial Intelligence Act (2024/1689)",
        "pageStart": 47,
        "pageEnd": 49,
        "text": "Article 6 classifies AI systems according to risk level...",
        "indexedAt": "2026-06-28T08:00:00Z"
      }
    ],
    "queryText": "EU AI Act liability provisions",
    "retrievedAt": "2026-06-28T10:00:00Z"
  },
  "synthesis": {
    "themes": [
      { "themeId": "risk-classification", "label": "Risk classification framework", "findingIds": ["f-3a1b2200"] }
    ],
    "findings": [
      { "findingId": "f-3a1b2200", "text": "Article 6 introduces a tiered risk classification [c-eu-ai-001].", "supportingChunkId": "c-eu-ai-001" }
    ],
    "synthesizedAt": "2026-06-28T10:00:05Z"
  },
  "report": {
    "title": "EU AI Act: Liability and Risk Provisions",
    "abstractText": "The Act establishes a tiered risk framework ...",
    "sections": [
      {
        "themeId": "risk-classification",
        "heading": "Risk classification framework",
        "body": "Article 6 classifies AI systems by risk tier and identifies high-risk categories in Annex III [c-eu-ai-001].",
        "citations": [
          { "chunkId": "c-eu-ai-001", "docTitle": "EU Artificial Intelligence Act (2024/1689)", "pageRange": "47-49" }
        ]
      }
    ],
    "writtenAt": "2026-06-28T10:00:10Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Theme coverage, citation density, chunk provenance, and section parity all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:10Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:10Z",
  "citationFlags": [
    {
      "sentence": "The Act imposes strict obligations on all AI developers.",
      "missingCitation": "no [chunkId] tag found",
      "windowSize": 10,
      "flaggedAt": "2026-06-28T10:00:03Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `citationFlags` is an array — empty on the happy path, populated when the agent generated uncited sentences and the guardrail requested revision.

### SSE event format

```
event: query-update
data: { "queryId": "q-7c4fa1...", "status": "SYNTHESIZED", "synthesis": { ... }, ... }
```

One event per state transition (`CREATED`, `RETRIEVING`, `RETRIEVED`, `SYNTHESIZING`, `SYNTHESIZED`, `REPORTING`, `REPORTED`, `EVALUATED`, `FAILED`) and one per `CitationFlagged` audit event:

```
event: query-citation-flag
data: { "queryId": "q-7c4fa1...", "sentence": "...", "missingCitation": "...", "windowSize": 10, "flaggedAt": "..." }
```

Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitQueryRequest` record and the `QueryCreated` event to carry it.
