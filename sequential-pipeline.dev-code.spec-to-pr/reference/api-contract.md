# API contract — spec-to-pr

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `SpecEndpoint` → `SpecRunEntity` |
| `GET` | `/api/runs` | — | `200 [ SpecRunRecord... ]` (newest-first) | `SpecEndpoint` ← `SpecRunView` |
| `GET` | `/api/runs/{id}` | — | `200 SpecRunRecord` / `404` | `SpecEndpoint` ← `SpecRunView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` | `SpecEndpoint` ← `SpecRunView` |
| `POST` | `/api/runs/{id}/approve` | `ReviewRequest` | `204` | `SpecEndpoint` → `SpecRunEntity` + workflow signal |
| `POST` | `/api/runs/{id}/reject` | `ReviewRequest` | `204` | `SpecEndpoint` → `SpecRunEntity` + workflow signal |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRunRequest (request body)

```json
{
  "specText": "Add rate-limiting middleware that rejects requests exceeding 100 req/s per client IP."
}
```

### ReviewRequest (request body — approve and reject endpoints)

```json
{
  "reviewerId": "eng-alice",
  "comment": "LGTM — diffs look scoped correctly."
}
```

### SpecRunRecord (response body)

```json
{
  "runId": "r-5af9c2...",
  "specText": "Add rate-limiting middleware ...",
  "parsedSpec": {
    "specTitle": "Add rate-limiting middleware",
    "requirements": [
      {
        "reqId": "r-a3f71c80",
        "text": "Introduce a rate-limiting middleware that rejects requests exceeding 100 req/s per client IP.",
        "priority": "high",
        "affectedFiles": ["src/middleware/RateLimiter.java", "src/config/AppConfig.java"]
      }
    ],
    "parsedAt": "2026-06-28T10:00:00Z"
  },
  "changePlan": {
    "changes": [
      {
        "filePath": "src/middleware/RateLimiter.java",
        "changeType": "add",
        "rationale": "New token-bucket rate limiter.",
        "proposedDiff": "+ public class RateLimiter { ... }"
      }
    ],
    "plannedAt": "2026-06-28T10:00:05Z"
  },
  "draftPr": {
    "title": "feat: add rate-limiting middleware (100 req/s per IP)",
    "description": "## Summary\n\nAdds a token-bucket rate limiter ...",
    "fileChanges": [ ... ],
    "draftedAt": "2026-06-28T10:00:10Z"
  },
  "reviewDecision": {
    "reviewerId": "eng-alice",
    "decision": "approved",
    "comment": "LGTM — diffs look scoped correctly.",
    "decidedAt": "2026-06-28T10:02:00Z"
  },
  "ciResult": {
    "passed": true,
    "checks": [
      { "name": "compile", "passed": true, "message": "All file paths found in manifest." },
      { "name": "test", "passed": true, "message": "No test-paired files deleted." },
      { "name": "lint", "passed": true, "message": "All diffs within 200-line limit." }
    ],
    "evaluatedAt": "2026-06-28T10:02:03Z"
  },
  "status": "MERGE_READY",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:02:03Z",
  "guardrailRejections": [
    {
      "phase": "PARSE",
      "tool": "composePrTitle",
      "reason": "phase-violation: composePrTitle requires status in {PLANNED, DRAFTING} with changePlan present, saw PARSING",
      "rejectedAt": "2026-06-28T10:00:01Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when the agent attempted a misordered tool call and the guardrail caught it.

### SSE event format

```
event: run-update
data: { "runId": "r-5af9c2...", "status": "AWAITING_REVIEW", "draftPr": { ... }, ... }
```

One event per state transition (`CREATED`, `PARSING`, `PARSED`, `PLANNING`, `PLANNED`, `DRAFTING`, `DRAFTED`, `AWAITING_REVIEW`, `CI_RUNNING`, `CI_PASSED`, `MERGE_READY`, `CI_FAILED`, `REJECTED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: run-rejection
data: { "runId": "r-5af9c2...", "phase": "PARSE", "tool": "composePrTitle", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware, stamp a `submittedBy` field from the authenticated principal (extend `SubmitRunRequest` and `RunCreated`), and validate that `reviewerId` in `ReviewRequest` matches an authorised reviewer principal.
