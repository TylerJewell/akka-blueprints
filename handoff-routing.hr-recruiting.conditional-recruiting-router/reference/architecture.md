# Architecture — conditional-recruiting-router

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`EmailSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundEmailReceived` events into `InboxQueue` (event-sourced for audit). A `CandidateSanitizer` Consumer subscribes to that queue, redacts the payload, registers an `ApplicationEntity`, and starts a `RecruitingWorkflow` instance.

The workflow orchestrates the handoff. It calls `EmailClassifierAgent` to classify the sanitized email, then branches: `INFO_REQUEST` invokes `InfoRequester`, `INTERVIEW_REQUEST` invokes `InterviewOrganizer`, `UNROUTABLE` terminates in `UNROUTABLE_CLOSED`. The chosen specialist owns its task end-to-end and returns a typed result (`OutboundReply` or `CalendarConfirmation`). Before every tool invocation inside `InterviewOrganizer`, the workflow's `toolGuardrailStep` calls `ToolCallGuardrail` to validate the pending call. On `allowed=false`, the workflow transitions to `TOOL_BLOCKED` and the application waits for a recruiter to unblock via `POST /api/applications/{id}/unblock`.

A second Consumer, `RoutingEvalScorer`, runs independently of the workflow. It subscribes to `ApplicationEntity` events; on every `RoutingDecided` it invokes `RoutingJudge` and writes a `RoutingScored` event back to the same entity. This is the **on-decision eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J2 (the interview-request happy path). Two distinct concurrent flows merge into `ApplicationEntity`:

1. The workflow path: `classifyStep` → `routeStep` → `scheduleStep` (with per-tool guardrail check) → `completeStep`.
2. The eval path: `RoutingEvalScorer` observes `RoutingDecided` and writes `RoutingScored` in parallel.

Both write to the same `ApplicationEntity`; the entity's commands are idempotent on `applicationId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI shows a routing score chip within ~10 s of the classification decision regardless of which path completes first.

## State machine

Ten states. The interesting branches:

- After `CLASSIFIED`, the `route` value branches the flow. `INFO_REQUEST` → `ROUTED_INFO`, `INTERVIEW_REQUEST` → `ROUTED_INTERVIEW`, `UNROUTABLE` jumps straight to the `UNROUTABLE_CLOSED` terminal — no specialist is invoked at all.
- `ROUTED_INFO` transitions to `REPLY_DRAFTED` on the info specialist's completion, then immediately to `COMPLETED`.
- `ROUTED_INTERVIEW` transitions to `SLOT_PROPOSED` on a successful tool call, then `COMPLETED`. If the before-tool-call guardrail blocks, it transitions to `TOOL_BLOCKED` instead.
- From `TOOL_BLOCKED`, the only forward transition is recruiter-initiated unblock → `COMPLETED`. There is no auto-timeout — blocked applications wait indefinitely for recruiter action.

`RoutingScored` events do not change `status`; they attach the eval score. The state diagram omits this as a no-op for clarity.

## Entity model

`ApplicationEntity` is the source of truth and emits ten event types covering registration, sanitization, routing, specialist output (reply or slot), tool-call blocking, completion, closure, and the eval score. `InboxQueue` is the upstream audit log — only `CandidateSanitizer` subscribes to it. `ApplicationView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any outbound reply or calendar hold that gets created, the application passed through:

1. **PII sanitizer** — the model never sees raw email addresses, phone numbers, or home addresses.
2. **EmailClassifierAgent** — typed classifier with a default-to-`UNROUTABLE` policy that prevents quietly mis-routing ambiguous content.
3. **Specialist agent** — owns the outcome end-to-end with a tightly-scoped prompt (no invented salary figures, no fabricated slot times, tool use restricted to slots returned by the availability lookup in the same session).
4. **ToolCallGuardrail** — typed before-tool-call check against the validation rubric. Blocks past-dated slots, unresolvable interviewer IDs, missing consent references, and unknown interview formats.
5. **RoutingEvalScorer** — out-of-band scoring of every routing decision.

Step 1 is preventive (the model can't leak what it never sees). Step 4 is preventive at the tool boundary (a bad argument never reaches the calendar or email API). Step 5 is continuous (a sustained drop in routing scores signals a classifier regression before it becomes a recruiter complaint).
