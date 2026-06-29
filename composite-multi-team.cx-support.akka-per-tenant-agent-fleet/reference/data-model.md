# Data model — per-tenant-agent-fleet

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CustomerBrief` | `customerId` | `String` | no | Id assigned at registration. |
| | `name` | `String` | no | Customer's display name. |
| | `email` | `String` | no | Customer's email address (sanitized at boundary). |
| | `tier` | `CrmTier` | no | Account tier at registration. |
| `CustomerProfile` | `customerId` | `String` | no | The customer this profile belongs to. |
| | `tier` | `CrmTier` | no | Account tier. |
| | `summary` | `String` | no | One-paragraph synthesis of the customer. |
| | `keyFacts` | `List<String>` | no | Three to five facts for the chat agent. |
| | `interactionHints` | `List<String>` | no | Two to three tone/topic guidance phrases. |
| `WelcomeEmail` | `recipientEmail` | `String` | no | Destination address (guardrail-vetted). |
| | `subject` | `String` | no | Email subject line. |
| | `body` | `String` | no | Email body (size-capped by guardrail). |
| | `templateId` | `String` | no | Template identifier (restrictions enforced by guardrail). |
| `WelcomeSendResult` | `customerId` | `String` | no | The customer this result belongs to. |
| | `sent` | `boolean` | no | `true` if the tool call completed; `false` if blocked. |
| | `blockReason` | `Optional<String>` | yes | Guardrail block reason when `sent=false`. |
| `ChatMessage` | `customerId` | `String` | no | The customer sending the message. |
| | `messageId` | `String` | no | Unique id for this message. |
| | `content` | `String` | no | Customer's message text (sanitized). |
| | `receivedAt` | `Instant` | no | When the endpoint received it. |
| `ChatReply` | `customerId` | `String` | no | The customer this reply is for. |
| | `messageId` | `String` | no | Matches the originating `ChatMessage`. |
| | `reply` | `String` | no | Agent's reply text (non-empty, quality-flagged if short). |
| | `repliedAt` | `Instant` | no | When the reply was generated. |
| `ChatTurn` | `messageId` | `String` | no | Unique id. |
| | `customerContent` | `String` | no | Customer's sanitized message. |
| | `agentReply` | `String` | no | Agent's reply. |
| | `turnAt` | `Instant` | no | When the turn completed. |
| `FleetEval` | `stage` | `String` | no | `welcome` or `chat`. |
| | `score` | `int` | no | Quality score 0–100 from `FleetEvaluator`. |
| | `flags` | `List<String>` | no | Issues found (may be empty). |
| | `evaluatedAt` | `Instant` | no | When the eval ran. |
| `FleetEntry` | `customerId` | `String` | no | Customer being registered. |
| | `name` | `String` | no | Customer's display name. |
| | `tier` | `CrmTier` | no | Account tier. |
| | `status` | `FleetStatus` | no | `SPAWNING` at creation. |
| | `registeredAt` | `Instant` | no | When recorded by the coordinator. |
| `FleetHealthSummary` | `customerId` | `String` | no | Dormant customer. |
| | `daysSinceLastInteraction` | `long` | no | Days since `lastInteractionAt`. |
| | `recommendation` | `String` | no | Short action phrase for the deployer. |

## Entity state — `Customer` (`CustomerRegistry`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `customerId` | `String` | no | Unique id; also the `AgentFleetWorkflow` id. |
| `name` | `String` | no | Customer display name. |
| `email` | `String` | no | Email address (sanitized). |
| `tier` | `CrmTier` | no | Account tier. |
| `status` | `CustomerStatus` | no | See enum. |
| `profile` | `Optional<CustomerProfile>` | yes | Populated on `ProfileUpdated`. |
| `profiledAt` | `Optional<Instant>` | yes | When the profile was stored. |
| `readyAt` | `Optional<Instant>` | yes | When `CustomerReadied` was emitted. |
| `registeredAt` | `Instant` | no | When `CustomerRegistered` was emitted. |

## Entity state — `AgentMemory` (`AgentMemoryEntity`)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `customerId` | `String` | no | Owning customer. |
| `profile` | `Optional<CustomerProfile>` | yes | Populated on `ProfileStored`. |
| `welcomeResult` | `Optional<WelcomeSendResult>` | yes | Populated on `WelcomeRecorded` or `WelcomeBlockRecorded`. |
| `turns` | `List<ChatTurn>` | no | All chat turns in order (may be empty). |
| `evals` | `List<FleetEval>` | no | All eval results in order (may be empty). |
| `lastInteractionAt` | `Optional<Instant>` | yes | Timestamp of the most recent chat turn; absent if no turns yet. |

## Enums

`CustomerStatus`: `REGISTERED`, `ONBOARDING`, `READY`.
`CrmTier`: `STANDARD`, `PREMIUM`, `ENTERPRISE`.
`FleetStatus`: `SPAWNING`, `ONBOARDING`, `READY`, `DORMANT`.

## Events — `CustomerRegistry`

| Event | Payload | Transition |
|---|---|---|
| `CustomerRegistered` | `customerId, name, email, tier, registeredAt` | → REGISTERED |
| `ProfileUpdated` | `profile, profiledAt` | REGISTERED/ONBOARDING → ONBOARDING (profile stored) |
| `CustomerReadied` | `readyAt` | ONBOARDING → READY |

## Events — `AgentMemoryEntity`

| Event | Payload | Transition (memory) |
|---|---|---|
| `MemoryInitialized` | `customerId` | creates empty memory |
| `ProfileStored` | `profile` | stores profile |
| `WelcomeRecorded` | `welcomeResult{sent=true}` | stores welcome result |
| `WelcomeBlockRecorded` | `welcomeResult{sent=false, blockReason}` | stores block reason |
| `ChatTurnAppended` | `turn` | appends turn, updates `lastInteractionAt` |
| `FleetEvalRecorded` | `eval` | appends eval |

## View rows

`CustomerRow` (row of `FleetStatusView`) mirrors `Customer` plus projection fields from `AgentMemoryEntity`: `welcomeSent`, `welcomeBlockReason`, `chatTurnCount`, `lastInteractionAt`, `latestEvalScore`, `profileSummary` (drops `keyFacts` and `interactionHints`). Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`ChatRow` (row of `ChatHistoryView`) carries `customerId`, `messageId`, `customerContent`, `agentReply`, `turnAt`. No heavy fields omitted — the individual turn is the row. Every field is non-nullable.

Both views expose unfiltered or keyed queries plus streaming queries for the SSE endpoints. `FleetStatusView.getAllCustomers` is unfiltered; `ChatHistoryView.getChatHistory(customerId)` filters by `customerId` (not an enum column — a foreign key).
