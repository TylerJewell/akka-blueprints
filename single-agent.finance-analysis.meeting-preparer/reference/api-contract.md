# API contract — meeting-preparer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/briefs` | `SubmitBriefRequest` | `201 { briefId }` | `BriefingEndpoint` → `BriefingEntity` |
| `GET` | `/api/briefs` | — | `200 [ Briefing... ]` (newest-first) | `BriefingEndpoint` ← `BriefingView` |
| `GET` | `/api/briefs/{id}` | — | `200 Briefing` / `404` | `BriefingEndpoint` ← `BriefingView` |
| `GET` | `/api/briefs/sse` | — | `text/event-stream` | `BriefingEndpoint` ← `BriefingView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitBriefRequest (request body)

```json
{
  "counterpartyName": "Meridian Capital Advisors",
  "meetingAt": "2026-07-03T14:00:00Z",
  "attendees": ["alice.chen@ourbank.com", "bob.ray@ourbank.com"],
  "agendaTopics": ["Q2 performance review", "fee structure negotiation", "upcoming product launch"],
  "requestedBy": "alice.chen",
  "crmSnapshot": {
    "contactName": "James Whitfield",
    "contactEmail": "j.whitfield@meridian.example.com",
    "contactPhone": "+1-212-555-0187",
    "dealStage": "Active — renewal",
    "lastInteractionNote": "Call with James on 2026-05-20: positive on Q1 results, some concern about fee compression in the sector.",
    "lastContactAt": "2026-05-20T10:00:00Z"
  }
}
```

### Briefing (response body)

```json
{
  "briefId": "b-7f3a...",
  "request": {
    "briefId": "b-7f3a...",
    "counterpartyName": "Meridian Capital Advisors",
    "meetingAt": "2026-07-03T14:00:00Z",
    "attendees": ["alice.chen@ourbank.com", "bob.ray@ourbank.com"],
    "agendaTopics": ["Q2 performance review", "fee structure negotiation", "upcoming product launch"],
    "requestedBy": "alice.chen",
    "requestedAt": "2026-06-29T09:00:00Z",
    "crmSnapshot": {
      "contactName": "James Whitfield",
      "contactEmail": "j.whitfield@meridian.example.com",
      "contactPhone": "+1-212-555-0187",
      "dealStage": "Active — renewal",
      "lastInteractionNote": "Call with James on 2026-05-20: positive on Q1 results, some concern about fee compression in the sector.",
      "lastContactAt": "2026-05-20T10:00:00Z"
    }
  },
  "sanitized": {
    "redactedContactName": "[REDACTED-NAME]",
    "redactedContactEmail": "[REDACTED-EMAIL]",
    "redactedContactPhone": "[REDACTED-PHONE]",
    "dealStage": "Active — renewal",
    "lastInteractionNote": "Call with [REDACTED-NAME] on 2026-05-20: positive on Q1 results, some concern about fee compression in the sector.",
    "lastContactAt": "2026-05-20T10:00:00Z",
    "piiCategoriesFound": ["person-name", "email", "phone"]
  },
  "brief": {
    "executiveSummary": "Meridian Capital Advisors is a mid-market PE-backed firm currently in active renewal. The meeting's timing coincides with their Q2 close, making a performance discussion natural.",
    "talkingPoints": [
      {
        "topic": "Q2 performance review",
        "point": "Their trailing AUM grew 6% in Q1 per public filings; acknowledge this before entering any fee discussion.",
        "evidenceSource": "financial-highlights.txt — ADV filing Q1 2026"
      },
      {
        "topic": "fee structure negotiation",
        "point": "The CRM note flags concern about fee compression; prepare a tiered-volume proposal rather than a flat rate.",
        "evidenceSource": "sanitized-crm.txt — CRM note 2026-05-20"
      },
      {
        "topic": "upcoming product launch",
        "point": "A Reuters item notes Meridian is preparing a new credit-focused vehicle; ask about distribution appetite before they approach competitors.",
        "evidenceSource": "news-items.txt — Reuters 2026-06-22"
      }
    ],
    "riskFlags": [
      {
        "label": "Key-person risk",
        "level": "MEDIUM",
        "detail": "The primary contact is the sole relationship owner per CRM; a personnel change at their firm would require re-establishing coverage."
      }
    ],
    "financialHighlights": "AUM: ~$2.1 B as of Q1 2026 (+6% YoY). No material debt events in trailing 12 months.",
    "recentNewsItems": [
      "Meridian Capital prepares new credit vehicle — Reuters, 2026-06-22",
      "Mid-market PE fee compression accelerates — FT, 2026-06-10"
    ],
    "preparedAt": "2026-06-29T09:00:28Z"
  },
  "eval": {
    "score": 5,
    "rationale": "All talking points cite sources; risk flags present; news items populated; executive summary length adequate.",
    "evaluatedAt": "2026-06-29T09:00:29Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-29T09:00:00Z",
  "finishedAt": "2026-06-29T09:00:29Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: briefing-update
data: { "briefId": "b-7f3a...", "status": "BRIEF_READY", "brief": { ... }, ... }
```

One event per state transition (`REQUESTED`, `DATA_SANITIZED`, `BRIEFING`, `BRIEF_READY`, `EVALUATED`, `FAILED`). Clients reconcile by `briefId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
