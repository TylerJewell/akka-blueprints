# API contract — swebench-evaluator-loop

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/issues` | `{ "repoName": String, "issueNumber": Integer, "description": String, "fileContext"?: String, "maxIterations"?: Integer, "tokenBudget"?: Integer }` | `202 { "issueId": String }` | `IssueEndpoint` → `IssueQueue` |
| `GET` | `/api/issues` | — | `200 [ Issue... ]` (optional `?status=PATCHING\|EVALUATING\|SOLVED\|EXHAUSTED` — filtered client-side from `getAllIssues`) | `IssueEndpoint` ← `IssuesView` |
| `GET` | `/api/issues/{id}` | — | `200 Issue` / `404` | `IssueEndpoint` ← `IssuesView` |
| `GET` | `/api/issues/sse` | — | `text/event-stream` (one event per issue change) | `IssueEndpoint` ← `IssuesView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/issues`

- Missing `fileContext` → `""` (empty string; the agent uses the description alone).
- Missing `maxIterations` → 5 (from `swebench.solving.max-iterations`).
- Missing `tokenBudget` → 50000 (from `swebench.solving.token-budget`).
- `maxIterations` must be in `[1, 20]`; otherwise `400`.
- `tokenBudget` must be in `[1000, 500000]`; otherwise `400`.
- Duplicate-detection window: 30 s on `(repoName, issueNumber)`; the second submission returns the first `issueId` (200) instead of starting a new workflow.

## JSON shapes

### Issue

```json
{
  "issueId": "iss-3a9f1…",
  "repoName": "example-org/myrepo",
  "issueNumber": 42,
  "description": "Function parse_date raises ValueError on dates before 1970",
  "maxIterations": 5,
  "tokenBudget": 50000,
  "status": "SOLVED",
  "iterations": [
    {
      "iterationNumber": 1,
      "patch": {
        "unifiedDiff": "--- a/utils/date_utils.py\n+++ b/utils/date_utils.py\n@@ -12,7 +12,8 @@\n ...",
        "lineCount": 12,
        "patchedAt": "2026-06-28T09:00:05Z"
      },
      "gate": { "passed": true, "reasonCode": "OK", "detail": "" },
      "testResult": {
        "verdict": "FAIL",
        "failures": {
          "failingTests": ["test_negative_epoch", "test_parse_date_utc_boundary"],
          "errorSnippets": ["AssertionError: expected -86400, got 0", "AssertionError: expected 0, got 3600"],
          "overallSummary": "Patch fixes the pre-1970 crash but introduces a timezone offset error."
        },
        "passCount": 8,
        "failCount": 2,
        "evaluatedAt": "2026-06-28T09:00:18Z"
      }
    },
    {
      "iterationNumber": 2,
      "patch": {
        "unifiedDiff": "--- a/utils/date_utils.py\n+++ b/utils/date_utils.py\n@@ -12,7 +12,9 @@\n ...",
        "lineCount": 14,
        "patchedAt": "2026-06-28T09:00:26Z"
      },
      "gate": { "passed": true, "reasonCode": "OK", "detail": "" },
      "testResult": {
        "verdict": "PASS",
        "failures": {
          "failingTests": [],
          "errorSnippets": [],
          "overallSummary": "All 10 tests pass with the revised parse_date implementation."
        },
        "passCount": 10,
        "failCount": 0,
        "evaluatedAt": "2026-06-28T09:00:41Z"
      }
    }
  ],
  "solvedAtIteration": 2,
  "acceptedPatch": "--- a/utils/date_utils.py\n...",
  "exhaustionReason": null,
  "createdAt": "2026-06-28T09:00:02Z",
  "finishedAt": "2026-06-28T09:00:42Z"
}
```

### CI gate verdict form

```json
{ "passed": true,  "reasonCode": "OK",                 "detail": "" }
{ "passed": false, "reasonCode": "MALFORMED_DIFF",      "detail": "Diff missing --- / +++ hunk header on line 1." }
{ "passed": false, "reasonCode": "EXCEEDS_LINE_BUDGET", "detail": "Diff is 612 lines; ceiling is 500." }
```

`reasonCode` is one of `OK`, `MALFORMED_DIFF`, `EXCEEDS_LINE_BUDGET`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: issue-update
data: { "issueId": "iss-3a9f1…", "status": "EVALUATING", "iterations": [...], ... }
```

One event per state transition. Clients reconcile by `issueId`. The full `Issue` JSON is included so a fresh client can render the row without a separate fetch.
