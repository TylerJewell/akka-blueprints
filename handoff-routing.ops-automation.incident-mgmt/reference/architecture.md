# Architecture — incident-management

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`IncidentFeeder` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundReportReceived` events into `IncidentQueue` (event-sourced for audit). A `ContextEnricher` Consumer subscribes to that queue, attaches environment metadata (host group, service tier, tags), registers an `IncidentEntity`, and starts an `IncidentWorkflow` instance.

The workflow orchestrates the handoff. It calls `ClassifierAgent` to categorize the enriched report, then branches: `INFRASTRUCTURE` invokes `InfraSpecialist`, `APPLICATION` invokes `AppSpecialist`, `AMBIGUOUS` terminates in `ESCALATED`. The chosen specialist owns the `INVESTIGATE` task end-to-end and returns a typed `RemediationPlan`. The workflow then calls `ToolCallGuardrail` to check the plan against the ops policy rubric before any action executes. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `BLOCKED` and the incident waits until an operator unblocks via `POST /api/incidents/{id}/unblock`.

A second Consumer, `RoutingEvalScorer`, runs independently of the workflow. It subscribes to `IncidentEntity` events; on every `IncidentClassified` it invokes `RoutingJudge` and writes a `RoutingScored` event back to the same entity. This is the **on-incident-reporter eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the infrastructure happy path). Two distinct concurrent flows merge into `IncidentEntity`:

1. The workflow path: `classifyStep` → `routeStep` → `infraStep` → `guardrailStep` → `publishStep`.
2. The eval path: `RoutingEvalScorer` observes `IncidentClassified` and writes `RoutingScored` in parallel.

Both write to the same `IncidentEntity`; the entity's commands are idempotent on `incidentId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a routing score appear within ~10 s of the classification decision regardless of which path completes first.

## State machine

Nine states. The interesting branches:

- After `CLASSIFIED`, the `category` value branches the flow. `INFRASTRUCTURE` and `APPLICATION` go to their respective `ROUTED_*` state, then converge at `PLAN_DRAFTED` once the specialist returns. `AMBIGUOUS` jumps straight to the `ESCALATED` terminal — no specialist is invoked at all.
- After `PLAN_DRAFTED`, the guardrail verdict branches again. `allowed=true` → `RESOLVED` terminal. `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is operator-initiated unblock → `RESOLVED`. There is no auto-timeout — blocked plans wait indefinitely.

`RoutingScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this as a no-op for clarity.

## Entity model

`IncidentEntity` is the source of truth and emits ten event types covering registration, enrichment, classification, routing, plan draft, guardrail verdict, publish, block, escalate, and the eval score. `IncidentQueue` is the upstream audit log — only `ContextEnricher` subscribes to it. `IncidentView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any remediation that executes, the incident passed through:

1. **ContextEnricher** — the model receives structured metadata (host group, service tier, extracted tags) rather than raw alert payload text. This reduces prompt-injection surface.
2. **ClassifierAgent** — typed classifier with a default-to-`AMBIGUOUS` policy that prevents quietly mis-routing mixed-signal incidents.
3. **Specialist agent** — owns the investigation end-to-end with a tightly-scoped prompt (authority limits, no-invented-runbooks, escalation triggers for out-of-scope actions).
4. **ToolCallGuardrail** — typed before-tool-call check against the ops policy rubric. Blocks peak-hours restarts, critical-host actions without change records, and alert-token echoes in summaries.
5. **RoutingEvalScorer** — out-of-band scoring of every routing decision.

Step 1 is preventive (the model works on enriched metadata, not raw alert content). Step 4 is detective (plans that violate ops policy are held). Step 5 is continuous (a sustained drop in routing scores signals a classifier regression before it causes widespread mis-routing).
