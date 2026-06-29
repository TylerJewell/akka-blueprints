# API contract — inbox-classifier

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/messages` | — | `200 [ Message... ]` (newest-first); supports optional `?label=…&status=…` filtered client-side | `InboxEndpoint` ← `MessageView` |
| `GET` | `/api/messages/{id}` | — | `200 Message` / `404` | `InboxEndpoint` ← `MessageView` |
| `POST` | `/api/messages` | `{ "senderId": String, "subject": String, "body": String, "channel": String }` | `201 { "messageId": String }` | `InboxEndpoint` → `InboxQueue` |
| `POST` | `/api/messages/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Message` (status now `ACTIONED`) / `409` if not `BLOCKED` | `InboxEndpoint` → `MessageEntity` |
| `GET` | `/api/messages/sse` | — | `text/event-stream` | `InboxEndpoint` ← `MessageView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundMessage (request body for `POST /api/messages`, minus `messageId` and `receivedAt`)

```json
{
  "senderId": "sender-42",
  "subject": "ALERT: payment processing failure — immediate attention required",
  "body": "Our payment gateway returned 500 errors for the last 15 minutes. Transactions are failing. Contact: ops@example.com.",
  "channel": "simulated"
}
```

### Message

```json
{
  "messageId": "msg-7f3a1b9c",
  "incoming": {
    "messageId": "msg-7f3a1b9c",
    "senderId": "sender-42",
    "subject": "ALERT: payment processing failure — immediate attention required",
    "body": "...raw with PII...",
    "channel": "simulated",
    "receivedAt": "2026-06-28T09:10:00Z"
  },
  "sanitized": {
    "redactedSubject": "ALERT: payment processing failure — immediate attention required",
    "redactedBody": "Our payment gateway returned 500 errors for the last 15 minutes. Transactions are failing. Contact: [REDACTED-EMAIL].",
    "piiCategoriesFound": ["email"]
  },
  "classification": {
    "label": "URGENT",
    "confidence": "high",
    "reason": "Production payment failure with explicit urgency signal."
  },
  "routing": {
    "action": "FLAG_URGENT",
    "targetFolder": null,
    "forwardAddress": null,
    "reason": "Production failure requires immediate attention.",
    "decidedAt": "2026-06-28T09:10:08Z"
  },
  "guardrail": null,
  "classificationScore": {
    "score": 5,
    "rationale": "Label and confidence correct; reason names the urgency signal directly.",
    "scoredAt": "2026-06-28T09:10:12Z"
  },
  "blockReason": null,
  "status": "ACTIONED",
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": "2026-06-28T09:10:09Z"
}
```

A blocked message has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated message has `blockReason` populated, `status = "ESCALATED"`.

### ClassificationDecision (returned by `ClassifierAgent`, embedded in `Message.classification`)

```json
{ "label": "URGENT" | "IMPORTANT" | "INFO" | "SPAM",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### RoutingDecision (returned by `RoutingAgent`, embedded in `Message.routing`)

```json
{ "action": "FLAG_URGENT" | "MOVE_TO_FOLDER" | "MARK_READ" | "MOVE_TO_SPAM" | "FORWARD_TO_HUMAN" | "DELETE",
  "targetFolder": "Projects" | "Admin" | "Newsletters" | "Archive" | null,
  "forwardAddress": "reviewer@example.com" | null,
  "reason": "one short sentence",
  "decidedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `ActionGuardrail`, embedded in `Message.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["unwarranted-delete", "misclassification-contradicts-action", …],
  "rubricVersion": "v1" }
```

### ClassificationScore (returned by `ClassificationJudge`, embedded in `Message.classificationScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: message-update
data: { "messageId": "msg-…", "status": "CLASSIFIED", … full Message JSON … }
```

One event per state transition on the `MessageView`. Clients reconcile by `messageId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/messages`).
- `404` — unknown `messageId`.
- `409` — `unblock` requested on a message not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
