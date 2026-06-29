# API contract — guided-intake-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `{ "goalId": String, "submittedBy"?: String }` | `202 { "conversationId": String }` | `ConversationEndpoint` → `SessionQueue` |
| `POST` | `/api/conversations/{id}/reply` | `{ "reply": String }` | `202` | `ConversationEndpoint` → `ConversationWorkflow.resume` |
| `GET` | `/api/conversations` | — | `200 [ ConversationRow... ]` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `ConversationEndpoint` ← `ConversationEntity` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` (one event per conversation change + one per question delivery) | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/goals` | — | `200 [ GoalDefinition... ]` | `ConversationEndpoint` ← `GoalCatalogEntity` |
| `POST` | `/api/goals` | `GoalDefinition` | `201 { "goalId": String }` | `ConversationEndpoint` → `GoalCatalogEntity` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `ConversationEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `ConversationEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `ConversationEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Conversation (full form, returned by `GET /api/conversations/{id}`)

```json
{
  "conversationId": "c-a3b…",
  "goalId": "support-ticket-intake",
  "submittedBy": "session-ui",
  "status": "ELICITING",
  "plan": {
    "goalId": "support-ticket-intake",
    "questionSequence": [
      { "questionId": "q1", "text": "What product or service are you having trouble with?", "fieldName": "product", "required": true },
      { "questionId": "q2", "text": "What version or release are you running?", "fieldName": "version", "required": false },
      { "questionId": "q3", "text": "Can you describe the steps that lead to the problem?", "fieldName": "stepsToReproduce", "required": true },
      { "questionId": "q4", "text": "How severe is the impact?", "fieldName": "severity", "required": true }
    ],
    "completionCriteria": ["product is identified", "steps to reproduce are described", "severity is stated"]
  },
  "turns": [
    {
      "turnNumber": 1,
      "questionId": "q1",
      "questionText": "What product or service are you having trouble with?",
      "sanitizedReply": "The billing console in the cloud portal.",
      "verdict": "OK",
      "blocker": null,
      "recordedAt": "2026-06-28T09:10:05Z"
    }
  ],
  "summary": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": null
}
```

### ConversationRow (list form, returned by `GET /api/conversations`)

The list form is `ConversationRow` — same fields as `Conversation`, but `turns` is truncated to the last 3 entries plus `truncatedFromTotal: int`. The UI fetches the full conversation by id on row expand.

### GoalDefinition

```json
{
  "goalId": "support-ticket-intake",
  "name": "Support Ticket Intake",
  "description": "Collect the information needed to file a support ticket.",
  "questionSet": [ ... ],
  "completionCriteria": [ "product is identified", "steps to reproduce are described", "severity is stated" ],
  "maxTurns": 8
}
```

### SSE event format

```
event: conversation-update
data: { "conversationId": "c-a3b…", "status": "ELICITING", ... }
```

One event per state transition. Clients reconcile by `conversationId`.

The SSE channel also emits `event: question-ready` when a new question is available for the user:

```
event: question-ready
data: { "conversationId": "c-a3b…", "turnNumber": 2, "questionId": "q2", "text": "What version or release are you running?" }
```

And `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating guardrail spike", "haltedAt": "2026-06-28T09:15:30Z" }
```
