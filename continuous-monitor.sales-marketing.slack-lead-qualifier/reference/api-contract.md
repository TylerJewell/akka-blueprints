# API contract — slack-lead-qualifier

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/leads` | — | `200 [ Lead... ]` (sorted newest-first; optional `?status=…&tier=…`) | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/leads/{id}` | — | `200 Lead` / `404` | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/leads/sse` | — | `text/event-stream` | `LeadEndpoint` ← `LeadView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### Lead

```json
{
  "leadId": "slk-3ae…",
  "event": {
    "eventId": "evt-7b1…",
    "userId": "U0123XYZ",
    "displayName": "Alex Kim",
    "email": "[REDACTED-EMAIL]",
    "channelId": "C0GENERAL",
    "joinedAt": "2026-06-28T09:12:00Z"
  },
  "enrichment": {
    "company": "Acme Cloud",
    "jobTitle": "VP Engineering",
    "industry": "SaaS",
    "linkedInUrl": null,
    "estimatedHeadcount": 320,
    "location": "Austin, US",
    "rawSearchSummary": "Alex Kim, VP Engineering at Acme Cloud — Series B SaaS platform…"
  },
  "sanitized": {
    "company": "Acme Cloud",
    "jobTitle": "VP Engineering",
    "industry": "SaaS",
    "estimatedHeadcount": 320,
    "piiCategoriesStripped": ["location"]
  },
  "scoring": {
    "score": 82,
    "tier": "HOT",
    "rationale": "VP Engineering at a 320-person SaaS company is a strong ICP match…",
    "matchedCriteria": ["job-seniority:VP", "industry:SaaS", "company-size:50-5000"],
    "missedCriteria": ["engagement-signal:channel-not-technical"]
  },
  "draft": {
    "channel": "C0SALES-LEADS",
    "headline": "New HOT lead: VP Engineering @ Acme Cloud",
    "body": "Score 82 — SaaS, ~320 employees. Joined #general. Matched: seniority, industry, size.",
    "scoreBadge": "82 · HOT",
    "draftedAt": "2026-06-28T09:12:14Z"
  },
  "slackTimestamp": "1751102034.123456",
  "status": "POSTED",
  "createdAt": "2026-06-28T09:12:00Z",
  "finishedAt": "2026-06-28T09:12:15Z"
}
```

### LeadRow (View)

`LeadRow` mirrors `Lead` but omits the raw `event.email` field (the audit log retains it in `SlackEventQueue`).

### SSE event format

```
event: lead-update
data: { "leadId": "slk-3ae…", "status": "SCORED", "tier": "HOT", "score": 82, ... }
```

One event per state transition. Clients reconcile by `leadId`.
