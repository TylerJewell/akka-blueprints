# API contract — code-agent-chat-ui

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `CreateSessionRequest` | `201 { sessionId }` | `ChatEndpoint` → `ChatSessionEntity` |
| `POST` | `/api/sessions/{id}/messages` | `SendMessageRequest` | `201 { messageId }` | `ChatEndpoint` → `ChatSessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ ChatSessionRow... ]` (newest-first) | `ChatEndpoint` ← `ChatView` |
| `GET` | `/api/sessions/{id}` | — | `200 ChatSession` / `404` | `ChatEndpoint` ← `ChatView` |
| `GET` | `/api/sessions/{id}/sse` | — | `text/event-stream` | `ChatEndpoint` ← `ChatView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateSessionRequest (request body)

```json
{
  "title": "Python version question"
}
```

`title` is optional. If omitted, the system derives a title from the first user message.

### SendMessageRequest (request body)

```json
{
  "content": "What is the current Python version and can you show me a hello-world?"
}
```

### ChatSessionRow (list response item)

```json
{
  "sessionId": "s-4a7f...",
  "title": "Python version question",
  "messageCount": 3,
  "status": "ACTIVE",
  "createdAt": "2026-06-28T12:00:00Z",
  "lastActiveAt": "2026-06-28T12:00:42Z"
}
```

### ChatSession (full session response)

```json
{
  "sessionId": "s-4a7f...",
  "title": "Python version question",
  "messages": [
    {
      "messageId": "m-001",
      "role": "USER",
      "content": "What is the current Python version and can you show me a hello-world?",
      "sentAt": "2026-06-28T12:00:00Z",
      "toolCalls": null
    },
    {
      "messageId": "m-002",
      "role": "ASSISTANT",
      "content": "Python **3.13** is the current stable release.\n\n```python\nprint('Hello, world!')\n```\n\nRunning that in the sandbox prints `Hello, world!`.",
      "sentAt": "2026-06-28T12:00:38Z",
      "toolCalls": [
        {
          "toolCallId": "tc-a1",
          "toolName": "web-search",
          "inputSummary": "current Python version 2025",
          "status": "COMPLETED",
          "blockReason": null,
          "outputSummary": "Python 3.13 released Oct 2024...",
          "requestedAt": "2026-06-28T12:00:05Z",
          "resolvedAt": "2026-06-28T12:00:08Z"
        },
        {
          "toolCallId": "tc-a2",
          "toolName": "code-execution",
          "inputSummary": "print('Hello, world!')",
          "status": "COMPLETED",
          "blockReason": null,
          "outputSummary": "Hello, world!",
          "requestedAt": "2026-06-28T12:00:10Z",
          "resolvedAt": "2026-06-28T12:00:12Z"
        }
      ]
    }
  ],
  "planRevisions": [],
  "status": "IDLE",
  "createdAt": "2026-06-28T12:00:00Z",
  "lastActiveAt": "2026-06-28T12:00:38Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

A blocked tool call looks like:

```json
{
  "toolCallId": "tc-b1",
  "toolName": "code-execution",
  "inputSummary": "import subprocess; subprocess.run(['ls', '/'])",
  "status": "BLOCKED",
  "blockReason": "code-execution-policy: subprocess spawn disallowed",
  "outputSummary": null,
  "requestedAt": "2026-06-28T12:01:00Z",
  "resolvedAt": "2026-06-28T12:01:00Z"
}
```

### SSE event format

```
event: session-update
data: { "sessionId": "s-4a7f...", "status": "ACTIVE", "latestMessage": { ... }, ... }
```

One event per state transition or new message (`SessionCreated`, `UserMessageReceived`, `AssistantMessageRecorded`, `ToolCallRequested`, `ToolCallResolved`, `PlanRevised`, `SessionFailed`). Clients reconcile by `sessionId`; an event always carries the full row at the moment of transition, so a late-joining client does not need to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `userId` from the authenticated principal rather than the request body.
