# API contract — code-interpreter-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `SubmitJobRequest` | `201 { jobId }` | `JobEndpoint` → `JobEntity` |
| `GET` | `/api/jobs` | — | `200 [ JobRow... ]` (newest-first) | `JobEndpoint` ← `JobView` |
| `GET` | `/api/jobs/{id}` | — | `200 Job` / `404` | `JobEndpoint` ← `JobView` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` | `JobEndpoint` ← `JobView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitJobRequest (request body)

```json
{
  "prompt": "Find the top 5 values in the revenue column and compute their mean.",
  "dataPayload": "product,revenue\nWidget A,12400\nWidget B,9800\nWidget C,22100\nWidget D,4300\nWidget E,17600\nWidget F,6700",
  "dataFormat": "CSV",
  "submittedBy": "analyst-7"
}
```

`dataFormat` is one of `CSV`, `JSON`, or `NUMERIC`.

### Job (response body — full record via GET /api/jobs/{id})

```json
{
  "jobId": "j-4a1...",
  "request": {
    "jobId": "j-4a1...",
    "prompt": "Find the top 5 values in the revenue column and compute their mean.",
    "dataPayload": "product,revenue\nWidget A,12400\n...",
    "dataFormat": "CSV",
    "submittedBy": "analyst-7",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "generatedCode": {
    "pythonSource": "import csv, statistics, io\nDATA_PAYLOAD = DATA_PAYLOAD\nreader = csv.DictReader(io.StringIO(DATA_PAYLOAD))\nvalues = sorted([float(r['revenue']) for r in reader], reverse=True)[:5]\nprint(f'ANSWER: mean={statistics.mean(values):.2f}')\n",
    "importsList": ["csv", "statistics", "io"],
    "generatedAt": "2026-06-28T14:00:04Z"
  },
  "result": {
    "kind": "SCALAR",
    "answer": "mean=15940.00",
    "rows": [],
    "columnNames": [],
    "pythonSourceRan": "import csv, statistics, io\n...",
    "decidedAt": "2026-06-28T14:00:05Z"
  },
  "status": "RESULT_RECORDED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:05Z"
}
```

### JobRow (response body — list item via GET /api/jobs)

`JobRow` mirrors `Job` but omits `request.dataPayload` (the payload can be large). The UI fetches the full record on demand via `GET /api/jobs/{id}`.

```json
{
  "jobId": "j-4a1...",
  "prompt": "Find the top 5 values in the revenue column and compute their mean.",
  "dataFormat": "CSV",
  "submittedBy": "analyst-7",
  "generatedCode": { "importsList": ["csv", "statistics", "io"], "generatedAt": "2026-06-28T14:00:04Z" },
  "result": { "kind": "SCALAR", "answer": "mean=15940.00", "decidedAt": "2026-06-28T14:00:05Z" },
  "status": "RESULT_RECORDED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:05Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: job-update
data: { "jobId": "j-4a1...", "status": "RESULT_RECORDED", "result": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `CODE_APPROVED`, `EXECUTING`, `RESULT_RECORDED`, `HALTED`, `FAILED`). Clients reconcile by `jobId`; each event carries the full row at the moment of transition, so a late-joining client never needs to replay.

### Halt response example (status = HALTED)

```json
{
  "jobId": "j-9z2...",
  "status": "HALTED",
  "haltReason": "TIMED_OUT",
  "wallClockMillis": 10003,
  "createdAt": "2026-06-28T14:02:00Z",
  "finishedAt": "2026-06-28T14:02:10Z"
}
```

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
