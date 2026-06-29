# Architecture — auto-insurance-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`RequestSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundRequestReceived` events into `RequestQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts the payload, registers a `RequestEntity`, and starts an `InsuranceWorkflow` instance.

The workflow orchestrates the handoff. It calls `ClaimTriageAgent` to classify the sanitized request, then branches across four paths: `CLAIM` invokes `ClaimsSpecialist`, `POLICY` invokes `PolicySpecialist`, `REWARDS` invokes `RewardsSpecialist`, `ROADSIDE` invokes `RoadsideSpecialist`, and `UNCLEAR` terminates immediately in `ESCALATED`. The chosen specialist owns the `HANDLE` task end-to-end and returns a typed `MemberResponse`. The workflow then calls `ResponseGuardrail` to check the draft against the insurance-communications policy rubric. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `BLOCKED` and the request sits in the supervisor queue until an operator calls `POST /api/requests/{id}/unblock`.

A second Consumer, `TriageEvalScorer`, runs independently of the workflow. It subscribes to `RequestEntity` events; on every `TriageDecided` it invokes `TriageJudge` and writes a `TriageScored` event back to the same entity. This is the **out-of-band eval** mechanism — the score is metadata, not a gate.

## Interaction sequence

The sequence diagram traces J1 (the claim happy path). Two distinct concurrent flows merge into `RequestEntity`:

1. The workflow path: `triageStep` → `routeStep` → `claimsStep` → `guardrailStep` → `publishStep`.
2. The eval path: `TriageEvalScorer` observes `TriageDecided` and writes `TriageScored` in parallel.

Both write to the same `RequestEntity`; the entity's commands are idempotent on `requestId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a triage score appear within ~10 s of the routing decision regardless of which path completes first.

## State machine

Eleven states. The notable branches:

- After `TRIAGED`, the `category` value fans out to four `ROUTED_*` states — one per specialist. An `UNCLEAR` decision jumps straight to the `ESCALATED` terminal; no specialist is invoked.
- All four `ROUTED_*` states converge at `RESPONSE_DRAFTED` once the specialist returns. The four branches collapse back to a single guardrail check.
- After `RESPONSE_DRAFTED`, the guardrail verdict branches again. `allowed=true` → `RESOLVED` terminal. `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward transition is supervisor-initiated unblock → `RESOLVED`. There is no auto-timeout — blocked drafts wait indefinitely.

`TriageScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this as a no-op for clarity.

## Entity model

`RequestEntity` is the source of truth and emits ten event types covering registration, sanitization, triage, routing, draft, guardrail verdict, publish, block, escalate, and the eval score. `RequestQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `RequestView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any response that is delivered to a member, the request passed through:

1. **PII sanitizer** — the model never sees raw member identifiers (policy numbers, VINs, member IDs, driver's license numbers).
2. **ClaimTriageAgent** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous content to the wrong specialist.
3. **Specialist agent** — owns the resolution end-to-end with a tightly-scoped prompt (no invented amounts, no invented ETAs, no out-of-scope advice, no redacted-token echoes).
4. **ResponseGuardrail** — typed before-agent-response check against the insurance-communications policy rubric. Blocks invented settlement amounts, invented coverage discounts, invented roadside ETAs, scope violations, and `[REDACTED]` echoes.
5. **TriageEvalScorer** — out-of-band scoring of every routing decision for continuous quality monitoring.

Step 1 is preventive (the model cannot leak what it never receives). Step 4 is detective (policy-violating drafts are held before delivery). Step 5 is continuous (a sustained drop in triage scores would signal a category-classification regression worth investigating).
