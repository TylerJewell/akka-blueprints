# SPEC — guardrails-pattern

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** GuardrailsPattern.
**One-line pitch:** A user submits a free-form prompt; one AI agent processes it and returns a typed structured reply — after an input guardrail screens the prompt and an output guardrail validates the response, both operating without the model ever seeing a rejected request or emitting a rejected reply.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `AssistantAgent` (AutonomousAgent) produces the entire reply; the surrounding components exist solely to guard its input and output. Two guardrail hooks are wired around the agent:

- A **before-llm-call guardrail** (`PromptGuardrail`) runs inside the agent before the prompt is forwarded to the model. It checks the incoming text against a configurable policy vocabulary — blocked topics, disallowed instruction patterns, length limits. A matched violation blocks the call entirely; the agent immediately returns a `BLOCKED` outcome without consuming a model token.
- A **before-agent-response guardrail** (`ReplyGuardrail`) runs on every candidate reply the model produces. It verifies that the reply parses into the expected `AgentReply` structure, that mandatory fields are populated, and that the reply text does not introduce topic violations the input guardrail missed (e.g., a model spontaneously producing disallowed content). A failed check forces a retry within the task's iteration budget.

The blueprint shows the full two-sided guardrail pattern: one gate before the model, one gate after. Together they enforce that (a) the model is never given a non-compliant prompt and (b) the caller is never given a non-compliant response.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a prompt into the **Prompt** textarea (or picks one of three seeded examples — a general knowledge question, a creative writing seed, a structured data extraction request).
2. The user picks a **policy profile** from a dropdown (General / Conservative / Strict) or leaves it at the default. Each profile selects a different blocked-topic vocabulary.
3. The user clicks **Submit**. The UI POSTs to `/api/interactions` and receives an `interactionId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `PROMPT_CHECKED` — the prompt guardrail result is visible (PASSED or BLOCKED). If BLOCKED, the card stays in `BLOCKED` state and shows the matched rule.
5. Within ~5–30 s (clean prompt), the workflow's `agentStep` completes. The card transitions to `REPLYING` then `REPLY_RECORDED`. The structured reply appears: a `category` chip, a `confidence` bar, the `text` body, and a `citations` list.
6. Both guardrail outcomes are visible in a compact timeline at the bottom of the detail pane — input check result, output check result.
7. The user can submit another prompt; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `InteractionEndpoint` | `HttpEndpoint` | `/api/interactions/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `InteractionEntity`, `InteractionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `InteractionEntity` | `EventSourcedEntity` | Per-interaction lifecycle: submitted → prompt-checked → replying → reply-recorded → failed / blocked. Source of truth. | `InteractionEndpoint`, `InteractionWorkflow` | `InteractionView` |
| `InteractionWorkflow` | `Workflow` | One workflow per interaction. Steps: `promptGuardStep` → `agentStep` → `replyGuardStep`. | started by `InteractionEndpoint` after `InteractionEntity.submit` | `AssistantAgent`, `InteractionEntity` |
| `AssistantAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user prompt and policy profile as task instructions; returns `AgentReply`. The `PromptGuardrail` (before-llm-call) and `ReplyGuardrail` (before-agent-response) are both registered on this agent. | invoked by `InteractionWorkflow` | returns reply |
| `PromptGuardrail` | supporting class | Implements the `before-llm-call` hook. Checks prompt against the selected policy profile's vocabulary. Returns `PASS` or `BLOCK(ruleId)`. | registered on `AssistantAgent` | blocks or allows the LLM call |
| `ReplyGuardrail` | supporting class | Implements the `before-agent-response` hook. Validates `AgentReply` structure and topic compliance. Returns `PASS` or `REJECT(reason)`. | registered on `AssistantAgent` | forces retry or passes reply |
| `InteractionView` | `View` | Read model: one row per interaction for the UI. | `InteractionEntity` events | `InteractionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum PolicyProfile { GENERAL, CONSERVATIVE, STRICT }

record PromptRequest(
    String interactionId,
    String promptText,
    PolicyProfile policyProfile,
    String submittedBy,
    Instant submittedAt
) {}

record GuardrailOutcome(
    boolean passed,
    String ruleId,       // null when passed
    String reason,       // null when passed
    Instant checkedAt
) {}

enum ReplyCategory { FACTUAL, CREATIVE, EXTRACTION, REFUSAL, OTHER }

record AgentReply(
    ReplyCategory category,
    String text,
    double confidence,   // 0.0..1.0
    List<String> citations,
    Instant repliedAt
) {}

record Interaction(
    String interactionId,
    Optional<PromptRequest> request,
    Optional<GuardrailOutcome> inputGuardResult,
    Optional<AgentReply> reply,
    Optional<GuardrailOutcome> outputGuardResult,
    InteractionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum InteractionStatus {
    SUBMITTED, PROMPT_CHECKED, BLOCKED, REPLYING, REPLY_RECORDED, FAILED
}
```

