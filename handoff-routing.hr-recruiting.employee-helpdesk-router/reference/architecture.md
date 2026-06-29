# Architecture — employee-helpdesk-router

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`QuestionSimulator` is the heartbeat — a TimedAction that ticks every 30 s and writes simulated `InboundQuestionReceived` events into `QuestionQueue` (event-sourced for audit). A `PiiSanitizer` Consumer subscribes to that queue, redacts employee identifiers from the payload, registers a `QuestionEntity`, and starts a `HelpWorkflow` instance.

The workflow orchestrates the handoff. It calls `TopicRouter` to classify the sanitized question, then branches: `HR` invokes `HRSpecialist`, `IT` invokes `ITSpecialist`, `POLICY` invokes `PolicySpecialist`, `UNCLEAR` terminates in `ESCALATED`. The chosen specialist owns the `ANSWER` task end-to-end and returns a typed `Answer`. The workflow then calls `AnswerGuardrail` to check the draft against the internal-policy rubric. On `allowed=true`, the workflow publishes; on `allowed=false`, it transitions to `BLOCKED` and the question sits there until an HR-team member unblocks via `POST /api/questions/{id}/unblock`.

A second Consumer, `RouteEvalScorer`, runs independently of the workflow. It subscribes to `QuestionEntity` events; on every `QuestionRouted` it invokes `RouteJudge` and writes a `RouteScored` event back to the same entity. This is the out-of-band quality signal for the routing decision — not a gate, but a continuous per-question score surfaced in the App UI.

## Interaction sequence

The sequence diagram traces J1 (the HR happy path). Two distinct concurrent flows merge into `QuestionEntity`:

1. The workflow path: `routeStep` → `branchStep` → `hrStep` → `guardrailStep` → `publishStep`.
2. The eval path: `RouteEvalScorer` observes `QuestionRouted` and writes `RouteScored` in parallel.

Both write to the same `QuestionEntity`; commands are idempotent on `questionId` and the events do not collide (different fields are mutated). The view materialises both events independently, so the App UI sees a route score appear within ~10 s of the routing decision regardless of which path completes first.

## State machine

Ten states. The key branches:

- After `SANITIZED`, the `QuestionRouted` event transitions to `ROUTED`. `branchStep` then immediately emits `RoutingBranched` which moves to `ROUTED_HR`, `ROUTED_IT`, or `ROUTED_POLICY`. `UNCLEAR` jumps straight to `ESCALATED` terminal — no specialist is invoked.
- After any `ROUTED_*` state, the specialist returns an `Answer` draft and the entity moves to `ANSWER_DRAFTED`. The guardrail verdict then branches: `allowed=true` → `ANSWERED` terminal; `allowed=false` → `BLOCKED`. From `BLOCKED`, the only forward path is operator-initiated unblock → `ANSWERED`. There is no auto-timeout on `BLOCKED` questions.

`RouteScored` events do not change `status`; they attach the eval score to the entity. The state diagram omits this transition for clarity.

## Entity model

`QuestionEntity` is the source of truth and emits ten event types: registration, sanitization, routing, branch selection, draft, guardrail verdict, publish, block, escalation, and the eval score. `QuestionQueue` is the upstream audit log — only `PiiSanitizer` subscribes to it. `QuestionView` projects the entity into a single read-model row.

## Defence-in-depth governance flow

For any answer that goes out to an employee, the question passed through:

1. **PII sanitizer** — the model never sees raw employee ids, email addresses, or phone numbers.
2. **TopicRouter** — typed classifier with a default-to-`UNCLEAR` policy that prevents quietly mis-routing ambiguous questions to the wrong specialist.
3. **Specialist agent** — owns the answer end-to-end with a tightly-scoped prompt (no inventing policy document ids, no fabricating benefit amounts, no legal/medical advice).
4. **AnswerGuardrail** — typed before-agent-response check against the internal-policy rubric. Blocks invented policy references, fabricated benefit amounts, scope violations, and `[REDACTED]` echoes.
5. **RouteEvalScorer** — out-of-band scoring of every routing decision, surfaced as a per-question chip in the UI.

Step 1 is preventive (the model cannot leak what it never sees). Step 4 is detective (drafts that violate are held). Step 5 is continuous (a sustained drop in route scores signals a classification regression).
