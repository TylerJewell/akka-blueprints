# API contract — code-assistant-loop

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `{ "prompt": String, "requestedBy"?: String }` | `202 { "sessionId": String }` | `SessionEndpoint` → `TaskQueue` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SessionEndpoint` ← `SessionEntity` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (one event per session change) | `SessionEndpoint` ← `SessionView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `SessionEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `SessionEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `SessionEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Session (full form, returned by `GET /api/sessions/{id}`)

```json
{
  "sessionId": "s-4a9…",
  "prompt": "Add a multiply method to Calculator.java and update the unit tests.",
  "status": "EXECUTING",
  "plan": {
    "targetFiles": [
      "/workspace/src/Calculator.java",
      "/workspace/test/CalculatorTest.java"
    ],
    "steps": [
      "Read Calculator.java to understand the existing structure",
      "Edit Calculator.java to add the multiply method",
      "Read CalculatorTest.java to understand test patterns",
      "Edit CalculatorTest.java to add multiply tests",
      "Run tests to validate"
    ],
    "currentDispatch": {
      "actionKind": "EDIT_FILE",
      "targetFile": "/workspace/src/Calculator.java",
      "instruction": "Add multiply(int a, int b) method after the divide method.",
      "rationale": "Method signature matches existing arithmetic methods."
    }
  },
  "editLog": {
    "entries": [
      {
        "attempt": 1,
        "actionKind": "READ_FILE",
        "targetFile": "/workspace/src/Calculator.java",
        "instruction": "Read Calculator.java to understand the existing structure",
        "verdict": "OK",
        "content": "public class Calculator { public int add(int a, int b) ...",
        "diff": null,
        "testOutput": null,
        "blocker": null,
        "recordedAt": "2026-06-28T09:10:04Z"
      }
    ]
  },
  "commitSummary": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:09:55Z",
  "finishedAt": null
}
```

### SessionRow (list form, returned by `GET /api/sessions`)

`SessionRow` mirrors `Session` but `editLog.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`, and each entry's `content` is capped at 240 characters. The UI fetches the full session by id on row expand.

### SSE event format

```
event: session-update
data: { "sessionId": "s-4a9…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `sessionId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "reviewing unexpected edit pattern", "haltedAt": "2026-06-28T09:11:30Z" }
```