Events on `InteractionEntity`: `PromptSubmitted`, `PromptGuardChecked`, `PromptBlocked`, `ReplyingStarted`, `ReplyRecorded`, `ReplyGuardChecked`, `InteractionFailed`.

Every nullable lifecycle field on the `Interaction` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/interactions` — body `{ promptText, policyProfile, submittedBy }` → `{ interactionId }`.
- `GET /api/interactions` — list all interactions, newest-first.
- `GET /api/interactions/{id}` — one interaction.
- `GET /api/interactions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Guardrails Pattern</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted interactions (status pill + policy-profile badge + age) and a right pane with the selected interaction's detail — prompt text, input guardrail outcome, reply body, output guardrail outcome, and a guardrail timeline.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-llm-call guardrail** (`PromptGuardrail`): runs inside `AssistantAgent` before the prompt is forwarded to the model. Checks the prompt text against the selected `PolicyProfile`'s blocked-topic vocabulary (a configurable list of disallowed patterns: restricted domains, jailbreak markers, instruction-injection patterns, excessive length). On a match, returns `Guardrail.block(ruleId)` — the agent immediately returns a `BLOCKED` outcome to the workflow; no model token is consumed. On pass, the prompt proceeds to the model unchanged.
- **G2 — before-agent-response guardrail** (`ReplyGuardrail`): runs on every candidate reply from `AssistantAgent`. Asserts the candidate parses into `AgentReply`, that `text` and `category` are non-null, that `confidence` is in `[0.0, 1.0]`, and that the reply text does not contain any of the policy profile's blocked-topic patterns. On failure, returns a structured rejection to the agent loop so it retries within its iteration budget. Passing responses flow to `InteractionWorkflow`.

## 9. Agent prompts

- `AssistantAgent` → `prompts/assistant-agent.md`. The single decision-making LLM. System prompt instructs it to respond to the user's prompt with a structured `AgentReply`: a category, a confidence estimate, the reply text, and any citations.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a general-knowledge question under the GENERAL profile; within 30 s a well-formed reply appears with a FACTUAL category chip, a non-zero confidence, and a guardrail timeline showing both checks passing.
2. **J2** — User submits a prompt that matches a blocked-topic pattern under the STRICT profile; the `before-llm-call` guardrail fires; the card reaches `BLOCKED` state with the matched `ruleId` shown; no model call is made.
3. **J3** — The agent produces a malformed reply on first try (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed reply; the UI never displays the malformed version.
4. **J4** — A prompt that passes the input guardrail but whose generated reply accidentally introduces a blocked topic is caught by the output guardrail; after retry, the final reply is clean; the timeline shows the first output-guard failure then the second-attempt pass.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named guardrails-pattern demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-guardrails-pattern. Java package io.akka.samples.guardrails. Akka 3.6.0.
HTTP port 9321.

Components to wire (exactly):

- 1 AutonomousAgent AssistantAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/assistant-agent.md>) and
  .capability(TaskAcceptance.of(AssistantTasks.GENERATE_REPLY).maxIterationsPerTask(3)).
  Two guardrails registered on the agent:
    (1) PromptGuardrail bound to the before-llm-call hook.
    (2) ReplyGuardrail bound to the before-agent-response hook.
  Output: AgentReply{category: ReplyCategory, text: String, confidence: double,
  citations: List<String>, repliedAt: Instant}.

