# API contract — sdr-lead-nurture

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/leads` | `IngestLeadRequest` | `201 { leadId }` | `LeadEndpoint` → `LeadEntity` |
| `POST` | `/api/leads/{id}/reply` | `LeadReplyRequest` | `202` | `LeadEndpoint` → `LeadEntity` |
| `GET` | `/api/leads` | — | `200 [ Lead... ]` (newest-first) | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/leads/{id}` | — | `200 Lead` / `404` | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/leads/sse` | — | `text/event-stream` | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### IngestLeadRequest (request body)

```json
{
  "firstName": "Jordan",
  "lastName": "Reyes",
  "company": "Meridian Logistics",
  "jobTitle": "Head of Engineering",
  "email": "jordan.reyes@meridian-logistics.example",
  "phone": "+1-415-555-0192",
  "channel": "WEBSITE_CHAT",
  "initialMessage": "Hi, I saw your post on AI governance. We're evaluating platforms for our agent fleet — can you tell me more about how compliance monitoring works?"
}
```

### LeadReplyRequest (request body)

```json
{
  "message": "That makes sense. What does the pricing look like for a 50-agent deployment?"
}
```

### Lead (response body)

```json
{
  "leadId": "l-7af3c...",
  "contact": {
    "leadId": "l-7af3c...",
    "firstName": "Jordan",
    "lastName": "Reyes",
    "company": "Meridian Logistics",
    "jobTitle": "Head of Engineering",
    "email": "jordan.reyes@meridian-logistics.example",
    "phone": "+1-415-555-0192",
    "channel": "WEBSITE_CHAT",
    "initialMessage": "Hi, I saw your post on AI governance ...",
    "receivedAt": "2026-06-28T14:00:00Z"
  },
  "sanitized": {
    "redactedRecord": "{ \"company\": \"Meridian Logistics\", \"jobTitle\": \"Head of Engineering\", \"email\": \"[REDACTED-EMAIL]\", \"phone\": \"[REDACTED-PHONE]\", \"channel\": \"WEBSITE_CHAT\", \"initialMessage\": \"Hi, I saw your post ...\" }",
    "piiCategoriesFound": ["email", "phone", "person-name"]
  },
  "conversation": [
    {
      "turnId": "t-001",
      "role": "LEAD",
      "message": "Hi, I saw your post on AI governance. We're evaluating platforms ...",
      "sentAt": "2026-06-28T14:00:00Z"
    },
    {
      "turnId": "t-002",
      "role": "AGENT",
      "message": "Thanks for reaching out! Happy to walk you through compliance monitoring. What's the main driver for your evaluation — an upcoming audit or a specific governance requirement?",
      "sentAt": "2026-06-28T14:00:03Z"
    }
  ],
  "booking": null,
  "quality": null,
  "status": "ENGAGING",
  "createdAt": "2026-06-28T14:00:00Z",
  "closedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### MeetingBooked Lead (response body — terminal state)

```json
{
  "leadId": "l-7af3c...",
  "booking": {
    "slot": "2026-07-01T10:00:00",
    "durationMinutes": 30,
    "accountExecutiveId": "ae-042",
    "zoomLink": "https://zoom.example/j/stub-9182736"
  },
  "quality": {
    "score": 4,
    "rationale": "Two discovery questions asked; one objection addressed; close was proportional to conversation length.",
    "evaluatedAt": "2026-06-28T14:08:22Z"
  },
  "status": "MEETING_BOOKED",
  "createdAt": "2026-06-28T14:00:00Z",
  "closedAt": "2026-06-28T14:08:22Z"
}
```

### SSE event format

```
event: lead-update
data: { "leadId": "l-7af3c...", "status": "MEETING_BOOKED", "booking": { ... }, "quality": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SANITIZED`, `ENGAGING`, `MEETING_BOOKED`, `DISMISSED`, `HANDED_OFF`, `FAILED`). Clients reconcile by `leadId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `submittedBy` from the authenticated session rather than accepting it from the request body.
