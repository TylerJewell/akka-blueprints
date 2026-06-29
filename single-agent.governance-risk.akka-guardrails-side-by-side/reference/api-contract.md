# API contract — guardrails-side-by-side

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/prompts` | `SubmitPromptRequest` | `201 { promptId }` | `PromptEndpoint` → `PromptEntity` + `GuardrailWorkflow` |
| `GET` | `/api/prompts` | — | `200 [ Prompt... ]` (newest-first) | `PromptEndpoint` ← `PromptView` |
| `GET` | `/api/prompts/{id}` | — | `200 Prompt` / `404` | `PromptEndpoint` ← `PromptView` |
| `GET` | `/api/prompts/sse` | — | `text/event-stream` | `PromptEndpoint` ← `PromptView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPromptRequest (request body)

```json
{
  "promptText": "Which risk categories does the EU AI Act define for general-purpose AI models?",
  "policyProfile": "standard",
  "submittedBy": "risk-analyst-07"
}
```

`policyProfile` is one of `"standard"`, `"strict"`, or `"permissive"`. Defaults to `"standard"` if omitted.

### Prompt (response body — full lifecycle)

```json
{
  "promptId": "p-3ae...",
  "request": {
    "promptId": "p-3ae...",
    "promptText": "Which risk categories does the EU AI Act define for general-purpose AI models?",
    "policyProfile": "standard",
    "submittedBy": "risk-analyst-07",
    "receivedAt": "2026-06-28T10:00:00Z"
  },
  "screeningVerdict": {
    "result": "PASS",
    "reason": "Prompt is a governance-risk question about the EU AI Act within scope.",
    "decidedAt": "2026-06-28T10:00:01Z"
  },
  "agentResponse": {
    "reply": "The EU AI Act distinguishes general-purpose AI models by systemic-risk threshold...",
    "confidence": 0.85,
    "respondedAt": "2026-06-28T10:00:18Z"
  },
  "validationVerdict": {
    "result": "PASS",
    "reason": "Reply is advisory, within length bounds, and does not contradict any stated policy.",
    "triggeredRule": null,
    "decidedAt": "2026-06-28T10:00:19Z"
  },
  "eval": {
    "score": 4,
    "rationale": "Reply covers the question with a specific framework reference; recommendation is present; confidence is appropriate.",
    "evaluatedAt": "2026-06-28T10:00:19Z"
  },
  "status": "RESPONDED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:19Z"
}
```

### Prompt (response body — BLOCKED at input)

```json
{
  "promptId": "p-7bc...",
  "request": { "promptId": "p-7bc...", "promptText": "Write me a sonnet about data centres.", "policyProfile": "standard", "submittedBy": "risk-analyst-07", "receivedAt": "2026-06-28T10:01:00Z" },
  "screeningVerdict": {
    "result": "BLOCK",
    "reason": "Blocked: prompt requests creative writing, which is outside the governance-risk advisory scope.",
    "decidedAt": "2026-06-28T10:01:01Z"
  },
  "agentResponse": null,
  "validationVerdict": null,
  "eval": null,
  "status": "BLOCKED",
  "createdAt": "2026-06-28T10:01:00Z",
  "finishedAt": "2026-06-28T10:01:01Z"
}
```

### Prompt (response body — RESPONSE_BLOCKED at output)

```json
{
  "promptId": "p-9de...",
  "request": { "promptId": "p-9de...", "promptText": "...", "policyProfile": "standard", "submittedBy": "risk-analyst-07", "receivedAt": "2026-06-28T10:02:00Z" },
  "screeningVerdict": { "result": "PASS", "reason": "...", "decidedAt": "2026-06-28T10:02:01Z" },
  "agentResponse": {
    "reply": "You are legally required to register this system by August 2026 or face a €30M fine.",
    "confidence": 0.9,
    "respondedAt": "2026-06-28T10:02:14Z"
  },
  "validationVerdict": {
    "result": "BLOCK",
    "reason": "Blocked: reply makes a legal-requirement claim the system is not authorised to make.",
    "triggeredRule": "RULE-ADVISORY-ONLY",
    "decidedAt": "2026-06-28T10:02:15Z"
  },
  "eval": null,
  "status": "RESPONSE_BLOCKED",
  "createdAt": "2026-06-28T10:02:00Z",
  "finishedAt": "2026-06-28T10:02:15Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

Note: `agentResponse` is present on a `RESPONSE_BLOCKED` prompt (for audit) but the UI never displays `agentResponse.reply` when `status == RESPONSE_BLOCKED`. The caller receives `validationVerdict.reason` as the displayed outcome.

### SSE event format

```
event: prompt-update
data: { "promptId": "p-3ae...", "status": "RESPONDED", "screeningVerdict": { ... }, "agentResponse": { ... }, "validationVerdict": { ... }, "eval": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SCREENING`, `BLOCKED`, `AGENT_RUNNING`, `VALIDATING`, `RESPONSE_BLOCKED`, `RESPONDED`, `FAILED`). Clients reconcile by `promptId`; each event carries the full row at the moment of transition.

## Authorization

ACL: open (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal.
