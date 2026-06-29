# API contract — code-eval-loop

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/solutions` | `{ "statement": String, "language"?: String, "submittedBy"?: String }` | `202 { "solutionId": String }` | `CodeEndpoint` → `ProblemQueue` |
| `GET` | `/api/solutions` | — | `200 [ Solution... ]` (optional `?status=GENERATING\|VERIFYING\|PASSED\|EXHAUSTED` — filtered client-side from `getAllSolutions`) | `CodeEndpoint` ← `SolutionsView` |
| `GET` | `/api/solutions/{id}` | — | `200 Solution` / `404` | `CodeEndpoint` ← `SolutionsView` |
| `GET` | `/api/solutions/sse` | — | `text/event-stream` (one event per solution change) | `CodeEndpoint` ← `SolutionsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/solutions`

- Missing `language` → `"java"` (from `code-eval-loop.solution.default-language`).
- Missing `submittedBy` → `"anonymous"`.
- `statement` must be non-empty and at most 4000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(statement, submittedBy)`; the second submission returns the first `solutionId` (200) instead of starting a new workflow.

## JSON shapes

### Solution

```json
{
  "solutionId": "s-9e4b1…",
  "statement": "Write a Java method that reverses a string without using StringBuilder.reverse().",
  "language": "java",
  "maxAttempts": 5,
  "status": "PASSED",
  "attempts": [
    {
      "attemptNumber": 1,
      "code": {
        "code": "public class StringReverser {\n    public static String reverse(String input) {\n        ...\n    }\n}",
        "language": "java",
        "lineCount": 12,
        "generatedAt": "2026-06-28T09:01:02Z"
      },
      "sandbox": { "passed": true, "reasonCode": "OK", "detail": "" },
      "verification": {
        "verdict": "FAIL",
        "notes": {
          "diagnostics": [
            "testNullInput failed: NullPointerException at line 3; null guard required.",
            "testEmptyInput failed: same root cause."
          ],
          "overallRationale": "Two test cases fail due to missing null/empty guard."
        },
        "score": 60,
        "verifiedAt": "2026-06-28T09:01:18Z"
      }
    },
    {
      "attemptNumber": 2,
      "code": {
        "code": "public class StringReverser {\n    public static String reverse(String input) {\n        if (input == null || input.isEmpty()) return input;\n        ...\n    }\n}",
        "language": "java",
        "lineCount": 14,
        "generatedAt": "2026-06-28T09:01:28Z"
      },
      "sandbox": { "passed": true, "reasonCode": "OK", "detail": "" },
      "verification": {
        "verdict": "PASS",
        "notes": { "diagnostics": [], "overallRationale": "All imports resolve; logic correct for all test cases; score 100." },
        "score": 100,
        "verifiedAt": "2026-06-28T09:01:44Z"
      }
    }
  ],
  "passedAttemptNumber": 2,
  "passedCode": "public class StringReverser { ... }",
  "exhaustionReason": null,
  "createdAt": "2026-06-28T09:00:58Z",
  "finishedAt": "2026-06-28T09:01:45Z"
}
```

### Sandbox verdict forms

```json
{ "passed": true,  "reasonCode": "OK",               "detail": "" }
{ "passed": false, "reasonCode": "FORBIDDEN_IMPORT",  "detail": "Forbidden import detected: java.lang.Runtime at line 4." }
{ "passed": false, "reasonCode": "UNSAFE_SYSCALL",    "detail": "Syscall pattern detected: ProcessBuilder usage at line 8." }
```

`reasonCode` is one of `OK`, `FORBIDDEN_IMPORT`, `UNSAFE_SYSCALL`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: solution-update
data: { "solutionId": "s-9e4b1…", "status": "VERIFYING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `solutionId`. The full `Solution` JSON is included so a fresh client can render the row without a separate fetch.
