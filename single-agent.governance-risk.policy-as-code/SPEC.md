# SPEC — policy-as-code

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** PolicyAsCode.
**One-line pitch:** A user submits a change request and a set of encoded policy rules; one AI agent reads the change payload (passed as a task attachment, never as inline prompt text) and returns a structured enforcement decision — ALLOW / DENY / WARN with a violation finding per rule.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the governance-risk domain. One `PolicyEnforcementAgent` (AutonomousAgent) carries the entire enforcement decision; the surrounding components prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** intercepts every external lookup the agent attempts — registry queries, external policy store lookups — and validates each call against an allowlist before it executes. Disallowed tool calls return a structured rejection so the agent can adapt its reasoning without reaching an external system.
- A **CI gate** enforces that every change payload must receive a finalized `PolicyDecision` before a downstream deployment step can proceed. The gate is checked at two points: when the decision first lands (`DECISION_RECORDED`) and when the deployer queries the gate status endpoint. A pending or missing decision blocks the gate.

The blueprint shows that a single-agent pattern can enforce hard build/deploy gates while the agent itself remains the sole decision-making LLM.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a change payload into the **Change payload** textarea (or picks one of three seeded examples — an infrastructure Terraform plan diff, a Kubernetes manifest change, a dependency-version bump).
2. The user picks a **policy set** from a dropdown (Infrastructure, Kubernetes, Supply-chain) or pastes a custom list of policy rules.
3. The user clicks **Submit for evaluation**. The UI POSTs to `/api/changes` and receives a `changeId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `VALIDATED` — the change metadata has been structurally verified and is visible in the card detail.
5. Within ~10–30 s, the workflow's `enforceStep` completes. The card transitions to `ENFORCING` then `DECISION_RECORDED`. The decision appears: a top-level badge (ALLOW / DENY / WARN), a short rationale paragraph, and a per-rule finding table (rule id, severity, affected resource, recommendation).
6. Within ~1 s of the decision, the `gateStep` finishes. The card shows a **CI gate** chip (OPEN / BLOCKED) based on whether any DENY finding exists.
7. The user can submit another change; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PolicyEndpoint` | `HttpEndpoint` | `/api/changes/*` — submit, list, get, SSE, gate-status; serves `/api/metadata/*`. | — | `ChangeRequestEntity`, `PolicyView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ChangeRequestEntity` | `EventSourcedEntity` | Per-change lifecycle: submitted → validated → enforcing → decision → gated. Source of truth. | `PolicyEndpoint`, `ChangeValidator`, `EvaluationWorkflow` | `PolicyView` |
| `ChangeValidator` | `Consumer` | Subscribes to `ChangeSubmitted` events; validates structure and metadata; calls `ChangeRequestEntity.attachValidated`. | `ChangeRequestEntity` events | `ChangeRequestEntity` |
| `EvaluationWorkflow` | `Workflow` | One workflow per change. Steps: `awaitValidatedStep` → `enforceStep` → `gateStep`. | started by `ChangeValidator` once validated event lands | `PolicyEnforcementAgent`, `ChangeRequestEntity` |
| `PolicyEnforcementAgent` | `AutonomousAgent` | The one decision-making LLM. Receives policy rules in the task definition and the change payload as a task attachment; returns `PolicyDecision`. | invoked by `EvaluationWorkflow` | returns decision |
| `PolicyView` | `View` | Read model: one row per change for the UI. | `ChangeRequestEntity` events | `PolicyEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PolicyRule(String ruleId, String text, String category, Severity severityFloor) {}

record ChangeRequest(
    String changeId,
    String changeTitle,
    String changeType,       // "terraform-plan" | "k8s-manifest" | "dependency-bump" | "custom"
    String rawPayload,
    List<PolicyRule> rules,
    String submittedBy,
    Instant submittedAt
) {}

record ValidatedChange(
    String changeType,
    List<String> affectedResources,
    String normalizedPayload     // whitespace-normalized, comment-stripped form
) {}

record Violation(
    String ruleId,
    Severity severity,
    String affectedResource,
    String evidence,
    String recommendation
) {}

record PolicyDecision(
    Outcome outcome,
    String rationale,
    List<Violation> violations,
    Instant decidedAt
) {}
enum Outcome { ALLOW, WARN, DENY }

record GateResult(
    GateStatus status,   // OPEN | BLOCKED
    String reason,
    Instant evaluatedAt
) {}
enum GateStatus { OPEN, BLOCKED }

record ChangeEvaluation(
    String changeId,
    Optional<ChangeRequest> request,
    Optional<ValidatedChange> validated,
    Optional<PolicyDecision> decision,
    Optional<GateResult> gate,
    EvaluationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum Severity { LOW, MEDIUM, HIGH, CRITICAL }

enum EvaluationStatus {
    SUBMITTED, VALIDATED, ENFORCING, DECISION_RECORDED, GATED, FAILED
}
```

Events on `ChangeRequestEntity`: `ChangeSubmitted`, `ChangeValidated`, `EnforcementStarted`, `DecisionRecorded`, `GateEvaluated`, `EvaluationFailed`.

