# API contract — structured-output-reflection

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/generations` | `{ "schemaName": String, "prompt": String, "requestedBy"?: String }` | `202 { "generationId": String }` | `GenerationEndpoint` → `SubmissionQueue` |
| `GET` | `/api/generations` | — | `200 [ Generation... ]` (optional `?status=GENERATING\|VALIDATING\|PASSED\|FAILED_FINAL` — filtered client-side from `getAllGenerations`) | `GenerationEndpoint` ← `GenerationsView` |
| `GET` | `/api/generations/{id}` | — | `200 Generation` / `404` | `GenerationEndpoint` ← `GenerationsView` |
| `GET` | `/api/generations/sse` | — | `text/event-stream` (one event per generation change) | `GenerationEndpoint` ← `GenerationsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/generations`

- Missing `requestedBy` → `"anonymous"`.
- `schemaName` must be one of `product`, `event-record`, `contact`; otherwise `400`.
- Duplicate-detection window: 10 s on `(schemaName, prompt, requestedBy)`; the second submission returns the first `generationId` (200) instead of starting a new workflow.

## JSON shapes

### Generation

```json
{
  "generationId": "g-9a1c7…",
  "schemaName": "product",
  "prompt": "a professional-grade wireless keyboard for developers",
  "maxAttempts": 4,
  "status": "PASSED",
  "attempts": [
    {
      "attemptNumber": 1,
      "output": {
        "json": "{\"name\":\"DevBoard Pro Wireless\",\"price\":\"149.99\",\"sku\":\"KB-DPW-001\",\"description\":\"...\",\"category\":\"Computer Peripherals\"}",
        "tokenCount": 42,
        "parseable": true,
        "generatedAt": "2026-06-28T09:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "report": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Field 'price' value is a string; expected a number.",
            "Field 'description' does not address the developer use-case in the prompt.",
            "Field 'sku' format is generic; consider a more specific code."
          ],
          "overallRationale": "Schema conformance fails on price type; prompt fidelity below threshold."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T09:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "output": {
        "json": "{\"name\":\"DevBoard Pro Wireless\",\"price\":149.99,\"sku\":\"KB-DPW-001\",\"description\":\"Compact tenkeyless wireless keyboard with mechanical switches, 90-day battery life.\",\"category\":\"Computer Peripherals\"}",
        "tokenCount": 48,
        "parseable": true,
        "generatedAt": "2026-06-28T09:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "report": {
        "verdict": "PASS",
        "notes": { "bullets": [], "overallRationale": "All required fields present with correct types; content matches prompt." },
        "score": 5,
        "evaluatedAt": "2026-06-28T09:01:33Z"
      }
    }
  ],
  "passedAttemptNumber": 2,
  "passedJson": "{\"name\":\"DevBoard Pro Wireless\",\"price\":149.99,...}",
  "failureReason": null,
  "createdAt": "2026-06-28T09:00:59Z",
  "finishedAt": "2026-06-28T09:01:34Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",            "detail": "" }
{ "passed": false, "reasonCode": "SCHEMA_INVALID", "detail": "Output is not valid JSON or is missing required schema fields." }
```

`reasonCode` is one of `OK`, `SCHEMA_INVALID`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: generation-update
data: { "generationId": "g-9a1c7…", "status": "VALIDATING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `generationId`. The full `Generation` JSON is included so a fresh client can render the row without a separate fetch.
