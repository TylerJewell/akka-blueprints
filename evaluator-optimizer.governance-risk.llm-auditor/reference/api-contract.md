# API contract — llm-auditor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `{ "text": String, "channel"?: String, "maxRevisions"?: Integer }` | `202 { "sessionId": String }` | `AuditEndpoint` → `ResponseQueue` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (optional `?status=AUDITING\|REVISING\|APPROVED\|ESCALATED` — filtered client-side from `getAllSessions`) | `AuditEndpoint` ← `SessionsView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `AuditEndpoint` ← `SessionsView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (one event per session change) | `AuditEndpoint` ← `SessionsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/sessions`

- Missing `channel` → `"default"`.
- Missing `maxRevisions` → `3` (from `llm-auditor.audit.max-revisions`).
- `maxRevisions` must be in `[1, 10]`; otherwise `400`.
- `text` must be non-empty; otherwise `400`.
- Duplicate-detection window: 10 s on a hash of `(text, channel)`; the second submission returns the first `sessionId` (200) instead of starting a new workflow.

## JSON shapes

### Session

```json
{
  "sessionId": "s-3b8e1…",
  "channel": "support",
  "maxRevisions": 3,
  "status": "APPROVED",
  "cycles": [
    {
      "cycleNumber": 1,
      "input": {
        "text": "Our team guarantees a 99.9% uptime SLA and you are entitled to a full refund.",
        "channel": "support",
        "receivedAt": "2026-06-28T10:01:00Z"
      },
      "guardrail": {
        "passed": true,
        "reasonCode": "OK",
        "detail": "",
        "initialSeverity": 4
      },
      "verdict": {
        "decision": "REVISE",
        "findings": {
          "findings": [
            {
              "dimension": "HALLUCINATION_RISK",
              "description": "Response cites '99.9% uptime SLA' not present in context.",
              "suggestedFix": "Replace with a reference to the published service agreement."
            },
            {
              "dimension": "POLICY_COMPLIANCE",
              "description": "The phrase 'entitled to a full refund' implies a legal commitment.",
              "suggestedFix": "Replace with eligibility language referencing the standard policy."
            }
          ],
          "overallRationale": "Two findings exceed threshold; severity driven by unverified SLA claim."
        },
        "severityScore": 6,
        "auditedAt": "2026-06-28T10:01:12Z"
      },
      "revision": {
        "text": "Our team works to the uptime targets described in our published service agreement…",
        "revisedAt": "2026-06-28T10:01:24Z"
      }
    },
    {
      "cycleNumber": 2,
      "input": {
        "text": "Our team works to the uptime targets described in our published service agreement…",
        "channel": "support",
        "receivedAt": "2026-06-28T10:01:24Z"
      },
      "guardrail": {
        "passed": true,
        "reasonCode": "OK",
        "detail": "",
        "initialSeverity": 1
      },
      "verdict": {
        "decision": "PASS",
        "findings": {
          "findings": [],
          "overallRationale": "All four dimensions within threshold; no unverified claims remain."
        },
        "severityScore": 1,
        "auditedAt": "2026-06-28T10:01:36Z"
      },
      "revision": null
    }
  ],
  "approvedAtCycle": 2,
  "approvedText": "Our team works to the uptime targets described in our published service agreement…",
  "escalationReason": null,
  "createdAt": "2026-06-28T10:00:58Z",
  "finishedAt": "2026-06-28T10:01:37Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",                          "detail": "", "initialSeverity": 4 }
{ "passed": false, "reasonCode": "SEVERITY_EXCEEDS_THRESHOLD",  "detail": "Initial severity 9 exceeds blocking threshold 9.", "initialSeverity": 9 }
```

`reasonCode` is one of `OK`, `SEVERITY_EXCEEDS_THRESHOLD`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: session-update
data: { "sessionId": "s-3b8e1…", "status": "REVISING", "cycles": [...], ... }
```

One event per state transition. Clients reconcile by `sessionId`. The full `Session` JSON is included so a fresh client can render the row without a separate fetch.
