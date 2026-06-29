# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same diagrams on the Architecture tab.

## Component graph

One topic enters through either the `ContentEndpoint` (a user submits) or the `TopicSimulator` (a scheduled tick). Both write to `InboundTopicQueue`, an event-sourced queue. `TopicConsumer` reacts to each `TopicEnqueued` event and starts one `ContentWorkflow` per topic. The workflow drives the three writer agents, then the brand reviewer, then the quality evaluator, recording each result on `CampaignEntity`. `CampaignsView` projects the entity's events into a read model the `ContentEndpoint` queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey is one topic flowing to a completed campaign. The workflow calls the research agent first because the blog and LinkedIn agents take the research report as input. After drafting, the brand reviewer inspects all three outputs; an off-brand verdict ends the workflow at `BLOCKED` without surfacing any output. A passing verdict moves the campaign to evaluation, where the quality evaluator records a score, and then to completion. Each entity transition fans out to `CampaignsView` and from there to any SSE subscriber.

## State machine

A campaign moves `RECEIVED → RESEARCHING → DRAFTING → REVIEWING`. From `REVIEWING` it forks: `BLOCKED` (terminal, outputs withheld) or `EVALUATING → COMPLETED` (terminal, outputs and score visible). The pipeline is forward-only; there is no compensation path because nothing is published until `COMPLETED`.

## Entity model

`CampaignEntity` is the single event-sourced source of truth. Each lifecycle stage emits one event (`TopicReceived`, `ResearchCompleted`, `ContentDrafted`, `BrandReviewPassed` or `BrandReviewBlocked`, `QualityEvaluated`, `CampaignCompleted`). `CampaignsView` holds one row per campaign, keyed by id, and is the only read path the UI uses.
