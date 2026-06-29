# API contract — multi-turn-simulator

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `{ "persona": String, "goal": String, "maxTurns"?: Integer, "submittedBy"?: String }` | `202 { "sessionId": String }` | `SimulationEndpoint` → `ScenarioQueue` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (optional `?status=RUNNING\|COMPLETED\|FLAGGED\|HALTED` — filtered client-side from `getAllSessions`) | `SimulationEndpoint` ← `SessionsView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SimulationEndpoint` ← `SessionsView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (one event per session change) | `SimulationEndpoint` ← `SessionsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/sessions`

- Missing `maxTurns` → 10 (from `multi-turn-simulator.simulation.max-turns`).
- Missing `submittedBy` → `"anonymous"`.
- `maxTurns` must be in `[1, 50]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(persona, goal, submittedBy)`; the second submission returns the first `sessionId` (200) instead of starting a new workflow.

## JSON shapes

### Session

```json
{
  "sessionId": "s-3c9a1…",
  "persona": "curious non-expert",
  "goal": "understand the refund policy",
  "maxTurns": 10,
  "status": "COMPLETED",
  "turns": [
    {
      "turnNumber": 1,
      "utterance": {
        "text": "Hi! I bought something last week. Can I get a refund?",
        "turnNumber": 1,
        "utteredAt": "2026-06-28T09:01:00Z"
      },
      "response": {
        "text": "Yes, refunds are available within 30 days of delivery.",
        "turnNumber": 1,
        "respondedAt": "2026-06-28T09:01:03Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "verdict": {
        "outcome": "PASS",
        "scores": { "coherence": 5, "policyAdherence": 5, "personaConsistency": 4, "bias": 5 },
        "overallScore": 4,
        "driftFlagged": false,
        "rationale": "All dimensions above threshold.",
        "evaluatedAt": "2026-06-28T09:01:07Z"
      }
    },
    {
      "turnNumber": 2,
      "utterance": {
        "text": "Got it! Thanks so much.\n[GOAL_COMPLETE]",
        "turnNumber": 2,
        "utteredAt": "2026-06-28T09:01:10Z"
      },
      "response": {
        "text": "You're welcome! Let me know if you have any other questions.",
        "turnNumber": 2,
        "respondedAt": "2026-06-28T09:01:12Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "verdict": {
        "outcome": "PASS",
        "scores": { "coherence": 5, "policyAdherence": 5, "personaConsistency": 5, "bias": 5 },
        "overallScore": 5,
        "driftFlagged": false,
        "rationale": "All dimensions score 5; goal-complete signal detected.",
        "evaluatedAt": "2026-06-28T09:01:15Z"
      }
    }
  ],
  "sessionVerdict": {
    "outcome": "CLEAN",
    "sessionScore": 4,
    "driftFlagged": false,
    "summaryRationale": "Both turns scored above threshold; no drift or policy issues detected.",
    "closedAt": "2026-06-28T09:01:16Z"
  },
  "createdAt": "2026-06-28T09:00:58Z",
  "finishedAt": "2026-06-28T09:01:16Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",               "detail": "" }
{ "passed": false, "reasonCode": "POLICY_VIOLATION",  "detail": "Response contains '[SYSTEM]' pattern at offset 42." }
```

`reasonCode` is one of `OK`, `POLICY_VIOLATION`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: session-update
data: { "sessionId": "s-3c9a1…", "status": "RUNNING", "turns": [...], ... }
```

One event per state transition or per new turn appended. Clients reconcile by `sessionId`. The full `Session` JSON is included so a fresh client can render the row without a separate fetch.
