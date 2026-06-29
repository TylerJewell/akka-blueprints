# Data model — discord-router

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncomingMessage` | `messageId` | `String` | no | Server-assigned UUID. |
| | `guildId` | `String` | no | Discord guild (server) identifier. |
| | `channelName` | `String` | no | `"general"` / `"help"` / `"tech-support"` / `"announcements"`. |
| | `authorHandle` | `String` | no | Raw Discord handle — PII; not passed to agents. |
| | `content` | `String` | no | Raw message content. |
| | `receivedAt` | `Instant` | no | When the inbound channel delivered it. |
| `SanitizedMessage` | `redactedContent` | `String` | no | PII redacted. |
| | `channelName` | `String` | no | Retained; not PII. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","discord-user-id","author-handle"]`. |
| `RoutingDecision` | `category` | `MessageCategory` | no | `COMMUNITY` / `TECHNICAL` / `UNCLEAR`. |
| | `confidence` | `String` | no | `"high"` / `"medium"` / `"low"`. |
| | `reason` | `String` | no | One short sentence. |
| `BotReply` | `replyBody` | `String` | no | 2–3 sentence reply. |
| | `action` | `ReplyAction` | no | `ANSWERED` / `DOCS_LINKED` / `FOLLOW_UP_REQUESTED` / `ACKNOWLEDGED` / `ESCALATED`. |
| | `specialistTag` | `String` | no | `"community"` / `"technical"`. |
| | `repliedAt` | `Instant` | no | When the specialist finished. |
| `GuardrailVerdict` | `allowed` | `boolean` | no | `true` to publish, `false` to block. |
| | `violations` | `List<String>` | no | Empty when `allowed=true`. Otherwise rubric-token list. |
| | `rubricVersion` | `String` | no | `"v1"`. |
| `RoutingScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One short sentence. |
| | `scoredAt` | `Instant` | no | When the judge finished. |
| `Message` (entity state) | `messageId` | `String` | no | — |
| | `incoming` | `IncomingMessage` | no | Captured once at create. |
| | `sanitized` | `Optional<SanitizedMessage>` | yes | Populated after `MessageSanitized`. |
| | `routing` | `Optional<RoutingDecision>` | yes | Populated after `MessageClassified`. |
| | `reply` | `Optional<BotReply>` | yes | Populated after `ReplyDrafted`. |
| | `guardrail` | `Optional<GuardrailVerdict>` | yes | Populated after `GuardrailVerdictAttached`. |
| | `routingScore` | `Optional<RoutingScore>` | yes | Populated after `RoutingScored`. |
| | `escalationReason` | `Optional<String>` | yes | Populated after `MessageEscalated`. |
| | `status` | `MessageStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MessageRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

Every nullable lifecycle field is `Optional<T>` per Lesson 6 — the view materializer rejects raw nullable types on a row record. The `Message` record is reused as the `MessageView` row type.

## Enums

`MessageCategory`: `COMMUNITY`, `TECHNICAL`, `UNCLEAR`.

`ReplyAction`: `ANSWERED`, `DOCS_LINKED`, `FOLLOW_UP_REQUESTED`, `ACKNOWLEDGED`, `ESCALATED`.

`MessageStatus`: `RECEIVED`, `SANITIZED`, `CLASSIFIED`, `ROUTED_COMMUNITY`, `ROUTED_TECHNICAL`, `REPLY_DRAFTED`, `BLOCKED`, `PUBLISHED`, `ESCALATED`.

## Events (`MessageEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `MessageRegistered` | `incoming` | `PiiSanitizer.registerIncoming` | → `RECEIVED` |
| `MessageSanitized` | `sanitized` | `PiiSanitizer.attachSanitized` | → `SANITIZED` |
| `MessageClassified` | `routing` | `DiscordWorkflow.classifyStep` | → `CLASSIFIED` |
| `MessageRouted` | `category` | `DiscordWorkflow.routeStep` | → `ROUTED_COMMUNITY` or `ROUTED_TECHNICAL` |
| `ReplyDrafted` | `reply` | `DiscordWorkflow.communityStep` / `technicalStep` | → `REPLY_DRAFTED` |
| `GuardrailVerdictAttached` | `guardrail` | `DiscordWorkflow.guardrailStep` | (no status change) |
| `ReplyPublished` | — | `DiscordWorkflow.publishStep` (guardrail allowed) | → `PUBLISHED` (terminal) |
| `ReplyBlocked` | `violations` | `DiscordWorkflow.publishStep` (guardrail denied) | → `BLOCKED` (terminal until unblock) |
| `MessageEscalated` | `escalationReason` | `DiscordWorkflow.escalateStep` (UNCLEAR or unrecoverable error) | → `ESCALATED` (terminal) |
| `RoutingScored` | `score, rationale, scoredAt` | `RoutingEvalScorer` Consumer | (no status change; populates `routingScore`) |

A `BLOCKED → PUBLISHED` transition is emitted via the `unblock` command, which writes both `GuardrailVerdictAttached` (`allowed=true` with an override marker in `rubricVersion`) and `ReplyPublished` in the same command.

## Events (`MessageQueue`)

| Event | Payload |
|---|---|
| `InboundMessageReceived` | `incoming` (the raw, pre-sanitization payload — used as the audit log) |

## View row

`MessageRow` is `Message` verbatim. Every nullable lifecycle field is `Optional<T>` so the view materializer accepts the row record without error. The view's single query is `SELECT * AS messages FROM message_view` with no `WHERE category` or `WHERE status` filter — callers (`DiscordEndpoint`) apply those client-side.
