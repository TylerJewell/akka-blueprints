# SPEC — guardrails-baseline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** GuardrailsBaseline.
**One-line pitch:** A user submits a text message and a named policy set; one AI agent evaluates the message against the policy and returns a structured moderation decision — ALLOW / BLOCK / ESCALATE with a reason per policy rule — while three governance controls prevent PII leakage, unauthorized tool use, and malformed outputs.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the governance-risk domain. One `ModerationAgent` (AutonomousAgent) carries the entire moderation decision; the surrounding components prepare its input and enforce its output contract. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw message submission and the agent call — the model never sees identifiers.
- A **before-agent-invocation guardrail** checks that every tool the agent intends to invoke is on the policy's allowed-tool list before the first LLM call on any task. Out-of-policy tool use is blocked at the invocation boundary.
- An **after-llm-response guardrail** validates the agent's decision on every turn: well-formed JSON, every rule-result references a known rule id, every action is in the allowed set, and the overall verdict is consistent with the individual rule results. A malformed decision triggers a retry inside the same task.

The blueprint shows that three independent controls can sit around one agent without the agent being aware of any of them. Each control addresses a distinct risk surface.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a message into the **Message** textarea (or picks one of three seeded examples — a synthetic financial-advice request, a synthetic healthcare-guidance inquiry, and a synthetic user-support escalation).
2. The user picks a **policy set** from a dropdown (finance, healthcare, support) or pastes a custom list of policy rules.
3. The user clicks **Submit for moderation**. The UI POSTs to `/api/moderations` and receives a `moderationId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted message is visible in the card detail, with a small list of PII categories the sanitizer found.
5. Within ~10–30 s, the workflow's `moderateStep` completes. The card transitions to `MODERATING` then `DECISION_RECORDED`. The decision appears: a top-level verdict badge (ALLOW / BLOCK / ESCALATE), a short rationale paragraph, and a per-rule table (rule id, action, confidence, explanation).
6. Within ~1 s of the decision, the `auditStep` finishes. The card shows an **audit score** chip (1–5) plus a one-line note describing whether the decision's reasoning is consistent.
7. The user can submit another message; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ModerationEndpoint` | `HttpEndpoint` | `/api/moderations/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ModerationEntity`, `ModerationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ModerationEntity` | `EventSourcedEntity` | Per-moderation lifecycle: submitted → sanitized → moderating → decision → audited. Source of truth. | `ModerationEndpoint`, `MessageSanitizer`, `ModerationWorkflow` | `ModerationView` |
| `MessageSanitizer` | `Consumer` | Subscribes to `MessageSubmitted` events; strips PII; calls `ModerationEntity.attachSanitized`. | `ModerationEntity` events | `ModerationEntity` |
| `ModerationWorkflow` | `Workflow` | One workflow per moderation. Steps: `awaitSanitizedStep` → `moderateStep` → `auditStep`. | started by `MessageSanitizer` once sanitized event lands | `ModerationAgent`, `ModerationEntity` |
| `ModerationAgent` | `AutonomousAgent` | The one decision-making LLM. Receives policy rules in the task definition and the sanitized message as a task attachment; returns `ModerationDecision`. | invoked by `ModerationWorkflow` | returns decision |
| `InputGuardrail` | supporting class | Registered on `ModerationAgent` via `before-agent-invocation` hook; enforces the tool-policy allowlist. | `ModerationAgent` | — |
| `OutputGuardrail` | supporting class | Registered on `ModerationAgent` via `after-llm-response` hook; validates decision structure before it leaves the agent loop. | `ModerationAgent` | — |
| `AuditScorer` | supporting class | Deterministic rule-based scorer; runs inside `auditStep`. Not an LLM call. | invoked by `ModerationWorkflow` | returns `AuditResult` |
| `ModerationView` | `View` | Read model: one row per moderation for the UI. | `ModerationEntity` events | `ModerationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PolicyRule(String ruleId, String description, String category) {}

record ModerationRequest(
    String moderationId,
    String messageTitle,
    String rawMessage,
    List<PolicyRule> rules,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedMessage(
    String redactedMessage,
    List<String> piiCategoriesFound
) {}

record RuleResult(
    String ruleId,
    RuleAction action,
    double confidence,    // 0.0..1.0
    String explanation
) {}
enum RuleAction { ALLOW, BLOCK, ESCALATE }

record ModerationDecision(
    Verdict verdict,
    String rationale,
    List<RuleResult> ruleResults,
    Instant decidedAt
) {}
enum Verdict { ALLOW, BLOCK, ESCALATE }

record AuditResult(
    int score,            // 1..5
    String note,
    Instant auditedAt
) {}

record Moderation(
    String moderationId,
    Optional<ModerationRequest> request,
    Optional<SanitizedMessage> sanitized,
    Optional<ModerationDecision> decision,
    Optional<AuditResult> audit,
    ModerationStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ModerationStatus {
    SUBMITTED, SANITIZED, MODERATING, DECISION_RECORDED, AUDITED, FAILED
}
```

Events on `ModerationEntity`: `MessageSubmitted`, `MessageSanitized`, `ModerationStarted`, `DecisionRecorded`, `AuditCompleted`, `ModerationFailed`.

