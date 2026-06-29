# Architecture — triage-expert-multi-agent-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph has two LLM-calling components arranged in a strict sequence. `SupportCaseEndpoint` accepts a `{issueDescription}` POST, writes `CaseCreated` onto `SupportCaseEntity`, and starts `SupportCaseWorkflow` keyed by `"workflow-" + caseId`. The workflow's first two steps call `TriageAgent` — `gatherStep` with `TaskDef.taskType(GATHER_CUSTOMER_INFO)` and `summarizeStep` with `TaskDef.taskType(SUMMARIZE_ISSUE)`. `TriageAgent` uses `IntakeTools` to look up the customer account and classify the issue, then distills the result into a typed `IssueSummary`.

Between the triage and expert phases, `sanitizeStep` runs `PiiSanitizer` in-process. No agent is involved; no LLM call is made. The sanitizer replaces name, email, account ID, and phone tokens with typed placeholders, producing a `SanitizedSummary`. This is the only input that `ExpertAgent` receives.

`recommendStep` calls `ExpertAgent` with `TaskDef.taskType(COMPOSE_RECOMMENDATION)` and the `SanitizedSummary` as instruction context. `ExpertAgent` searches the knowledge base using `KnowledgeBaseTools`, fetches relevant articles, and composes a `Recommendation`. Before the typed result is returned to the workflow, `RecommendationGuardrail` runs its `before-agent-response` check. A non-compliant recommendation is blocked and a `GuardrailBlocked` event records the violation; the agent retries inside its 3-iteration budget.

After a compliant `Recommendation` lands on the entity, `evalStep` runs `RecommendationScorer` in-process and writes `EvaluationScored`. `SupportCaseView` projects every entity event into a read-model row; `SupportCaseEndpoint` serves that read model over REST and SSE.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth noting:

1. **The agent boundary is the data-minimization boundary.** Between `summarizeStep` and `recommendStep`, the workflow runs `sanitizeStep`. `ExpertAgent` never receives a task with the customer's actual name, email, account ID, or phone number — only the sanitized placeholders. The typed handoff from `TriageAgent` to `ExpertAgent` passes through an explicit scrub step.
2. **The guardrail runs before the result is committed.** `RecommendationGuardrail` intercepts the `ExpertAgent` response before `RecommendationComposed` is written onto the entity. If the response is blocked, no non-compliant text is persisted to the entity's event log.
3. **Tool separation is enforced by agent registration.** `IntakeTools` is registered only on `TriageAgent`; `KnowledgeBaseTools` is registered only on `ExpertAgent`. There is no runtime mechanism preventing a mistakenly cross-registered tool — the separation is enforced at build time by the agent's `AgentDefinition`.

The agent calls are bounded by per-step timeouts: 60 s each for `gatherStep` and `summarizeStep`, 90 s for `recommendStep` (three guardrail retry iterations plus LLM latency). `sanitizeStep` and `evalStep` are synchronous and finish in milliseconds.

## State machine

Eleven states. The interesting paths:

- The happy path walks `CREATED → GATHERING → GATHERED → SUMMARIZING → SUMMARIZED → SANITIZING → SANITIZED → RECOMMENDING → RECOMMENDED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `GATHERING`, `SUMMARIZING`, or `RECOMMENDING`. A failed case's prior data is preserved on the entity — the UI shows the partial state and the support team can act on the triage data even if the recommendation failed.
- `GuardrailBlocked` is a side-event recorded for audit; it does not change `status`. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `RESOLVED` or `CLOSED` state. The recommendation is advisory; the support agent acts on it outside the system. The blueprint stops at `EVALUATED`.

## Entity model

`SupportCaseEntity` is the source of truth. It emits twelve event types: four phase-start events, four phase-completion events, an evaluation, the guardrail audit event, the failure, and the initial creation. `SupportCaseView` projects every event into a row. `SupportCaseWorkflow` both reads and writes on the entity across all five steps. `TriageAgent` returns `CustomerInfo` and `IssueSummary`; `ExpertAgent` returns `Recommendation` — each typed result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any case that lands in the entity log, the customer's issue passed through:

1. **PII sanitizer** — runs between the two agent phases. Every item in the `IssueSummary` that contains a personal-data token is scrubbed before `ExpertAgent` receives it. The sanitization is deterministic: the same input always produces the same sanitized output.
2. **ExpertAgent (1 task run)** — one model call, one structured output. The task's instruction context contains only the sanitized summary; no raw PII reaches the LLM.
3. **Before-agent-response guardrail** — every `ExpertAgent` response is checked for disallowed-content patterns before it is written onto the entity. A blocked response is never persisted.
4. **On-decision evaluator** — every committed recommendation gets a 1–5 grounding score. Article citation, article provenance, and category alignment are each worth one point.

Each step is independent. The sanitizer does not check content policy; the guardrail does not check citation grounding; the evaluator does not check PII. Removing any one of them opens an explicit gap the others do not cover.
