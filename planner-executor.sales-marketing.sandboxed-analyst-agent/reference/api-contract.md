# API contract — sandboxed-analyst-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "datasetName": String, "uploaderEmail"?: String, "csvContent"?: String }` | `202 { "jobId": String }` | `AnalysisEndpoint` → `UploadQueue` |
| `GET` | `/api/jobs` | — | `200 [ AnalysisJob... ]` | `AnalysisEndpoint` ← `AnalysisJobView` |
| `GET` | `/api/jobs/{id}` | — | `200 AnalysisJob` / `404` | `AnalysisEndpoint` ← `AnalysisJobEntity` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` (one event per job change) | `AnalysisEndpoint` ← `AnalysisJobView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `AnalysisEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `AnalysisEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `AnalysisEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### AnalysisJob (full form, returned by `GET /api/jobs/{id}`)

```json
{
  "jobId": "j-8a3…",
  "datasetName": "q1-2026-inbound-calls.csv",
  "uploaderEmail": "analyst@example.com",
  "status": "EXECUTING",
  "ledger": {
    "datasetFacts": ["Q1 2026 inbound call records, ~4200 rows"],
    "hypotheses": ["longer calls correlate with closed deals"],
    "plan": [
      "LOAD_INSPECT — print columns, dtypes, row count",
      "AGGREGATE — mean call duration by outcome",
      "AGGREGATE — close rate per sales rep",
      "SUMMARISE — compile findings"
    ],
    "currentStep": {
      "scriptKind": "AGGREGATE",
      "pythonScript": "import pandas as pd\ndf = pd.read_csv('/workspace/dataset.csv')\nprint(df.groupby('outcome')['duration_s'].mean())",
      "rationale": "Compute mean call duration per outcome to test the first hypothesis."
    }
  },
  "progress": {
    "entries": [
      {
        "attempt": 1,
        "scriptKind": "LOAD_INSPECT",
        "script": "import pandas as pd\ndf = pd.read_csv('/workspace/dataset.csv')\nprint(df.dtypes)\nprint(df.shape)",
        "verdict": "OK",
        "scrubbedOutput": "duration_s      float64\noutcome          object\nrep_name         object\ndtype: object\n(4218, 8)",
        "blocker": null,
        "recordedAt": "2026-06-28T09:11:04Z"
      }
    ]
  },
  "report": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:10:57Z",
  "finishedAt": null
}
```

### AnalysisJob (list form, returned by `GET /api/jobs`)

The list form is `AnalysisJobRow` — same fields as `AnalysisJob`, but `progress.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full job by id on row expand.

### SSE event format

```
event: job-update
data: { "jobId": "j-8a3…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `jobId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "reviewing unexpected script output", "haltedAt": "2026-06-28T09:14:55Z" }
```
