# Architecture

This file narrates the four diagrams in [`PLAN.md`](../PLAN.md). The generated system renders the same mermaid source on the Architecture tab.

## Component graph

Engagements enter two ways: the `RequestSimulator` TimedAction drips a canned brief from `sample-events/engagements.jsonl` every 30 seconds, and the `EngagementEndpoint` accepts a `POST /api/engagements`. Both land on `InboundRequestQueue`, an event-sourced entity whose `InboundRequestQueued` event the `EngagementRequestConsumer` reacts to by starting one `EngagementWorkflow` per brief.

The workflow is the supervisor. It calls `ConsultingCoordinator` to route, then either `JuniorResearcher` (delegate) or `SeniorConsultant` (handoff) to produce the deliverable, writing each transition to `EngagementEntity`. The entity's events project into `EngagementsView`, which the endpoint queries and streams over SSE. A second consumer, `RoutingEvalConsumer`, reacts to `EngagementRouted` to record a non-blocking routing eval.

## Interaction sequence

The sequence diagram traces a high-stakes engagement. The coordinator returns a `HANDOFF` decision with a high complexity score; the workflow records routing, the eval consumer scores the decision, the senior consultant produces a recommendation that passes the before-agent-response guardrail, and the engagement reaches `DELIVERED`. Compliance review is noted as pending — it happens after delivery and does not gate it.

## State machine

An engagement moves `RECEIVED → ROUTED`, then branches to `RESEARCHING` (delegate) or `CONSULTING` (handoff), both converging on `DELIVERED`. From `DELIVERED` a human compliance reviewer can move a senior recommendation to `COMPLIANCE_REVIEWED` or `FLAGGED`. The state-label and edge-label CSS overrides from Lesson 24 are required so the labels render white and uncut.

## Entity model

`InboundRequestQueue` starts the workflow that drives `EngagementEntity`. `EngagementEntity` is both the durable state and the row type for `EngagementsView`; every lifecycle field is `Optional<T>` so the view materializer accepts the record before those fields are populated.
