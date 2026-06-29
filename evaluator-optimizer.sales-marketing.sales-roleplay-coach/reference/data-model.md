# Data model — sales-roleplay-coach

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `DealContext` | `buyerPersona` | `String` | no | Free-text description of the buyer's role and company. |
| | `product` | `String` | no | Name of the product being pitched. |
| | `dealStage` | `DealStage` | no | Current stage of the hypothetical deal. |
| | `dealAmountUsd` | `Optional<Integer>` | yes | Indicative deal size; optional. |
| | `requestedBy` | `String` | no | UI identifier of the rep. |
| `RepTurn` | `text` | `String` | no | The rep's pitch text for this turn. |
| | `submittedAt` | `Instant` | no | When the rep submitted the turn. |
| `BuyerTurn` | `text` | `String` | no | The buyer's spoken response. |
| | `signal` | `BuyerSignal` | no | `INTERESTED`, `SKEPTICAL`, `RESISTANT`, or `READY_TO_BUY`. |
| | `respondedAt` | `Instant` | no | When the buyer simulator returned. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the turn. |
| | `reasonCode` | `String` | no | `OK` or `PROHIBITED_CONTENT`. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `CoachingNotes` | `bullets` | `List<String>` | no | 0–4 short bullets (empty on `ACCEPT`). |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `CoachVerdict` | `decision` | `CoachDecision` | no | `ACCEPT` or `REVISE`. |
| | `notes` | `CoachingNotes` | no | See above. |
| | `score` | `int` | no | 1–5 rubric. |
| | `evaluatedAt` | `Instant` | no | When the coach returned. |
| `Turn` | `turnNumber` | `int` | no | 1-indexed; monotonic across the session (includes guardrail-blocked turns). |
| | `repTurn` | `RepTurn` | no | The rep's pitch for this turn. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `buyerResponse` | `Optional<BuyerTurn>` | yes | Empty when guardrail blocked; populated otherwise. |
| | `coachVerdict` | `Optional<CoachVerdict>` | yes | Empty until coach scores; populated after `coachStep`. |
| `Session` (entity state) | `sessionId` | `String` | no | Unique id. |
| | `dealContext` | `DealContext` | no | The scenario's deal parameters. |
| | `maxTurns` | `int` | no | Per-session retry ceiling (default 5). |
| | `status` | `SessionStatus` | no | See enum. |
| | `turns` | `List<Turn>` | no | Bounded at `maxTurns`; starts empty. |
| | `passedAtTurnNumber` | `Optional<Integer>` | yes | Populated on `SessionPassed`. |
| | `passedTurnText` | `Optional<String>` | yes | Populated on `SessionPassed`. |
| | `exhaustionReason` | `Optional<String>` | yes | Populated on `SessionExhausted`. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the session reached a terminal state. |

## Enums

`SessionStatus`: `PITCHING`, `EVALUATING`, `PASSED`, `EXHAUSTED`.

`CoachDecision`: `ACCEPT`, `REVISE`.

`DealStage`: `DISCOVERY`, `DEMO`, `PROPOSAL`, `NEGOTIATION`, `CLOSING`.

`BuyerSignal`: `INTERESTED`, `SKEPTICAL`, `RESISTANT`, `READY_TO_BUY`.

## Events (`SessionEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `SessionCreated` | `sessionId, dealContext, maxTurns, createdAt` | Workflow `startStep` | → `PITCHING` |
| `BuyerResponseRecorded` | `turnNumber, buyerTurn: BuyerTurn` | After `openBuyerStep` or `buyerResponseStep` returns | (no status change for opener; subsequent responses recorded in `turns[n].buyerResponse`) |
| `RepTurnRecorded` | `turnNumber, repTurn: RepTurn` | After `repTurnStep` delivers a turn | (no status change; appends to `turns[]`) |
| `TurnGuardrailVerdictRecorded` | `turnNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `EVALUATING`; `passed=false` → `PITCHING` (re-pitch wait) |
| `TurnCoachVerdicted` | `turnNumber, coachVerdict: CoachVerdict` | After `coachStep` returns | (no status change; populates `turns[n].coachVerdict`) |
| `SessionPassed` | `turnNumber, passedTurnText` | `CoachVerdict.decision = ACCEPT` | → `PASSED`, `finishedAt = now` |
| `SessionExhausted` | `bestTurnNumber, bestTurnText, exhaustionReason` | `turns.size() == maxTurns` AND last coach verdict is `REVISE`; OR `defaultStepRecovery` failover | → `EXHAUSTED`, `finishedAt = now` |
| `EvalRecorded` | `turnNumber, decision, score, guardrailFlagged, recordedAt` | `EvalSampler` per coached turn; workflow on terminal transition | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`ScenarioQueue`)

| Event | Payload |
|---|---|
| `ScenarioSubmitted` | `sessionId, buyerPersona, product, dealStage, dealAmountUsd, requestedBy, submittedAt` |

## View row

`SessionRow` is structurally identical to `Session` — the `turns` list is bounded at `maxTurns` (default 5) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `SessionEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllSessions` returning the full list. Callers filter by `status` client-side.
