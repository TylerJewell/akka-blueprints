# API contract — chatbot-sim-eval

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/simulations` | `{ "personaKey": String, "issueDescription": String, "maxTurns"?: Integer, "submittedBy"?: String }` | `202 { "simulationId": String }` | `SimulationEndpoint` → `ScenarioQueue` |
| `GET` | `/api/simulations` | — | `200 [ Simulation... ]` (optional `?status=RUNNING\|EVALUATING\|PASSED_EVALUATION\|FAILED_EVALUATION` — filtered client-side from `getAllSimulations`) | `SimulationEndpoint` ← `SimulationsView` |
| `GET` | `/api/simulations/{id}` | — | `200 Simulation` / `404` | `SimulationEndpoint` ← `SimulationsView` |
| `GET` | `/api/simulations/sse` | — | `text/event-stream` (one event per simulation change) | `SimulationEndpoint` ← `SimulationsView` |
| `GET` | `/api/simulations/ci-gate` | — | `200 { "passed": Boolean, "failed": [String...], "pending": [String...] }` (optional `?set=<comma-sep-ids>`) | `SimulationEndpoint` ← `SimulationsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/simulations`

- Missing `maxTurns` → 10 (from `chatbot-sim-eval.simulation.max-turns`).
- Missing `submittedBy` → `"anonymous"`.
- `personaKey` must be one of `frustrated-customer`, `first-time-caller`, `enterprise-admin`, `returns-specialist`; otherwise `400`.
- `maxTurns` must be in `[2, 30]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(personaKey, issueDescription)`; the second submission returns the first `simulationId` (200) instead of starting a new workflow.

## CI gate semantics

`GET /api/simulations/ci-gate` without `?set=` evaluates all simulations ever submitted.
`GET /api/simulations/ci-gate?set=sim-abc,sim-def` evaluates only those two ids.
- `passed: true` — every requested id is in `PASSED_EVALUATION`.
- `passed: false` — at least one id is in `FAILED_EVALUATION`.
- `pending` — ids still in `RUNNING` or `EVALUATING` (not yet counted); the caller should poll until `pending` is empty before trusting `passed`.

## JSON shapes

### Simulation

```json
{
  "simulationId": "sim-9f1b2…",
  "personaKey": "frustrated-customer",
  "issueDescription": "My order from three weeks ago has not arrived.",
  "maxTurns": 10,
  "status": "PASSED_EVALUATION",
  "turns": [
    {
      "turnNumber": 1,
      "userTurn": {
        "text": "I ordered three weeks ago and nothing arrived.",
        "signaledResolution": false,
        "turnedAt": "2026-06-28T09:01:02Z"
      },
      "assistantTurn": {
        "text": "I'm sorry to hear that. Can you confirm the email on the account?",
        "safeContentFlagged": false,
        "flagDetail": "",
        "turnedAt": "2026-06-28T09:01:08Z"
      }
    },
    {
      "turnNumber": 2,
      "userTurn": {
        "text": "orders@example.com",
        "signaledResolution": false,
        "turnedAt": "2026-06-28T09:01:14Z"
      },
      "assistantTurn": {
        "text": "Thank you. I've flagged the shipment for re-dispatch. Reference: [TICKET-PENDING].",
        "safeContentFlagged": false,
        "flagDetail": "",
        "turnedAt": "2026-06-28T09:01:20Z"
      }
    },
    {
      "turnNumber": 3,
      "userTurn": {
        "text": "Fine. I'll wait for the confirmation email.",
        "signaledResolution": true,
        "turnedAt": "2026-06-28T09:01:26Z"
      },
      "assistantTurn": null
    }
  ],
  "verdict": {
    "outcome": "PASS",
    "finding": {
      "findings": [],
      "overallSummary": "All four dimensions at threshold; resolution confirmed on turn 2 with a tracking reference."
    },
    "overallScore": 9,
    "evaluatedAt": "2026-06-28T09:01:45Z"
  },
  "haltReason": "RESOLVED",
  "createdAt": "2026-06-28T09:00:58Z",
  "finishedAt": "2026-06-28T09:01:46Z"
}
```

### Guardrail-flagged assistant turn

```json
{
  "text": "I can guarantee your refund will arrive by Friday.",
  "safeContentFlagged": true,
  "flagDetail": "Pattern 'guarantee … refund … by [date]' matched safe-content rule SC-03.",
  "turnedAt": "2026-06-28T09:02:10Z"
}
```

### CI gate response

```json
{ "passed": false, "failed": ["sim-abc"], "pending": [] }
{ "passed": true,  "failed": [],          "pending": [] }
{ "passed": false, "failed": [],          "pending": ["sim-xyz"] }
```

### SSE event format

```
event: simulation-update
data: { "simulationId": "sim-9f1b2…", "status": "EVALUATING", "turns": [...], ... }
```

One event per state transition. Clients reconcile by `simulationId`. The full `Simulation` JSON is included so a fresh client can render the row without a separate fetch.
