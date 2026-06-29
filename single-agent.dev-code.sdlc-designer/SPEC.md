# SPEC — sdlc-technical-designer

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SDLC Technical Designer.
**One-line pitch:** A user submits a feature description and selects a project context profile; one AI agent reads the feature description (passed as a task attachment, never as inline prompt text) and returns a structured `DesignProposal` — chosen components, a data model, an API surface sketch, and a decision log entry for every significant choice.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `DesignAgent` (AutonomousAgent) carries the entire design decision; the surrounding components only prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **before-agent-response guardrail** validates the agent's proposal on every turn: well-formed JSON, every decision-log entry references a real component id, every selected pattern is in the allowed-pattern registry, and all required sections are present. A malformed proposal triggers a retry inside the same task.
- An **on-decision eval** runs immediately after each `ProposalRecorded` event, scoring the proposal on completeness and rationale depth (does every component choice have a non-empty rationale? do data-model decisions reference concrete field types? is the API surface actionable?).

The blueprint shows that the single-agent pattern does not mean "ungoverned" — two independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a feature description into the **Feature** textarea (or picks one of three seeded examples — a user-notification service, a real-time metrics aggregator, and a document-processing pipeline).
2. The user picks a **Project context** from a dropdown (microservices, monolith, event-driven) or pastes a custom context YAML block.
3. The user fills **Requested by** and clicks **Submit for design**. The UI POSTs to `/api/designs` and receives a `designId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `CONTEXT_LOADED` — the assembled context package is visible in the card detail.
5. Within ~10–30 s, the workflow's `designStep` completes. The card transitions to `DESIGNING` then `PROPOSAL_RECORDED`. The proposal appears: a component table, a data model sketch, an API surface section, and an expandable decision log.
6. Within ~1 s of the proposal, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the proposal's decisions are well-reasoned.
7. The user can submit another feature; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DesignEndpoint` | `HttpEndpoint` | `/api/designs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `DesignRequestEntity`, `DesignView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `DesignRequestEntity` | `EventSourcedEntity` | Per-request lifecycle: submitted → context-loaded → designing → proposal → eval. Source of truth. | `DesignEndpoint`, `ContextLoader`, `DesignWorkflow` | `DesignView` |
| `ContextLoader` | `Consumer` | Subscribes to `DesignRequestSubmitted` events; assembles project context package; calls `DesignRequestEntity.attachContext`. | `DesignRequestEntity` events | `DesignRequestEntity` |
| `DesignWorkflow` | `Workflow` | One workflow per design request. Steps: `awaitContextStep` → `designStep` → `evalStep`. | started by `ContextLoader` once context-loaded event lands | `DesignAgent`, `DesignRequestEntity` |
| `DesignAgent` | `AutonomousAgent` | The one decision-making LLM. Receives project context profile in the task definition and the feature description as a task attachment; returns `DesignProposal`. | invoked by `DesignWorkflow` | returns proposal |
| `DesignView` | `View` | Read model: one row per design request for the UI. | `DesignRequestEntity` events | `DesignEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TechConstraint(String constraintId, String description, String category) {}
// category: "language", "framework", "infra", "team-skill", "compliance"

record DesignRequest(
    String designId,
    String featureTitle,
    String featureDescription,
    String projectContextKey,
    List<TechConstraint> constraints,
    String requestedBy,
    Instant submittedAt
) {}

record ProjectContext(
    String contextKey,
    String architecturePattern,   // e.g. "microservices", "monolith", "event-driven"
    List<String> existingComponents,
    List<String> preferredPatterns,
    String targetLanguage
) {}

record ComponentChoice(
    String componentId,
    String componentKind,         // e.g. "EventSourcedEntity", "Workflow", "HttpEndpoint", "View"
    String rationale,
    List<String> dependsOn
) {}

record DataField(
    String fieldName,
    String fieldType,
    boolean required,
    String purpose
) {}

record DataModelSketch(
    String entityName,
    List<DataField> fields,
    List<String> eventTypes
) {}

record ApiEndpointSketch(
    String method,
    String path,
    String requestSummary,
    String responseSummary,
    String owningComponent
) {}

record DecisionLogEntry(
    String decisionId,
    String componentId,
    String decision,
    String rationale,
    List<String> alternativesConsidered
) {}

record DesignProposal(
    List<ComponentChoice> components,
    List<DataModelSketch> dataModel,
    List<ApiEndpointSketch> apiSurface,
    List<DecisionLogEntry> decisionLog,
    String executiveSummary,
    Instant decidedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record DesignRequestState(
    String designId,
    Optional<DesignRequest> request,
    Optional<ProjectContext> context,
    Optional<DesignProposal> proposal,
    Optional<EvalResult> eval,
    DesignStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum DesignStatus {
    SUBMITTED, CONTEXT_LOADED, DESIGNING, PROPOSAL_RECORDED, EVALUATED, FAILED
}
```

