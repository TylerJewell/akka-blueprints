# API contract — akka-research-bot

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "question": String, "requestedBy"?: String }` | `202 { "jobId": String }` | `JobEndpoint` → `JobQueue` |
| `GET` | `/api/jobs` | — | `200 [ JobRow... ]` | `JobEndpoint` ← `ResearchJobView` |
| `GET` | `/api/jobs/{id}` | — | `200 ResearchJob` / `404` | `JobEndpoint` ← `ResearchJobEntity` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` (one event per job change) | `JobEndpoint` ← `ResearchJobView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `JobEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `JobEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `JobEndpoint` ← `SystemControlEntity` |
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
  "question": "What are the key provisions of the EU AI Act affecting autonomous decision-making?",
  "status": "WRITING",
  "plan": {
    "question": "What are the key provisions of the EU AI Act affecting autonomous decision-making?",
    "queries": [
      {
        "kind": "WEB",
        "query": "EU AI Act Article 22 autonomous decision-making prohibited practices",
        "expectedContribution": "primary legal text on prohibitions"
      },
      {
        "kind": "ACADEMIC",
        "query": "EU AI Act algorithmic decision-making compliance obligations 2024",
        "expectedContribution": "academic analysis of compliance burden"
      },
      {
        "kind": "NEWS",
        "query": "EU AI Act enforcement timeline autonomous systems 2025",
        "expectedContribution": "current implementation status"
      }
    ],
    "notes": null
  },
  "results": {
    "entries": [
      {
        "attempt": 1,
        "query": "EU AI Act Article 22 autonomous decision-making prohibited practices",
        "kind": "WEB",
        "verdict": "OK",
        "scrubbedContent": "Article 22 prohibits solely automated decisions with significant effects... (akka.io, 'EU AI Act Overview')",
        "blocker": null,
        "recordedAt": "2026-06-28T10:12:04Z"
      }
    ]
  },
  "report": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T10:11:55Z",
  "finishedAt": null
}
```

### JobRow (list form, returned by `GET /api/jobs`)

The list form is `JobRow` — same fields as `ResearchJob`, but `results.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. Each entry's `scrubbedContent` is capped at 240 characters. The UI fetches the full job by id on row expand.

### ResearchReport (nested in a COMPLETED job)

```json
{
  "title": "EU AI Act Provisions on Autonomous Decision-Making: Key Requirements",
  "summary": "The EU AI Act establishes strict obligations for high-risk AI systems that make autonomous decisions with significant effects on individuals. Article 22 prohibits fully automated consequential decisions without meaningful human oversight. Deployers must conduct conformity assessments and maintain technical documentation. Academic analysis confirms broad applicability across financial, healthcare, and employment sectors.",
  "sections": [
    {
      "heading": "Prohibited Practices",
      "body": "Article 22 of the EU AI Act prohibits autonomous decisions based solely on automated processing where those decisions produce legal or similarly significant effects. Exceptions apply when the individual provides explicit consent or when the decision is necessary for contract performance.",
      "citations": ["[WEB] EU AI Act Article 22 (akka.io, 'EU AI Act Overview')"]
    }
  ],
  "bibliography": [
    "[WEB] EU AI Act Article 22 (akka.io, 'EU AI Act Overview')",
    "[ACADEMIC] Compliance obligations for high-risk AI systems (arxiv.org, 'EU AI Act Analysis 2024')",
    "[NEWS] EU AI Act enforcement timeline (doc.akka.io, 'AI Regulation News Q1 2025')"
  ],
  "producedAt": "2026-06-28T10:14:33Z"
}
```

### SSE event format

```
event: job-update
data: { "jobId": "rj-4a1…", "status": "WRITING", ... }
```

One event per state transition. Clients reconcile by `jobId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "reviewing unexpected query pattern", "haltedAt": "2026-06-28T10:15:01Z" }
```
