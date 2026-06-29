# SPEC — safety-plugins

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SafetyPlugins.
**One-line pitch:** A user submits a text payload (an incoming user message or a candidate model response) and a safety profile; one AI agent reads the payload (passed as a task attachment, never as inline prompt text) and returns a structured safety decision — ALLOW / BLOCK / REDACT with a per-category finding for each configured safety rule.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the governance-risk domain. One `SafetyAgent` (AutonomousAgent) carries the entire classification decision; the surrounding components prepare its input and audit its output. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw payload submission and the agent call — so the model never sees identifiers embedded in the submitted text.
- A **before-llm-call guardrail** inspects the incoming payload for known harmful patterns (prompt-injection signatures, hard-blocked strings, payload length limits) before any LLM call is made. Payloads that match a hard block are rejected immediately without consuming an agent iteration.
- An **after-llm-response guardrail** validates the agent's safety decision on every turn: well-formed JSON, every finding references a real category in the submitted safety profile, every action is in the allowed set. A malformed response triggers a retry inside the same task.
- An **on-decision eval** runs immediately after each `DecisionRecorded` event, scoring the decision on rationale quality: does every BLOCK or REDACT finding cite specific evidence from the payload? Is the severity proportional? Are ALLOW decisions accompanied by a brief justification?

The blueprint shows that a single-agent content-safety system can carry multiple independent governance layers — the guardrails operate before and after the LLM, not as alternatives to it.

## 3. User-facing flows

The user opens the App UI tab.

1. The user pastes a text payload into the **Payload** textarea (or picks one of three seeded examples — a benign user message, a prompt-injection attempt, a response containing PII and policy-violating advice).
2. The user picks a **safety profile** from a dropdown (GENERAL, CHILDREN, ENTERPRISE) or pastes a custom list of safety rules.
3. The user clicks **Submit for screening**. The UI POSTs to `/api/screenings` and receives a `screeningId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted payload is visible in the card detail, with a small list of PII categories the sanitizer found.
5. Within ~10–30 s, the workflow's `screenStep` completes. The card transitions to `SCREENING` then `DECISION_RECORDED`. The decision appears: a top-level action badge (ALLOW / BLOCK / REDACT), a short summary paragraph, and a per-finding table (category id, action, evidence quote, rationale).
6. Within ~1 s of the decision, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the decision's evidence is solid.
7. The user can submit another payload; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SafetyEndpoint` | `HttpEndpoint` | `/api/screenings/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `SafetyEntity`, `SafetyView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SafetyEntity` | `EventSourcedEntity` | Per-screening lifecycle: submitted → sanitized → screening → decision → evaluated. Source of truth. | `SafetyEndpoint`, `PayloadSanitizer`, `SafetyWorkflow` | `SafetyView` |
| `PayloadSanitizer` | `Consumer` | Subscribes to `PayloadSubmitted` events; redacts PII; calls `SafetyEntity.attachSanitized`. | `SafetyEntity` events | `SafetyEntity` |
| `SafetyWorkflow` | `Workflow` | One workflow per screening. Steps: `awaitSanitizedStep` → `screenStep` → `evalStep`. | started by `PayloadSanitizer` once sanitized event lands | `SafetyAgent`, `SafetyEntity` |
| `SafetyAgent` | `AutonomousAgent` | The one decision-making LLM. Receives safety rules in the task definition and the sanitized payload as a task attachment; returns `SafetyDecision`. | invoked by `SafetyWorkflow` | returns decision |
| `InputGuardrail` | supporting class | before-llm-call hook: pattern-match + length checks on the raw payload before the LLM call. | wired on `SafetyAgent` | allows or hard-blocks |
| `OutputGuardrail` | supporting class | after-llm-response hook: structural validation of `SafetyDecision` JSON. | wired on `SafetyAgent` | passes or forces retry |
| `DecisionEvaluator` | supporting class | Rule-based rationale scorer. No LLM call. | invoked by `SafetyWorkflow.evalStep` | returns `EvalResult` |
| `SafetyView` | `View` | Read model: one row per screening for the UI. | `SafetyEntity` events | `SafetyEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SafetyRule(String ruleId, String category, String description, String actionFloor) {}
// category values: HATE_SPEECH, SELF_HARM, SEXUAL_CONTENT, VIOLENCE, PROMPT_INJECTION,
//                  DATA_EXFILTRATION, HARASSMENT, ILLEGAL_GOODS, MISINFORMATION, OTHER

