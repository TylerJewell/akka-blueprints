# API contract — query-planner-parallel-executor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `{ "question": String, "requestedBy"?: String }` | `202 { "sessionId": String }` | `QueryEndpoint` → `QueryQueue` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` | `QueryEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 QuerySession` / `404` | `QueryEndpoint` ← `QuerySessionEntity` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (one event per session change) | `QueryEndpoint` ← `SessionView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `QueryEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `QueryEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `QueryEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### QuerySession (full form, returned by `GET /api/sessions/{id}`)

```json
{
  "sessionId": "s-a3f…",
  "question": "What are the key regulatory requirements for deploying autonomous AI agents in the EU and US?",
  "status": "EXECUTING",
  "currentPlan": {
    "subQueries": [
      {
        "subQueryId": "sq-1",
        "queryText": "EU AI Act obligations for high-risk AI systems",
        "strategy": "CORPUS",
        "round": 1
      },
      {
        "subQueryId": "sq-2",
        "queryText": "US federal AI governance executive orders 2023-2025",
        "strategy": "WEB",
        "round": 1
      },
      {
        "subQueryId": "sq-3",
        "queryText": "autonomous agent definition regulatory frameworks",
        "strategy": "KNOWLEDGE_BASE",
        "round": 1
      }
    ],
    "coverageGoal": "Identify compliance obligations for autonomous AI agents under the EU AI Act and US federal AI policy.",
    "round": 1,
    "revisionReason": null
  },
  "lastEval": {
    "score": 0.87,
    "passing": true,
    "rationale": "Plan uses all three strategies; sub-queries are independent; 3 sub-queries meet coverage minimum.",
    "evaluatedAt": "2026-06-28T09:12:04Z"
  },
  "results": [
    {
      "subQueryId": "sq-1",
      "strategy": "CORPUS",
      "queryText": "EU AI Act obligations for high-risk AI systems",
      "ok": true,
      "content": "The EU AI Act classifies autonomous decision-making agents as high-risk when deployed in regulated sectors...",
      "errorReason": null,
      "verdict": "OK",
      "retrievedAt": "2026-06-28T09:12:08Z"
    }
  ],
  "answer": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:12:01Z",
  "finishedAt": null
}
```

### SessionRow (list form, returned by `GET /api/sessions`)

`SessionRow` mirrors `QuerySession` but `results` is truncated to the last 5 `SubQueryResult` entries plus `truncatedFromTotal: int`. Each entry's `content` is capped at 240 characters. The UI fetches the full session by id on row expand.

Every nullable field on the row record is declared `Optional<T>` (Lesson 6).

### SSE event format

```
event: session-update
data: { "sessionId": "s-a3f…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `sessionId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating unexpected CORPUS results", "haltedAt": "2026-06-28T09:15:33Z" }
```

### PlanEvaluation

```json
{
  "score": 0.87,
  "passing": true,
  "rationale": "Plan uses all three retrieval strategies; sub-queries are independent and non-duplicate; 3+ sub-queries present.",
  "evaluatedAt": "2026-06-28T09:12:04Z"
}
```

### ResearchAnswer

```json
{
  "summary": "EU AI Act Article 6 classifies autonomous decision-making agents as high-risk when deployed in regulated sectors, requiring conformity assessments and human oversight. US Executive Order 14110 directs federal agencies to mandate pre-deployment safety evaluations for AI systems with autonomous action capabilities. Both frameworks converge on transparency, audit logging, and operator override mechanisms as baseline requirements.",
  "citations": [
    "CORPUS: EU AI Act obligations for high-risk AI systems",
    "WEB: US federal AI governance executive orders 2023-2025",
    "KNOWLEDGE_BASE: autonomous agent definition regulatory frameworks"
  ],
  "confidence": 0.9,
  "roundsCompleted": 1,
  "producedAt": "2026-06-28T09:14:22Z"
}
```
