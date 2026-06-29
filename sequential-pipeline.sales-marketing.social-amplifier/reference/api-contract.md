# API contract — social-amplifier

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/amplifications` | `SubmitAmplificationRequest` | `201 { amplificationId }` | `AmplificationEndpoint` → `AmplificationEntity` |
| `GET` | `/api/amplifications` | — | `200 [ AmplificationRecord... ]` (newest-first) | `AmplificationEndpoint` ← `AmplificationView` |
| `GET` | `/api/amplifications/{id}` | — | `200 AmplificationRecord` / `404` | `AmplificationEndpoint` ← `AmplificationView` |
| `GET` | `/api/amplifications/sse` | — | `text/event-stream` | `AmplificationEndpoint` ← `AmplificationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAmplificationRequest (request body)

```json
{
  "sourceUrl": "https://akka.io/blog/akka-3-6-launch",
  "targetPlatforms": ["LINKEDIN", "X", "BLUESKY"]
}
```

`targetPlatforms` defaults to `["LINKEDIN", "X", "BLUESKY"]` when omitted.

### AmplificationRecord (response body)

```json
{
  "amplificationId": "amp-7bc4d1...",
  "sourceUrl": "https://akka.io/blog/akka-3-6-launch",
  "targetPlatforms": ["LINKEDIN", "X", "BLUESKY"],
  "parsedArticle": {
    "headline": "Akka 3.6: Native AutonomousAgent Pipeline Support",
    "sourceUrl": "https://akka.io/blog/akka-3-6-launch",
    "keyMessages": [
      {
        "messageId": "km-001",
        "text": "AutonomousAgent pipelines now support typed task handoffs across phases.",
        "platformHint": "ALL"
      },
      {
        "messageId": "km-002",
        "text": "Phase-gated guardrails enforce dependency contracts at the agent-tool boundary.",
        "platformHint": "LINKEDIN"
      }
    ],
    "parsedAt": "2026-06-28T10:00:02Z"
  },
  "draftSet": {
    "drafts": [
      {
        "draftId": "d-a1b2c3",
        "platform": "LINKEDIN",
        "text": "Akka 3.6 is here. This release delivers native support for AutonomousAgent pipelines, making it straightforward to build multi-phase AI workflows with typed task handoffs and runtime guardrails. #AkkaBuilt #AkkaSample",
        "hashtags": ["#AkkaBuilt", "#AkkaSample"],
        "characterCount": 223,
        "brandCheckAttempts": 0
      },
      {
        "draftId": "d-d4e5f6",
        "platform": "X",
        "text": "Akka 3.6 ships AutonomousAgent pipelines: typed task handoffs, phase-gated guardrails, and deterministic evals. #AkkaBuilt",
        "hashtags": ["#AkkaBuilt"],
        "characterCount": 122,
        "brandCheckAttempts": 0
      },
      {
        "draftId": "d-g7h8i9",
        "platform": "BLUESKY",
        "text": "Akka 3.6 brings AutonomousAgent pipeline support with typed task handoffs and guardrails. Build AI workflows you can audit. #AkkaBuilt",
        "hashtags": ["#AkkaBuilt"],
        "characterCount": 133,
        "brandCheckAttempts": 0
      }
    ],
    "draftedAt": "2026-06-28T10:00:08Z"
  },
  "publishedSet": {
    "receipts": [
      {
        "receiptId": "rcpt-001",
        "platform": "LINKEDIN",
        "postUrlStub": "https://stub.example.com/linkedin/rcpt-001",
        "publishedAt": "2026-06-28T10:00:15Z"
      },
      {
        "receiptId": "rcpt-002",
        "platform": "X",
        "postUrlStub": "https://stub.example.com/x/rcpt-002",
        "publishedAt": "2026-06-28T10:00:16Z"
      },
      {
        "receiptId": "rcpt-003",
        "platform": "BLUESKY",
        "postUrlStub": "https://stub.example.com/bluesky/rcpt-003",
        "publishedAt": "2026-06-28T10:00:17Z"
      }
    ],
    "publishedAt": "2026-06-28T10:00:17Z"
  },
  "brandAudit": {
    "score": 5,
    "rationale": "All checks passed: receipt coverage, no stub errors, low retry counts, all drafts within character limits.",
    "auditedAt": "2026-06-28T10:00:17Z"
  },
  "status": "PUBLISHED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:17Z",
  "brandCheckRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `brandCheckRejections` is an array — empty on the happy path, populated when the brand-policy guardrail rejected one or more drafts.

A run with brand-check rejections shows:

```json
{
  "brandCheckRejections": [
    {
      "draftId": "d-d4e5f6",
      "platform": "X",
      "rule": "character-limit",
      "reason": "brand-policy-violation: character-limit: draft for X has 312 characters, limit is 280",
      "rejectedAt": "2026-06-28T10:00:06Z"
    }
  ]
}
```

### SSE event format

```
event: amplification-update
data: { "amplificationId": "amp-7bc4d1...", "status": "DRAFTED", "draftSet": { ... }, ... }
```

One event per state transition (`CREATED`, `PARSING`, `PARSED`, `DRAFTING`, `DRAFTED`, `PUBLISHING`, `PUBLISHED`, `FAILED`) and one per `BrandCheckFailed` or `PublishToolBlocked` audit event:

```
event: brand-check-failed
data: { "amplificationId": "amp-7bc4d1...", "draftId": "d-d4e5f6", "platform": "X", "rule": "character-limit", "reason": "...", "rejectedAt": "..." }

event: publish-tool-blocked
data: { "amplificationId": "amp-7bc4d1...", "platform": "LINKEDIN", "tool": "publishToLinkedIn", "reason": "publish-gated: ...", "rejectedAt": "..." }
```

Clients reconcile by `amplificationId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitAmplificationRequest` record and the `AmplificationCreated` event to carry it.
