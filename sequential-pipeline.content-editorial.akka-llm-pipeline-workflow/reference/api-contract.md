# API contract — akka-llm-pipeline-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/posts` | `SubmitPostRequest` | `201 { postId }` | `PostEndpoint` → `PostEntity` |
| `GET` | `/api/posts` | — | `200 [ PostRecord... ]` (newest-first) | `PostEndpoint` ← `PostView` |
| `GET` | `/api/posts/{id}` | — | `200 PostRecord` / `404` | `PostEndpoint` ← `PostView` |
| `GET` | `/api/posts/sse` | — | `text/event-stream` | `PostEndpoint` ← `PostView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPostRequest (request body)

```json
{
  "topic": "The future of edge computing",
  "wordTarget": 600
}
```

`wordTarget` is optional; defaults to 600 if omitted.

### PostRecord (response body)

```json
{
  "postId": "p-3bc7a1...",
  "topic": "The future of edge computing",
  "wordTarget": 600,
  "outline": {
    "title": "Edge Computing: Where Processing Moves Next",
    "wordTarget": 600,
    "sections": [
      {
        "heading": "What edge computing solves",
        "keyPoints": [
          "Latency reduction for real-time workloads",
          "Bandwidth cost savings vs. centralised cloud"
        ]
      },
      {
        "heading": "Deployment patterns in practice",
        "keyPoints": [
          "Micro data centres at cell towers",
          "On-device inference for IoT"
        ]
      },
      {
        "heading": "Open challenges",
        "keyPoints": [
          "Operational overhead of distributed fleet management",
          "Security surface expansion at the edge"
        ]
      }
    ],
    "outlinedAt": "2026-06-28T10:00:05Z"
  },
  "draft": {
    "title": "Edge Computing: Where Processing Moves Next",
    "introduction": "As data volumes continue to climb, moving computation closer to the source has shifted from an optimisation to a necessity.",
    "sections": [
      {
        "heading": "What edge computing solves",
        "body": "Centralised cloud architectures impose round-trip latency on every request..."
      },
      {
        "heading": "Deployment patterns in practice",
        "body": "Operators are placing micro data centres at cellular tower sites..."
      },
      {
        "heading": "Open challenges",
        "body": "Managing a geographically distributed fleet of edge nodes introduces operational overhead..."
      }
    ],
    "conclusion": "Edge computing is not a replacement for centralised infrastructure; it is a complement that extends the reach of distributed systems to where latency and bandwidth matter most.",
    "wordCount": 587,
    "draftedAt": "2026-06-28T10:00:22Z"
  },
  "review": {
    "passed": true,
    "reason": "all checks passed",
    "reviewedAt": "2026-06-28T10:00:22Z"
  },
  "validationFailure": null,
  "status": "READY",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:22Z"
}
```

Any lifecycle field that has not yet been populated is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `validationFailure` is `null` on the happy path and populated when `SchemaGuardrail` rejected the outline. `review.passed` is `false` and `reason` names the failing rule when `EditorialGuardrail` flagged the draft.

### Flagged post example

```json
{
  "postId": "p-9de2f0...",
  "status": "FLAGGED",
  "review": {
    "passed": false,
    "reason": "editorial-flag: prohibited-topic: phrase 'guaranteed returns' found in section 'Open challenges'",
    "reviewedAt": "2026-06-28T10:05:14Z"
  }
}
```

### Failed post example (schema validation)

```json
{
  "postId": "p-1a4bc8...",
  "status": "FAILED",
  "validationFailure": {
    "field": "sections",
    "reason": "validation-failed: sections is empty; at least one OutlineSection is required",
    "failedAt": "2026-06-28T10:03:01Z"
  }
}
```

### SSE event format

```
event: post-update
data: { "postId": "p-3bc7a1...", "status": "OUTLINED", "outline": { ... }, ... }
```

One event per state transition (`CREATED`, `OUTLINING`, `OUTLINED`, `DRAFTING`, `DRAFTED`, `REVIEWING`, `READY`, `FLAGGED`, `FAILED`).

```
event: post-flagged
data: { "postId": "p-9de2f0...", "status": "FLAGGED", "review": { "passed": false, "reason": "..." }, ... }
```

```
event: post-failed
data: { "postId": "p-1a4bc8...", "status": "FAILED", "validationFailure": { "field": "sections", "reason": "..." }, ... }
```

Clients reconcile by `postId`; every SSE event carries the full row at the moment of transition, so a late-joining client never needs to replay history.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitPostRequest` record and the `PostCreated` event to carry it.
