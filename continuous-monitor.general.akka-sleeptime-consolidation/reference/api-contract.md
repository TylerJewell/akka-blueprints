# API contract — sleeptime-consolidation

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/memory/blocks` | — | `200 [ MemoryBlockRow... ]` (sorted newest-first). Optional `?sessionId=…` or `?status=…`. | `MemoryEndpoint` ← `MemoryView` |
| `GET` | `/api/memory/blocks/{blockId}` | — | `200 MemoryBlock` / `404` | `MemoryEndpoint` ← `MemoryBlockEntity` |
| `POST` | `/api/memory/blocks` | `{ "sessionId": String, "rawContent": String, "topicHints": [String] }` | `201 MemoryBlockRow` | `MemoryEndpoint` → `MemoryBlockEntity` |
| `POST` | `/api/memory/blocks/{blockId}/content` | `{ "rawContent": String, "topicHints": [String] }` | `200 MemoryBlockRow` (version incremented) | `MemoryEndpoint` → `MemoryBlockEntity` + `InteractionCounterEntity` |
| `GET` | `/api/memory/sessions/{sessionId}/counter` | — | `200 SessionCounter` | `MemoryEndpoint` ← `InteractionCounterEntity` |
| `POST` | `/api/memory/sessions/{sessionId}/step` | — | `200 SessionCounter` (stepsSinceLastConsolidation incremented) | `MemoryEndpoint` → `InteractionCounterEntity` |
| `POST` | `/api/memory/agent/query` | `{ "sessionId": String, "prompt": String }` | `200 AgentResponse` | `MemoryEndpoint` → `PrimaryAgent` |
| `GET` | `/api/memory/sse` | — | `text/event-stream` | `MemoryEndpoint` ← `MemoryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### MemoryBlockRow (view row — content omitted)

```json
{
  "blockId": "blk-a3f…",
  "sessionId": "sess-019…",
  "version": 3,
  "status": "CONSOLIDATED",
  "consolidatedSummary": "User prefers async patterns; budget cap $5k; no vendor lock-in.",
  "driftScore": 42,
  "driftRationale": "New vendor constraint added; prior preference preserved.",
  "stepsSinceLastConsolidation": 2,
  "createdAt": "2026-06-28T09:00:00Z",
  "lastConsolidatedAt": "2026-06-28T09:14:35Z"
}
```

### MemoryBlock (full — from entity)

```json
{
  "blockId": "blk-a3f…",
  "sessionId": "sess-019…",
  "rawContent": {
    "blockId": "blk-a3f…",
    "sessionId": "sess-019…",
    "rawContent": "User asked about async patterns; mentioned $5k budget; asked to avoid proprietary SDKs.",
    "topicHints": ["async", "budget", "vendor"],
    "capturedAt": "2026-06-28T09:13:10Z"
  },
  "consolidated": {
    "consolidatedText": "User prefers async patterns; budget cap $5k; no vendor lock-in.",
    "changeRationale": "Added vendor preference from latest interaction.",
    "priorVersion": 2,
    "consolidatedAt": "2026-06-28T09:14:35Z"
  },
  "version": 3,
  "drift": {
    "driftScore": 42,
    "rationale": "New vendor constraint added; prior preference preserved.",
    "priorSummary": "User prefers async patterns; budget cap $5k.",
    "currentSummary": "User prefers async patterns; budget cap $5k; no vendor lock-in."
  },
  "lastGuardrailTrip": null,
  "status": "CONSOLIDATED",
  "createdAt": "2026-06-28T09:00:00Z",
  "lastConsolidatedAt": "2026-06-28T09:14:35Z"
}
```

### SessionCounter

```json
{
  "sessionId": "sess-019…",
  "totalSteps": 12,
  "stepsSinceLastConsolidation": 2,
  "lastConsolidatedAt": "2026-06-28T09:14:35Z"
}
```

### AgentResponse

```json
{
  "reply": "Your stated budget is $5k with no vendor lock-in requirement. Async event-sourced patterns fit that constraint well.",
  "blockIdRead": "blk-a3f…",
  "blockVersionRead": 3,
  "respondedAt": "2026-06-28T09:15:02Z"
}
```

### SSE event format

```
event: block-update
data: { "blockId": "blk-a3f…", "status": "CONSOLIDATED", "version": 3, ... }

event: guardrail-trip
data: { "blockId": "blk-a3f…", "callerId": "agent-ext-007", "rejectionReason": "No active consolidation lease for this block.", "triggeredAt": "2026-06-28T09:20:01Z" }
```

One event per state transition or guardrail trip. Clients reconcile by `blockId`.
