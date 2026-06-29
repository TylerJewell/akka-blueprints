# API contract — research-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "query": String, "requestedBy"?: String }` | `202 { "jobId": String }` | `ResearchEndpoint` → `QueryQueue` |
| `GET` | `/api/jobs` | — | `200 [ JobRow... ]` | `ResearchEndpoint` ← `ResearchJobView` |
| `GET` | `/api/jobs/{id}` | — | `200 ResearchJob` / `404` | `ResearchEndpoint` ← `ResearchJobEntity` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` (one event per job change) | `ResearchEndpoint` ← `ResearchJobView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `ResearchEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `ResearchEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `ResearchEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### ResearchJob (full form, returned by `GET /api/jobs/{id}`)

```json
{
  "jobId": "rj-4a1…",
  "query": "Summarise the EU AI Act's transparency requirements for high-risk AI systems.",
  "status": "SEARCHING",
  "searchLedger": {
    "context": "User wants a structured summary of transparency obligations under the EU AI Act.",
    "knownFacts": ["The EU AI Act is in force as of 2024."],
    "missingFacts": ["Which articles cover transparency?", "What are provider obligations?"],
    "plan": [
      "Web-search for EU AI Act transparency articles",
      "Document-search for cached regulatory text",
      "Compose a structured summary with citations"
    ],
    "currentDispatch": {
      "searcher": "WEB",
      "query": "EU AI Act transparency requirements high-risk systems",
      "rationale": "Need to anchor the search in the primary regulatory text."
    }
  },
  "findingsLedger": {
    "entries": [
      {
        "attempt": 1,
        "searcher": "WEB",
        "query": "EU AI Act transparency requirements high-risk systems",
        "content": "EU AI Act Article 13 requires high-risk AI providers to ensure transparency...\nhttps://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689",
        "citationVerdict": "CITED",
        "confidence": 0.75,
        "noSourceReason": null,
        "recordedAt": "2026-06-28T09:11:05Z"
      }
    ]
  },
  "report": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:10:58Z",
  "finishedAt": null
}
```

### JobRow (list form, returned by `GET /api/jobs`)

The list form is `JobRow` — same fields as `ResearchJob`, but `findingsLedger.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full job by id on row expand.

### SSE event format

```
event: job-update
data: { "jobId": "rj-4a1…", "status": "SEARCHING", ... }
```

One event per state transition. Clients reconcile by `jobId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "reviewing unexpected UNCITED rate", "haltedAt": "2026-06-28T09:13:40Z" }
```