Events on `DesignRequestEntity`: `DesignRequestSubmitted`, `ContextLoaded`, `DesignStarted`, `ProposalRecorded`, `EvaluationScored`, `DesignFailed`.

Every nullable lifecycle field on `DesignRequestState` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/designs` — body `{ featureTitle, featureDescription, projectContextKey, constraints: [TechConstraint], requestedBy }` → `{ designId }`.
- `GET /api/designs` — list all design requests, newest-first.
- `GET /api/designs/{id}` — one design request.
- `GET /api/designs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SDLC Technical Designer</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted design requests (status pill + eval score chip + age) and a right pane with the selected request's detail — submitted constraints list, project context preview, proposal summary, component table, data model sketch, API surface table, decision log, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `DesignAgent`. Asserts the candidate response is well-formed `DesignProposal` JSON, every `decisionLog[].componentId` matches a `componentId` in the `components` list, every `componentKind` is in the allowed set (`EventSourcedEntity`, `Workflow`, `HttpEndpoint`, `View`, `Consumer`, `AutonomousAgent`, `TimedAction`), the `components` list is non-empty, `executiveSummary` is non-empty, and `apiSurface` is non-empty. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `ProposalRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call — the eval is rule-based on purpose, so the same proposal always scores the same) checks that every `ComponentChoice.rationale` is non-empty, every `DecisionLogEntry.rationale` is non-empty, every `DataField.purpose` is non-empty, and that `alternativesConsidered` lists at least one alternative per decision. Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `DesignAgent` → `prompts/design-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached feature description, apply the project context profile, and return one `ComponentChoice` per selected component, a `DataModelSketch` per proposed entity, an `ApiEndpointSketch` per endpoint, and one `DecisionLogEntry` per significant design decision.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the notification-service seed; within 30 s the proposal appears with at least one component, one data model, one API endpoint, and one decision-log entry per component choice.
2. **J2** — The agent's first response on a design request is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed proposal; the UI never displays the malformed response.
3. **J3** — A proposal whose decision-log entries all have empty `rationale` fields receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A proposal for the event-driven context seed contains at least one `AutonomousAgent` or `Consumer` component choice, demonstrating the agent adapts its output to the context profile.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sdlc-technical-designer demonstrating the single-agent × dev-code cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-sdlc-designer. Java package io.akka.samples.sdlctechnicaldesigner.
Akka 3.6.0. HTTP port 9369.

Components to wire (exactly):

- 1 AutonomousAgent DesignAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/design-agent.md>) and
  .capability(TaskAcceptance.of(DESIGN_FEATURE).maxIterationsPerTask(3)). The task receives
  the project context profile as its instruction text and the feature description as a task
  ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is
  the canonical call). Output: DesignProposal{components: List<ComponentChoice>, dataModel:
  List<DataModelSketch>, apiSurface: List<ApiEndpointSketch>, decisionLog:
  List<DecisionLogEntry>, executiveSummary: String, decidedAt: Instant}. The agent is
  configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml) registered
  via the agent's guardrail-configuration block. On guardrail rejection the agent loop retries
  the response within its 3-iteration budget.