record ScreeningRequest(
    String screeningId,
    String payloadTitle,
    String rawPayload,
    String payloadDirection,  // "user-to-model" or "model-to-user"
    List<SafetyRule> rules,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedPayload(
    String redactedPayload,
    List<String> piiCategoriesFound
) {}

record CategoryFinding(
    String ruleId,
    String category,
    RuleAction action,
    String evidenceQuote,
    String rationale
) {}
enum RuleAction { ALLOW, REDACT, BLOCK }

record SafetyDecision(
    OverallAction overallAction,
    String summary,
    List<CategoryFinding> findings,
    Instant decidedAt
) {}
enum OverallAction { ALLOW, REDACT, BLOCK }

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Screening(
    String screeningId,
    Optional<ScreeningRequest> request,
    Optional<SanitizedPayload> sanitized,
    Optional<SafetyDecision> decision,
    Optional<EvalResult> eval,
    ScreeningStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ScreeningStatus {
    SUBMITTED, SANITIZED, SCREENING, DECISION_RECORDED, EVALUATED, FAILED
}
```

Events on `SafetyEntity`: `PayloadSubmitted`, `PayloadSanitized`, `ScreeningStarted`, `DecisionRecorded`, `EvaluationScored`, `ScreeningFailed`.

Every nullable lifecycle field on the `Screening` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/screenings` — body `{ payloadTitle, rawPayload, payloadDirection, rules: [SafetyRule], submittedBy }` → `{ screeningId }`.
- `GET /api/screenings` — list all screenings, newest-first.
- `GET /api/screenings/{id}` — one screening.
- `GET /api/screenings/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: SafetyPlugins</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted screenings (status pill + action badge + age) and a right pane with the selected screening's detail — submitted safety rules list, sanitized payload preview, decision summary, finding table, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `PayloadSanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from the raw payload before any LLM call. Records which categories were found.
- **G1 — before-llm-call guardrail**: runs on every invocation of `SafetyAgent` before the LLM call is made. Asserts the incoming payload does not exceed the configured maximum token budget, does not contain known prompt-injection signature strings (e.g., `IGNORE PREVIOUS INSTRUCTIONS`, `[SYSTEM]` overrides), and does not contain hard-blocked patterns from the safety profile. A hard-block match terminates the task immediately and writes `ScreeningFailed` with the matching pattern id — no LLM credit consumed.
- **G2 — after-llm-response guardrail**: runs on every turn of `SafetyAgent`. Asserts the candidate response is well-formed `SafetyDecision` JSON, every `findings[].ruleId` matches a submitted rule, every `action` is in `{ALLOW, REDACT, BLOCK}`, and the findings list covers every submitted rule (no silent omissions). On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `DecisionRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call) checks that every BLOCK or REDACT finding has a non-empty `evidenceQuote`, that rationales are non-empty, that an ALLOW decision across all rules has at least one justification sentence, and that the `overallAction` is consistent with the most severe per-finding action. Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `SafetyAgent` → `prompts/safety-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached payload, walk every submitted safety rule, and return one `CategoryFinding` per rule with an evidence quote where a violation was detected.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the benign-message seed with GENERAL profile; within 30 s the decision appears with one finding per submitted rule, all ALLOW, and an eval score chip.
2. **J2** — The agent's first response on a screening is intentionally malformed (mock LLM path) — the `after-llm-response` guardrail rejects it; the second iteration produces a well-formed decision; the UI never displays the malformed response.
3. **J3** — A screening whose BLOCK findings all carry empty `evidenceQuote` strings receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A payload containing `jane.doe@example.com` and `SSN 123-45-6789` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-SSN]`; the entity's `request.rawPayload` retains the raw text for audit.
5. **J5** — A payload containing a known prompt-injection string (`IGNORE PREVIOUS INSTRUCTIONS`) hits the `before-llm-call` guardrail; the workflow records `ScreeningFailed` with `inputGuardrail.hardBlock` as the reason; no LLM call is made; the UI shows the FAILED state immediately.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named safety-plugins demonstrating the single-agent × governance-risk cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-governance-risk-safety-plugins. Java package io.akka.samples.safetyplugins.
Akka 3.6.0. HTTP port 9367.

Components to wire (exactly):

- 1 AutonomousAgent SafetyAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/safety-agent.md>) and
  .capability(TaskAcceptance.of(SCREEN_PAYLOAD).maxIterationsPerTask(3)). The task receives
  safety rules as its instruction text and the sanitized payload as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: SafetyDecision{overallAction: OverallAction (ALLOW/REDACT/BLOCK), summary: String,
  findings: List<CategoryFinding>, decidedAt: Instant}. The agent is configured with an
  InputGuardrail (before-llm-call hook) and an OutputGuardrail (after-llm-response hook)
  registered via the agent's guardrail-configuration block. On OutputGuardrail rejection the
  agent loop retries within its 3-iteration budget. On InputGuardrail hard-block, the task
  terminates immediately with a rejection result; the workflow transitions to error.

- 1 Workflow SafetyWorkflow per screeningId with three steps:
  * awaitSanitizedStep — polls SafetyEntity.getScreening every 1s; on screening.sanitized().isPresent()
    advances to screenStep. WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * screenStep — emits ScreeningStarted, then calls componentClient.forAutonomousAgent(
    SafetyAgent.class, "screener-" + screeningId).runSingleTask(
      TaskDef.instructions(formatRules(screening.request.rules))
        .attachment("payload.txt", screening.sanitized.redactedPayload.getBytes())
    ) — returns a taskId, then forTask(taskId).result(SCREEN_PAYLOAD) to fetch the decision.
    On success calls SafetyEntity.recordDecision(decision). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(SafetyWorkflow::error).
  * evalStep — runs a deterministic rule-based DecisionEvaluator (NOT an LLM call) over the
    recorded decision: checks that every BLOCK/REDACT finding has non-empty evidenceQuote and
    rationale, that ALLOW decisions have at least one justification sentence in the summary,
    and that overallAction is consistent with the most severe per-finding action. Emits
    EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SafetyEntity (one per screeningId). State Screening{screeningId: String,
  request: Optional<ScreeningRequest>, sanitized: Optional<SanitizedPayload>,
  decision: Optional<SafetyDecision>, eval: Optional<EvalResult>, status: ScreeningStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. ScreeningStatus enum: SUBMITTED,
  SANITIZED, SCREENING, DECISION_RECORDED, EVALUATED, FAILED. Events: PayloadSubmitted{request},
  PayloadSanitized{sanitized}, ScreeningStarted{}, DecisionRecorded{decision},
  EvaluationScored{eval}, ScreeningFailed{reason}. Commands: submit, attachSanitized,
  markScreening, recordDecision, recordEvaluation, fail, getScreening. emptyState() returns
  Screening.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer PayloadSanitizer subscribed to SafetyEntity events; on PayloadSubmitted runs
  a regex+heuristic redaction pipeline (emails, phone numbers, SSN-like, payment-card-like,
  postal addresses, person-name heuristic, account-id-like tokens) over rawPayload, computes
  the list of categories found, builds SanitizedPayload, then calls
  SafetyEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer
  starts a SafetyWorkflow with id = "screening-" + screeningId.

- 1 View SafetyView with row type ScreeningRow (mirrors Screening minus request.rawPayload — the
  audit log keeps the raw; the view holds the sanitized form for the UI). Table updater
  consumes SafetyEntity events. ONE query getAllScreenings: SELECT * AS screenings FROM safety_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * SafetyEndpoint at /api with POST /screenings (body
    {payloadTitle, rawPayload, payloadDirection, rules: [{ruleId, category, description, actionFloor}],
    submittedBy}; mints screeningId; calls SafetyEntity.submit; returns {screeningId}), GET /screenings
    (list from getAllScreenings, sorted newest-first), GET /screenings/{id} (one row), GET
    /screenings/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SafetyTasks.java declaring one Task<R> constant: SCREEN_PAYLOAD = Task.name("Screen payload")
  .description("Read the attached payload and produce a SafetyDecision per safety rule")
  .resultConformsTo(SafetyDecision.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records SafetyRule, ScreeningRequest, SanitizedPayload, CategoryFinding, RuleAction,
  SafetyDecision, OverallAction, EvalResult, Screening, ScreeningStatus.

- InputGuardrail.java implementing the before-llm-call hook. Checks (1) payload byte length
  does not exceed configured limit, (2) the redacted payload does not contain known
  prompt-injection signatures, (3) the rule set includes at least one rule. On hard-block,
  returns Guardrail.reject(<blocking-pattern-id>). The list of injection signatures is loaded
  from src/main/resources/safety-config/injection-patterns.txt at startup.

- OutputGuardrail.java implementing the after-llm-response hook. Reads the candidate
  SafetyDecision from the LLM response, runs three checks: (1) every findings[].ruleId matches
  a submitted rule, (2) every action is in {ALLOW, REDACT, BLOCK}, (3) every submitted rule has
  at least one corresponding finding. On failure returns Guardrail.reject(<structured-error>)
  to force the agent loop to retry.

- DecisionEvaluator.java — pure deterministic logic (no LLM). Inputs: SafetyDecision and
  the list of submitted SafetyRules. Outputs: EvalResult. Scoring rubric documented in
  Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9367 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The SafetyAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/safety-profiles.jsonl with 3 seeded safety profiles:
  a GENERAL profile (8 rules covering common categories), a CHILDREN profile (10 rules with
  stricter thresholds), and an ENTERPRISE profile (6 rules focused on data-exfiltration and
  prompt-injection).

- src/main/resources/sample-events/seed-payloads.jsonl with 3 paired example payloads:
  a benign user message (150 words), a prompt-injection attempt (80 words, contains
  IGNORE PREVIOUS INSTRUCTIONS and a [SYSTEM] block), and a candidate model response
  containing PII references and policy-violating medical advice (300 words). Each GENERAL
  payload contains 2–3 plausible PII strings so S1 has work to do.

- src/main/resources/safety-config/injection-patterns.txt with a list of known
  prompt-injection trigger strings (one per line, case-insensitive matching).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 4 controls (S1, G1, G2, E1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = blocking
  (the agent's decision can block content automatically), oversight.human_in_loop = false
  (the safety decision is enforced inline, no human review per item), oversight.human_on_loop
  = true (an operator monitors aggregate metrics and adjusts thresholds), failure.failure_modes
  including "false-positive-block", "false-negative-allow", "pii-leakage-via-llm",
  "guardrail-bypass-via-encoding", "prompt-injection-in-payload"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/safety-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Safety Plugins", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of screening cards; right = selected-screening detail with submitted safety rules,
  sanitized payload preview, decision summary, finding table, and eval-score chip).
  Browser title exactly: <title>Akka Sample: SafetyPlugins</title>. No subtitle on the
  Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(screeningId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    screen-payload.json — 8 SafetyDecision entries covering the three OverallAction values.
      Each entry has a summary paragraph and a `findings` array with one CategoryFinding per
      rule in the matched profile (GENERAL / CHILDREN / ENTERPRISE). Each finding has a
      non-empty evidenceQuote (a paraphrase referring to a passage of the seeded payload) and
      an actionable rationale. Actions vary realistically across ALLOW, REDACT, BLOCK. Plus
      2 deliberately MALFORMED entries (one with a finding whose ruleId is not in the submitted
      list; one with an action value outside the enum) — the OutputGuardrail blocks both,
      exercising the retry path. The mock should select a malformed entry on the FIRST
      iteration of every 3rd screening (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(screeningId) helper makes per-screening selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SafetyAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SafetyTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (screenStep
  60s, awaitSanitizedStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Screening row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: SafetyTasks.java with SCREEN_PAYLOAD = Task.name(...).description(...)
  .resultConformsTo(SafetyDecision.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9367 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (SafetyAgent). The
  InputGuardrail and OutputGuardrail are hooks on that one agent, not separate agents.
  DecisionEvaluator is rule-based and does NOT make an LLM call — keeping the pattern's
  "one agent" promise honest.
- The payload is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated screenStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-llm-call guardrail (InputGuardrail) and after-llm-response guardrail
  (OutputGuardrail) are wired via the agent's guardrail-configuration mechanism, not as
  external checks. Lesson 1's AutonomousAgent contract is the authoritative reference.
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
