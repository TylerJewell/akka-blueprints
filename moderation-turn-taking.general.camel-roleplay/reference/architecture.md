# Architecture

This file is the narrative around the four diagrams in [`../PLAN.md`](../PLAN.md). The generated system renders those diagrams on the Architecture tab; this prose is what an author reads to understand why each component exists.

## The turn-taking loop

The system has one job: run a moderated role-play collaboration. A request arrives — a task description, an assistant role, and a user role — and the `CollaborationWorkflow` becomes the Coordinator. It does not collaborate itself; it owns the turn order. Each round it gives the User agent one turn, then the Assistant agent one turn, then asks the Coordinator agent to judge the round. That judgement routes the workflow: a complete solution ends the loop with a summary and final answer, an impasse ends it without one, and anything else advances the round. Twelve rounds is the hard cap; the thirteenth would conclude as impasse.

This is the moderation-turn-taking pattern applied to collaborative problem-solving. The two parties never call each other. The Coordinator is the only component that talks to both, and the Coordinator alone decides when a round is over and when the whole collaboration is over. Keeping the turn order inside a durable workflow means a crash mid-round resumes at the same round with the same turn history, because the round counter and every turn live on the `CollaborationEntity`, not only in workflow memory.

See the **component graph** (PLAN.md §1) for who calls whom. Solid arrows are commands, dashed arrows are event subscriptions, dotted arrows are the two scheduled ticks.

## How a collaboration flows

The **interaction sequence** (PLAN.md §2) traces one collaboration from request to score. A task request lands in the `InboundTaskQueue` — either from the App UI tab or from the `TaskSimulator` that drips a canned task every thirty seconds. The `TaskRequestConsumer` picks up the queued event, checks the operator halt flag on `SystemControl`, and if the system is running it starts a workflow with a fresh id.

Inside the loop, each agent turn is a workflow step that calls `forAutonomousAgent(...).runSingleTask(...)` and then `forTask(taskId).result(...)` to retrieve the typed `DialogueTurn` or `CoordinatorDecision`. Because those calls hit an LLM, each step overrides the default five-second timeout to sixty seconds — leaving the default in place is the single most common cause of a turn-taking loop that stalls on LLM latency. After the Coordinator concludes, the entity emits `CollaborationConcluded`, and the `SolutionEvaluator` consumer scores the result and writes it back. The score is the last thing to appear beside the collaboration in the UI.

## The collaboration lifecycle

The **state machine** (PLAN.md §3) shows the `CollaborationEntity` moving `CREATED → COLLABORATING → CONCLUDED`, with a side exit to `ESCALATED` when the `StalledCollaborationMonitor` finds a collaboration still running after three minutes. The solved-or-impasse distinction is not in the enum — it rides on a separate `outcome` string — so the read model never has to index an enum column, which the runtime cannot auto-index. `OutcomeEvaluated` is a self-transition on `CONCLUDED`: it enriches the row without changing the lifecycle stage.

## What is stored

The **entity model** (PLAN.md §4) shows the `Collaboration` aggregate with its append-only `TurnLine` history, the events it emits, and the one-to-one projection into the view row the UI reads. The `InboundTaskQueue` and `SystemControl` are the two supporting pieces of durable state: one records incoming tasks, the other holds the halt flag. Everything is in one service — the task stream, the two agents, the coordinator, and the operator switch are all Akka primitives, so the whole system builds and runs with `/akka:build` and nothing else.

## Why these primitives

- **Three AutonomousAgents, not three request/response Agents.** Each party runs a short bounded task per turn and returns a typed result; the runtime drives the task to completion. The User and Assistant are structurally symmetric roles with distinct prompts; the Coordinator is the neutral third.
- **A Workflow for the Coordinator.** Turn order is durable, restartable state. A plain loop in an endpoint would lose its place on a crash; the workflow resumes mid-collaboration.
- **An EventSourcedEntity for the collaboration.** Every dialogue turn is history. Replaying the events reconstructs the exact exchange that led to a solution or impasse, which is what a review of the collaboration outcome needs.
- **A KeyValueEntity for the halt switch.** The flag is a single mutable value with no history worth keeping; a key-value entity is the right fit.
- **Two Consumers.** One turns queued tasks into workflows; one turns concluded collaborations into scored outcomes. Each reacts to events without the producer knowing it exists.
- **Two TimedActions.** One supplies load so the system is never idle; one enforces the three-minute escalation objective.
