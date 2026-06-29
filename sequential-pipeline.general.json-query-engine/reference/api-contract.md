# API contract — json-query-engine

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
  "question": "What is the price of the Pro plan?",
  "docId": "saas-pricing-catalog"
}
```

### QueryRecord (response body)

```json
{
  "queryId": "q-3f7d1a...",
  "question": "What is the price of the Pro plan?",
  "docId": "saas-pricing-catalog",
  "parsed": {
    "questionText": "What is the price of the Pro plan?",
    "intent": {
      "intentType": "lookup",
      "targetEntity": "Pro plan price",
      "candidateRootKeys": ["plans", "pricing"]
    },
    "parsedAt": "2026-06-28T10:00:00Z"
  },
  "traversal": {
    "matches": [
      {
        "pathExpression": "$.plans[?(@.name=='Pro')].price",
        "resolvedValue": "49.00",
        "depthLevel": 3
      }
    ],
    "docId": "saas-pricing-catalog",
    "traversedAt": "2026-06-28T10:00:05Z"
  },
  "result": {
    "answer": "The Pro plan is priced at $49.00 per month.",
    "citations": [
      {
        "pathExpression": "$.plans[?(@.name=='Pro')].price",
        "value": "49.00",
        "label": "price"
      }
    ],
    "composedAt": "2026-06-28T10:00:10Z"
  },
  "accuracy": {
    "score": 5,
    "rationale": "Path coverage, answer grounding, citation completeness, and intent alignment all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:10Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:10Z",
  "guardrailRejections": [
    {
      "phase": "TRAVERSE",
      "tool": "resolvePathExpression",
      "pathExpr": "plans.Pro.price",
      "reason": "path-violation: expression must begin with '$', saw 'plans.Pro.price'",
      "rejectedAt": "2026-06-28T10:00:03Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path, populated when the agent generated an invalid path expression or a misordered tool call.

### SSE event format

```
event: query-update
data: { "queryId": "q-3f7d1a...", "status": "TRAVERSED", "traversal": { ... }, ... }
```

One event per state transition (`CREATED`, `PARSING`, `PARSED`, `TRAVERSING`, `TRAVERSED`, `RESPONDING`, `RESPONDED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: query-rejection
data: { "queryId": "q-3f7d1a...", "phase": "TRAVERSE", "tool": "resolvePathExpression", "pathExpr": "...", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitQueryRequest` record and the `QueryCreated` event to carry it.
