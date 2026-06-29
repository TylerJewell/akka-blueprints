# API contract — team-poems-multi-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/poems` | `{ "title": String, "inspiration": String, "submittedBy"?: String }` | `202 { "poemId": String }` | `PoetryEndpoint` → `PromptQueue` |
| `GET` | `/api/poems/{id}` | — | `200 Poem` / `404` | `PoetryEndpoint` ← `PoemEntity` |
| `GET` | `/api/stanzas` | — | `200 [ Stanza... ]` | `PoetryEndpoint` ← `VerseBoardView` |
| `GET` | `/api/stanzas?poemId=…&status=…` | — | `200 [ Stanza... ]` (filtered client-side) | `PoetryEndpoint` ← `VerseBoardView` |
| `GET` | `/api/stanzas/sse` | — | `text/event-stream` (one event per stanza change) | `PoetryEndpoint` ← `VerseBoardView` |
| `GET` | `/api/mailbox/{poetId}` | — | `200 [ CoordinationMessage... ]` | `PoetryEndpoint` ← `CoordinationMailbox` |
| `POST` | `/api/mailbox/{poetId}/messages/{messageId}/reply` | `{ "reply": String }` | `200` / `404` | `PoetryEndpoint` → `CoordinationMailbox` |
| `POST` | `/api/control/pause` | `{ "reason": String, "by": String }` | `200 EditorialControl` | `PoetryEndpoint` → `EditorialControl` |
| `POST` | `/api/control/resume` | — | `200 EditorialControl` | `PoetryEndpoint` → `EditorialControl` |
| `GET` | `/api/control` | — | `200 EditorialControl` | `PoetryEndpoint` ← `EditorialControl` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Poem

```json
{
  "poemId": "pm-7c1…",
  "title": "After the Rain",
  "inspiration": "A walk through a city just after a summer storm.",
  "submittedBy": "ui",
  "status": "DRAFTING",
  "stanzaIds": ["pm-7c1-s0", "pm-7c1-s1", "pm-7c1-s2"],
  "planNote": "Three-stanza arc — sensory opening, reflective middle, quiet close.",
  "createdAt": "2026-06-28T09:10:00Z",
  "publishedAt": null
}
```

### Stanza

```json
{
  "stanzaId": "pm-7c1-s0",
  "poemId": "pm-7c1",
  "title": "Opening image",
  "verseBrief": "Describe the wet pavement, neon reflections, the smell of rain.",
  "form": "FREE_VERSE",
  "dependsOn": [],
  "status": "DONE",
  "claimedBy": "poet-1",
  "claimedAt": "2026-06-28T09:10:08Z",
  "approachNote": "Free-verse tercet; short first line, longer second, returning short.",
  "lineCount": 3,
  "qualityReport": { "passed": true, "issues": [], "ranAt": "2026-06-28T09:10:22Z" },
  "blockedReason": null,
  "completedAt": "2026-06-28T09:10:22Z",
  "createdAt": "2026-06-28T09:10:04Z"
}
```

Lifecycle fields (`claimedBy`, `claimedAt`, `approachNote`, `lineCount`, `qualityReport`, `blockedReason`, `completedAt`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### CoordinationMessage

```json
{
  "messageId": "cm-44f…",
  "fromPoet": "poet-1",
  "toPoet": "poet-2",
  "stanzaId": "pm-7c1-s1",
  "question": "What is the last line of the opening image stanza so I can match the rhythm?",
  "sentAt": "2026-06-28T09:10:30Z",
  "reply": null,
  "repliedAt": null
}
```

### EditorialControl

```json
{ "paused": false, "pausedReason": null, "pausedBy": null, "pausedAt": null }
```

### SSE event format

```
event: stanza-update
data: { "stanzaId": "pm-7c1-s0", "status": "IN_REVIEW", ... }
```

One event per stanza state transition. Clients reconcile by `stanzaId` and group the board by `status`.
