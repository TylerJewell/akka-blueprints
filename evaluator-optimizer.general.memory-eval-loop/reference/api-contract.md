# API contract — memory-eval-loop

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `{ "text": String, "userId"?: String }` | `202 { "sessionId": String }` | `SessionEndpoint` → `SessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (optional `?status=ANSWERING\|SCORING\|ACCEPTED\|REJECTED_FINAL` — filtered client-side from `getAllSessions`) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (one event per session turn change) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/memory/{userId}` | — | `200 [ MemoryEntry... ]` | `SessionEndpoint` ← `MemoryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/sessions`

- Missing `userId` → `"anonymous"`.
- `text` is required and must be non-empty; otherwise `400`.
- Duplicate-detection window: 10 s on `(text, userId)`; the second submission returns the first `sessionId` (200) instead of starting a new workflow.

## JSON shapes

### Session

```json
{
  "sessionId": "s-4a1f2…",
  "userId": "demo-user",
  "questionText": "What programming languages do I use at work?",
  "maxAttempts": 3,
  "status": "ACCEPTED",
  "attempts": [
    {
      "attemptNumber": 1,
      "answerText": "Based on your notes, you primarily work in Python and Go, with Rust as a language you began learning in 2025.",
      "citedEntryIds": ["entry-01", "entry-02"],
      "score": {
        "verdict": "IMPROVE",
        "notes": {
          "bullets": [
            "Answer does not distinguish primary from learning-stage languages.",
            "entry-02 content confirms Rust is exploratory — make this explicit.",
            "Completeness adequate; coherence acceptable."
          ],
          "overallRationale": "Relevance and grounding pass; completeness dimension just below threshold."
        },
        "score": 3,
        "scoredAt": "2026-06-28T09:01:14Z"
      },
      "answeredAt": "2026-06-28T09:01:02Z"
    },
    {
      "attemptNumber": 2,
      "answerText": "Your primary work languages are Python and Go. Rust appears in your notes as a language you began learning in 2025, not yet a primary tool.",
      "citedEntryIds": ["entry-01", "entry-02"],
      "score": {
        "verdict": "PASS",
        "notes": {
          "bullets": [],
          "overallRationale": "All four dimensions meet threshold; distinction between primary and learning-stage languages now explicit."
        },
        "score": 5,
        "scoredAt": "2026-06-28T09:01:33Z"
      },
      "answeredAt": "2026-06-28T09:01:21Z"
    }
  ],
  "acceptedAttemptNumber": 2,
  "acceptedAnswer": "Your primary work languages are Python and Go. Rust appears in your notes as a language you began learning in 2025, not yet a primary tool.",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### MemoryEntry

```json
{
  "entryId": "entry-01",
  "userId": "demo-user",
  "content": "User works primarily in Python and Go.",
  "sourceSessionId": "s-3b0e1…",
  "recordedAt": "2026-06-28T08:45:00Z"
}
```

### SSE event format

```
event: session-update
data: { "sessionId": "s-4a1f2…", "status": "SCORING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `sessionId`. The full `Session` JSON is included so a fresh client can render the row without a separate fetch.

### Drift check (via GET `/api/sessions/drift-watch-singleton`)

The sentinel entity exposes its `DriftCheckRecorded` events through the standard `GET /api/sessions/{id}` endpoint with id `drift-watch-singleton`. The response's `attempts` array is empty (it is not a real session turn); the relevant fields are surfaced in a top-level `driftHistory` extension:

```json
{
  "sessionId": "drift-watch-singleton",
  "status": "ACCEPTED",
  "driftHistory": [
    {
      "computedAt": "2026-06-28T09:10:00Z",
      "passRate": 0.75,
      "avgScore": 4.1,
      "driftFlagged": false
    }
  ]
}
```
