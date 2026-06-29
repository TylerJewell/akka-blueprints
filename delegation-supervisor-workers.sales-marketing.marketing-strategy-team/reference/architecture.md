# Architecture

The system demonstrates the delegation-supervisor-workers pattern: one supervisor agent breaks a brief into assignments, three worker agents each return a typed artifact, and the supervisor assembles a final plan. A durable `StrategyWorkflow` drives the delegation; an event-sourced `CampaignEntity` holds each campaign's lifecycle; a `CampaignsView` projects it for the UI.

## Component graph

```mermaid
flowchart TB
  classDef agent fill:#0e1e2a,stroke:#7EC8E3,color:#7EC8E3;
  classDef wf fill:#1c1330,stroke:#A855F7,color:#A855F7;
  classDef ese fill:#1f1900,stroke:#F5C518,color:#F5C518;
  classDef view fill:#0e2010,stroke:#3fb950,color:#3fb950;
  classDef cons fill:#251503,stroke:#F97316,color:#F97316;
  classDef ta fill:#1a1c20,stroke:#aab3bd,color:#aab3bd;
  classDef ep fill:#161616,stroke:#fff,color:#fff;

  CE[CampaignEndpoint]:::ep -->|enqueueBrief| BQ[BriefQueue]:::ese
  SIM[BriefSimulator]:::ta -.->|enqueueBrief| BQ
  BQ -.->|BriefEnqueued| BC[BriefConsumer]:::cons
  BC -->|start| WF[StrategyWorkflow]:::wf
  WF -->|runSingleTask| RA[MarketResearchAgent]:::agent
  WF -->|runSingleTask| SA[StrategyAgent]:::agent
  WF -->|runSingleTask| CA[ContentAgent]:::agent
  WF -->|runSingleTask| LA[CampaignLeadAgent]:::agent
  RA -->|GET search| WS[WebSearchEndpoint]:::ep
  WF -->|record*| CAMP[CampaignEntity]:::ese
  CAMP -.->|events| CV[CampaignsView]:::view
  CE -->|query / SSE| CV
```

The workflow is the supervisor's spine: it issues each worker task in turn and writes each result onto the campaign. The lead agent runs last — once as the assembler, once as the evaluator.

## Interaction sequence

See PLAN.md for the full sequence diagram. The path is: brief enqueued → consumer starts the workflow → research → strategy → content → assemble (with the G1 claim-grounding guardrail) → self-evaluation.

## State machine

`CampaignStatus` advances `RECEIVED → RESEARCHED → STRATEGIZED → CONTENT_DRAFTED → ASSEMBLED → EVALUATED`. Any phase can fail to `FAILED`. The G1 guardrail gates the `CONTENT_DRAFTED → ASSEMBLED` transition: an ungrounded plan re-runs the assemble step rather than advancing.

## Entity model

`CampaignEntity` emits `CampaignEvent`s that the `CampaignsView` table updater consumes into a `Campaign` row. `BriefQueue` emits `BriefEnqueued`, which `BriefConsumer` reacts to. The View's single non-streaming query returns all campaigns; status filtering is done client-side because Akka cannot auto-index the enum column (Lesson 2).

## Grounding source

The web-search grounding is modeled in-process. `WebSearchEndpoint` answers `GET /api/search?q=...` from `src/main/resources/sample-docs/search-index.json`, matching query terms against canned result entries. The production analog would call a real search API; the env-var swap to a real client is the only change.
