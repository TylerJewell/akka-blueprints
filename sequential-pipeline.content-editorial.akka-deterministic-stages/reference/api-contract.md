# API contract — deterministic-multi-stage-agent-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/stories` | `SubmitStoryRequest` | `201 { storyId }` | `StoryEndpoint` → `StoryEntity` |
| `GET` | `/api/stories` | — | `200 [ StoryRecord... ]` (newest-first) | `StoryEndpoint` ← `StoryView` |
| `GET` | `/api/stories/{id}` | — | `200 StoryRecord` / `404` | `StoryEndpoint` ← `StoryView` |
| `GET` | `/api/stories/sse` | — | `text/event-stream` | `StoryEndpoint` ← `StoryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitStoryRequest (request body)

```json
{
  "prompt": "The lighthouse keeper's last night"
}
```

### StoryRecord (response body)

```json
{
  "storyId": "s-5af9c2...",
  "prompt": "The lighthouse keeper's last night",
  "outline": {
    "title": "The lighthouse keeper's last night",
    "genre": "literary-fiction",
    "beats": [
      {
        "beatId": "beat-01",
        "text": "The keeper discovers the lighthouse logbook contains an entry dated tomorrow.",
        "position": 0
      },
      {
        "beatId": "beat-02",
        "text": "Reading the entry forward, the keeper realises it was written in their own hand.",
        "position": 1
      }
    ],
    "outlinedAt": "2026-06-28T10:00:00Z"
  },
  "body": {
    "paragraphs": [
      {
        "paragraphId": "p-7f3a1b22",
        "beatId": "beat-01",
        "text": "On the last patrol of the night, the keeper pulled the logbook from its shelf..."
      },
      {
        "paragraphId": "p-9c0e44d1",
        "beatId": "beat-02",
        "text": "The keeper's hands trembled as the handwriting came into focus..."
      }
    ],
    "writtenAt": "2026-06-28T10:00:05Z"
  },
  "ending": {
    "closingText": "The keeper set the logbook down, walked to the lamp room, and watched the beam sweep out across the water.",
    "arcs": [
      {
        "arcId": "arc-foreknowledge",
        "label": "Foreknowledge and inevitability",
        "paragraphIds": ["p-7f3a1b22", "p-9c0e44d1"]
      }
    ],
    "endingWrittenAt": "2026-06-28T10:00:10Z"
  },
  "validation": {
    "score": 5,
    "rationale": "Genre detection, beat coverage, arc linkage, and title consistency all satisfied.",
    "validatedAt": "2026-06-28T10:00:11Z"
  },
  "status": "VALIDATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:11Z",
  "guardrailRejections": [
    {
      "stage": "OUTLINE",
      "field": "beats.size",
      "reason": "structural-violation: beats.size() must be >= 2, saw 1",
      "rejectedAt": "2026-06-28T10:00:00Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path, populated when a stage result failed structural validation and the guardrail caught it.

### SSE event format

```
event: story-update
data: { "storyId": "s-5af9c2...", "status": "BODY_WRITTEN", "body": { ... }, ... }
```

One event per state transition (`CREATED`, `OUTLINING`, `OUTLINED`, `BODY_WRITING`, `BODY_WRITTEN`, `ENDING_WRITING`, `ENDING_WRITTEN`, `VALIDATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: story-rejection
data: { "storyId": "s-5af9c2...", "stage": "OUTLINE", "field": "beats.size", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `storyId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitStoryRequest` record and the `StoryCreated` event to carry it.
