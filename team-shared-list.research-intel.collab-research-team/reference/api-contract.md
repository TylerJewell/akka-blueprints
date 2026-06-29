# API contract — collab-research-team

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/questions` | `{ "title": String, "context"?: String, "submittedBy"?: String }` | `202 { "questionId": String }` | `ResearchEndpoint` → `IntakeQueue` |
| `GET` | `/api/questions/{id}` | — | `200 Question` / `404` | `ResearchEndpoint` ← `QuestionEntity` |
| `GET` | `/api/tasks` | — | `200 [ TaskRow... ]` | `ResearchEndpoint` ← `TaskBoardView` |
| `GET` | `/api/tasks?questionId=…&status=…` | — | `200 [ TaskRow... ]` (filtered client-side) | `ResearchEndpoint` ← `TaskBoardView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `ResearchEndpoint` ← `TaskBoardView` |
| `GET` | `/api/mailbox/{researcherId}` | — | `200 [ CoordinationMessage... ]` | `ResearchEndpoint` ← `ResearcherMailbox` |
| `POST` | `/api/mailbox/{researcherId}/messages/{messageId}/reply` | `{ "reply": String }` | `200` / `404` | `ResearchEndpoint` → `ResearcherMailbox` |
| `POST` | `/api/questions/{id}/approve` | `{ "reviewedBy": String, "note"?: String }` | `200 Question` | `ResearchEndpoint` → `QuestionEntity` |
| `POST` | `/api/questions/{id}/reject` | `{ "reviewedBy": String, "reason": String }` | `200 Question` | `ResearchEndpoint` → `QuestionEntity` |
| `POST` | `/api/control/halt` | `{ "reason": String, "by": String }` | `200 OperatorControl` | `ResearchEndpoint` → `OperatorControl` |
| `POST` | `/api/control/resume` | — | `200 OperatorControl` | `ResearchEndpoint` → `OperatorControl` |
| `GET` | `/api/control` | — | `200 OperatorControl` | `ResearchEndpoint` ← `OperatorControl` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Question

```json
{
  "questionId": "q-7a3…",
  "title": "What are the leading causes of urban heat islands?",
  "context": "Focus on peer-reviewed literature from the last decade.",
  "submittedBy": "ui",
  "status": "IN_PROGRESS",
  "taskIds": ["q-7a3-t0", "q-7a3-t1", "q-7a3-t2"],
  "planSummary": "Investigate physical, land-use, and mitigation dimensions of urban heat islands.",
  "report": null,
  "reviewNote": null,
  "createdAt": "2026-06-28T08:10:00Z",
  "completedAt": null
}
```

### TaskRow

```json
{
  "taskId": "q-7a3-t1",
  "questionId": "q-7a3",
  "title": "Land-use and infrastructure drivers",
  "focusStatement": "How impervious surfaces and building density contribute to urban heat.",
  "keywords": ["impervious surface", "building density", "anthropogenic heat"],
  "status": "DONE",
  "claimedBy": "researcher-2",
  "claimedAt": "2026-06-28T08:10:45Z",
  "findingsSummary": "Identified five impervious-surface categories linked to elevated daytime temperatures.",
  "sourceCount": 3,
  "blockedReason": null,
  "completedAt": "2026-06-28T08:11:30Z",
  "createdAt": "2026-06-28T08:10:10Z"
}
```

Lifecycle fields (`claimedBy`, `claimedAt`, `findingsSummary`, `sourceCount`, `blockedReason`, `completedAt`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### CoordinationMessage

```json
{
  "messageId": "m-c21…",
  "fromResearcher": "researcher-3",
  "toResearcher": "researcher-2",
  "taskId": "q-7a3-t2",
  "question": "Which impervious-surface categories did you identify? I need the taxonomy to select mitigation interventions.",
  "sentAt": "2026-06-28T08:11:05Z",
  "reply": null,
  "repliedAt": null
}
```

### ResearchReport (embedded in Question once COMPLETED)

```json
{
  "questionId": "q-7a3",
  "conclusions": [
    {
      "statement": "Low-albedo impervious surfaces absorb significantly more solar energy than vegetated land.",
      "citedUrls": ["https://example-journal.org/uhi-albedo-2023"]
    },
    {
      "statement": "Cool-roof interventions reduce roof surface temperatures by 10–20°C in field studies.",
      "citedUrls": ["https://example-journal.org/cool-roofs-2022"]
    }
  ],
  "executiveSummary": "Urban heat islands result from low surface albedo, suppressed evapotranspiration, and waste heat. Targeted cool-roof and urban-forestry programs show measurable cooling effects.",
  "synthesizedAt": "2026-06-28T08:13:00Z"
}
```

### OperatorControl

```json
{ "halted": false, "haltedReason": null, "haltedBy": null, "haltedAt": null }
```

### SSE event format

```
event: task-update
data: { "taskId": "q-7a3-t1", "status": "DONE", ... }
```

One event per task state transition. Clients reconcile by `taskId` and group the board by `status`.
