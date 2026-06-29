# API contract — citation-rag

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
  "queryText": "What are the main findings on transformer attention efficiency?"
}
```

### QueryRecord (response body)

```json
{
  "queryId": "q-5af9c2a1",
  "queryText": "What are the main findings on transformer attention efficiency?",
  "passages": {
    "passages": [
      {
        "passageId": "p-4a1f9e2b",
        "source": "Attention Is All You Need",
        "documentTitle": "Vaswani et al. 2017",
        "text": "Multi-head attention allows the model to jointly attend to information from different representation subspaces at different positions.",
        "relevanceScore": 0.94,
        "retrievedAt": "2026-06-28T10:00:00Z"
      }
    ],
    "retrievedAt": "2026-06-28T10:00:00Z"
  },
  "claimSet": {
    "claims": [
      {
        "claimId": "cl-7f3a1b22",
        "text": "Multi-head attention enables joint attention across different representation subspaces.",
        "citedPassageId": "p-4a1f9e2b",
        "confidence": 0.95
      }
    ],
    "attributedAt": "2026-06-28T10:00:05Z"
  },
  "answer": {
    "queryId": "q-5af9c2a1",
    "queryText": "What are the main findings on transformer attention efficiency?",
    "responseBody": "Transformer attention efficiency research focuses on two complementary directions...",
    "claims": [
      {
        "claimId": "cl-7f3a1b22",
        "text": "Multi-head attention enables joint attention across different representation subspaces.",
        "citedPassageId": "p-4a1f9e2b",
        "confidence": 0.95
      }
    ],
    "citations": [
      {
        "claimId": "cl-7f3a1b22",
        "passageId": "p-4a1f9e2b",
        "passageSnippet": "Multi-head attention allows the model to jointly attend to information from different representation subspaces at different positions."
      }
    ],
    "composedAt": "2026-06-28T10:00:10Z"
  },
  "citationScore": {
    "score": 5,
    "rationale": "Full attribution, resolved citations, non-empty answer, and no decoration citations all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:11Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:11Z",
  "guardrailRejections": [
    {
      "claimId": "cl-bad0000",
      "reason": "uncited-claim: claim cl-bad0000 has no valid citation in the recorded PassageSet",
      "rejectedAt": "2026-06-28T10:00:09Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when the agent produced an uncited claim and the guardrail caught it.

### SSE event format

```
event: query-update
data: { "queryId": "q-5af9c2a1", "status": "ATTRIBUTED", "claimSet": { ... }, ... }
```

One event per state transition (`CREATED`, `RETRIEVING`, `RETRIEVED`, `ATTRIBUTING`, `ATTRIBUTED`, `COMPOSING`, `COMPOSED`, `EVALUATED`, `FAILED`) and one per `CitationGuardrailRejected` audit event:

```
event: query-rejection
data: { "queryId": "q-5af9c2a1", "claimId": "cl-bad0000", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitQueryRequest` record and the `QueryCreated` event to carry it.
