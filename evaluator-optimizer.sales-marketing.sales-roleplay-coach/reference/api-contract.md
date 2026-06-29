# API contract — sales-roleplay-coach

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `{ "buyerPersona": String, "product": String, "dealStage"?: String, "dealAmountUsd"?: Integer, "requestedBy"?: String }` | `202 { "sessionId": String }` | `CoachEndpoint` → `ScenarioQueue` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (optional `?status=PITCHING\|EVALUATING\|PASSED\|EXHAUSTED` — filtered client-side from `getAllSessions`) | `CoachEndpoint` ← `SessionsView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `CoachEndpoint` ← `SessionsView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (one event per session change) | `CoachEndpoint` ← `SessionsView` |
| `POST` | `/api/sessions/{id}/turns` | `{ "text": String }` | `202 { "turnNumber": Integer }` | `CoachEndpoint` → `SessionWorkflow` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/sessions`

- Missing `dealStage` → `DISCOVERY` (from `sales-coach.session.default-deal-stage`).
- Missing `requestedBy` → `"anonymous"`.
- `buyerPersona` and `product` are required; omitting either returns `400`.
- `dealAmountUsd` must be a positive integer when present; otherwise `400`.
- Duplicate-detection window: 10 s on `(buyerPersona, product, requestedBy)`; the second submission returns the first `sessionId` (200) instead of starting a new workflow.

## Validation on `POST /api/sessions/{id}/turns`

- `text` is required and must be 1–2000 characters; otherwise `400`.
- Submitting to a session in `PASSED` or `EXHAUSTED` state returns `409 Conflict`.
- Submitting when the workflow is not waiting for a rep turn (e.g., the buyer response is in flight) returns `409 Conflict`.

## JSON shapes

### Session

```json
{
  "sessionId": "s-4a1b8…",
  "dealContext": {
    "buyerPersona": "VP of Engineering at a mid-market SaaS company",
    "product": "AI-powered observability platform",
    "dealStage": "DEMO",
    "dealAmountUsd": 48000,
    "requestedBy": "rep-42"
  },
  "maxTurns": 5,
  "status": "PASSED",
  "turns": [
    {
      "turnNumber": 1,
      "repTurn": {
        "text": "Our platform gives your engineers real-time traces across every service…",
        "submittedAt": "2026-06-28T09:10:01Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "buyerResponse": {
        "text": "That sounds useful in theory, but we already have a dashboard tool. What does yours do that ours doesn't?",
        "signal": "SKEPTICAL",
        "respondedAt": "2026-06-28T09:10:08Z"
      },
      "coachVerdict": {
        "decision": "REVISE",
        "notes": {
          "bullets": [
            "Opening was product-feature-led; connect to the buyer's specific pain (MTTR, on-call fatigue) before describing capabilities.",
            "Discovery question not asked; the buyer's current tool and its gaps are unknown.",
            "No next step or commitment ask."
          ],
          "overallRationale": "Value articulation and discovery both fall below threshold."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T09:10:15Z"
      }
    },
    {
      "turnNumber": 2,
      "repTurn": {
        "text": "Fair point — what's your biggest pain when something goes wrong at 2 AM?",
        "submittedAt": "2026-06-28T09:11:04Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "buyerResponse": {
        "text": "Honestly? Knowing which service to look at first. We spend 20 minutes correlating logs before anyone writes a single line of fix.",
        "signal": "INTERESTED",
        "respondedAt": "2026-06-28T09:11:10Z"
      },
      "coachVerdict": {
        "decision": "ACCEPT",
        "notes": {
          "bullets": [],
          "overallRationale": "Discovery unlocked buyer's core pain; all five rubric dimensions clear."
        },
        "score": 5,
        "evaluatedAt": "2026-06-28T09:11:17Z"
      }
    }
  ],
  "passedAtTurnNumber": 2,
  "passedTurnText": "Fair point — what's your biggest pain when something goes wrong at 2 AM?",
  "exhaustionReason": null,
  "createdAt": "2026-06-28T09:09:58Z",
  "finishedAt": "2026-06-28T09:11:17Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",                "detail": "" }
{ "passed": false, "reasonCode": "PROHIBITED_CONTENT", "detail": "Turn references competitor brand 'Datadog' in a fabricated claim." }
```

`reasonCode` is one of `OK`, `PROHIBITED_CONTENT`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: session-update
data: { "sessionId": "s-4a1b8…", "status": "EVALUATING", "turns": [...], ... }
```

One event per state transition. Clients reconcile by `sessionId`. The full `Session` JSON is included so a fresh client can render the row without a separate fetch.
