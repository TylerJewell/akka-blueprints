# API contract — shared-memory-multi-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/blocks` | `{ "blockName": String, "initialContent": String, "seededBy"?: String }` | `202 { "blockId": String }` | `MemoryEndpoint` → `MemoryEventLog`, `MemoryBlockEntity` |
| `GET` | `/api/blocks` | — | `200 [ BlockRow... ]` | `MemoryEndpoint` ← `MemoryBlockView` |
| `GET` | `/api/blocks?status=…` | — | `200 [ BlockRow... ]` (filtered client-side) | `MemoryEndpoint` ← `MemoryBlockView` |
| `GET` | `/api/blocks/{id}` | — | `200 MemoryBlock` / `404` | `MemoryEndpoint` ← `MemoryBlockEntity` |
| `GET` | `/api/blocks/sse` | — | `text/event-stream` (one event per block or fragment change) | `MemoryEndpoint` ← `MemoryBlockView` |
| `GET` | `/api/blocks/{id}/fragments` | — | `200 [ FragmentRow... ]` | `MemoryEndpoint` ← `MemoryBlockView` |
| `POST` | `/api/blocks/{id}/consolidate` | — | `202 { "workflowId": String }` | `MemoryEndpoint` → `ConsolidationWorkflow` |
| `GET` | `/api/agents` | — | `200 AgentRegistry` | `MemoryEndpoint` ← `AgentRegistry` |
| `POST` | `/api/agents` | `{ "agentId": String, "displayName": String }` | `200 AgentRegistry` | `MemoryEndpoint` → `AgentRegistry` |
| `DELETE` | `/api/agents/{agentId}` | — | `200` / `404` | `MemoryEndpoint` → `AgentRegistry` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### MemoryBlock

```json
{
  "blockId": "b-7c1…",
  "blockName": "project-glossary",
  "currentContent": "**Agent**: an autonomous process that reads and writes shared memory blocks. **Memory Block**: a named container for shared knowledge.",
  "status": "CONSOLIDATED",
  "pendingFragmentCount": 0,
  "lastConsolidationSummary": "Merged three writer contributions adding four glossary terms.",
  "lastConsolidatedAt": "2026-06-28T09:14:22Z",
  "lastWriterAgent": "writer-2",
  "createdAt": "2026-06-28T09:10:05Z",
  "updatedAt": "2026-06-28T09:14:22Z"
}
```

### BlockRow (view projection — no large content field)

```json
{
  "blockId": "b-7c1…",
  "blockName": "project-glossary",
  "status": "CONSOLIDATED",
  "pendingFragmentCount": 0,
  "lastConsolidationSummary": "Merged three writer contributions adding four glossary terms.",
  "lastConsolidatedAt": "2026-06-28T09:14:22Z",
  "lastWriterAgent": "writer-2",
  "createdAt": "2026-06-28T09:10:05Z"
}
```

### FragmentRow

```json
{
  "fragmentId": "b-7c1-f-writer-1-2",
  "blockId": "b-7c1…",
  "authorId": "writer-1",
  "content": "**Consolidation**: the process of merging all pending fragments into a single coherent block snapshot.",
  "status": "MERGED",
  "sanitized": false,
  "sanitizedNote": null,
  "createdAt": "2026-06-28T09:12:08Z",
  "mergedAt": "2026-06-28T09:14:22Z"
}
```

Lifecycle fields (`sanitizedNote`, `mergedAt`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### AgentRegistry

```json
{
  "agents": {
    "writer-1": { "agentId": "writer-1", "displayName": "Writer 1", "registeredAt": "2026-06-28T09:10:00Z", "lastActiveAt": "2026-06-28T09:13:40Z" },
    "writer-2": { "agentId": "writer-2", "displayName": "Writer 2", "registeredAt": "2026-06-28T09:10:00Z", "lastActiveAt": "2026-06-28T09:13:55Z" },
    "writer-3": { "agentId": "writer-3", "displayName": "Writer 3", "registeredAt": "2026-06-28T09:10:00Z", "lastActiveAt": null }
  }
}
```

### SSE event format

```
event: block-update
data: { "blockId": "b-7c1…", "status": "CONSOLIDATING", "pendingFragmentCount": 5, ... }

event: fragment-update
data: { "fragmentId": "b-7c1-f-writer-1-2", "blockId": "b-7c1…", "status": "MERGED", ... }
```

One event per block or fragment state transition. Clients reconcile by `blockId` / `fragmentId` and group the block board by `status`.
