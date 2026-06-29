# Architecture

This file is the narrative around the four diagrams in [`../PLAN.md`](../PLAN.md). The generated system renders those diagrams on the Architecture tab; this prose is what an author reads to understand why each component exists.

## The turn-taking loop

The system has one job: run a moderated group chat. A request arrives — a topic — and the `ChatSessionWorkflow` becomes the Orchestrator. It does not contribute to the conversation; it owns the turn order. Each round it gives the Researcher one turn, then the Critic one turn, then asks the Orchestrator agent to assess the round. That assessment routes the workflow: consensus ends the loop with a summary, an escalation ends it immediately, and anything else advances the turn. Twenty turns is the hard cap; the twenty-first would conclude as `MAX_TURNS_REACHED`.

This is the moderation-turn-taking pattern applied to multi-assistant conversation. The two assistants never call each other. The moderator is the only component that reads the full transcript, and the moderator alone decides when a round is over and when the whole session is over. Keeping the turn order inside a durable workflow means a crash mid-round resumes at the same turn with the same transcript, because the turn counter and every message live on the `ChatSessionEntity`, not only in workflow memory.

See the **component graph** (PLAN.md §1) for who calls whom. Solid arrows are commands, dashed arrows are event subscriptions, dotted arrows are the two scheduled ticks.

## How a session flows

The **interaction sequence** (PLAN.md §2) traces one session from request to quality score. A request lands in the `SessionRequestQueue` — either from the App UI tab or from the `RequestSimulator` that drips a canned topic every thirty seconds. The `SessionRequestConsumer` picks up the queued event, checks the operator halt flag on `SystemControl`, and if the system is running it starts a workflow with a fresh id.

Inside the loop, each assistant turn is a workflow step that calls `forAutonomousAgent(...).runSingleTask(...)` and then `forTask(taskId).result(...)` to retrieve the typed `ChatTurn` or `OrchestratorDecision`. Because those calls hit an LLM, each step overrides the default five-second timeout to sixty seconds. Before the result is committed to the entity, the before-agent-response guardrail inspects the `ChatTurn` for the `flagged` field and a non-empty message, and the `OrchestratorDecision` for a non-empty `summary` when the verdict is `CONCLUDE`. A rejected result causes the step to fail over to the error step, ending the session as `ESCALATED`. After the Orchestrator concludes, the entity emits `SessionConcluded`, and the `SessionQualityEvaluator` consumer scores the result and writes it back.

## The session lifecycle

The **state machine** (PLAN.md §3) shows the `ChatSessionEntity` moving `CREATED → CHATTING → CONCLUDED`, with a side exit to `ESCALATED` when either the `StalledSessionMonitor` finds a session still running after three minutes or a guardrail rejection routes the workflow to the error step. The termination-reason distinction — `CONSENSUS`, `MAX_TURNS_REACHED`, or `ESCALATED` — is not in the enum; it rides on a separate `terminationReason` string, so the read model never has to index an enum column. `QualityEvaluated` is a self-transition on `CONCLUDED`: it enriches the row without changing the lifecycle stage.

## What is stored

The **entity model** (PLAN.md §4) shows the `ChatSession` aggregate with its append-only `TurnLine` history, the events it emits, and the one-to-one projection into the view row the UI reads. The `SessionRequestQueue` and `SystemControl` are the two supporting pieces of durable state: one records incoming requests, the other holds the halt flag. Everything is in one service — the assistant personas, the moderator, the transcript, and the operator switch are all Akka primitives.

## Why these primitives

- **Three AutonomousAgents, not three request/response Agents.** Each participant runs a short bounded task per turn and returns a typed result; the runtime drives the task to completion. The Researcher and Critic are symmetric participants; the Orchestrator is the neutral third.
- **A Workflow for the moderator.** Turn order is durable, restartable state. A plain loop in an endpoint would lose its place on a crash; the workflow resumes mid-session.
- **An EventSourcedEntity for the session.** Every turn is history. Replaying the events reconstructs the exact sequence of messages, which is what an audit of a concluded session needs.
- **A KeyValueEntity for the halt switch.** The flag is a single mutable value with no history worth keeping; a key-value entity is the right fit.
- **Two Consumers.** One turns queued requests into workflows; one turns concluded sessions into scored outcomes. Each reacts to events without the producer knowing it exists.
- **Two TimedActions.** One supplies load so the system is never idle; one enforces the three-minute escalation objective.
