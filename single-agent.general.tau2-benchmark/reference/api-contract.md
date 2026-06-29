# API contract — tau2-benchmark-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `BenchmarkEndpoint` → `BenchmarkRunEntity` |
| `GET` | `/api/runs` | — | `200 [ BenchmarkRun... ]` (newest-first) | `BenchmarkEndpoint` ← `BenchmarkView` |
| `GET` | `/api/runs/{id}` | — | `200 BenchmarkRun` / `404` | `BenchmarkEndpoint` ← `BenchmarkView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` | `BenchmarkEndpoint` ← `BenchmarkView` |
| `GET` | `/api/runs/aggregate` | — | `200 AggregateStats` | `BenchmarkEndpoint` ← computed |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRunRequest (request body)

```json
{
  "task": {
    "taskId": "tau2-webnav-001",
    "category": "web-navigation",
    "description": "Find the primary contact email on example.com and state its domain.",
    "steps": [
      "Locate the contact email on example.com",
      "State the domain of that email address"
    ],
    "referenceAnswer": "example.com",
    "maxSteps": 2
  },
  "submittedBy": "researcher-42"
}
```

### BenchmarkRun (response body)

```json
{
  "runId": "r-8c3...",
  "request": {
    "runId": "r-8c3...",
    "task": {
      "taskId": "tau2-webnav-001",
      "category": "web-navigation",
      "description": "Find the primary contact email on example.com and state its domain.",
      "steps": ["Locate the contact email on example.com", "State the domain of that email address"],
      "referenceAnswer": "example.com",
      "maxSteps": 2
    },
    "submittedBy": "researcher-42",
    "submittedAt": "2026-06-28T12:34:00Z"
  },
  "result": {
    "outcome": "PASS",
    "stepOutputs": [
      {
        "stepIndex": 0,
        "output": "The contact email listed on the example.com About page is contact@example.com.",
        "attempted": true
      },
      {
        "stepIndex": 1,
        "output": "The domain of contact@example.com is example.com.",
        "attempted": true
      }
    ],
    "finalAnswer": "example.com",
    "latencyMs": 4200,
    "completedAt": "2026-06-28T12:34:12Z"
  },
  "score": {
    "score": 5,
    "rationale": "All steps attempted with non-empty outputs; final answer matches reference answer; latency within bounds.",
    "scoredAt": "2026-06-28T12:34:13Z"
  },
  "status": "SCORED",
  "createdAt": "2026-06-28T12:34:00Z",
  "finishedAt": "2026-06-28T12:34:13Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### AggregateStats (response body)

```json
{
  "totalRuns": 12,
  "passCount": 7,
  "partialCount": 3,
  "failCount": 2,
  "passRate": 0.583,
  "meanScore": 3.8,
  "byCategory": {
    "web-navigation": { "total": 4, "pass": 3, "partial": 1, "fail": 0, "meanScore": 4.5 },
    "tool-use": { "total": 4, "pass": 2, "partial": 2, "fail": 0, "meanScore": 3.75 },
    "multi-step-reasoning": { "total": 4, "pass": 2, "partial": 0, "fail": 2, "meanScore": 3.0 }
  }
}
```

Computed server-side from the full run list on each request. Only SCORED runs are included in aggregate statistics; SUBMITTED, EXECUTING, RESULT_RECORDED, and FAILED runs are excluded from `passCount`, `passRate`, and `meanScore` but counted in `totalRuns`.

### SSE event format

```
event: run-update
data: { "runId": "r-8c3...", "status": "SCORED", "result": { ... }, "score": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `EXECUTING`, `RESULT_RECORDED`, `SCORED`, `FAILED`). Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
