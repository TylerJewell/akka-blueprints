# Architecture — 311-triage

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`RequestSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundRequestReceived` events into `RequestQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the payload, registers a `ServiceRequestEntity`, and starts a `RequestWorkflow` instance.

The workflow orchestrates the handoff in two stages before any specialist is touched. First, `TriageAgent` classifies the sanitized request. Then — before routing to a department — `RouteGuardrail` audits the proposed route. Only after the route is approved does the workflow branch: `PUBLIC_WORKS` invokes `PublicWorksSpecialist`, `PERMITS_ZONING` invokes `PermitsSpecialist`, `UNCLEAR` terminates in `ESCALATED`. A route that fails the guardrail terminates in `FLAGGED_FOR_REVIEW`; no specialist is ever called. A human reviewer can approve the route (which resumes the workflow) or escalate.

A second Consumer, `TriageEvalScorer`, runs independently of the workflow. It subscribes to `ServiceRequestEntity` events; on every `TriageDecided` it invokes `TriageJudge` and writes a `TriageScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the public-works happy path). Two distinct concurrent flows merge into `ServiceRequestEntity`:

1. The workflow path: `triageStep` → `routeGuardrailStep` → `routeStep` → `publicWorksStep` → `publishStep`.
2. The eval path: `TriageEvalScorer` observes `TriageDecided` and writes `TriageScored` in parallel.

Both write to the same `ServiceRequestEntity`; the entity's commands are idempotent on `requestId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a triage score appear within ~10 s of the routing decision regardless of which path completes first.

## State machine

Ten states, including two terminal states (`RESOLVED`, `ESCALATED`) and one holding state (`FLAGGED_FOR_REVIEW`). The interesting branches:

- After `TRIAGED`, the route guardrail check fires first. `FLAGGED_FOR_REVIEW` captures routes the guardrail cannot approve. `ESCALATED` captures `UNCLEAR` triage decisions (and guardrail-triggered escalations after human review). `ROUTE_APPROVED` gates the department branch.
- After `ROUTE_APPROVED`, the `category` value branches the flow. `PUBLIC_WORKS` and `PERMITS_ZONING` proceed to their respective `ROUTED_*` state, then converge at `RESPONSE_DRAFTED` once the specialist returns.
- From `FLAGGED_FOR_REVIEW`, a reviewer's `decision="approve"` transitions back to `ROUTE_APPROVED` and resumes the workflow; `decision="escalate"` transitions to `ESCALATED`.

`TriageScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this as a no-op for clarity.

## Entity model

`ServiceRequestEntity` is the source of truth and emits ten event types covering registration, sanitization, triage, route-verdict, routing, draft, publish, flag, escalate, and the eval score. `RequestQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `RequestView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any reply that goes out, the request passed through:

1. **PII sanitizer** — the model never sees raw constituent identifiers.
2. **TriageAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly misrouting ambiguous content.
3. **RouteGuardrail** — typed before-agent-invocation check against the route rubric. Flags low-confidence routes, content-category mismatches, and out-of-jurisdiction descriptions before any specialist is called.
4. **Department specialist** — owns the response end-to-end with a tightly-scoped prompt (no invented work order numbers, no SLA-specific timelines, no invented permit application numbers).
5. **TriageEvalScorer** — out-of-band scoring of every routing decision.

Step 1 is preventive (the model can't leak what it never sees). Step 3 is preventive at the routing layer (the wrong specialist is never called). Step 5 is continuous (a sustained drop in triage scores would signal a classification regression).
