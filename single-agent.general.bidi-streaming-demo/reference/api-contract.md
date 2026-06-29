# API contract — bidi-demo

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/channels` | `OpenChannelRequest` | `201 { channelId }` | `ChannelEndpoint` → `ChannelEntity` |
| `POST` | `/api/channels/{id}/messages` | `SendMessageRequest` | `201 { messageId, turnId }` / `409` if closed | `ChannelEndpoint` → `ChannelEntity` |
| `GET` | `/api/channels` | — | `200 [ ChannelRow... ]` (newest-first) | `ChannelEndpoint` ← `ChannelView` |
| `GET` | `/api/channels/{id}` | — | `200 Channel` / `404` | `ChannelEndpoint` ← `ChannelView` |
| `GET` | `/api/channels/{id}/sse` | — | `text/event-stream` | `ChannelEndpoint` ← `ChannelEntity` events |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `ChannelEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `ChannelEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `ChannelEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### OpenChannelRequest (request body)

```json
{
  "channelName": "general-q-and-a",
  "turnBudget": 10,
  "openedBy": "user-42"
}
```

`turnBudget` is optional; defaults to 10.

### SendMessageRequest (request body)

```json
{
  "content": "What is bidirectional streaming and why does it matter for agents?",
  "sentBy": "user-42"
}
```

### Channel (response body — GET /api/channels/{id})

```json
{
  "channelId": "ch-7a3...",
  "channelName": "general-q-and-a",
  "turns": [
    {
      "turnId": "turn-msg-9b1...",
      "message": {
        "messageId": "msg-9b1...",
        "channelId": "ch-7a3...",
        "content": "What is bidirectional streaming and why does it matter for agents?",
        "sentBy": "user-42",
        "sentAt": "2026-06-28T14:00:00Z"
      },
      "frames": [
        {
          "turnId": "turn-msg-9b1...",
          "frameIndex": 0,
          "content": "A bidirectional channel keeps a persistent connection open so both the user and agent can initiate messages without waiting for the other side to finish.",
          "done": false,
          "emittedAt": "2026-06-28T14:00:05Z"
        },
        {
          "turnId": "turn-msg-9b1...",
          "frameIndex": 1,
          "content": "For agents, this means the agent can stream partial replies frame by frame rather than waiting until a full response is ready.",
          "done": true,
          "emittedAt": "2026-06-28T14:00:05Z"
        }
      ],
      "status": "COMPLETE",
      "startedAt": "2026-06-28T14:00:01Z",
      "completedAt": "2026-06-28T14:00:06Z"
    }
  ],
  "status": "ACTIVE",
  "turnBudget": 10,
  "openedAt": "2026-06-28T13:59:50Z",
  "closedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### ChannelRow (list response — GET /api/channels)

```json
{
  "channelId": "ch-7a3...",
  "channelName": "general-q-and-a",
  "turns": [
    {
      "turnId": "turn-msg-9b1...",
      "status": "COMPLETE",
      "frameCount": 2,
      "completedAt": "2026-06-28T14:00:06Z"
    }
  ],
  "status": "ACTIVE",
  "turnBudget": 10,
  "openedAt": "2026-06-28T13:59:50Z",
  "closedAt": null
}
```

The list view omits frame `content` from turn summaries; fetch `/api/channels/{id}` for the full frame text.

### SSE event format

```
event: frame
data: { "channelId": "ch-7a3...", "turnId": "turn-msg-9b1...", "frameIndex": 0, "content": "...", "done": false, "emittedAt": "2026-06-28T14:00:05Z" }

event: frame
data: { "channelId": "ch-7a3...", "turnId": "turn-msg-9b1...", "frameIndex": 1, "content": "...", "done": true, "emittedAt": "2026-06-28T14:00:05Z" }

event: lifecycle
data: { "channelId": "ch-7a3...", "event": "TurnCompleted", "turnId": "turn-msg-9b1..." }

event: lifecycle
data: { "channelId": "ch-7a3...", "event": "ChannelClosed", "reason": "turn-budget-exhausted" }
```

Two SSE event types:

- `frame` — one event per `FramePublished` entity event. Carries the full `ResponseFrame` fields plus `channelId`.
- `lifecycle` — emitted on `TurnStarted`, `TurnCompleted`, `TurnFailed`, `ChannelClosed`. Clients use lifecycle events to update status indicators without polling.

Late-joining subscribers receive all prior `FramePublished` events from the entity's event stream before receiving new ones — the SSE endpoint uses the entity's full event log as its source, not a snapshot.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `openedBy` / `sentBy` from the authenticated principal rather than the request body. Per-channel access control (only the opener may send messages) is a deployer extension.
