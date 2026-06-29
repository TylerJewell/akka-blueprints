# API contract — reply-classifier

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/replies` | `SubmitReplyRequest` | `201 { replyId }` | `ReplyEndpoint` → `ReplyEntity` |
| `GET` | `/api/replies` | — | `200 [ Reply... ]` (newest-first) | `ReplyEndpoint` ← `ReplyView` |
| `GET` | `/api/replies/{id}` | — | `200 Reply` / `404` | `ReplyEndpoint` ← `ReplyView` |
| `GET` | `/api/replies/sse` | — | `text/event-stream` | `ReplyEndpoint` ← `ReplyView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitReplyRequest (request body)

```json
{
  "dealId": "deal-00421",
  "sender": "alice.chen@prospect.example",
  "subject": "Re: Governance tooling for your AI stack",
  "rawReplyText": "Hi, thanks for reaching out. We've actually been looking at this space for Q3. Happy to set up a 30-minute call to learn more — does next Thursday work?"
}
```

### Reply (response body)

```json
{
  "replyId": "rpl-7f3c...",
  "submission": {
    "replyId": "rpl-7f3c...",
    "dealId": "deal-00421",
    "sender": "alice.chen@prospect.example",
    "subject": "Re: Governance tooling for your AI stack",
    "rawReplyText": "(full text preserved for audit)",
    "receivedAt": "2026-06-28T10:00:00Z"
  },
  "classification": {
    "intent": "INTERESTED",
    "confidenceScore": 92,
    "rationale": "Prospect explicitly offers to schedule a call, signalling readiness to engage further.",
    "crmAction": {
      "type": "UPDATE_STAGE",
      "newStage": "QUALIFIED",
      "skipReason": null
    },
    "classifiedAt": "2026-06-28T10:00:18Z"
  },
  "crmResult": {
    "success": true,
    "previousStage": "PROSPECTING",
    "newStage": "QUALIFIED",
    "guardrailRejectionReason": null,
    "updatedAt": "2026-06-28T10:00:20Z"
  },
  "status": "CRM_UPDATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:20Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: reply-update
data: { "replyId": "rpl-7f3c...", "status": "CRM_UPDATED", "classification": { ... }, "crmResult": { ... } }
```

One event per state transition (`RECEIVED`, `CLASSIFYING`, `CLASSIFIED`, `CRM_UPDATED`, `CRM_SKIPPED`, `FAILED`). Clients reconcile by `replyId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `sender` from the authenticated email identity rather than the request body. Pipedrive API token is supplied via `${?PIPEDRIVE_API_TOKEN}` in `application.conf`; `PipedriveClient` reads it at startup and fails fast if the token is absent and the simulator mode is off.
