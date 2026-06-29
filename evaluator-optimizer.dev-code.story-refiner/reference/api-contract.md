# API contract — story-refiner

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/stories` | `{ "featureDescription": String, "targetTeam"?: String, "storyTypeHint"?: String, "requestedBy"?: String }` | `202 { "storyId": String }` | `StoryEndpoint` → `RequestQueue` |
| `GET` | `/api/stories` | — | `200 [ Story... ]` (optional `?status=DRAFTING\|REVIEWING\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllStories`) | `StoryEndpoint` ← `StoriesView` |
| `GET` | `/api/stories/{id}` | — | `200 Story` / `404` | `StoryEndpoint` ← `StoriesView` |
| `GET` | `/api/stories/sse` | — | `text/event-stream` (one event per story change) | `StoryEndpoint` ← `StoriesView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/stories`

- Missing `targetTeam` → `"unspecified"` (from `story-refiner.refinement.default-team`).
- Missing `requestedBy` → `"anonymous"`.
- `featureDescription` must be between 10 and 2000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(featureDescription, requestedBy)`; the second submission returns the first `storyId` (200) instead of starting a new workflow.

## JSON shapes

### Story

```json
{
  "storyId": "s-3b9f1…",
  "featureDescription": "Allow platform engineers to rotate service account keys without downtime.",
  "targetTeam": "platform",
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "draft": {
        "role": "a platform engineer managing service account credentials",
        "goal": "rotate a service account key without interrupting active connections",
        "benefit": "so that security key rotation can happen on any schedule without a maintenance window",
        "acceptanceCriteria": [
          "Given an active key, when I initiate rotation, then the old key remains valid during a configurable grace period.",
          "Given the new key is distributed, when the grace period expires, then the old key is revoked automatically.",
          "Given rotation is in progress, when I check status, then I see both keys with expiry timestamps."
        ],
        "fieldCount": 4,
        "draftedAt": "2026-06-28T10:01:02Z"
      },
      "review": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "benefit: 'any schedule' is not quantifiable — add a measurable rotation SLA.",
            "acceptanceCriteria[1]: grace period duration is not constrained — specify a default or range."
          ],
          "overallRationale": "Benefit value and AC testability fall below threshold."
        },
        "score": 3,
        "reviewedAt": "2026-06-28T10:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "draft": {
        "role": "a platform engineer managing service account credentials",
        "goal": "rotate a service account key without interrupting active connections",
        "benefit": "so that key rotation completes within 5 minutes with zero connection drops, enabling continuous compliance without manual coordination",
        "acceptanceCriteria": [
          "Given an active key, when I initiate rotation, then the old key remains valid for a grace period between 1 and 60 minutes (configurable).",
          "Given the grace period expires and the new key is distributed, when I check key status, then the old key is revoked and no active connections report errors.",
          "Given rotation is in progress, when I check status, then I see both keys with their individual expiry timestamps."
        ],
        "fieldCount": 4,
        "draftedAt": "2026-06-28T10:01:24Z"
      },
      "review": {
        "verdict": "APPROVE",
        "notes": {
          "bullets": [],
          "overallRationale": "All five dimensions score 4 or above; benefit is measurable and ACs are independently testable."
        },
        "score": 5,
        "reviewedAt": "2026-06-28T10:01:36Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedDraft": {
    "role": "a platform engineer managing service account credentials",
    "goal": "rotate a service account key without interrupting active connections",
    "benefit": "so that key rotation completes within 5 minutes with zero connection drops…",
    "acceptanceCriteria": ["…"],
    "fieldCount": 4,
    "draftedAt": "2026-06-28T10:01:24Z"
  },
  "rejectionReason": null,
  "createdAt": "2026-06-28T10:00:59Z",
  "finishedAt": "2026-06-28T10:01:37Z"
}
```

### SSE event format

```
event: story-update
data: { "storyId": "s-3b9f1…", "status": "REVIEWING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `storyId`. The full `Story` JSON is included so a fresh client can render the row without a separate fetch.
