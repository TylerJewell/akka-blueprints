# Data model — guided-intake-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SessionRequest` | `goalId` | `String` | no | ID of the operator-defined goal to execute. |
| | `submittedBy` | `String` | no | Session initiator identifier. |
| `IntakePlan` | `goalId` | `String` | no | The goal this plan was produced for. |
| | `questionSequence` | `List<QuestionDef>` | no | Ordered list of questions (up to `maxTurns`). |
| | `completionCriteria` | `List<String>` | no | Conditions that, when all met, allow `GoalMet`. |
| `QuestionDef` | `questionId` | `String` | no | Stable ID within the goal. |
| | `text` | `String` | no | Natural-language question text. |
| | `fieldName` | `String` | no | Domain field this question fills. |
| | `required` | `boolean` | no | Whether this field is mandatory for goal completion. |
| `QuestionProposal` | `questionId` | `String` | no | Which question the agent proposes to ask next. |
| | `text` | `String` | no | Proposed question text (may differ from plan default). |
| | `rationale` | `String` | no | One-sentence rationale. |
| `TurnRecord` | `turnNumber` | `int` | no | 1-based turn counter for this session. |
| | `questionId` | `String` | no | Which question was asked this turn. |
| | `questionText` | `String` | no | Text of the question as delivered to the user. |
| | `sanitizedReply` | `String` | no | User reply after PII scrubbing. |
| | `verdict` | `TurnVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / INCOMPLETE / ESCALATED. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED. |
| | `recordedAt` | `Instant` | no | When the turn was recorded. |
| `SanitizedReply` | `original` | `String` | no | Unmodified raw reply (held in memory only; never stored). |
| | `sanitized` | `String` | no | Redacted form. |
| | `redactionTags` | `List<String>` | no | Distinct tag types applied. |
| `IntakeSummary` | `goalId` | `String` | no | Goal this summary covers. |
| | `filledFields` | `Map<String,String>` | no | Field name → sanitized reply excerpt. |
| | `confidence` | `double` | no | 0.0–1.0 confidence that all required fields are substantively answered. |
| | `producedAt` | `Instant` | no | When the agent produced the summary. |
| `Conversation` (entity state) | `conversationId` | `String` | no | Unique session id. |
| | `goalId` | `String` | no | Goal being executed. |
| | `submittedBy` | `String` | no | Session initiator. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `plan` | `Optional<IntakePlan>` | yes | Populated after `ConversationPlanned`. |
| | `turns` | `List<TurnRecord>` | no | Append-only; empty before first `TurnRecorded`. |
| | `summary` | `Optional<IntakeSummary>` | yes | Populated after `ConversationCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `ConversationAbandoned`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `ConversationHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `ConversationCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal state. |
| `GoalDefinition` | `goalId` | `String` | no | Stable goal identifier. |
| | `name` | `String` | no | Human-readable name. |
| | `description` | `String` | no | One-sentence goal description. |
| | `questionSet` | `List<QuestionDef>` | no | All questions for this goal. |
| | `completionCriteria` | `List<String>` | no | Field-level conditions for goal completion. |
| | `maxTurns` | `int` | no | Maximum number of turns before escalation. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `TurnDecision` | (sealed interface) | — | — | Permits `Continue(QuestionProposal next)`, `Replan(IntakePlan revised)`, `GoalMet(IntakeSummary stub)`, `Escalate(String reason)`. |

## Enums

- `ConversationStatus` → `PLANNING`, `ELICITING`, `COMPLETED`, `ESCALATED`, `HALTED`, `ABANDONED`.
- `TurnVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `INCOMPLETE`, `ESCALATED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ConversationCreated` | `conversationId, goalId, submittedBy, createdAt` | → PLANNING |
| `ConversationPlanned` | `plan: IntakePlan` | → ELICITING |
| `QuestionAsked` | `turnNumber, questionId, questionText` | no status change |
| `QuestionBlocked` | `turnNumber, questionId, blockerReason` | no status change; appends a `TurnRecord` with verdict `BLOCKED_BY_GUARDRAIL` |
| `TurnRecorded` | `turn: TurnRecord` | no status change; appends to `turns` |
| `PlanRevised` | `plan: IntakePlan` | no status change; replaces `plan` |
| `ConversationCompleted` | `summary: IntakeSummary, finishedAt` | → COMPLETED |
| `ConversationEscalated` | `reason, finishedAt` | → ESCALATED |
| `ConversationHaltedOperator` | `haltReason, finishedAt` | → HALTED |
| `ConversationAbandoned` | `failureReason, finishedAt` | → ABANDONED |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`SessionQueue`)

| Event | Payload |
|---|---|
| `SessionSubmitted` | `conversationId, goalId, submittedBy, submittedAt` |

## Events (`GoalCatalogEntity`)

| Event | Payload |
|---|---|
| `GoalAdded` | `goal: GoalDefinition` |
| `GoalUpdated` | `goalId, goal: GoalDefinition` |
| `GoalRemoved` | `goalId` |

## View row

`ConversationRow` mirrors `Conversation` minus the full turn list — `turns` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each `TurnRecord.sanitizedReply` is capped at 240 characters. The UI fetches the full conversation by id on click via `GET /api/conversations/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