- 1 Workflow DesignWorkflow per designId with three steps:
  * awaitContextStep — polls DesignRequestEntity.getDesignRequest every 1s; on
    request.context().isPresent() advances to designStep. WorkflowSettings.stepTimeout 15s
    (context loader is in-process and fast).
  * designStep — emits DesignStarted, then calls componentClient.forAutonomousAgent(
    DesignAgent.class, "designer-" + designId).runSingleTask(
      TaskDef.instructions(formatContext(request.context()))
        .attachment("feature.md", request.request().featureDescription().getBytes())
    ) — returns a taskId, then forTask(taskId).result(DESIGN_FEATURE) to fetch the proposal.
    On success calls DesignRequestEntity.recordProposal(proposal). WorkflowSettings.stepTimeout
    90s with defaultStepRecovery maxRetries(2).failoverTo(DesignWorkflow::error).
  * evalStep — runs a deterministic rule-based ProposalScorer (NOT an LLM call) over the
    recorded proposal: checks that every ComponentChoice.rationale is non-empty, every
    DecisionLogEntry.rationale is non-empty, every DataField.purpose is non-empty, and that
    alternativesConsidered is non-empty per entry. Emits EvaluationScored{score: 1-5,
    rationale: String}. WorkflowSettings.stepTimeout 5s. error step transitions the entity
    to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity DesignRequestEntity (one per designId). State DesignRequestState{
  designId: String, request: Optional<DesignRequest>, context: Optional<ProjectContext>,
  proposal: Optional<DesignProposal>, eval: Optional<EvalResult>, status: DesignStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. DesignStatus enum: SUBMITTED,
  CONTEXT_LOADED, DESIGNING, PROPOSAL_RECORDED, EVALUATED, FAILED. Events:
  DesignRequestSubmitted{request}, ContextLoaded{context}, DesignStarted{},
  ProposalRecorded{proposal}, EvaluationScored{eval}, DesignFailed{reason}. Commands:
  submit, attachContext, markDesigning, recordProposal, recordEvaluation, fail,
  getDesignRequest. emptyState() returns DesignRequestState.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer ContextLoader subscribed to DesignRequestEntity events; on
  DesignRequestSubmitted resolves the ProjectContext for the given projectContextKey
  from src/main/resources/sample-events/project-contexts.jsonl (falls back to a generic
  context for unknown keys), then calls DesignRequestEntity.attachContext(context). After
  attachContext lands, the same Consumer starts a DesignWorkflow with id = "design-" + designId.

- 1 View DesignView with row type DesignRow (mirrors DesignRequestState minus
  request.featureDescription text — the entity holds the full text for the API; the view
  holds a truncated preview for the UI list). Table updater consumes DesignRequestEntity
  events. ONE query getAllDesigns: SELECT * AS designs FROM design_view. No WHERE status
  filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * DesignEndpoint at /api with POST /designs (body
    {featureTitle, featureDescription, projectContextKey,
    constraints: [{constraintId, description, category}], requestedBy};
    mints designId; calls DesignRequestEntity.submit; returns {designId}), GET /designs
    (list from getAllDesigns, sorted newest-first), GET /designs/{id} (one row), GET
    /designs/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- DesignTasks.java declaring one Task<R> constant: DESIGN_FEATURE = Task.name("Design feature")
  .description("Read the attached feature description and produce a DesignProposal for the
  given project context").resultConformsTo(DesignProposal.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records TechConstraint, DesignRequest, ProjectContext, ComponentChoice, DataField,
  DataModelSketch, ApiEndpointSketch, DecisionLogEntry, DesignProposal, EvalResult,
  DesignRequestState, DesignStatus.

- ProposalGuardrail.java implementing the before-agent-response hook. Reads the candidate
  DesignProposal from the LLM response, runs the six checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- ProposalScorer.java — pure deterministic logic (no LLM). Inputs: DesignProposal. Outputs:
  EvalResult. Scoring rubric documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9369 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The DesignAgent.definition() binds
  the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/project-contexts.jsonl with 3 seeded project context
  profiles: a microservices platform context, a monolith-to-modular migration context, and
  an event-driven streaming context. Each includes existingComponents, preferredPatterns,
  and targetLanguage.

- src/main/resources/sample-events/feature-descriptions.jsonl with 3 paired example feature
  descriptions: a user-notification service, a real-time metrics aggregator, and a
  document-processing pipeline. Each is 200–400 words describing the feature, its inputs,
  outputs, and non-functional requirements.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — dev-code
  is the use case, not the anchor.

- risk-survey.yaml at the project root with decisions.authority_level = recommend-only
  (the agent's proposal is advisory; a human engineer reviews before implementation),
  oversight.human_in_loop = true, failure.failure_modes including "hallucinated-component",
  "missing-decision-rationale", "incompatible-pattern-selection",
  "incomplete-data-model"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/design-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: SDLC Technical Designer",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of design-request cards; right = selected-request detail with submitted
  constraints, project context preview, component table, data model sketch, API surface
  table, decision log, and eval-score chip). Browser title exactly:
  <title>Akka Sample: SDLC Technical Designer</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(designId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    design-feature.json — 6 DesignProposal entries covering all three project-context keys.
      Each entry has an executiveSummary paragraph, a components list (3–6 entries), a
      dataModel list (1–3 entities with fields), an apiSurface list (2–5 endpoints), and a
      decisionLog list with one entry per component (non-empty rationale, at least one
      alternative). Plus 2 deliberately MALFORMED entries (one with a decisionLog entry whose
      componentId does not match any component in the components list; one with a
      componentKind value outside the allowed set) — the guardrail blocks both, exercising the
      retry path. The mock should select a malformed entry on the FIRST iteration of every 3rd
      design request (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(designId) helper makes per-request selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. DesignAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion DesignTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (designStep
  90s, awaitContextStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the DesignRequestState record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: DesignTasks.java with DESIGN_FEATURE = Task.name(...).description(...)
  .resultConformsTo(DesignProposal.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9369 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (DesignAgent). The
  on-decision eval is rule-based (ProposalScorer.java) and does NOT make an LLM call —
  keeping the pattern's "one agent" promise honest.
- The feature description is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated designStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns. Lesson 1's
  AutonomousAgent contract is the authoritative reference for how the hook is registered.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