Every nullable lifecycle field on the `Moderation` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/moderations` — body `{ messageTitle, rawMessage, rules: [PolicyRule], submittedBy }` → `{ moderationId }`.
- `GET /api/moderations` — list all moderations, newest-first.
- `GET /api/moderations/{id}` — one moderation.
- `GET /api/moderations/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Guardrails Baseline</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted moderations (status pill + verdict badge + age) and a right pane with the selected moderation's detail — submitted policy rules list, sanitized message preview, decision rationale, rule-result table, and audit-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MessageSanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from the raw message before any LLM call. Records which categories were found.
- **G1 — before-agent-invocation guardrail**: runs at the start of every task on `ModerationAgent`. Checks the agent's planned tool invocations against the policy's tool-allowlist. Any tool not on the allowlist triggers a structured rejection before the LLM call is made; the agent loop retries without the blocked tool within its iteration budget.
- **G2 — after-llm-response guardrail**: runs on every turn of `ModerationAgent`. Asserts the candidate response is well-formed `ModerationDecision` JSON, every `ruleResults[].ruleId` matches a submitted rule, every `action` is in `{ALLOW, BLOCK, ESCALATE}`, `ruleResults` covers every submitted rule (no silent omissions), and the overall `verdict` is consistent with the rule results (if any rule is BLOCK, verdict must not be ALLOW). On failure returns a structured `invalid-response` error so the agent loop retries within its iteration budget.
- **AuditScorer** — deterministic, rule-based (no LLM call); runs inside `auditStep` immediately after `DecisionRecorded`. Checks that every `RuleResult` has a non-empty explanation, that confidence values are in `[0.0, 1.0]`, and that the verdict is consistent with the highest-severity rule result. Emits `AuditCompleted` with a 1–5 score.

## 9. Agent prompts

- `ModerationAgent` → `prompts/moderation-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached message, walk every policy rule, and return one `RuleResult` per rule with an explanation.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the finance-policy seed message; within 30 s the decision appears with one rule result per submitted policy rule and an audit score chip.
2. **J2** — The agent's task attempts to invoke a forbidden tool (mock LLM path) — the `before-agent-invocation` guardrail rejects it; the second iteration produces a decision without the forbidden tool; the UI never displays a failed moderation card for this input.
3. **J3** — The agent's first response on a moderation is intentionally malformed (mock LLM path) — the `after-llm-response` guardrail rejects it; the second iteration produces a well-formed decision; the UI shows the final decision only.
4. **J4** — A message containing `john.smith@example.com` and a phone number is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`; the entity's `request.rawMessage` retains the raw text for audit.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named guardrails-baseline demonstrating the single-agent × governance-risk
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-governance-risk-guardrails-baseline. Java package io.akka.samples.guardrails.
Akka 3.6.0. HTTP port 9680.

Components to wire (exactly):

- 1 AutonomousAgent ModerationAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/moderation-agent.md>) and
  .capability(TaskAcceptance.of(MODERATE_MESSAGE).maxIterationsPerTask(3)). The task receives
  policy rules as its instruction text and the sanitized message as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: ModerationDecision{verdict: Verdict (ALLOW/BLOCK/ESCALATE), rationale: String,
  ruleResults: List<RuleResult>, decidedAt: Instant}. The agent is configured with:
    (a) a before-agent-invocation guardrail (see G1 in eval-matrix.yaml) that blocks any tool
        not on the policy's allowlist — registered via the agent's guardrail-configuration block
        for the before-agent-invocation hook.
    (b) an after-llm-response guardrail (see G2 in eval-matrix.yaml) that validates response
        structure — registered via the agent's guardrail-configuration block for the
        after-llm-response hook.
  On guardrail rejection the agent loop retries within its 3-iteration budget.

