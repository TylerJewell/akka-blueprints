# API contract — self-improving-deep-researcher

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `{ "topic": String, "depthHint"?: String, "sourceTypePreference"?: String, "requestedBy"?: String }` | `202 { "sessionId": String }` | `ResearchEndpoint` → `QueryQueue` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (optional `?status=RESEARCHING\|EVALUATING\|ACCEPTED\|MAX_ATTEMPTS_REACHED` — filtered client-side from `getAllSessions`) | `ResearchEndpoint` ← `SessionsView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `ResearchEndpoint` ← `SessionsView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (one event per session change; add `?channel=memory-updates` for memory-only stream) | `ResearchEndpoint` ← `SessionsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/sessions`

- Missing `depthHint` → `"standard"`.
- Missing `sourceTypePreference` → `"any"`.
- Missing `requestedBy` → `"anonymous"`.
- `depthHint` must be one of `brief`, `standard`, `deep`; otherwise `400`.
- Duplicate-detection window: 15 s on `(topic, requestedBy)`; the second submission returns the first `sessionId` (200) instead of starting a new workflow.

## JSON shapes

### Session

```json
{
  "sessionId": "s-9a1b2…",
  "topic": "regulatory approaches to large language model transparency in the EU",
  "depthHint": "standard",
  "maxAttempts": 3,
  "status": "ACCEPTED",
  "attempts": [
    {
      "attemptNumber": 1,
      "report": {
        "executiveSummary": "EU regulatory efforts center on the AI Act's transparency obligations …",
        "evidence": [
          {
            "claim": "The EU AI Act distinguishes GPAI models by systemic-risk threshold …",
            "sourceRef": "https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:32024R1689",
            "confidence": 0.95
          }
        ],
        "sources": { "sourceRefs": ["https://eur-lex.europa.eu/…"], "totalSources": 4 },
        "researchedAt": "2026-06-28T09:01:02Z"
      },
      "evaluation": {
        "verdict": "REFINE",
        "notes": {
          "bullets": [
            "Coverage omits the open-weight model exemption.",
            "Three of four evidence segments cite the same base URL.",
            "Executive summary lists findings rather than synthesizing them."
          ],
          "overallRationale": "Coverage and source diversity fall below threshold."
        },
        "score": 2,
        "evaluatedAt": "2026-06-28T09:01:18Z"
      }
    },
    {
      "attemptNumber": 2,
      "report": {
        "executiveSummary": "EU regulatory efforts center on the AI Act's transparency obligations, with differentiated requirements for open-weight models …",
        "evidence": [ "…6 segments…" ],
        "sources": { "sourceRefs": ["…5 distinct URLs…"], "totalSources": 5 },
        "researchedAt": "2026-06-28T09:01:31Z"
      },
      "evaluation": {
        "verdict": "SUFFICIENT",
        "notes": {
          "bullets": [],
          "overallRationale": "All four dimensions score 4 or above."
        },
        "score": 4,
        "evaluatedAt": "2026-06-28T09:01:47Z"
      }
    }
  ],
  "acceptedAttemptNumber": 2,
  "acceptedReport": { "…": "same as attempts[1].report" },
  "memoryDiff": {
    "changes": [
      {
        "blockId": "source-prefs",
        "changeType": "APPEND",
        "before": "Prefer peer-reviewed publications and official regulatory sources.",
        "after": "Prefer peer-reviewed publications and official regulatory sources. When regulatory topics span multiple jurisdictions, include at least one source from each relevant regulator's own publication channel.",
        "reason": "Source diversity was the near-miss on session s-9a1b2; regulator-per-jurisdiction guidance prevents recurrence."
      }
    ],
    "triggerSessionId": "s-9a1b2…",
    "triggerVerdict": "SUFFICIENT",
    "diffedAt": "2026-06-28T09:01:53Z"
  },
  "createdAt": "2026-06-28T09:00:58Z",
  "finishedAt": "2026-06-28T09:01:53Z"
}
```

### SSE event format

```
event: session-update
data: { "sessionId": "s-9a1b2…", "status": "EVALUATING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `sessionId`. The full `Session` JSON is included so a fresh client can render the row without a separate fetch.

```
event: memory-update
data: { "sessionId": "s-9a1b2…", "memoryDiff": { "changes": [...], ... } }
```

Emitted only when a `MemoryUpdated` event is projected. Available on both the default SSE stream and the `?channel=memory-updates` filtered stream.

### PromptDriftRecorded event in SSE

```
event: drift-detected
data: {
  "oldFingerprint": "63b2275b7c13e8a8",
  "newFingerprint": "a1f4c9d02b7e3f11",
  "changedBlockCount": 1,
  "detectedAt": "2026-06-28T09:02:15Z"
}
```
