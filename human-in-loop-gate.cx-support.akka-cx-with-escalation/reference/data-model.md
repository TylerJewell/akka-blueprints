# Data model

Every record, event, enum, and view row the generated system defines.

## `Conversation` (ConversationEntity state + ConversationsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Conversation id; equals the workflow id and entity id |
| `customerMessage` | `Optional<String>` | yes | The PII-sanitized inbound customer message |
| `status` | `ConversationStatus` | no | Lifecycle status |
| `openedAt` | `Optional<Instant>` | yes | When the conversation was opened |
| `agentReply` | `Optional<String>` | yes | Text of the reply SupportAgent produced |
| `repliedAt` | `Optional<Instant>` | yes | When the reply was recorded |
| `escalationReason` | `Optional<String>` | yes | Reason supplied when escalation was requested |
| `escalationRequestedAt` | `Optional<Instant>` | yes | When escalation was requested |
| `acceptedBy` | `Optional<String>` | yes | Identity of the human agent who accepted |
| `acceptedAt` | `Optional<Instant>` | yes | When the human agent accepted |
| `handoffSummary` | `Optional<String>` | yes | EscalationAgent-produced briefing for the human agent |
| `handoffPriority` | `Optional<String>` | yes | `low`, `medium`, `high`, or `urgent` |
| `resolvedAt` | `Optional<Instant>` | yes | When the conversation was resolved directly |
| `resolveNote` | `Optional<String>` | yes | Note recorded on direct resolution |

Every nullable lifecycle field is `Optional<T>` because `Conversation` is the view row type (Lesson 6). `emptyState()` returns `Conversation.initial("")` with no `commandContext()` reference (Lesson 3).

## `ConversationStatus` enum

`ACTIVE`, `ESCALATING`, `ESCALATED`, `RESOLVED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `ConversationOpened` | `recordReply` after SupportWorkflow starts and SupportAgent returns | sets `openedAt`, `customerMessage`, `agentReply`, `repliedAt`, status → `ACTIVE` |
| `ReplyRecorded` | subsequent `recordReply` calls within the same conversation | updates `agentReply`, `repliedAt` |
| `EscalationRequested` | `requestEscalation` command (customer or agent-action via API) | sets `escalationReason`, `escalationRequestedAt`, status → `ESCALATING` |
| `EscalationAccepted` | `acceptEscalation` command (human agent via API) | sets `acceptedBy`, `acceptedAt` (status stays `ESCALATING` until handoff recorded) |
| `HandoffRecorded` | `recordHandoff` after EscalationAgent returns | sets `handoffSummary`, `handoffPriority`, status → `ESCALATED` |
| `ConversationResolved` | `resolve` command (agent action or human via API) | sets `resolvedAt`, `resolveNote`, status → `RESOLVED` |

## Domain records

- `AgentReply(String message, String action)` — SupportAgent result; `action` is `"resolve"` or `"escalate"`.
- `EscalationRequest(String reason)` — escalate command payload.
- `HandoffSummary(String summary, String priority)` — EscalationAgent result.

## View row type

`ConversationsView` rows are `Conversation`. One query: `getAllConversations` → `SELECT * AS conversations FROM conversations_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (SupportTasks.java)

- `REPLY` — `Task.name(...).resultConformsTo(AgentReply.class)`.
- `HANDOFF` — `Task.name(...).resultConformsTo(HandoffSummary.class)`.
