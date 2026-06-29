# Architecture

The system realises the delegation-supervisor-workers pattern: a workflow supervises three worker agents, delegating one phase each and recording each typed result into an event-sourced entity before advancing. The four diagrams below are the source for the Architecture tab; they render with the Lesson 24 mermaid overrides.

## Component graph

See `PLAN.md` → Component graph. A `RequestSimulator` TimedAction and the `ProspectEndpoint` both feed `RequestQueueEntity`. Each `AnalysisQueued` event drives `AnalysisRequestConsumer`, which starts one `ProspectWorkflow` per prospect. The workflow delegates to `CompanyResearchAgent`, then `OrgAnalysisAgent`, then `OutreachStrategyAgent`, writing results into `ProspectEntity`. `ProspectsView` projects entity events for the endpoint's list and SSE stream. `StalledAnalysisMonitor` watches the view and marks in-progress prospects stalled.

## Interaction sequence

See `PLAN.md` → Interaction sequence. The primary journey runs a single prospect through research, analysis, and strategy. The `recordAnalysis` write applies the PII sanitizer (S1) that masks each decision-maker contact hint before the `OrgAnalyzed` event persists.

## State machine

See `PLAN.md` → State machine. A prospect moves `RESEARCHING` → `ANALYZING` → `STRATEGIZING` → `COMPLETED`. From any in-progress state the monitor can drive it to the terminal `STALLED` state.

## Entity model

See `PLAN.md` → Entity model. `RequestQueueEntity` spawns workflows; `ProspectEntity` emits the five lifecycle events; `ProspectsView` holds one row per prospect.

## Governance touch-points

- The before-tool-call guardrail (G1) sits between `CompanyResearchAgent` and `ResearchToolEndpoint`, refusing off-domain calls.
- The PII sanitizer (S1) sits inside `ProspectEntity.recordAnalysis`, masking contact hints at the write boundary.