Every nullable lifecycle field on the `ChangeEvaluation` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/changes` — body `{ changeTitle, changeType, rawPayload, rules: [PolicyRule], submittedBy }` → `{ changeId }`.
- `GET /api/changes` — list all evaluations, newest-first.
- `GET /api/changes/{id}` — one evaluation.
- `GET /api/changes/{id}/gate` — gate status for CI integration: `{ changeId, gateStatus, reason }`.
- `GET /api/changes/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: PolicyAsCode</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted change evaluations (status pill + outcome badge + age) and a right pane with the selected evaluation's detail — submitted policy rules list, validated change preview, decision rationale, violation table, and CI gate chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs before every tool call initiated by `PolicyEnforcementAgent`. Maintains an allowlist of permitted tool names (`lookup-policy-metadata`, `query-resource-registry`). Any tool call not on the allowlist is blocked and returns a structured rejection to the agent so it can continue without the disallowed data source. On rejection the agent receives the blocked-tool error and must reason from the attached payload and the permitted tools only.
- **C1 — CI gate** (`ci-gate`, `configuration-gate`): implemented inside `EvaluationWorkflow.gateStep`, fired immediately after `DecisionRecorded`. The gate scorer examines the `PolicyDecision.outcome`: if any violation carries `severity == CRITICAL` or the overall `outcome == DENY`, the gate sets `GateStatus = BLOCKED`; otherwise `OPEN`. Emits `GateEvaluated` with reason text. The `GET /api/changes/{id}/gate` endpoint exposes gate status for CI system polling; a BLOCKED gate means the pipeline step must not proceed.

## 9. Agent prompts

