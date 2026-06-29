# API contract — competitive-coding-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/submissions` | `{ "title": String, "statementText": String, "sampleCases"?: TestCase[], "timeLimitMs"?: Integer, "memoryLimitMb"?: Integer, "submittedBy"?: String }` | `202 { "submissionId": String }` | `SolverEndpoint` → `ProblemQueue` |
| `GET` | `/api/submissions` | — | `200 [ Submission... ]` (optional `?status=GENERATING\|JUDGING\|ACCEPTED\|REJECTED_FINAL` — filtered client-side from `getAllSubmissions`) | `SolverEndpoint` ← `SubmissionsView` |
| `GET` | `/api/submissions/{id}` | — | `200 Submission` / `404` | `SolverEndpoint` ← `SubmissionsView` |
| `GET` | `/api/submissions/sse` | — | `text/event-stream` (one event per submission change) | `SolverEndpoint` ← `SubmissionsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/submissions`

- Missing `sampleCases` → `[]` (zero test cases; the sandbox will still compile and report any runtime error).
- Missing `timeLimitMs` → `2000` (from `competitive-coding-agent.solving.default-time-limit-ms`).
- Missing `memoryLimitMb` → `256` (from `competitive-coding-agent.solving.default-memory-limit-mb`).
- Missing `submittedBy` → `"anonymous"`.
- `timeLimitMs` must be in `[100, 10000]`; `memoryLimitMb` must be in `[16, 1024]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(title, submittedBy)`; the second submission returns the first `submissionId` (`200`) instead of starting a new workflow.

## JSON shapes

### TestCase

```json
{ "inputData": "3\n1 2 3", "expectedOutput": "6" }
```

### Submission

```json
{
  "submissionId": "s-4a9c1…",
  "problemTitle": "Prefix Sum Maximum",
  "statementText": "Given N integers, find the maximum prefix sum…",
  "timeLimitMs": 2000,
  "memoryLimitMb": 256,
  "maxAttempts": 5,
  "status": "ACCEPTED",
  "attempts": [
    {
      "attemptNumber": 1,
      "solution": {
        "sourceCode": "import java.io.*;\npublic class Main { … }",
        "language": "java",
        "lineCount": 18,
        "generatedAt": "2026-06-28T09:01:02Z"
      },
      "sandbox": {
        "resourcesOk": true,
        "reasonCode": "OK",
        "detail": "",
        "caseResults": [
          { "caseIndex": 0, "actualOutput": "6", "passed": true, "verdict": "AC" },
          { "caseIndex": 1, "actualOutput": "0", "passed": false, "verdict": "WA" }
        ],
        "peakMemoryMb": 24,
        "wallTimeMs": 145
      },
      "verdict": {
        "outcome": "FAIL",
        "notes": {
          "bullets": [
            "Case 1 fails; solution does not handle all-negative input — prefix sum never resets to zero.",
            "Edge case: n=1 produces incorrect output when the single value is negative.",
            "Time complexity O(n^2) in the inner scan loop; may TLE at n=10^5."
          ],
          "overallRationale": "Correctness and complexity fall below the pass threshold."
        },
        "passedCases": 1,
        "totalCases": 2,
        "evaluatedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "solution": {
        "sourceCode": "import java.io.*;\npublic class Main { … }",
        "language": "java",
        "lineCount": 22,
        "generatedAt": "2026-06-28T09:01:21Z"
      },
      "sandbox": {
        "resourcesOk": true,
        "reasonCode": "OK",
        "detail": "",
        "caseResults": [
          { "caseIndex": 0, "actualOutput": "6", "passed": true, "verdict": "AC" },
          { "caseIndex": 1, "actualOutput": "0", "passed": true, "verdict": "AC" }
        ],
        "peakMemoryMb": 26,
        "wallTimeMs": 132
      },
      "verdict": {
        "outcome": "PASS",
        "notes": { "bullets": [], "overallRationale": "All 2 cases match expected output; resource usage within limits." },
        "passedCases": 2,
        "totalCases": 2,
        "evaluatedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "acceptedAttemptNumber": 2,
  "acceptedSourceCode": "import java.io.*;\npublic class Main { … }",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Sandbox report form

```json
{ "resourcesOk": true,  "reasonCode": "OK",  "detail": "" }
{ "resourcesOk": false, "reasonCode": "TLE", "detail": "Wall time 3218ms exceeded limit 2000ms." }
{ "resourcesOk": false, "reasonCode": "MLE", "detail": "Peak memory 312MB exceeded limit 256MB." }
```

`reasonCode` is one of `OK`, `TLE`, `MLE`, `CE` (compile error). Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: submission-update
data: { "submissionId": "s-4a9c1…", "status": "JUDGING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `submissionId`. The full `Submission` JSON is included so a fresh client can render the row without a separate fetch.
