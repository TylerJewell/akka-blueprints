# Architecture

Narrative for the four mermaid diagrams in `PLAN.md`. The Architecture tab of the generated UI renders the same diagrams with the Lesson 24 CSS overrides (white state-box labels, `overflow:visible` edge labels) and Akka theme variables.

## Component graph

`RequestSimulator` (a TimedAction) drips canned requests into `InboundRequestQueue`, an event-sourced entity that emits `InboundRequestQueued`. `RequestConsumer` subscribes to those events and starts one `ApprovalWorkflow` per request, keyed by a fresh `actionId`. The workflow calls `ActionAgent` to plan a tool call, persists the plan to `ActionEntity`, then pauses. After human approval the workflow calls `ExecutorAgent` to run the tool and persists the result. `ActionEntity` events project into `ActionsView`, which `ActionEndpoint` queries and streams over SSE. `StuckActionMonitor` reads the view and escalates stale plans. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey: a client POSTs a request; the workflow plans an action and pauses in `awaitApprovalStep`, re-arming a 5-second resume timer while the status is `PLANNED`. A reviewer approves; the workflow observes `APPROVED`, the before-tool-call guardrail confirms the status, `ExecutorAgent` runs the tool, and the action reaches `EXECUTED`.

## State machine

`ActionEntity` moves `CREATED → PLANNED → {APPROVED → EXECUTED | REJECTED | ESCALATED}`. `REJECTED`, `ESCALATED`, and `EXECUTED` are terminal. Each transition corresponds to one `ActionEvent`.

## Entity model

`InboundRequestQueue` is a single-instance log that triggers workflows. `ActionEntity` is the per-action aggregate; its events project to the `ActionsView` read model. The view row is the same `Action` record the entity holds, with every nullable lifecycle field declared `Optional<T>` so the view materializer accepts it.
