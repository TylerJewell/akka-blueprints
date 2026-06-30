# API contract — screenplay-writer-marketplace

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/screenplays` | `SubmitScreenplayRequest` | `201 { screenplayId }` | `ScreenplayEndpoint` → `PiiSanitizer` → `ScreenplayEntity` |
| `GET` | `/api/screenplays` | — | `200 [ ScreenplayRecord... ]` (newest-first) | `ScreenplayEndpoint` ← `ScreenplayView` |
| `GET` | `/api/screenplays/{id}` | — | `200 ScreenplayRecord` / `404` | `ScreenplayEndpoint` ← `ScreenplayView` |
| `GET` | `/api/screenplays/sse` | — | `text/event-stream` | `ScreenplayEndpoint` ← `ScreenplayView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitScreenplayRequest (request body)

```json
{
  "sourceTitle": "Merger announcement email thread",
  "sourceText": "From: [NAME_1] <[EMAIL_1]>\nTo: [NAME_2] <[EMAIL_2]>\nSubject: Merger announcement\n\nDear team,\nAs of Monday we are merging with Acme Corp..."
}
```

Note: the `sourceText` field is submitted as-is; `PiiSanitizer` runs server-side before any entity write. The example above already shows sanitized text for illustration — actual submitted text may contain raw PII that the sanitizer will replace.

### ScreenplayRecord (response body)

```json
{
  "screenplayId": "sp-7b3f1c...",
  "sourceTitle": "Merger announcement email thread",
  "parsedSource": {
    "characters": [
      { "characterId": "ch-01", "placeholder": "[NAME_1]", "archetype": "CEO announcing merger" },
      { "characterId": "ch-02", "placeholder": "[NAME_2]", "archetype": "lead negotiator" }
    ],
    "settings": [
      { "settingId": "s-01", "placeholder": "the boardroom", "type": "INT" }
    ],
    "beats": [
      { "beatId": "b-01", "description": "[NAME_1] reveals the merger decision.", "dramatic_function": "inciting incident" }
    ],
    "parsedAt": "2026-06-30T10:00:00Z"
  },
  "scenePlan": {
    "scenes": [
      { "sceneId": "sc-01", "slugline": "INT. THE BOARDROOM - DAY", "beatIds": ["b-01"], "characterIds": ["ch-01", "ch-02"], "settingId": "s-01" }
    ],
    "developedAt": "2026-06-30T10:00:05Z"
  },
  "screenplay": {
    "title": "The Announcement",
    "logline": "[NAME_1] gathers the leadership team to announce a merger that will change the company forever.",
    "scenes": [
      {
        "sceneId": "sc-01",
        "slugline": "INT. THE BOARDROOM - DAY",
        "action": "[NAME_1] stands at the head of the table. The room is silent.",
        "dialogue": [
          { "characterId": "ch-01", "line": "Effective Monday, we are joining forces with Acme Corp." },
          { "characterId": "ch-02", "line": "I have the term sheet here. We move forward today." }
        ]
      }
    ],
    "formattedAt": "2026-06-30T10:00:10Z"
  },
  "deliveryCheck": {
    "passed": true,
    "detectedTokens": [],
    "checkedAt": "2026-06-30T10:00:10Z"
  },
  "status": "DELIVERED",
  "createdAt": "2026-06-30T10:00:00Z",
  "finishedAt": "2026-06-30T10:00:10Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). The `placeholderMap` is never returned — it is a private field on the entity, not projected onto `ScreenplayView`.

A `DELIVERY_BLOCKED` record:

```json
{
  "screenplayId": "sp-9a0d2e...",
  "sourceTitle": "Product launch campaign emails",
  "status": "DELIVERY_BLOCKED",
  "deliveryCheck": {
    "passed": false,
    "detectedTokens": ["alice@example.com"],
    "checkedAt": "2026-06-30T10:01:20Z"
  },
  "createdAt": "2026-06-30T10:01:00Z",
  "finishedAt": "2026-06-30T10:01:20Z"
}
```

### SSE event format

```
event: screenplay-update
data: { "screenplayId": "sp-7b3f1c...", "status": "DEVELOPED", "scenePlan": { ... }, ... }
```

One event per state transition (`CREATED`, `PARSING`, `PARSED`, `DEVELOPING`, `DEVELOPED`, `FORMATTING`, `FORMATTED`, `DELIVERED`, `DELIVERY_BLOCKED`, `FAILED`). Each event carries the full row at the moment of transition so a late-joining client never needs to replay.

A delivery-block event:

```
event: screenplay-delivery-blocked
data: { "screenplayId": "sp-9a0d2e...", "detectedTokens": ["alice@example.com"], "reason": "pii-detected: alice@example.com found in formatted output", "blockedAt": "2026-06-30T10:01:20Z" }
```

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend `SubmitScreenplayRequest` and `ScreenplayCreated` to carry it.