- 1 Workflow InteractionWorkflow per interactionId with three steps:
  * promptGuardStep — invokes PromptGuardrail.check(request) directly (not via agent). On
    BLOCK records a PromptBlocked event on InteractionEntity and calls the workflow error path.
    On PASS records PromptGuardChecked(passed=true) and advances to agentStep.
    WorkflowSettings.stepTimeout 5s.
  * agentStep — emits ReplyingStarted, then calls
    componentClient.forAutonomousAgent(AssistantAgent.class, "assistant-" + interactionId)
    .runSingleTask(TaskDef.instructions(formatPrompt(request.promptText(), request.policyProfile())))
    — the task carries the raw prompt as instructions (no attachment needed for text-only input).
    Returns taskId, then forTask(taskId).result(AssistantTasks.GENERATE_REPLY). On success calls
    InteractionEntity.recordReply(reply). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(InteractionWorkflow::error).
  * replyGuardStep — runs ReplyGuardrail.check(reply) directly in-workflow (non-blocking eval;
    the ReplyGuardrail is also registered on the agent, but this post-hoc check lets the workflow
    record the outcome as an event). Emits ReplyGuardChecked(passed=true/false, reason). Advances
    to finished state. WorkflowSettings.stepTimeout 5s.
  error step transitions the entity to FAILED or BLOCKED as appropriate.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity InteractionEntity (one per interactionId). State Interaction{
  interactionId: String, request: Optional<PromptRequest>, inputGuardResult:
  Optional<GuardrailOutcome>, reply: Optional<AgentReply>, outputGuardResult:
  Optional<GuardrailOutcome>, status: InteractionStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. InteractionStatus enum: SUBMITTED, PROMPT_CHECKED,
  BLOCKED, REPLYING, REPLY_RECORDED, FAILED. Events: PromptSubmitted{request},
  PromptGuardChecked{outcome}, PromptBlocked{ruleId, reason}, ReplyingStarted{},
  ReplyRecorded{reply}, ReplyGuardChecked{outcome}, InteractionFailed{reason}. Commands:
  submit, recordPromptGuardResult, markBlocked, markReplying, recordReply,
  recordReplyGuardResult, fail, getInteraction. emptyState() returns
  Interaction.initial("") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View InteractionView with row type InteractionRow (mirrors Interaction). Table updater
  consumes InteractionEntity events. ONE query getAllInteractions: SELECT * AS interactions
  FROM interaction_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * InteractionEndpoint at /api with POST /interactions (body {promptText, policyProfile,
    submittedBy}; mints interactionId; calls InteractionEntity.submit; starts
    InteractionWorkflow; returns {interactionId}), GET /interactions (list from
    getAllInteractions, sorted newest-first), GET /interactions/{id} (one row), GET
    /interactions/sse (Server-Sent Events from view's stream-updates), and three
    /api/metadata/* endpoints serving YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AssistantTasks.java declaring one Task<R> constant: GENERATE_REPLY = Task
  .name("Generate reply").description("Read the user prompt and produce a structured
  AgentReply with category, confidence, text, and citations")
  .resultConformsTo(AgentReply.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records PromptRequest, GuardrailOutcome, AgentReply, Interaction,
  PolicyProfile enum, ReplyCategory enum, InteractionStatus enum.

- PromptGuardrail.java implementing the before-llm-call hook. Reads the PolicyProfile from
  the task context. Checks the prompt text against the profile's blocked pattern set (three
  sets: GENERAL — jailbreak markers only; CONSERVATIVE — jailbreak + restricted domains;
  STRICT — jailbreak + restricted domains + excessive length). Returns Guardrail.pass() or
  Guardrail.block(ruleId) where ruleId is the matched pattern identifier.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  AgentReply from the response. Checks: (1) text is non-null and non-empty, (2) category is
  in the enum, (3) confidence is in [0.0, 1.0], (4) reply text does not contain blocked-topic
  patterns from the active profile. On any failure returns Guardrail.reject(reason); on pass
  returns Guardrail.pass().

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9321 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The AssistantAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-prompts.jsonl with 3 seeded prompts: a general
  knowledge question (What is the boiling point of water at high altitude?), a creative
  writing seed (Write the opening sentence of a mystery novel set on a space station.), and
  a structured extraction request (Extract all dates and amounts from the following invoice
  text: ...). Also include 2 prompts that trigger the STRICT policy (one jailbreak-pattern
  prompt, one excessive-length prompt) so J2 is exercisable from the UI.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, G2) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root pre-filled for the general domain; deployer-specific
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/assistant-agent.md loaded as the agent system prompt.

- README.md at the project root matching the blueprint README.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of interaction cards; right = selected-interaction detail with prompt text, input
  guardrail outcome badge, reply body, output guardrail outcome badge, and a guardrail
  timeline row). Browser title exactly: <title>Akka Sample: Guardrails Pattern</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on the Task<R> id.
- Per-task mock-response shapes for THIS blueprint:
    generate-reply.json — 8 AgentReply entries covering the four ReplyCategory values.
      Each entry has a non-empty text, a category, a confidence in [0.5, 1.0], and a
      citations list (0-2 items). Plus 2 deliberately MALFORMED entries (one with confidence
      = 1.7, one with null category) — the ReplyGuardrail blocks both, exercising the retry
      path. The mock selects a malformed entry on the FIRST iteration of every 3rd interaction
      (modulo seed) so J3 is reproducible.
- MockModelProvider.seedFor(interactionId) makes per-interaction selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. AssistantAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AssistantTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (promptGuardStep 5s, agentStep
  60s, replyGuardStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Interaction row record is Optional<T>.
- Lesson 7: AssistantTasks.java with GENERATE_REPLY = Task.name(...).description(...)
  .resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated model names.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9321 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (AssistantAgent). Both
  guardrail hooks are registered on that same agent. The workflow's replyGuardStep calls
  ReplyGuardrail.check() directly as a post-hoc record step — it does NOT invoke a second
  agent.
- The before-llm-call guardrail must block calls BEFORE any token is sent to the model. The
  overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
