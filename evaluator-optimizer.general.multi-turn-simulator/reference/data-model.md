# Data model — multi-turn-simulator

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Scenario` | `scenarioId` | `String` | no | Unique id assigned at submission time. |
| | `persona` | `String` | no | User character label (e.g., "curious non-expert"). |
| | `goal` | `String` | no | What the simulated user is trying to achieve. |
| | `submittedBy` | `String` | no | UI identifier of the submitter; defaults to "anonymous". |
| `ActorUtterance` | `text` | `String` | no | The generated user message; may end with `[GOAL_COMPLETE]`. |
| | `turnNumber` | `int` | no | 1-indexed; monotonic across the session. |
| | `utteredAt` | `Instant` | no | When the ActorAgent returned the utterance. |
| `TargetResponse` | `text` | `String` | no | The target agent's reply. |
| | `turnNumber` | `int` | no | Matches the utterance's turn number. |
| | `respondedAt` | `Instant` | no | When the proxy returned the response. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the response. |
| | `reasonCode` | `String` | no | `OK` or `POLICY_VIOLATION`. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed = true`. |
| `TurnScore` | `coherence` | `int` | no | 1–5 dimension score. |
| | `policyAdherence` | `int` | no | 1–5 dimension score. |
| | `personaConsistency` | `int` | no | 1–5 dimension score. |
| | `bias` | `int` | no | 1–5 dimension score. |
| `TurnVerdict` | `outcome` | `TurnOutcome` | no | `PASS`, `REVISE`, or `POLICY_VIOLATION`. |
| | `scores` | `TurnScore` | no | Four dimension scores. |
| | `overallScore` | `int` | no | Minimum of the four dimension scores. |
| | `driftFlagged` | `boolean` | no | `true` if `overallScore < 3` or response contradicts a prior turn. |
| | `rationale` | `String` | no | One sentence; required either way. |
| | `evaluatedAt` | `Instant` | no | When the EvaluatorAgent returned. |
| `TurnRecord` | `turnNumber` | `int` | no | 1-indexed; monotonic across the session. |
| | `utterance` | `ActorUtterance` | no | The actor's message for this turn. |
| | `response` | `TargetResponse` | no | The target agent's reply. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `verdict` | `Optional<TurnVerdict>` | yes | Empty when the guardrail blocked and synthesized the verdict outside the evaluator call; populated otherwise. |
| `SessionVerdict` | `outcome` | `SessionOutcome` | no | `CLEAN`, `FLAGGED`, or `HALTED`. |
| | `sessionScore` | `int` | no | Floor of mean `overallScore` across all turns. |
| | `driftFlagged` | `boolean` | no | `true` if any turn was flagged. |
| | `summaryRationale` | `String` | no | One sentence describing overall quality. |
| | `closedAt` | `Instant` | no | When the session reached a terminal state. |
| `Session` (entity state) | `sessionId` | `String` | no | Unique id. |
| | `persona` | `String` | no | User persona label. |
| | `goal` | `String` | no | Session goal. |
| | `maxTurns` | `int` | no | Turn ceiling (default 10). |
| | `status` | `SessionStatus` | no | See enum. |
| | `turns` | `List<TurnRecord>` | no | Bounded at `maxTurns`; starts empty. |
| | `sessionVerdict` | `Optional<SessionVerdict>` | yes | Populated on `SessionClosed`. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal state. |

## Enums

`SessionStatus`: `RUNNING`, `COMPLETED`, `FLAGGED`, `HALTED`.

`TurnOutcome`: `PASS`, `REVISE`, `POLICY_VIOLATION`.

`SessionOutcome`: `CLEAN`, `FLAGGED`, `HALTED`.

## Events (`SessionEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `SessionCreated` | `sessionId, persona, goal, maxTurns, createdAt` | Workflow `startStep` | → `RUNNING` |
| `TurnUtteranceRecorded` | `turnNumber, utterance: ActorUtterance` | After `actorStep` returns | (no status change; appends a partial `TurnRecord` to `turns[]`) |
| `TurnResponseRecorded` | `turnNumber, response: TargetResponse` | After `targetStep` returns | (no status change; populates `turns[n].response`) |
| `TurnGuardrailVerdictRecorded` | `turnNumber, verdict: GuardrailVerdict` | After `guardrailStep` | (no status change; populates `turns[n].guardrail`) |
| `TurnVerdictRecorded` | `turnNumber, verdict: TurnVerdict` | After `scoreStep` returns or synthetic verdict synthesized | (no status change; populates `turns[n].verdict`) |
| `SessionClosed` | `sessionVerdict: SessionVerdict, finishedAt` | Goal-complete signal detected, `maxTurns` reached, or `defaultStepRecovery` failover | → `COMPLETED`, `FLAGGED`, or `HALTED` based on `SessionVerdict.outcome`; `finishedAt = now` |
| `DriftEvalRecorded` | `turnNumber, outcome, overallScore, driftFlagged, recordedAt` | `DriftSampler` per scored turn; workflow on `SessionClosed` | (no status change; drives per-turn quality stream) |

## Events (`ScenarioQueue`)

| Event | Payload |
|---|---|
| `ScenarioSubmitted` | `sessionId, persona, goal, maxTurns, submittedBy, submittedAt` |

## View row

`SessionRow` is structurally identical to `Session` — the `turns` list is bounded at `maxTurns` (default 10) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `SessionEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllSessions` returning the full list. Callers filter by `status` client-side.
