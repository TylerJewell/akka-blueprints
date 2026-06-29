# API contract — mr-reviewer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/mrs` | `{ "projectPath": String, "mrIid": Integer, "targetBranch": String, "diffText": String, "submittedBy"?: String }` | `202 { "mrId": String }` | `ReviewEndpoint` → `MrQueue` |
| `GET` | `/api/mrs` | — | `200 [ MergeRequest... ]` (optional `?status=RECEIVED\|REVIEWING\|GATE_CHECKING\|CI_PASS\|CI_FAIL\|SANITIZER_BLOCKED` — filtered client-side from `getAllMrs`) | `ReviewEndpoint` ← `MrView` |
| `GET` | `/api/mrs/{id}` | — | `200 MergeRequest` / `404` | `ReviewEndpoint` ← `MrView` |
| `GET` | `/api/mrs/{id}/ci-signal` | — | `200 { "mrId": String, "signal": "CI_PASS"\|"CI_FAIL"\|"PENDING" }` | `ReviewEndpoint` ← `MrView` |
| `GET` | `/api/mrs/sse` | — | `text/event-stream` (one event per MR change) | `ReviewEndpoint` ← `MrView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `ReviewEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `ReviewEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `ReviewEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/mrs`

- Missing `submittedBy` → `"anonymous"`.
- `diffText` must be non-empty and under 512 KB; otherwise `400`.
- Duplicate-detection window: 10 s on `(projectPath, mrIid)`; the second submission returns the first `mrId` (200) instead of starting a new workflow.

## JSON shapes

### MergeRequest

```json
{
  "mrId": "mr-9a1b2…",
  "projectPath": "acme/backend",
  "mrIid": 417,
  "targetBranch": "main",
  "maxPasses": 3,
  "status": "CI_PASS",
  "passes": [
    {
      "passNumber": 1,
      "sanitizer": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "findings": [
          {
            "filePath": "internal/auth/handler.go",
            "lineNumber": 88,
            "severity": "WARNING",
            "message": "Return value of token.Validate() is unchecked."
          }
        ],
        "summary": "The MR adds a new token-validation path. One unchecked return was found. Test coverage for the new code path is absent.",
        "qualityScore": 68,
        "commentary": "handler.go line 88: the return value of token.Validate() should be checked — a failure here would be silently swallowed. No tests were added for the new validation path.",
        "reviewedAt": "2026-06-28T09:01:05Z"
      },
      "gate": {
        "decision": "REFINE",
        "feedback": {
          "bullets": [
            "scripts/seed.py has changes but no finding or summary mention.",
            "The summary does not address whether tests were added.",
            "Finding at handler.go line 44 lacks a specific action."
          ],
          "overallRationale": "Coverage and actionability fall below threshold."
        },
        "gateScore": 55,
        "evaluatedAt": "2026-06-28T09:01:18Z"
      }
    },
    {
      "passNumber": 2,
      "sanitizer": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "findings": [
          {
            "filePath": "internal/auth/handler.go",
            "lineNumber": 88,
            "severity": "WARNING",
            "message": "Return value of token.Validate() is unchecked; wrap in an if-err block and return 401."
          },
          {
            "filePath": "scripts/seed.py",
            "lineNumber": 0,
            "severity": "INFO",
            "message": "Seed script updated to use the new token fixture; no security concerns."
          }
        ],
        "summary": "Addresses prior feedback: seed.py now covered, actionability improved. No test additions detected for the new validation path — this remains a gap.",
        "qualityScore": 82,
        "commentary": "handler.go line 88: wrap token.Validate() return value. scripts/seed.py: safe fixture update. No tests were added for the new validation path.",
        "reviewedAt": "2026-06-28T09:01:31Z"
      },
      "gate": {
        "decision": "PASS",
        "feedback": {
          "bullets": [],
          "overallRationale": "All files covered; findings are line-cited and actionable; no secrets reproduced; test gap is noted."
        },
        "gateScore": 78,
        "evaluatedAt": "2026-06-28T09:01:44Z"
      }
    }
  ],
  "acceptedPassNumber": 2,
  "acceptedReview": { "...": "same as passes[1].review" },
  "ciSignal": "CI_PASS",
  "receivedAt": "2026-06-28T09:00:58Z",
  "finishedAt": "2026-06-28T09:01:45Z"
}
```

### Sanitizer verdict forms

```json
{ "passed": true,  "reasonCode": "OK",               "detail": "" }
{ "passed": false, "reasonCode": "SECRET_DETECTED",  "detail": "Diff matches PEM private key header at file config/keys.pem." }
```

`reasonCode` is one of `OK`, `SECRET_DETECTED`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### CI signal form

```json
{ "mrId": "mr-9a1b2…", "signal": "CI_PASS" }
{ "mrId": "mr-9a1b2…", "signal": "CI_FAIL" }
{ "mrId": "mr-9a1b2…", "signal": "PENDING" }
```

### Guardrail verdict form

```json
{ "passed": true,  "reasonCode": "OK",                              "detail": "" }
{ "passed": false, "reasonCode": "COMMENTARY_REPRODUCES_SECRET",   "detail": "Commentary contains a token matching high-entropy pattern at character offset 214." }
```

### SSE event format

```
event: mr-update
data: { "mrId": "mr-9a1b2…", "status": "GATE_CHECKING", "passes": [...], ... }
```

One event per state transition. Clients reconcile by `mrId`. The full `MergeRequest` JSON is included so a fresh client can render the row without a separate fetch.
