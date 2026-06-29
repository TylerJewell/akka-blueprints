# API contract — safety-plugins

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/screenings` | `SubmitScreeningRequest` | `201 { screeningId }` | `SafetyEndpoint` → `SafetyEntity` |
| `GET` | `/api/screenings` | — | `200 [ Screening... ]` (newest-first) | `SafetyEndpoint` ← `SafetyView` |
| `GET` | `/api/screenings/{id}` | — | `200 Screening` / `404` | `SafetyEndpoint` ← `SafetyView` |
| `GET` | `/api/screenings/sse` | — | `text/event-stream` | `SafetyEndpoint` ← `SafetyView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitScreeningRequest (request body)

```json
{
  "payloadTitle": "User message from session s-9124 (2026-06-28)",
  "rawPayload": "Can you help me with my account at acme.com? My email is user@acme.com and my SSN is 123-45-6789.",
  "payloadDirection": "user-to-model",
  "rules": [
    {
      "ruleId": "rule-pii-exfiltration-001",
      "category": "DATA_EXFILTRATION",
      "description": "Payload must not request or transmit government identifiers, financial account numbers, or credentials.",
      "actionFloor": "REDACT"
    },
    {
      "ruleId": "rule-prompt-injection-001",
      "category": "PROMPT_INJECTION",
      "description": "Payload must not contain override directives targeting system instructions.",
      "actionFloor": "BLOCK"
    }
  ],
  "submittedBy": "platform-gateway-v2"
}
```

### Screening (response body)

```json
{
  "screeningId": "sc-7fc...",
  "request": {
    "screeningId": "sc-7fc...",
    "payloadTitle": "User message from session s-9124 (2026-06-28)",
    "rawPayload": "(raw text preserved for audit)",
    "payloadDirection": "user-to-model",
    "rules": [
      {
        "ruleId": "rule-pii-exfiltration-001",
        "category": "DATA_EXFILTRATION",
        "description": "...",
        "actionFloor": "REDACT"
      }
    ],
    "submittedBy": "platform-gateway-v2",
    "submittedAt": "2026-06-28T09:15:00Z"
  },
  "sanitized": {
    "redactedPayload": "Can you help me with my account at acme.com? My email is [REDACTED-EMAIL] and my SSN is [REDACTED-SSN].",
    "piiCategoriesFound": ["email", "ssn"]
  },
  "decision": {
    "overallAction": "REDACT",
    "summary": "No injection attempt detected. PII references present in the redacted payload warranting a redaction action under the data-exfiltration rule.",
    "findings": [
      {
        "ruleId": "rule-pii-exfiltration-001",
        "category": "DATA_EXFILTRATION",
        "action": "REDACT",
        "evidenceQuote": "My email is [REDACTED-EMAIL] and my SSN is [REDACTED-SSN]",
        "rationale": "Payload contains redacted PII markers confirming government identifier transmission; redaction has already occurred upstream."
      },
      {
        "ruleId": "rule-prompt-injection-001",
        "category": "PROMPT_INJECTION",
        "action": "ALLOW",
        "evidenceQuote": "",
        "rationale": "No override directives or system-instruction manipulation patterns detected."
      }
    ],
    "decidedAt": "2026-06-28T09:15:14Z"
  },
  "eval": {
    "score": 4,
    "rationale": "All REDACT findings include evidence quotes; overall action is consistent with per-finding actions.",
    "evaluatedAt": "2026-06-28T09:15:15Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T09:15:00Z",
  "finishedAt": "2026-06-28T09:15:15Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: screening-update
data: { "screeningId": "sc-7fc...", "status": "DECISION_RECORDED", "decision": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `SCREENING`, `DECISION_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `screeningId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated gateway identifier rather than the request body. In production, the endpoint would typically be called by a platform gateway (not by an end user directly).