- 1 Workflow ModerationWorkflow per moderationId with three steps:
  * awaitSanitizedStep — polls ModerationEntity.getModeration every 1s; on
    moderation.sanitized().isPresent() advances to moderateStep.
    WorkflowSettings.stepTimeout 15s.
  * moderateStep — emits ModerationStarted, then calls componentClient.forAutonomousAgent(
    ModerationAgent.class, "moderator-" + moderationId).runSingleTask(
      TaskDef.instructions(formatRules(moderation.request.rules))
        .attachment("message.txt", moderation.sanitized.redactedMessage.getBytes())
    ) — returns a taskId, then forTask(taskId).result(MODERATE_MESSAGE) to fetch the decision.
    On success calls ModerationEntity.recordDecision(decision).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ModerationWorkflow::error).
  * auditStep — runs deterministic rule-based AuditScorer (NOT an LLM call) over the
    recorded decision: checks every RuleResult has non-empty explanation, confidence is in
    [0.0, 1.0], and verdict is consistent with rule results. Emits
    AuditCompleted{score: 1-5, note: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ModerationEntity (one per moderationId). State
  Moderation{moderationId: String, request: Optional<ModerationRequest>,
  sanitized: Optional<SanitizedMessage>, decision: Optional<ModerationDecision>,
  audit: Optional<AuditResult>, status: ModerationStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. ModerationStatus enum: SUBMITTED, SANITIZED,
  MODERATING, DECISION_RECORDED, AUDITED, FAILED. Events: MessageSubmitted{request},
  MessageSanitized{sanitized}, ModerationStarted{}, DecisionRecorded{decision},
  AuditCompleted{audit}, ModerationFailed{reason}. Commands: submit, attachSanitized,
  markModerating, recordDecision, recordAudit, fail, getModeration. emptyState() returns
  Moderation.initial("") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer MessageSanitizer subscribed to ModerationEntity events; on MessageSubmitted
  runs a regex+heuristic redaction pipeline (emails, phone numbers, SSN-like,
  payment-card-like, postal addresses, person-name heuristic, account-id-like tokens)
  over rawMessage, computes the list of categories found, builds SanitizedMessage, then
  calls ModerationEntity.attachSanitized(sanitized). After attachSanitized lands, the
  same Consumer starts a ModerationWorkflow with id = "moderation-" + moderationId.

- 1 View ModerationView with row type ModerationRow (mirrors Moderation minus
  request.rawMessage — the audit log keeps the raw; the view holds the sanitized form
  for the UI). Table updater consumes ModerationEntity events. ONE query
  getAllModerations: SELECT * AS moderations FROM moderation_view. No WHERE status
  filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ModerationEndpoint at /api with POST /moderations (body
    {messageTitle, rawMessage, rules: [{ruleId, description, category}], submittedBy};
    mints moderationId; calls ModerationEntity.submit; returns {moderationId}), GET /moderations
    (list from getAllModerations, sorted newest-first), GET /moderations/{id} (one row),
    GET /moderations/sse (Server-Sent Events forwarded from the view's stream-updates), and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ModerationTasks.java declaring one Task<R> constant: MODERATE_MESSAGE =
  Task.name("Moderate message").description("Evaluate the attached message against each
  policy rule and produce a ModerationDecision").resultConformsTo(ModerationDecision.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PolicyRule, ModerationRequest, SanitizedMessage, RuleResult, RuleAction,
  ModerationDecision, Verdict, AuditResult, Moderation, ModerationStatus.

- InputGuardrail.java implementing the before-agent-invocation hook. Reads the agent's
  planned tool list, checks each tool name against the policy's ALLOWED_TOOLS set, and
  either passes through or returns Guardrail.reject(<structured-error>) naming the
  forbidden tool. Registered on ModerationAgent via the before-agent-invocation hook.

- OutputGuardrail.java implementing the after-llm-response hook. Reads the candidate
  ModerationDecision from the LLM response, runs the four checks listed in eval-matrix.yaml
  G2, and either passes the response through or returns Guardrail.reject(<structured-error>)
  to force the agent loop to retry. Registered on ModerationAgent via the after-llm-response
  hook.

- AuditScorer.java — pure deterministic logic (no LLM). Inputs: ModerationDecision and the
  list of submitted PolicyRule. Outputs: AuditResult. Scoring rubric documented in Javadoc.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9680 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/messages.jsonl with 3 seeded message+policy examples:
  a synthetic financial-advice request (4 policy rules), a synthetic healthcare-guidance
  inquiry (5 policy rules), and a synthetic user-support escalation (3 policy rules). Each
  message contains 2–3 plausible PII strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the metadata endpoints to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, G2) matching Section 8.
  Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root as described in Section 8.

- prompts/moderation-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Guardrails Baseline", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of moderation cards; right = selected-moderation detail with submitted rules,
  sanitized message preview, decision rationale, rule-result table, and audit-score chip).
  Browser title exactly: <title>Akka Sample: Guardrails Baseline</title>. No subtitle on
  the Overview tab.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with per-task
  dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly per
  call (seedFor(moderationId)), and deserialises into the task's typed return.
- moderate-message.json — 8 ModerationDecision entries covering the three Verdict values.
  Each entry has a rationale paragraph and a ruleResults array with one RuleResult per
  rule in the matched policy set (finance / healthcare / support). Each RuleResult has a
  non-empty explanation and a confidence in [0.0, 1.0]. Verdicts vary. Plus 2 deliberately
  MALFORMED entries: one where a ruleResult.ruleId is not in the submitted list; one where
  a verdict is ALLOW despite a BLOCK rule result — both exercising guardrail G2. The mock
  selects a malformed entry on the FIRST iteration of every 3rd moderation (modulo seed).
  One entry additionally contains a forbidden-tool reference for the G1 path (J2).
- A MockModelProvider.seedFor(moderationId) helper makes selection deterministic across
  restarts.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key (options a–e as in the canonical
  pattern). NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ModerationAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ModerationTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (moderateStep 60s,
  awaitSanitizedStep 15s, auditStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on Moderation is Optional<T>.
- Lesson 7: ModerationTasks.java with MODERATE_MESSAGE is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9680 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute; DOM contains
  exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (ModerationAgent). AuditScorer
  is rule-based and does NOT make an LLM call.
- The message is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated moderateStep uses TaskDef.attachment(...).
- Both guardrails are wired via the agent's guardrail-configuration mechanism, not as
  external post-hoc checks.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
