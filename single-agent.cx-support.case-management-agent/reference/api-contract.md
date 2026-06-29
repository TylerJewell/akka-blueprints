# API contract — case-management-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/messages` | `SubmitMessageRequest` | `201 { messageId, caseId }` | `CaseEndpoint` → `CaseEntity` |
| `GET` | `/api/cases` | — | `200 [ CaseRow... ]` (newest-first) | `CaseEndpoint` ← `CaseView` |
| `GET` | `/api/cases/{id}` | — | `200 CaseRow` / `404` | `CaseEndpoint` ← `CaseView` |
| `GET` | `/api/cases/sse` | — | `text/event-stream` | `CaseEndpoint` ← `CaseView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitMessageRequest (request body)

```json
{
  "customerId": "cust-8842",
  "existingCaseId": null,
  "rawText": "I was charged twice for my subscription this month. My email is john.smith@example.com. Please help.",
  "channel": "WEB_CHAT"
}
```

`existingCaseId` is optional — omit or set to `null` for the CREATE path; set to an existing case id to test the UPDATE/ESCALATE/CLOSE paths.

### SubmitMessageResponse (response body)

```json
{
  "messageId": "msg-c9a1...",
  "caseId": "case-3f7b..."
}
```

For a CREATE action the `caseId` is a freshly minted UUID. For UPDATE/ESCALATE/CLOSE it echoes the `existingCaseId` from the request.

### CaseRow (response body — list and get)

```json
{
  "caseId": "case-3f7b...",
  "lastMessage": {
    "messageId": "msg-c9a1...",
    "customerId": "cust-8842",
    "existingCaseId": null,
    "rawText": "(raw text preserved for audit)",
    "channel": "WEB_CHAT",
    "receivedAt": "2026-06-28T14:05:00Z"
  },
  "sanitizedMessage": {
    "redactedText": "I was charged twice for my subscription this month. My email is [REDACTED-EMAIL]. Please help.",
    "piiCategoriesFound": ["email"]
  },
  "lastAction": {
    "actionType": "CREATE",
    "targetCaseId": null,
    "category": "billing",
    "priority": "MEDIUM",
    "tier": "TIER_1",
    "summary": "Customer reports a duplicate charge on their subscription account.",
    "agentReasoning": "No prior case exists; message describes a billing dispute within tier-1 scope."
  },
  "crmRecord": {
    "caseId": "case-3f7b...",
    "customerId": "cust-8842",
    "category": "billing",
    "priority": "MEDIUM",
    "tier": "TIER_1",
    "status": "OPEN",
    "summary": "Customer reports a duplicate charge on their subscription account.",
    "messageIds": ["msg-c9a1..."],
    "openedAt": "2026-06-28T14:05:01Z",
    "resolvedAt": null
  },
  "eval": {
    "score": 4,
    "rationale": "Action is well-typed and summary is factual; reasoning could be more specific about the duplicate-charge signal.",
    "evaluatedAt": "2026-06-28T14:05:12Z"
  },
  "status": "OPEN",
  "createdAt": "2026-06-28T14:05:00Z",
  "resolvedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: case-update
data: { "caseId": "case-3f7b...", "status": "OPEN", "lastAction": { ... }, "eval": { ... }, ... }
```

One event per state transition (`OPEN`, `IN_PROGRESS`, `ESCALATED`, `RESOLVED`, `FAILED`). Clients reconcile by `caseId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `customerId` from the authenticated principal rather than the request body.
