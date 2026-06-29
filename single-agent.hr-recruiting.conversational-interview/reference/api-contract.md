# API contract — conversational-interview

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `StartSessionRequest` | `201 { sessionId }` | `SessionEndpoint` → `InterviewSessionEntity` |
| `POST` | `/api/sessions/{id}/answers` | `SubmitAnswerRequest` | `202 accepted` | `SessionEndpoint` → `InterviewSessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 SessionRow` / `404` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartSessionRequest (request body)

```json
{
  "candidateHandle": "candidate-77",
  "role": {
    "roleId": "swe-backend",
    "roleTitle": "Software Engineer (Backend)",
    "competencies": [
      {
        "competencyId": "swe-problem-solving",
        "name": "Problem Solving",
        "description": "Breaks down complex technical challenges into tractable sub-problems."
      },
      {
        "competencyId": "swe-collaboration",
        "name": "Collaboration",
        "description": "Works effectively across team boundaries under disagreement or time pressure."
      },
      {
        "competencyId": "swe-system-design",
        "name": "System Design",
        "description": "Designs scalable, maintainable systems with clear tradeoff reasoning."
      }
    ],
    "targetTurnCount": 5,
    "gradingNote": "Focus on concrete examples, not hypotheticals."
  },
  "startedBy": "recruiter-42"
}
```

### SubmitAnswerRequest (request body)

```json
{
  "turnIndex": "2",
  "rawAnswer": "In my last role I worked on a distributed cache layer where ..."
}
```

### SessionRow (response body)

```json
{
  "sessionId": "s-9ac...",
  "request": {
    "sessionId": "s-9ac...",
    "candidateHandle": "candidate-77",
    "role": {
      "roleId": "swe-backend",
      "roleTitle": "Software Engineer (Backend)",
      "competencies": [ { "competencyId": "swe-problem-solving", "name": "Problem Solving", "description": "..." } ],
      "targetTurnCount": 5,
      "gradingNote": "Focus on concrete examples."
    },
    "startedBy": "recruiter-42",
    "startedAt": "2026-06-28T14:00:00Z"
  },
  "turns": [
    {
      "turnIndex": "1",
      "turn": {
        "turnIndex": "1",
        "question": "Walk me through a technical problem you had to break down into smaller parts to solve.",
        "competencyId": "swe-problem-solving",
        "sessionComplete": false,
        "generatedAt": "2026-06-28T14:00:05Z"
      },
      "submittedAnswer": {
        "turnIndex": "1",
        "rawAnswer": "In my last role I worked on a distributed cache layer ...",
        "answeredAt": "2026-06-28T14:02:10Z"
      },
      "screenedAnswer": {
        "turnIndex": "1",
        "redactedAnswer": "In my last role I worked on a distributed cache layer ...",
        "protectedCategoriesFound": []
      }
    }
  ],
  "status": "CONDUCTING",
  "createdAt": "2026-06-28T14:00:00Z",
  "completedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). The `turns` list grows with each conducted turn. `submittedAnswer` and `screenedAnswer` inside a `TurnRecord` are `null` until the candidate answers that turn.

### SSE event format

```
event: session-update
data: { "sessionId": "s-9ac...", "status": "CONDUCTING", "turns": [...], ... }
```

One event per state transition (`OPEN`, `ANSWER_RECEIVED`, `ANSWER_SCREENED`, `CONDUCTING`, `COMPLETED`, `FAILED`). Clients reconcile by `sessionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `startedBy` and `candidateHandle` from the authenticated principal rather than the request body. The `candidateHandle` should remain anonymised — real candidate names must not appear in the session data passed to the model.
