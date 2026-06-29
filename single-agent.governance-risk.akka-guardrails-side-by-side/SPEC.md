# SPEC — guardrails-side-by-side

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** GuardrailsSideBySide.
**One-line pitch:** A user submits a prompt; an input-screener agent decides whether the prompt is safe and in-scope before the main agent is called, and an output-validator agent decides whether the main agent's reply meets policy rules before it is returned — mirroring the `input_guardrails` / `output_guardrails` pattern with Akka's native AutonomousAgent hooks.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the governance-risk domain. `PolicyAgent` (AutonomousAgent) is the single decision-making LLM. Two additional AutonomousAgents act purely as guardrails: `InputScreenerAgent` gates the prompt before `PolicyAgent` is ever invoked, and `OutputValidatorAgent` gates the reply before it leaves the system. Both guardrail agents are short-lived — they evaluate and return a structured verdict; they do not carry conversation state.

Three governance controls are wired around the pipeline:

- A **before-agent-invocation guardrail** runs `InputScreenerAgent` synchronously inside `GuardrailWorkflow.inputScreenStep`. If the verdict is BLOCK, the workflow short-circuits and the entity records a `PromptBlocked` event. `PolicyAgent` is never called.
- A **before-agent-response guardrail** runs `OutputValidatorAgent` inside `GuardrailWorkflow.outputValidateStep`. If the verdict is BLOCK, the workflow records `ResponseBlocked`; the caller receives the block reason instead of the model's reply.
- A non-blocking **on-decision eval** scores every PASS prompt-response pair for response quality immediately after the response passes the output guardrail. The evaluator is rule-based — no additional LLM call.

The blueprint shows how `input_guardrails` and `output_guardrails` from the OpenAI Agents SDK map directly to Akka workflow steps that call purpose-built AutonomousAgents before and after the main agent.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a prompt into the **Prompt** textarea (or picks one of three seeded examples — a governance question, an off-topic request, and a prompt whose expected response would violate output policy).
2. The user picks a **Policy profile** from a dropdown (standard / strict / permissive) or accepts the default.
3. The user clicks **Submit**. The UI POSTs to `/api/prompts` and receives a `promptId`.
4. The card appears in the live list in `RECEIVED` state. Within ~1 s, it transitions to `SCREENING` as the input-screener agent runs.
5. If the input screener blocks the prompt: the card transitions to `BLOCKED` and shows the screener's reason. `PolicyAgent` is never invoked.
6. If the input screener passes: the card transitions to `AGENT_RUNNING` within ~1 s. Within ~10–30 s, `PolicyAgent` returns its reply and the card transitions to `VALIDATING` as the output validator runs.
7. If the output validator blocks the reply: the card transitions to `RESPONSE_BLOCKED` and shows the validator's reason; the agent's raw reply is never displayed.
8. If the output validator passes: the card transitions to `RESPONDED` and displays the agent's reply. Within ~1 s, an eval score chip appears (1–5).
9. The user can submit another prompt; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PromptEndpoint` | `HttpEndpoint` | `/api/prompts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `PromptEntity`, `PromptView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PromptEntity` | `EventSourcedEntity` | Per-prompt lifecycle: received → screening → agent_running → validating → responded / blocked / response_blocked. Source of truth. | `PromptEndpoint`, `GuardrailWorkflow` | `PromptView` |
| `GuardrailWorkflow` | `Workflow` | One workflow per prompt. Steps: `inputScreenStep` → `agentCallStep` → `outputValidateStep` → `evalStep`. Short-circuits to `blockStep` if either guardrail returns BLOCK. | started by `PromptEndpoint` after `PromptEntity.receive` | `InputScreenerAgent`, `PolicyAgent`, `OutputValidatorAgent`, `PromptEntity` |
| `InputScreenerAgent` | `AutonomousAgent` | Evaluates the user prompt against scope and safety rules. Returns `ScreeningVerdict{result: PASS|BLOCK, reason}`. | invoked by `GuardrailWorkflow.inputScreenStep` | returns verdict |
| `PolicyAgent` | `AutonomousAgent` | The main decision-making LLM. Receives the user prompt and returns a `PolicyResponse{reply, confidence}`. | invoked by `GuardrailWorkflow.agentCallStep` | returns response |
| `OutputValidatorAgent` | `AutonomousAgent` | Evaluates `PolicyAgent`'s reply against output policy rules. Returns `ValidationVerdict{result: PASS|BLOCK, reason, triggeredRule}`. | invoked by `GuardrailWorkflow.outputValidateStep` | returns verdict |
| `ResponseEvaluator` | supporting class (no LLM) | Rule-based scorer; runs in `evalStep`. Scores the passed response 1–5 on relevance, completeness, and citation of evidence. | invoked by `GuardrailWorkflow.evalStep` | returns `EvalResult` |
| `PromptView` | `View` | Read model: one row per prompt for the UI. | `PromptEntity` events | `PromptEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PromptRequest(
    String promptId,
    String promptText,
    String policyProfile,   // "standard" | "strict" | "permissive"
    String submittedBy,
    Instant receivedAt
) {}

record ScreeningVerdict(
    ScreeningResult result,       // PASS | BLOCK
    String reason,
    Instant decidedAt
) {}
enum ScreeningResult { PASS, BLOCK }

record PolicyResponse(
    String reply,
    double confidence,            // 0.0 – 1.0
    Instant respondedAt
) {}

record ValidationVerdict(
    ValidationResult result,      // PASS | BLOCK
    String reason,
    String triggeredRule,         // null if PASS
    Instant decidedAt
) {}
enum ValidationResult { PASS, BLOCK }

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Prompt(
    String promptId,
    Optional<PromptRequest> request,
    Optional<ScreeningVerdict> screeningVerdict,
    Optional<PolicyResponse> agentResponse,
    Optional<ValidationVerdict> validationVerdict,
    Optional<EvalResult> eval,
    PromptStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PromptStatus {
    RECEIVED, SCREENING, AGENT_RUNNING, VALIDATING,
    RESPONDED, BLOCKED, RESPONSE_BLOCKED, FAILED
}
```

