# Architecture — akka-stategraph-bridge

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`GraphEndpoint` is the entry point. A submission writes a `RunSubmitted` event to `RunQueueEntity` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `GraphWorkflow` per submission. The workflow drives the graph execution loop: it first asks `PlannerAgent` to validate the graph definition, then on each node iteration it resolves the current node, runs the dispatch guardrail (for tool-annotated nodes), calls `NodeAgent`, sanitizes the output, records the updated state, asks `EdgeRouterAgent` to route to the next node, and checks cycle budgets. Every transition emits an event on `GraphRunEntity`; `GraphRunView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `RunSimulator` drips sample graph definitions every 90 s; `StuckRunMonitor` ticks every 30 s to mark `EXECUTING` runs older than 5 minutes as `STUCK`.

`SystemControlEntity` is a single-instance entity keyed by the literal `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag at the top of every loop iteration.

## Interaction sequence

The sequence diagram traces the happy path (J1). The loop in the middle is one full node execution cycle: check-halt → resolve-node → guardrail → execute-node → sanitize → record-state → route → check-cycle → decide. The diagram uses `NodeAgent` to label the execution participant; in code the workflow always calls `NodeAgent` for execution and `EdgeRouterAgent` for routing, with `PlannerAgent` called at startup and on guardrail revision requests.

## State machine

`GraphRunEntity` has six states. `PLANNING` is the initial state while the `PlannerAgent` validates the definition. `EXECUTING` is the loop state — most node events fire here without changing the top-level status. A run lands in one of four terminal states: `COMPLETED` (all nodes reached a declared terminal node), `FAILED` (planning failed, cycle limit exceeded, or revision budget exhausted), `HALTED` (operator pressed Halt or the auto-safety evaluator flagged an unsafe output), or `STUCK` (no node progress for 5 minutes).

## Entity model

`GraphRunEntity` is the system's source of truth, holding the graph plan, the current state map, the execution trace, and the final result. `SystemControlEntity` carries the operator halt flag. `RunQueueEntity` is the audit log of submitted graph definitions. `GraphRunView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeouts: `planStep` 60 s, `executeNodeStep` 120 s, `routeStep` 45 s, `completeStep` 60 s.
- Cycle budget: each `NodeDef.maxVisits` defaults to 3; `checkCycleStep` increments a per-node counter in workflow-local state.
- Guardrail revision budget: 2 revision attempts per blocked node; the third block on the same node transitions to `failStep`.
- Halt poll: synchronous read of `SystemControlEntity` at the top of every iteration; no caching.
- Idempotency: `(graphJson hash, requestedBy)` over a 10 s window deduplicates `POST /api/runs`.
- Stuck detection: `StuckRunMonitor` every 30 s; runs `EXECUTING` for > 5 minutes are marked `STUCK`.
