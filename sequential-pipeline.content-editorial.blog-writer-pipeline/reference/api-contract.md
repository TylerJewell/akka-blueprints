# API contract — blog-writer-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/posts` | `SubmitPostRequest` | `201 { postId }` | `BlogPostEndpoint` → `BlogPostEntity` |
| `GET` | `/api/posts` | — | `200 [ BlogPostRecord... ]` (newest-first) | `BlogPostEndpoint` ← `BlogPostView` |
| `GET` | `/api/posts/{id}` | — | `200 BlogPostRecord` / `404` | `BlogPostEndpoint` ← `BlogPostView` |
| `GET` | `/api/posts/sse` | — | `text/event-stream` | `BlogPostEndpoint` ← `BlogPostView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPostRequest (request body)

```json
{
  "topic": "The future of developer tooling",
  "style": "Technical"
}
```

`style` is one of `Technical`, `Conversational`, or `Thought leadership`. If omitted, defaults to `Conversational`.

### BlogPostRecord (response body)

```json
{
  "postId": "p-7bc34e...",
  "topic": "The future of developer tooling",
  "style": "Technical",
  "research": {
    "references": [
      {
        "source": "DevTools State Report 2026",
        "url": "https://example.org/devtools-state-2026",
        "keyInsight": "81% of teams report >30% reduction in review cycle time with AI-assisted code review.",
        "capturedAt": "2026-06-28T10:00:00Z"
      }
    ],
    "topicSummary": "Developer tooling in 2026 is shaped by AI-assisted workflows and self-service platform engineering.",
    "researchedAt": "2026-06-28T10:00:00Z"
  },
  "outline": {
    "proposedTitle": "Developer tooling in 2026: AI review and platform self-service",
    "introduction": "Two forces are reshaping how engineers write, review, and ship code in 2026.",
    "sections": [
      {
        "sectionId": "ai-code-review",
        "heading": "AI-assisted code review cuts cycle time",
        "keyPoints": [
          "81% of teams report >30% reduction in review cycle time",
          "AI review surfaces common issues before human reviewers see them"
        ]
      }
    ],
    "outlinedAt": "2026-06-28T10:00:05Z"
  },
  "post": {
    "title": "Developer tooling in 2026: AI review and platform self-service",
    "introduction": "Two forces are reshaping how engineers write, review, and ship code in 2026: AI-assisted code review and self-service internal developer platforms.",
    "sections": [
      {
        "sectionId": "ai-code-review",
        "heading": "AI-assisted code review cuts cycle time",
        "body": "According to the DevTools State Report 2026, 81% of teams that adopted AI-assisted code review saw review cycle times fall by more than 30%..."
      }
    ],
    "conclusion": "The trajectory is clear: teams that invest in AI-assisted review tooling today carry a compounding productivity advantage. Get started with whichever bottleneck your team hits most often.",
    "style": "Technical",
    "draftedAt": "2026-06-28T10:00:10Z"
  },
  "quality": {
    "score": 5,
    "rationale": "Section coverage, section depth, title fidelity, and section parity all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:11Z"
  },
  "guardrailRejections": [],
  "status": "QUALITY_CHECKED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:11Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when the agent produced a non-conformant draft and the guardrail caught it.

#### Example with a brand-voice rejection

```json
{
  "guardrailRejections": [
    {
      "rule": "word-count",
      "detail": "Draft has 187 words; minimum is 300.",
      "rejectedAt": "2026-06-28T10:00:09Z"
    }
  ]
}
```

### SSE event format

```
event: post-update
data: { "postId": "p-7bc34e...", "status": "OUTLINED", "outline": { ... }, ... }
```

One event per state transition (`CREATED`, `RESEARCHING`, `RESEARCHED`, `OUTLINING`, `OUTLINED`, `DRAFTING`, `DRAFTED`, `QUALITY_CHECKED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: post-rejection
data: { "postId": "p-7bc34e...", "rule": "word-count", "detail": "Draft has 187 words; minimum is 300.", "rejectedAt": "..." }
```

Clients reconcile by `postId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitPostRequest` record and the `PostCreated` event to carry it.
