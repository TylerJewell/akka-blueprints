# Architecture — screenplay-writer

The system is a sequential pipeline. A thread enters once, passes through four ordered stages — sanitize, analyze, write dialogue, format — and lands as a finished screenplay on an event-sourced entity that a view streams to the UI. The four diagrams below are the source for the Architecture tab; they carry the Lesson 24 mermaid CSS overrides when rendered in `index.html`.

## Component graph

The flowchart in `PLAN.md` ("Component graph") shows every component coloured by Akka primitive. Two entry points feed the pipeline: `ThreadIntakeEndpoint` (a user submitting a thread) and `ThreadSimulator` (a timed action dripping canned threads). Both land in `InboundThreadQueue`; `ThreadIntakeConsumer` reads its events and starts one `ScreenplayWorkflow` per thread. The workflow calls the three agents in turn and writes each result to `ScreenplayEntity`, whose events project into `ScreenplayView` for the read path.

## Interaction sequence

The sequence diagram in `PLAN.md` ("Interaction sequence") traces the primary journey: submit → sanitize → analyze → write dialogue → format. The `PiiSanitizer` runs before any agent, so agent prompts never contain raw personal data. The `alt containsPii` branch shows the final guardrail outcome — `BLOCKED` on a leak, `COMPLETED` when clean.

## State machine

The `stateDiagram-v2` in `PLAN.md` ("State machine") is the lifecycle of one `ScreenplayJob`: `RECEIVED → SANITIZED → ANALYZED → DIALOGUE_WRITTEN → COMPLETED`, with `DIALOGUE_WRITTEN → BLOCKED` for the guardrail path and `→ FAILED` available from every working stage on an unrecoverable step error.

## Entity model

The `erDiagram` in `PLAN.md` ("Entity model") shows the two entities and their events. `InboundThreadQueue` emits `ThreadQueued`. `ScreenplayEntity` emits the lifecycle events and projects into `ScreenplayView`, whose row carries the id, source label, status name, and redaction count for the list and SSE surfaces.

## Why these primitives

- A `Workflow` gives the pipeline durable, resumable ordering across stages that each take an LLM call.
- Request/response `Agent`s fit single-shot transforms; each stage runs once per job and returns a typed record. No durable agent loop is needed, so none of the three is an `AutonomousAgent` (and none carries a `Tasks.java`).
- `EventSourcedEntity` + `View` give an auditable history and a streamable read model with no extra store.
- A `Consumer` + `TimedAction` model the inbound feed entirely in-process, so the sample runs out of the box.
