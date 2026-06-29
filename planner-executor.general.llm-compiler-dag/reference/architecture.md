# Architecture — llm-compiler-dag

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`JobEndpoint` is the entry point. A submission writes a `QuerySubmitted` event to `RequestQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `TaskFetchingUnit` workflow per submission. The workflow first calls `PlannerAgent` to compile the query into a `CompilationPlan` — a DAG of `ToolCall` nodes with explicit dependency edges. It then enters a frontier-evaluation loop: compute the set of nodes whose dependencies are all resolved, vet them through `DispatchGuardrail`, and dispatch the approved nodes in parallel. Every result is scrubbed by `SecretScrubber`, then recorded on `JobEntity`. The loop repeats until the DAG is exhausted. `JoinerAgent` then synthesizes the complete `ResultSet` into a `QueryAnswer`. `JobView` projects `JobEntity` events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `RequestSimulator` drips sample queries for demo purposes; `StaleJobMonitor` ticks every 30 s to mark long-running `RUNNING` jobs as `STALE`.

`SystemControlEntity` is a single-instance entity keyed by the literal string `"global"`. Operators flip its halt flag from the UI; the workflow polls the flag before every new frontier dispatch.

## Interaction sequence

The sequence diagram traces the happy path (J1). It shows a query that produces three `ToolCall` nodes: two independent siblings (`c1` and `c2`) that dispatch in parallel in the first batch, and one dependent node (`c3`) that dispatches in the second batch after both are resolved. The parallel dispatch fan-out is the defining characteristic of this pattern — the diagram makes the concurrent branches explicit. After the DAG is exhausted, the Joiner synthesizes the answer in one call.

## State machine

`JobEntity` has six states. `COMPILING` is the initial state — the system waits for `PlannerAgent` to return the `CompilationPlan`. `RUNNING` is the loop state — most events fire here as parallel batches resolve without changing the job status. A job lands in one of four terminal states: `COMPLETED` (all nodes resolved, Joiner answered), `FAILED` (error budget exhausted or all frontier nodes blocked), `HALTED` (operator pressed Halt before a new frontier was dispatched), or `STALE` (no frontier progress for 5 minutes).

## Entity model

`JobEntity` is the system's source of truth; every transition writes one of ten event types. `SystemControlEntity` carries the operator halt flag. `RequestQueue` is the audit log of submissions. `JobView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency and timeouts

- Per-step timeouts: `compileStep` 60 s, `guardStep` 30 s, `parallelDispatchStep` 120 s (covers all concurrent branches), `joinStep` 60 s.
- **Parallel dispatch:** all approved frontier nodes are dispatched simultaneously as concurrent workflow branches inside `parallelDispatchStep`. The step does not advance until the last branch completes.
- **Frontier rule:** a node enters the frontier when every `callId` in its `dependsOn` list is present in `resolvedIds`. `SKIPPED` nodes (rejected by the guardrail) are also added to `resolvedIds` so downstream nodes that do not strictly require the skipped output can still proceed.
- **Error budget:** three `ToolCallFailed` events on the same job → `failStep`.
- **Stale detection:** `StaleJobMonitor` every 30 s; jobs `RUNNING` for > 5 minutes are marked `STALE`. The workflow's `frontierStep` checks the job status and exits on `STALE`.
- **Halt poll:** synchronous read of `SystemControlEntity` at the top of every loop iteration, after the previous batch has fully settled; no caching.