- `PolicyEnforcementAgent` → `prompts/policy-enforcement-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached change payload, walk every policy rule, and return one `Violation` per rule that is breached plus the overall `Outcome`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the Terraform seed change; within 30 s the decision appears with one violation entry per breached rule and a CI gate chip.
2. **J2** — The agent attempts a disallowed tool call (mock path) — the `before-tool-call` guardrail blocks it; the agent continues using only the attached payload; a decision still lands within the step timeout.
3. **J3** — A DENY decision causes the gate endpoint to return `BLOCKED`; a subsequent GET confirms the pipeline would be halted.
4. **J4** — A change payload with no matching policy violations receives an ALLOW decision and an OPEN gate; the UI shows neither violations nor a blocked chip.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named policy-as-code demonstrating the single-agent × governance-risk cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-governance-risk-policy-as-code. Java package io.akka.samples.policyascode.
Akka 3.6.0. HTTP port 9701.

Components to wire (exactly):

- 1 AutonomousAgent PolicyEnforcementAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/policy-enforcement-agent.md>) and
  .capability(TaskAcceptance.of(ENFORCE_POLICY).maxIterationsPerTask(3)). The task receives
  policy rules as its instruction text and the validated change payload as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: PolicyDecision{outcome: Outcome (ALLOW/WARN/DENY), rationale: String,
  violations: List<Violation>, decidedAt: Instant}. The agent is configured with a
  before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent receives a structured error
  and adapts without retrying the whole task.

- 1 Workflow EvaluationWorkflow per changeId with three steps:
  * awaitValidatedStep — polls ChangeRequestEntity.getChange every 1s; on
    change.validated().isPresent() advances to enforceStep. WorkflowSettings.stepTimeout 15s
    (validator is in-process and fast).
  * enforceStep — emits EnforcementStarted, then calls componentClient.forAutonomousAgent(
    PolicyEnforcementAgent.class, "enforcer-" + changeId).runSingleTask(
      TaskDef.instructions(formatRules(change.request.rules))
        .attachment("change.txt", change.validated.normalizedPayload.getBytes())
    ) — returns a taskId, then forTask(taskId).result(ENFORCE_POLICY) to fetch the decision.
    On success calls ChangeRequestEntity.recordDecision(decision).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(EvaluationWorkflow::error).
  * gateStep — runs a deterministic GateEvaluator (NOT an LLM call) over the recorded
    decision: if outcome == DENY or any violation.severity == CRITICAL, gate is BLOCKED;
    otherwise OPEN. Emits GateEvaluated{status, reason: String}. WorkflowSettings.stepTimeout
    5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ChangeRequestEntity (one per changeId). State ChangeEvaluation{
  changeId: String, request: Optional<ChangeRequest>, validated: Optional<ValidatedChange>,
  decision: Optional<PolicyDecision>, gate: Optional<GateResult>,
  status: EvaluationStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  EvaluationStatus enum: SUBMITTED, VALIDATED, ENFORCING, DECISION_RECORDED, GATED, FAILED.
  Events: ChangeSubmitted{request}, ChangeValidated{validated}, EnforcementStarted{},
  DecisionRecorded{decision}, GateEvaluated{gate}, EvaluationFailed{reason}.
  Commands: submit, attachValidated, markEnforcing, recordDecision, recordGate, fail,
  getChange. emptyState() returns ChangeEvaluation.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer ChangeValidator subscribed to ChangeRequestEntity events; on ChangeSubmitted
  runs a structural validation pass over rawPayload (checks it is non-empty, detects changeType
  from content heuristics, extracts a list of affectedResources, and produces a
  normalizedPayload by stripping comments and collapsing whitespace), then calls
  ChangeRequestEntity.attachValidated(validated). After attachValidated lands, the same
  Consumer starts an EvaluationWorkflow with id = "eval-" + changeId.

- 1 View PolicyView with row type ChangeRow (mirrors ChangeEvaluation minus
  request.rawPayload — the audit log keeps the raw; the view holds the normalized form for
  the UI). Table updater consumes ChangeRequestEntity events. ONE query getAllChanges:
  SELECT * AS changes FROM policy_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * PolicyEndpoint at /api with POST /changes (body
    {changeTitle, changeType, rawPayload, rules: [{ruleId, text, category, severityFloor}],
    submittedBy}; mints changeId; calls ChangeRequestEntity.submit; returns {changeId}),
    GET /changes (list from getAllChanges, sorted newest-first), GET /changes/{id} (one row),
    GET /changes/{id}/gate (gate status for CI: {changeId, gateStatus, reason}), GET
    /changes/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- PolicyTasks.java declaring one Task<R> constant: ENFORCE_POLICY = Task.name("Enforce
  policy").description("Read the attached change payload and evaluate it against every
  submitted policy rule, returning a PolicyDecision").resultConformsTo(PolicyDecision.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PolicyRule, ChangeRequest, ValidatedChange, Violation, Severity,
  PolicyDecision, Outcome, GateResult, GateStatus, ChangeEvaluation, EvaluationStatus.

- ToolCallGuardrail.java implementing the before-tool-call hook. Maintains an allowlist set
  {lookup-policy-metadata, query-resource-registry}. For each tool call, checks the tool name
  against the allowlist. If blocked, returns Guardrail.reject(<structured-error>) carrying
  the tool name and the blocking reason; the agent receives this and must reason without that
  tool. Allowed calls pass through.

- GateEvaluator.java — pure deterministic logic (no LLM). Inputs: PolicyDecision and the list
  of submitted PolicyRule. Outputs: GateResult. If outcome == DENY or any
  violation.severity == CRITICAL, gate = BLOCKED. Otherwise OPEN. Javadoc on the class
  documents the gate logic.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9701 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The PolicyEnforcementAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/policy-rules.jsonl with 3 seeded policy sets:
  a 5-rule infrastructure-change rulebook, a 6-rule Kubernetes-manifest policy, and a
  4-rule software-supply-chain policy.

- src/main/resources/sample-events/seed-changes.jsonl with 3 paired example change payloads:
  a synthetic Terraform plan diff (infrastructure, ~800 words), a synthetic Kubernetes
  manifest change (~600 words), and a synthetic dependency-version bump (~300 words). Each
  contains 1–2 policy violations so the agent has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, C1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with decisions.authority_level = enforce
  (the agent's DENY decision is authoritative, not advisory), oversight.human_in_loop = false
  (the gate fires automatically), failure.failure_modes including "false-positive-block",
  "missed-violation", "disallowed-tool-data-used", "gate-bypass-on-timeout";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/policy-enforcement-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Policy-as-Code", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of change-evaluation cards; right = selected-evaluation detail with submitted
  policy rules, validated change preview, decision rationale, violation table, and CI gate
  chip). Browser title exactly: <title>Akka Sample: PolicyAsCode</title>. No subtitle on
  the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(changeId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    enforce-policy.json — 8 PolicyDecision entries covering all three Outcome values.
      Each entry has a rationale paragraph and a violations array covering the matched
      rule set (Infrastructure / Kubernetes / Supply-chain). Each Violation has a non-empty
      affectedResource (e.g., "aws_s3_bucket.data-store"), a non-empty evidence string
      (a paraphrase of a passage in the seeded payload), and an actionable recommendation.
      Severities vary realistically. Plus 2 entries that trigger the before-tool-call
      guardrail path — they instruct the agent to call a disallowed tool first; the mock
      guardrail rejects those and the agent's second pass uses only the attachment.
      The mock should select a guardrail-triggering entry on the FIRST iteration of every
      3rd change (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(changeId) helper makes per-change selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. PolicyEnforcementAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion PolicyTasks.java
  MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (enforceStep
  60s, awaitValidatedStep 15s, gateStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the ChangeEvaluation row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: PolicyTasks.java with ENFORCE_POLICY = Task.name(...).description(...)
  .resultConformsTo(PolicyDecision.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9701 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent
  (PolicyEnforcementAgent). The CI gate is rule-based (GateEvaluator.java) and does NOT
  make an LLM call — keeping the pattern's "one agent" promise honest.
- The change payload is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated enforceStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent returns. Lesson 1's AutonomousAgent
  contract is the authoritative reference for how the hook is registered.
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
