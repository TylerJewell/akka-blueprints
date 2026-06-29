# Data model — service-agent-omnichannel

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `InboundMessage` | `caseId` | `String` | no | UUID minted by `CaseEndpoint`. |
| | `channel` | `InboundChannel` | no | Enum: the channel the message arrived on. |
| | `customerId` | `String` | no | User-supplied customer identifier. |
| | `rawMessage` | `String` | no | Pre-sanitization message body. Audit-only. |
| | `scenario` | `String` | no | User-supplied label (e.g., "billing-dispute"). |
| | `receivedAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedMessage` | `redactedMessage` | `String` | no | PII redacted; this is what the agent sees. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","account-number","postal-address"]`. |
| `CaseContext` | `caseId` | `String` | no | Matches the owning `CaseEntity`. |
| | `channel` | `InboundChannel` | no | — |
| | `category` | `CaseCategory` | no | Assigned by `CaseTriage`. |
| | `scenario` | `String` | no | Passed through from `InboundMessage`. |
| `AgentReply` | `resolutionIntent` | `ResolutionIntent` | no | Enum: RESOLVED, NEEDS_FOLLOW_UP, or ESCALATE. |
| | `replyText` | `String` | no | The message to send to the customer. |
| | `channelFormat` | `String` | no | One of `sms-short`, `voice-ssml`, `chat-markdown`. |
| | `crmWritesApplied` | `List<String>` | no | Describes each CRM write the agent applied. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `EscalationScorer` finished. |
| `Case` (entity state) | `caseId` | `String` | no | — |
| | `message` | `Optional<InboundMessage>` | yes | Populated after `MessageReceived`. |
| | `sanitized` | `Optional<SanitizedMessage>` | yes | Populated after `MessageSanitized`. |
| | `category` | `Optional<CaseCategory>` | yes | Populated after `CaseTriaged`. |
| | `reply` | `Optional<AgentReply>` | yes | Populated after `ReplySent`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `CaseStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Case` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`InboundChannel`: `WHATSAPP`, `VOICE`, `WEB`, `FACEBOOK`.
`CaseCategory`: `BILLING`, `TECHNICAL`, `RETURNS`, `ACCOUNT`, `UNKNOWN`.
`ResolutionIntent`: `RESOLVED`, `NEEDS_FOLLOW_UP`, `ESCALATE`.
`CaseStatus`: `RECEIVED`, `SANITIZED`, `TRIAGING`, `HANDLING`, `REPLIED`, `ESCALATED`, `RESOLVED`, `FAILED`.

## Events (`CaseEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MessageReceived` | `message` | → RECEIVED |
| `MessageSanitized` | `sanitized` | → SANITIZED |
| `CaseTriaged` | `category` | → TRIAGING |
| `HandlingStarted` | — | → HANDLING |
| `ReplySent` | `reply` | → REPLIED |
| `CaseEscalated` | `reason: String` | → ESCALATED (terminal) |
| `CaseResolved` | — | → RESOLVED (terminal happy) |
| `EvaluationScored` | `eval` | updates eval field (no status change) |
| `CaseFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Case.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`CaseRow` mirrors `Case` minus `message.rawMessage` (the audit log keeps that). The UI fetches the raw message on demand via `GET /api/cases/{id}` and reads `message.rawMessage` from the JSON.

The view declares ONE query: `getAllCases: SELECT * AS cases FROM case_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`CaseTasks.java`)

```java
public final class CaseTasks {
  public static final Task<AgentReply> HANDLE_CASE = Task
      .name("Handle support case")
      .description("Read the attached customer message and produce an AgentReply with a resolution intent and channel-formatted reply text")
      .resultConformsTo(AgentReply.class);

  private CaseTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
