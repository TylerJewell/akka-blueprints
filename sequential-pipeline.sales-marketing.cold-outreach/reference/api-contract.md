# API contract — cold-outreach

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/outreach` | `SubmitOutreachRequest` | `201 { prospectId }` | `OutreachEndpoint` → `ProspectEntity` |
| `GET` | `/api/outreach` | — | `200 [ ProspectRecord... ]` (newest-first) | `OutreachEndpoint` ← `ProspectView` |
| `GET` | `/api/outreach/{id}` | — | `200 ProspectRecord` / `404` | `OutreachEndpoint` ← `ProspectView` |
| `POST` | `/api/outreach/{id}/review` | `ReviewRequest` | `200 { prospectId, status }` | `OutreachEndpoint` → `ProspectEntity` |
| `GET` | `/api/outreach/sse` | — | `text/event-stream` | `OutreachEndpoint` ← `ProspectView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitOutreachRequest (request body)

```json
{
  "contactEmail": "dev@acme.example",
  "companyName": "Acme DevTools"
}
```

### ReviewRequest (request body)

```json
{
  "approved": true,
  "reason": "Good personalization, tone is right."
}
```

### ProspectRecord (response body)

```json
{
  "prospectId": "p-acme-001",
  "contactEmail": "dev@acme.example",
  "profile": {
    "prospectId": "p-acme-001",
    "contactEmail": "dev@acme.example",
    "firmographics": {
      "companyName": "Acme DevTools",
      "domain": "acme.example",
      "industry": "developer-tooling",
      "employeeCount": 120,
      "hqCountry": "US",
      "intentSignals": [
        {
          "signalType": "hiring-engineering",
          "description": "Acme posted 8 senior backend engineering roles in the last 30 days.",
          "observedAt": "2026-06-20T00:00:00Z"
        }
      ],
      "researchedAt": "2026-06-28T10:00:00Z"
    },
    "personalizationHooks": [
      "Acme is scaling its backend team — 8 open roles in the last 30 days"
    ],
    "profiledAt": "2026-06-28T10:00:05Z"
  },
  "draft": {
    "subject": "Scaling Acme's backend team — how we can help",
    "body": "Hi,\n\nI noticed Acme DevTools is scaling its backend engineering team quickly — 8 open roles in the last 30 days is a strong signal. We help developer-tooling companies like yours onboard distributed teams 40% faster.\n\nWould a 20-minute call this week make sense?\n\n[sender-address]\n[unsubscribe]",
    "personalizationFields": ["companyName", "intentSignal.hiring-engineering"],
    "templateId": "saas-technical-buyer",
    "draftedAt": "2026-06-28T10:00:20Z"
  },
  "compliance": {
    "passed": true,
    "failedRules": [],
    "checkedAt": "2026-06-28T10:00:21Z"
  },
  "reviewDecision": {
    "approved": true,
    "reviewerNote": "Good personalization, tone is right.",
    "decidedAt": "2026-06-28T10:05:00Z"
  },
  "outreachEmail": {
    "subject": "Scaling Acme's backend team — how we can help",
    "body": "Hi,\n\nI noticed Acme DevTools is scaling its backend engineering team quickly...\n\n[sender-address]\n[unsubscribe]",
    "recipient": "dev@acme.example",
    "compliance": {
      "passed": true,
      "failedRules": [],
      "checkedAt": "2026-06-28T10:00:21Z"
    },
    "receipt": {
      "messageId": "msg-7a3f1c9e",
      "recipient": "dev@acme.example",
      "sentAt": "2026-06-28T10:05:05Z"
    },
    "finishedAt": "2026-06-28T10:05:05Z"
  },
  "status": "SENT",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:05:05Z",
  "guardrailRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when a guardrail caught a misordered or non-compliant attempt.

### ReviewResponse (POST /api/outreach/{id}/review response body)

```json
{
  "prospectId": "p-acme-001",
  "status": "SENDING"
}
```

If `approved == false`, the returned status is `REVIEW_REJECTED`.

### SSE event format

```
event: outreach-update
data: { "prospectId": "p-acme-001", "status": "AWAITING_REVIEW", "draft": { ... }, ... }
```

One event per state transition (`CREATED`, `RESEARCHING`, `RESEARCHED`, `DRAFTING`, `DRAFTED`, `AWAITING_REVIEW`, `SENDING`, `SENT`, `REVIEW_REJECTED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: outreach-rejection
data: { "prospectId": "p-acme-001", "phase": "SEND", "tool": "sendEmail", "reason": "send-blocked: ...", "rejectedAt": "..." }
```

And one per `ComplianceChecked` event (pass or fail during DRAFT):

```
event: outreach-compliance
data: { "prospectId": "p-acme-001", "passed": false, "failedRules": ["unsubscribe-missing"], "checkedAt": "..." }
```

Clients reconcile by `prospectId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend `SubmitOutreachRequest` and `ProspectCreated` to carry it. The review endpoint should similarly stamp `reviewedBy` from the authenticated reviewer principal.