Events on `PromptEntity`: `PromptReceived`, `ScreeningStarted`, `PromptBlocked`, `AgentCallStarted`, `AgentResponded`, `ValidationStarted`, `ResponseBlocked`, `ResponseAccepted`, `EvaluationScored`, `PromptFailed`.

Every nullable lifecycle field on the `Prompt` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/prompts` — body `{ promptText, policyProfile, submittedBy }` → `{ promptId }`.
- `GET /api/prompts` — list all prompts, newest-first.
- `GET /api/prompts/{id}` — one prompt.
- `GET /api/prompts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: GuardrailsSideBySide</title>`.

The App UI tab is a two-column layout: a left column with the prompt submission form and the live list of submitted prompts (status pill + decision badge + age) and a right pane with the selected prompt's detail — prompt text, screening verdict, agent reply (or block reason), validation verdict, and eval score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-invocation guardrail**: `InputScreenerAgent` runs inside `GuardrailWorkflow.inputScreenStep` before `PolicyAgent` is ever called. Checks: (1) prompt is not empty, (2) prompt is in scope for governance-risk questions (not a request to generate code, write content, or perform an action unrelated to the governance domain), (3) prompt does not contain injection-pattern tokens (system-prompt override attempts, role-switching instructions), (4) prompt does not solicit harmful content. On BLOCK, `GuardrailWorkflow` short-circuits to `blockStep` and writes `PromptBlocked` to `PromptEntity`. `PolicyAgent` is never invoked.
- **G2 — before-agent-response guardrail**: `OutputValidatorAgent` runs inside `GuardrailWorkflow.outputValidateStep` after `PolicyAgent` returns. Checks: (1) reply is not empty, (2) reply does not claim capabilities or make commitments beyond the system's advisory role, (3) reply does not contain personal data about third parties, (4) reply does not contradict stated policy rules. On BLOCK, writes `ResponseBlocked` to `PromptEntity`; the caller receives `validationVerdict.reason` rather than the agent's reply.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `ResponseAccepted` lands, as `evalStep` inside the workflow. A rule-based `ResponseEvaluator` (no LLM call — keeping the single-agent invariant honest) checks: reply length is between 20 and 1000 words, reply references at least one specific governance concept from the prompt, and reply does not hedge excessively (fewer than 30% of sentences begin with "it depends" or "may"). Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `InputScreenerAgent` → `prompts/input-screener.md`. Screens incoming prompts for scope, safety, and injection risk.
- `PolicyAgent` → `prompts/policy-agent.md`. Main governance advisor. Answers governance-risk questions with citations and clear recommendations.
- `OutputValidatorAgent` → `prompts/output-validator.md`. Validates `PolicyAgent`'s reply against output policy rules before it is returned.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a seeded governance question with `policyProfile = standard`; within 45 s the response appears with screening verdict PASS, validation verdict PASS, and an eval score chip.
2. **J2** — User submits the seeded off-topic prompt; `InputScreenerAgent` returns BLOCK; the card shows `BLOCKED` with the screener's reason; `PolicyAgent` is never invoked (no `AgentCallStarted` event on the entity).
3. **J3** — User submits the seeded policy-violating-response prompt (one whose expected answer would trigger the output validator); `OutputValidatorAgent` returns BLOCK; the card shows `RESPONSE_BLOCKED`; the agent's raw reply is never displayed.
4. **J4** — A response whose reply is below 20 words receives an eval score of 2 with a rationale citing brevity; the UI flags the card.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named guardrails-side-by-side demonstrating the single-agent ×
governance-risk cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact single-agent-governance-risk-akka-guardrails-side-by-side.
Java package io.akka.samples.openaiagentsinputoutputguardrails. Akka 3.6.0. HTTP port 9255.

Components to wire (exactly):

- 3 AutonomousAgents:
  * InputScreenerAgent — definition() returns AgentDefinition with
    .instructions(<system prompt loaded from prompts/input-screener.md>) and
    .capability(TaskAcceptance.of(GuardrailTasks.SCREEN_INPUT).maxIterationsPerTask(1)).
    Output: ScreeningVerdict{result: ScreeningResult (PASS/BLOCK), reason: String, decidedAt: Instant}.
    Single-iteration only — the input screener makes one decision and returns.
  * PolicyAgent — definition() returns AgentDefinition with
    .instructions(<system prompt loaded from prompts/policy-agent.md>) and
    .capability(TaskAcceptance.of(GuardrailTasks.ANSWER_PROMPT).maxIterationsPerTask(3)).
    Output: PolicyResponse{reply: String, confidence: double, respondedAt: Instant}.
  * OutputValidatorAgent — definition() returns AgentDefinition with
    .instructions(<system prompt loaded from prompts/output-validator.md>) and
    .capability(TaskAcceptance.of(GuardrailTasks.VALIDATE_OUTPUT).maxIterationsPerTask(1)).
    Output: ValidationVerdict{result: ValidationResult (PASS/BLOCK), reason: String,
    triggeredRule: String (nullable), decidedAt: Instant}.
    Single-iteration only — the output validator makes one decision and returns.

- 1 Workflow GuardrailWorkflow per promptId with five steps:
  * inputScreenStep — calls componentClient.forAutonomousAgent(InputScreenerAgent.class,
    "screener-" + promptId).runSingleTask(TaskDef.instructions(prompt.promptText)). On
    BLOCK verdict writes PromptBlocked via PromptEntity.recordBlock, then transitions to
    blockStep. On PASS writes ScreeningStarted then transitions to agentCallStep.
    WorkflowSettings.stepTimeout 30s.
  * agentCallStep — emits AgentCallStarted, then calls componentClient.forAutonomousAgent(
    PolicyAgent.class, "policy-" + promptId).runSingleTask(
      TaskDef.instructions(prompt.promptText).context("policyProfile", prompt.policyProfile)
    ). On success records AgentResponded and transitions to outputValidateStep.
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2).failoverTo(GuardrailWorkflow::error).
  * outputValidateStep — calls componentClient.forAutonomousAgent(OutputValidatorAgent.class,
    "validator-" + promptId).runSingleTask(
      TaskDef.instructions(agentResponse.reply).context("originalPrompt", prompt.promptText)
    ). On BLOCK writes ResponseBlocked and transitions to blockStep. On PASS writes
    ValidationStarted then ResponseAccepted then transitions to evalStep.
    WorkflowSettings.stepTimeout 30s.
  * evalStep — runs a deterministic rule-based ResponseEvaluator (NOT an LLM call) over
    the PolicyResponse: checks word count (20–1000), governance-concept reference, and
    hedging ratio. Emits EvaluationScored{score: 1-5, rationale: String}.
    WorkflowSettings.stepTimeout 5s.
  * blockStep / error step — transitions entity to appropriate terminal state (BLOCKED,
    RESPONSE_BLOCKED, or FAILED). WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity PromptEntity (one per promptId). State Prompt{promptId: String,
  request: Optional<PromptRequest>, screeningVerdict: Optional<ScreeningVerdict>,
  agentResponse: Optional<PolicyResponse>, validationVerdict: Optional<ValidationVerdict>,
  eval: Optional<EvalResult>, status: PromptStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. PromptStatus enum: RECEIVED, SCREENING, AGENT_RUNNING,
  VALIDATING, RESPONDED, BLOCKED, RESPONSE_BLOCKED, FAILED.
  Events: PromptReceived{request}, ScreeningStarted{}, PromptBlocked{screeningVerdict},
  AgentCallStarted{}, AgentResponded{agentResponse}, ValidationStarted{},
  ResponseBlocked{validationVerdict}, ResponseAccepted{validationVerdict},
  EvaluationScored{eval}, PromptFailed{reason}.
  Commands: receive, startScreening, recordBlock, startAgentCall, recordAgentResponse,
  startValidation, recordResponseBlock, acceptResponse, recordEvaluation, fail, getPrompt.
  emptyState() returns Prompt.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 View PromptView with row type PromptRow (mirrors Prompt; includes all Optional lifecycle
  fields). Table updater consumes PromptEntity events. ONE query getAllPrompts: SELECT * AS
  prompts FROM prompt_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * PromptEndpoint at /api with POST /prompts (body {promptText, policyProfile, submittedBy};
    mints promptId; calls PromptEntity.receive; starts GuardrailWorkflow; returns {promptId}),
    GET /prompts (list from getAllPrompts, sorted newest-first), GET /prompts/{id} (one row),
    GET /prompts/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- GuardrailTasks.java declaring three Task<R> constants:
    SCREEN_INPUT = Task.name("Screen input").description("Evaluate the prompt for scope, safety, and injection risk").resultConformsTo(ScreeningVerdict.class)
    ANSWER_PROMPT = Task.name("Answer prompt").description("Produce a governance-risk advisory response to the prompt").resultConformsTo(PolicyResponse.class)
    VALIDATE_OUTPUT = Task.name("Validate output").description("Check the agent reply against output policy rules").resultConformsTo(ValidationVerdict.class)
  DO NOT skip this — every AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PromptRequest, ScreeningVerdict, ScreeningResult, PolicyResponse,
  ValidationVerdict, ValidationResult, EvalResult, Prompt, PromptStatus.

- ResponseEvaluator.java — pure deterministic logic (no LLM). Inputs: PolicyResponse and
  PromptRequest. Outputs: EvalResult. Scoring rubric: word count in range (1 point),
  governance-concept reference present (1 point), hedging ratio below 30% (1 point),
  confidence >= 0.6 (1 point), reply has at least one sentence ending in a clear
  recommendation (1 point). Scoring rubric documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9255 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Each agent's definition() binds the
  configured provider via .modelProvider("${akka.javasdk.agent.default}") or the per-agent
  override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-prompts.jsonl with 3 seeded prompt scenarios:
  (1) a well-formed governance question about AI risk categorisation (expected: PASS/PASS/score≥3),
  (2) an off-topic prompt (request to write a poem — expected: BLOCK at input),
  (3) a prompt whose straightforward answer would encourage policy non-compliance — expected:
  PASS input, BLOCK output.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, G2, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root pre-filled for governance-risk domain.

- prompts/input-screener.md, prompts/policy-agent.md, prompts/output-validator.md loaded as
  agent system prompts.

- README.md at the project root: title "Akka Sample: GuardrailsSideBySide", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission form + live list of prompt cards; right = selected-prompt detail with prompt
  text, screening verdict, agent reply or block reason, validation verdict, and eval chip).
  Browser title exactly: <title>Akka Sample: GuardrailsSideBySide</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(promptId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    screen-input.json — 5 ScreeningVerdict entries: 3 PASS, 2 BLOCK (with realistic reasons).
    answer-prompt.json — 6 PolicyResponse entries with varying replies and confidence values.
    validate-output.json — 5 ValidationVerdict entries: 3 PASS, 2 BLOCK (with triggeredRule).
  Plus mock selection ensures seed prompt 2 (off-topic) always picks a BLOCK screener verdict
  and seed prompt 3 always picks a PASS screener + BLOCK output validator.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: all three AutonomousAgents (InputScreenerAgent, PolicyAgent, OutputValidatorAgent)
  extend akka.javasdk.agent.autonomous.AutonomousAgent. The companion GuardrailTasks.java MUST
  exist with all three Task constants. None may be silently downgraded to Agent.
- Lesson 4: every workflow step has an explicit stepTimeout: inputScreenStep 30s,
  agentCallStep 60s, outputValidateStep 30s, evalStep 5s, blockStep 5s, error 5s.
- Lesson 6: every nullable lifecycle field on the Prompt row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: GuardrailTasks.java with all three Task constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9255 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words anywhere in user-facing prose or generated comments.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE agent (PolicyAgent) answers the user's prompt.
  InputScreenerAgent and OutputValidatorAgent are guardrail-only — they evaluate and return
  a structured verdict; they do not produce end-user answers. The on-decision eval is
  rule-based (ResponseEvaluator.java) and does NOT make an LLM call.
- InputScreenerAgent is invoked as a task in inputScreenStep — it is a before-agent-invocation
  guardrail in the workflow's control flow sense. It does not use the Akka
  before-agent-response hook on PolicyAgent itself; instead it is a separate agent called
  BEFORE PolicyAgent is started, which is the idiomatic Akka mapping of input_guardrails.
- OutputValidatorAgent is invoked as a task in outputValidateStep — it is a
  before-agent-response guardrail in the workflow's control flow sense, called AFTER
  PolicyAgent returns but BEFORE the response is written to the entity as accepted. This
  is the idiomatic Akka mapping of output_guardrails.
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
