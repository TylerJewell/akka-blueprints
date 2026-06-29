# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A query enters through `RunEndpoint` (`POST /api/runs`) or is dripped by `QuerySimulator` every 60 seconds. Either path writes a `QuerySubmitted` event onto `QueryQueue`. `RunRequestConsumer` subscribes to those events and starts one `RunWorkflow` per submission, keyed by `runId`.

The workflow is the supervisor. It asks `ManagerAgent` to decompose the query into a `SearchPlan`, dispatches `SearchAgent` with that plan (intercepting each tool call through the URL allow-list guardrail), assembles a `RunTrace` from the emitted `PhoenixSpan` records, asks `TraceInspector` to produce a `TraceReport`, then asks `ManagerAgent` to synthesise the final answer. Every transition is written as a command to `RunEntity`, whose events project into `RunView`. The endpoint reads and streams the view; `EvalSampler` reads it to pick a run to score.

## Interaction sequence

The sequence diagram traces the primary journey: plan, search with per-call URL checking, trace assembly, trace inspection, synthesis, and persistence. The URL allow-list check appears as an inlined note on the `SearchAgent` lifeline — it runs synchronously before each HTTP fetch and can route the workflow to `BLOCKED_TOOL` without returning to the manager. The `assembleTraceStep` is a deterministic Java step with no LLM call.

## State machine

`AgentRun` moves `DISPATCHED → SEARCHING`, then to one of three terminals: `TRACED` (full pipeline complete), `PARTIAL` (search exceeded `maxPages`), or `BLOCKED_TOOL` (URL allow-list violation). `TRACED` accepts one further `RunEvalScored` self-transition when `EvalSampler` records a quality score.

## Entity model

`QueryQueue` seeds one `AgentRun` per query. A run owns at most one `SearchPlan`, one `SearchResultBundle`, one `RunTrace`, one `TraceReport`, and one `SynthesisedAnswer`. The view row mirrors the run with `Optional<T>` on every field that is null before its lifecycle transition (Lesson 6). Token totals from the `RunTrace` are promoted to the view row so the UI can display efficiency without deserialising nested spans.
