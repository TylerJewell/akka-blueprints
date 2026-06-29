# Data model — chatbot-sim-eval

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ScenarioBrief` | `personaKey` | `String` | no | One of the four supported persona identifiers. |
| | `issueDescription` | `String` | no | Free-text description of the customer's problem. |
| | `maxTurns` | `int` | no | Turn ceiling for the dialogue. |
| | `submittedBy` | `String` | no | UI identifier of the submitter. |
| `UserTurn` | `text` | `String` | no | The customer's message text. |
| | `signaledResolution` | `boolean` | no | `true` when the persona considers the issue resolved. |
| | `turnedAt` | `Instant` | no | When SimulatedUserAgent returned the turn. |
| `AssistantTurn` | `text` | `String` | no | The assistant's reply text. |
| | `safeContentFlagged` | `boolean` | no | `true` when the guardrail matched a policy pattern. |
| | `flagDetail` | `String` | no | Matching pattern and offending substring; empty when not flagged. |
| | `turnedAt` | `Instant` | no | When ChatbotAgent returned the turn. |
| `DialogueTurn` | `turnNumber` | `int` | no | 1-indexed; monotonic across the dialogue. |
| | `userTurn` | `UserTurn` | no | The customer's side of the turn. |
| | `assistantTurn` | `Optional<AssistantTurn>` | yes | Empty on the final user turn when it signals resolution with no pending reply. |
| `DimensionFinding` | `dimension` | `String` | no | One of: `Resolution`, `Tone`, `Accuracy`, `PolicyCompliance`. |
| | `score` | `int` | no | 1–10. |
| | `observation` | `String` | no | Specific observation citing the turn number and quoting relevant text. |
| `EvalFinding` | `findings` | `List<DimensionFinding>` | no | One entry per dimension that scored below 7; empty on `PASS`. |
| | `overallSummary` | `String` | no | One-sentence summary of the outcome; required either way. |
| `EvalVerdict` | `outcome` | `EvalOutcome` | no | `PASS` or `FAIL`. |
| | `finding` | `EvalFinding` | no | See above. |
| | `overallScore` | `int` | no | 1–10; minimum of all four dimension scores. |
| | `evaluatedAt` | `Instant` | no | When EvaluatorAgent returned. |
| `Simulation` (entity state) | `simulationId` | `String` | no | Unique id. |
| | `personaKey` | `String` | no | Persona used. |
| | `issueDescription` | `String` | no | Original issue description. |
| | `maxTurns` | `int` | no | Per-simulation turn ceiling. |
| | `status` | `SimulationStatus` | no | See enum. |
| | `turns` | `List<DialogueTurn>` | no | Bounded at `maxTurns`; starts empty. |
| | `verdict` | `Optional<EvalVerdict>` | yes | Populated on `EvalVerdictRecorded`. |
| | `haltReason` | `Optional<String>` | yes | `RESOLVED` or `MAX_TURNS_REACHED`; populated on `ConversationConcluded`. |
| | `createdAt` | `Instant` | no | When `SimulationCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the simulation reached a terminal state. |

## Enums

`SimulationStatus`: `RUNNING`, `EVALUATING`, `PASSED_EVALUATION`, `FAILED_EVALUATION`.

`EvalOutcome`: `PASS`, `FAIL`.

## Events (`SimulationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `SimulationCreated` | `simulationId, personaKey, issueDescription, maxTurns, createdAt` | Workflow `startStep` | → `RUNNING` |
| `UserTurnRecorded` | `turnNumber, userTurn: UserTurn` | After `openDialogueStep` or `userReplyStep` returns | (no status change; appends to `turns[]`) |
| `AssistantTurnRecorded` | `turnNumber, assistantTurn: AssistantTurn` | After `guardrailStep` (which populates `safeContentFlagged`) | (no status change; populates `turns[n].assistantTurn`) |
| `ConversationConcluded` | `haltReason, turnCount` | `UserTurn.signaledResolution = true` OR `turnCount >= maxTurns` | → `EVALUATING` |
| `EvalVerdictRecorded` | `outcome, finding, overallScore, evaluatedAt` | After `evaluateStep` returns | `PASS` → `PASSED_EVALUATION`; `FAIL` → `FAILED_EVALUATION`; sets `finishedAt = now` |
| `EvalRecorded` | `simulationId, outcome, overallScore, anyFlagged, recordedAt` | `EvalSampler` per completed simulation | (no status change; appends to an internal `evalEvents[]` view-side projection) |

## Events (`ScenarioQueue`)

| Event | Payload |
|---|---|
| `ScenarioSubmitted` | `simulationId, personaKey, issueDescription, maxTurns, submittedBy, submittedAt` |

## View row

`SimulationRow` is structurally identical to `Simulation` — the `turns` list is bounded at `maxTurns` (default 10) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `SimulationEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllSimulations` returning the full list. Callers filter by `status` client-side.
