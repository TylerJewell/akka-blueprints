# Architecture — with Signals & Queries

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one long-running workflow that drives one agent. `ChatEndpoint` accepts a new session and writes a `SessionOpened` event onto `ChatSessionEntity`, then starts a `ChatSessionWorkflow` instance. The workflow's `processTurnStep` calls `ChatAgent` — the single AutonomousAgent — with the accumulated conversation context plus the new prompt as `TaskDef.instructions(...)`. Before the agent is invoked, the `SignalValidator` guardrail validates the inbound signal payload; rejections fail the step without touching the agent. Once the agent returns an `AgentReply`, the workflow writes `TurnReplied` back to the entity. The workflow then waits for the next signal rather than terminating.

Two signal paths branch off the workflow: `pauseSignal` suspends it in `pausedStep`; `resumeSignal` resumes it, optionally prepending an operator corrective note. A third path, `closeSignal`, transitions the workflow to `closeStep` and terminates it. `getSessionQuery` reads the workflow's in-memory state without writing any event.

`SessionView` projects every entity event into a read-model row; `ChatEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. `SignalValidator` is a supporting class (a guardrail hook), not an LLM call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path for two turns (J1 + J2). Note two distinct control-flow moments:

1. Each turn through `processTurnStep` blocks on the LLM call — bounded by the step's 60 s timeout.
2. Between turns, the workflow waits for the next `addTurnSignal`; this waiting is not polling — it is the workflow runtime's native signal-receipt mechanism, consuming no compute resources while idle.

The operator pause path (J3) is not shown in the sequence diagram to keep it readable, but the state machine diagram captures both the `PAUSED` state and the `SessionResumed` transition.

## State machine

Seven states. The notable paths:

- Happy path: `STARTING → ACTIVE → … → CLOSING → CLOSED`. The workflow loops through `ACTIVE` on every new turn, emitting `TurnAdded` + `TurnReplied` pairs. No state change is needed between turns — `ACTIVE` is a stable steady state.
- Operator intervention: `ACTIVE → PAUSED → ACTIVE`. The session re-enters `ACTIVE` after `resumeSignal` with the corrective note prepended to the next turn.
- Two failure terminals: from `ACTIVE` or `PAUSED` via `SessionFailed` (guardrail rejection or unrecoverable agent error). The partial turn log is preserved on the entity; the UI shows the failure reason.
- There is no `APPROVED` or `RESOLVED` state. The session is advisory; callers act on the agent's replies outside the system. The blueprint stops at `CLOSED`.

## Entity model

`ChatSessionEntity` is the source of truth. It emits seven event types across the session lifecycle. `SessionView` projects every event into a row used by the UI. `ChatSessionWorkflow` both reads (`getSession`) and writes (`openSession`, `addTurn`, `recordReply`, `pauseSession`, `resumeSession`, `closeSession`, `failSession`) on the entity. The relationship between `ChatAgent` and `AgentReply` is "returns" — the agent's task result is the reply record appended to the turn log.

## Signals and queries as governance primitives

The HITL mechanism (H1) and the query endpoint are not HTTP workarounds — they are Akka Workflow primitives that carry the same transactional guarantees as any other workflow step transition. A signal write is durable: if the service restarts after a `pauseSignal` arrives but before `resumeSignal` does, the workflow wakes in `pausedStep` exactly where it left off, and the pause reason is preserved on the entity. The query (`getSessionQuery`) is read-only at the workflow level and has no entity side-effect; it does not require any event to be emitted, which means calling it repeatedly never grows the entity's event log.
