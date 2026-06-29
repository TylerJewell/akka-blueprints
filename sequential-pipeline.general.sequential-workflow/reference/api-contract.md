# API contract — sequential-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `SubmitJobRequest` | `201 { jobId }` | `JobEndpoint` → `JobEntity` |
| `GET` | `/api/jobs` | — | `200 [ JobRecord... ]` (newest-first) | `JobEndpoint` ← `JobView` |
| `GET` | `/api/jobs/{id}` | — | `200 JobRecord` / `404` | `JobEndpoint` ← `JobView` |
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
  "jobName": "quarterly-report-csv",
  "jobType": "csv-transform",
  "parameters": {
    "sourceFile": "q2-2026-raw.csv",
    "targetSchema": "finance-v3"
  }
}
```

### JobRecord (response body)

```json
{
  "jobId": "j-7c3fa2...",
  "spec": {
    "jobName": "quarterly-report-csv",
    "jobType": "csv-transform",
    "parameters": { "sourceFile": "q2-2026-raw.csv", "targetSchema": "finance-v3" }
  },
  "validation": {
    "fieldResults": [
      { "fieldName": "sourceFile", "status": "OK", "note": "" },
      { "fieldName": "targetSchema", "status": "OK", "note": "" }
    ],
    "valid": true,
    "validatedAt": "2026-06-28T10:00:05Z"
  },
  "enrichedJob": {
    "validation": { "...": "..." },
    "context": {
      "params": [
        { "key": "sourceFile", "value": "q2-2026-raw.csv", "source": "validation" },
        { "key": "targetSchema", "value": "finance-v3", "source": "validation" }
      ],
      "metadata": { "jobType": "csv-transform", "version": "3" },
      "resolvedAt": "2026-06-28T10:00:10Z"
    },
    "enrichedAt": "2026-06-28T10:00:10Z"
  },
  "output": {
    "steps": [
      {
        "stepId": "s-parse",
        "stepName": "Parse input CSV",
        "outcome": "success",
        "artifacts": [
          { "artifactId": "a-3f1c9a02", "name": "parsed-rows.json", "kind": "intermediate", "ref": "mem://parsed-rows" }
        ]
      },
      {
        "stepId": "s-transform",
        "stepName": "Apply column mappings",
        "outcome": "success",
        "artifacts": [
          { "artifactId": "a-8b44d71e", "name": "transformed-output.csv", "kind": "output", "ref": "mem://transformed-output" }
        ]
      }
    ],
    "executedAt": "2026-06-28T10:00:20Z"
  },
  "summary": {
    "title": "csv-transform job, 2026-06-28",
    "outcomeStatement": "Both steps completed successfully; the transformed output is available as artifact a-8b44d71e.",
    "entries": [
      {
        "stepId": "s-parse",
        "heading": "Parse input CSV",
        "body": "The input file was parsed and all rows were accepted.",
        "artifactIds": ["a-3f1c9a02"]
      },
      {
        "stepId": "s-transform",
        "heading": "Apply column mappings",
        "body": "Column mappings were applied to every row. The final CSV is the primary deliverable.",
        "artifactIds": ["a-8b44d71e"]
      }
    ],
    "summarizedAt": "2026-06-28T10:00:25Z"
  },
  "quality": {
    "score": 5,
    "rationale": "Step coverage, artifact attribution, artifact provenance, and entry parity all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:25Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:25Z",
  "guardrailRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path, populated when the agent attempted a misordered tool call and the guardrail caught it.

A `guardrailRejections` entry when a violation fires:

```json
{
  "phase": "VALIDATE",
  "tool": "runStep",
  "reason": "phase-violation: runStep requires status in {ENRICHED, EXECUTING} with enrichedJob present, saw VALIDATING",
  "rejectedAt": "2026-06-28T10:00:02Z"
}
```

### SSE event format

```
event: job-update
data: { "jobId": "j-7c3fa2...", "status": "ENRICHED", "enrichedJob": { ... }, ... }
```

One event per state transition (`CREATED`, `VALIDATING`, `VALIDATED`, `ENRICHING`, `ENRICHED`, `EXECUTING`, `EXECUTED`, `SUMMARIZING`, `SUMMARIZED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: job-rejection
data: { "jobId": "j-7c3fa2...", "phase": "VALIDATE", "tool": "runStep", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `jobId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitJobRequest` record and the `JobCreated` event to carry it.
