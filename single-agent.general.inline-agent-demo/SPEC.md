# SPEC — inline-agent-demo

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** InlineAgentDemo.
**One-line pitch:** A caller submits a question alongside a complete agent definition — system prompt, tools, and output schema — at runtime; one AI agent is assembled on the fly and returns a typed structured answer, with no class or configuration change required between runs.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain, using the inline-agent API. Instead of declaring a fixed `AutonomousAgent` subclass with a static `definition()`, the `InlineAgentRunner` component constructs an `AgentDefinition` at call time from the caller-supplied `InlineAgentRequest`. This lets callers vary the system prompt, tool list, and expected output schema per request.

The surrounding components prepare input and persist run state — a pattern consistent with every other single-agent blueprint — but the distinguishing feature is that the agent's identity is ephemeral: it exists only for the duration of one task.

This blueprint has no governance controls wired by default. The `eval-matrix.yaml` is empty of controls. A deployer who adds a real use case adds controls appropriate to that use case.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **seeded agent definition** from the dropdown (Q&A assistant, keyword extractor, tone classifier) or pastes a custom definition JSON into the **Agent definition** textarea.
2. The user types a **question** (or picks a seeded example that goes with the selected definition).
3. The user clicks **Run**. The UI POSTs to `/api/runs` and receives a `runId`.
4. The card appears in the live list in `RECEIVED` state. Within ~1 s it transitions to `RUNNING` as the workflow hands off to the inline agent.
5. Within ~10–30 s the run completes. The card transitions to `COMPLETED`. The right pane shows: the agent definition summary, the question, and the structured answer produced by the agent.
6. If the agent definition is malformed (missing `systemPrompt` or `outputSchema`), the card transitions immediately to `FAILED` with a validation error visible in the right pane.
7. The user can submit more runs; the live list keeps history visible newest-first.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RunEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `RunEntity`, `RunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `RunEntity` | `EventSourcedEntity` | Per-run lifecycle: received → running → completed / failed. Source of truth. | `RunEndpoint`, `RunWorkflow` | `RunView` |
| `RunWorkflow` | `Workflow` | One workflow per run. Steps: `validateStep` → `runStep` → (record result). | started by `RunEndpoint` after entity write | `InlineAgentRunner`, `RunEntity` |
| `InlineAgentRunner` | `AutonomousAgent` | The single decision-making LLM. Receives a fully-constructed `AgentDefinition` from the workflow; returns `AgentResponse`. | invoked by `RunWorkflow` | returns answer |
| `RunView` | `View` | Read model: one row per run for the UI. | `RunEntity` events | `RunEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record AgentDefinition(
    String agentName,
    String systemPrompt,
    String outputSchema,         // JSON Schema string describing the expected answer shape
    List<String> allowedTools    // empty list = no tools; named tools the runtime knows about
) {}

record InlineAgentRequest(
    String runId,
    String question,
    AgentDefinition agentDefinition,
    String submittedBy,
    Instant submittedAt
) {}

record AgentResponse(
    String answer,               // structured text or JSON matching outputSchema
    int tokenCount,
    Instant answeredAt
) {}

record Run(
    String runId,
    Optional<InlineAgentRequest> request,
    Optional<AgentResponse> response,
    Optional<String> failureReason,
    RunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus {
    RECEIVED, RUNNING, COMPLETED, FAILED
}
```

Events on `RunEntity`: `RunReceived`, `RunStarted`, `RunCompleted`, `RunFailed`.

Every nullable lifecycle field on the `Run` record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ question, agentDefinition: { agentName, systemPrompt, outputSchema, allowedTools }, submittedBy }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Inline Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted runs (status pill + age) and a right pane with the selected run's detail — agent definition summary, the question, and the structured answer.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. This baseline has **no controls** wired by default. The generated system runs without any sanitizer, guardrail, or evaluator. A deployer adds controls appropriate to their use case after forking this blueprint.

## 9. Agent prompts

- `InlineAgentRunner` → `prompts/inline-agent-runner.md`. The base system prompt included with every inline run, before the caller-supplied `systemPrompt` is appended.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User picks the Q&A seeded definition and submits a question; within 30 s the answer appears with a non-empty `answer` field.
2. **J2** — User submits a definition with `systemPrompt` missing; the service returns `400` immediately, the run card shows `FAILED`, and no LLM call is made.
3. **J3** — Two runs with different agent definitions are submitted within 1 s of each other; both complete successfully and neither answer contains text from the other run's context.
4. **J4** — The live list SSE stream delivers a status event within 1 s of each state transition; no polling is required.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named inline-agent-demo demonstrating the single-agent × general domain cell,
specifically the inline-agent API pattern. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact single-agent-general-inline-agent-demo.
Java package io.akka.samples.inlineagent. Akka 3.6.0. HTTP port 9801.

Components to wire (exactly):

- 1 AutonomousAgent InlineAgentRunner — definition() returns an AgentDefinition assembled
  from the InlineAgentRequest received via the workflow step. The key difference from a
  static agent: the AgentDefinition is constructed at runtime from
  request.agentDefinition.systemPrompt, request.agentDefinition.outputSchema, and
  request.agentDefinition.allowedTools. The base instructions from
  prompts/inline-agent-runner.md are prepended to the caller-supplied systemPrompt.
  The agent's capability is TaskAcceptance.of(RunTasks.ANSWER_QUESTION).maxIterationsPerTask(3).
  Output: AgentResponse{answer: String, tokenCount: int, answeredAt: Instant}.

- 1 Workflow RunWorkflow per runId with two steps:
  * validateStep — reads RunEntity.getRun, checks that agentDefinition.systemPrompt is
    non-blank and agentDefinition.outputSchema is non-blank. On failure calls
    RunEntity.fail(reason) and transitions to the error step. WorkflowSettings.stepTimeout 5s.
  * runStep — emits RunStarted, then calls componentClient.forAutonomousAgent(
    InlineAgentRunner.class, "runner-" + runId).runSingleTask(
      TaskDef.instructions(request.question)
    ) — the agent definition (system prompt + schema + tools) is injected into the
    AgentDefinition object passed to the agent at construction time, not embedded in the
    instructions text. Returns a taskId, then forTask(taskId).result(RunTasks.ANSWER_QUESTION)
    to fetch the response. On success calls RunEntity.complete(response).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2).failoverTo(RunWorkflow::error).
  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity RunEntity (one per runId). State Run{runId: String,
  request: Optional<InlineAgentRequest>, response: Optional<AgentResponse>,
  failureReason: Optional<String>, status: RunStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. RunStatus enum: RECEIVED, RUNNING, COMPLETED, FAILED.
  Events: RunReceived{request}, RunStarted{}, RunCompleted{response}, RunFailed{reason}.
  Commands: submit, markRunning, complete, fail, getRun. emptyState() returns Run.initial("")
  with no commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 View RunView with row type RunRow (mirrors Run). Table updater consumes RunEntity events.
  ONE query getAllRuns: SELECT * AS runs FROM run_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * RunEndpoint at /api with POST /runs (body {question, agentDefinition: {agentName,
    systemPrompt, outputSchema, allowedTools: []}, submittedBy}; mints runId; calls
    RunEntity.submit; starts RunWorkflow; returns {runId}), GET /runs (list from getAllRuns,
    sorted newest-first), GET /runs/{id} (one row), GET /runs/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- RunTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer question")
  .description("Read the question and the agent definition and produce an AgentResponse")
  .resultConformsTo(AgentResponse.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records AgentDefinition, InlineAgentRequest, AgentResponse, Run, RunStatus.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9801 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. InlineAgentRunner.definition() binds
  the configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/agent-definitions.jsonl with 3 seeded agent definitions:
  a Q&A assistant (general knowledge), a keyword extractor, and a tone classifier.

- src/main/resources/sample-events/seed-questions.jsonl with 3 paired example questions,
  one per seeded agent definition.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 0 controls (empty controls list). No
  simplified_view rows.

- risk-survey.yaml at the project root with data.data_classes matching general/unspecified
  defaults. Deployer-specific fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/inline-agent-runner.md loaded as the base agent instructions prepended before the
  caller-supplied systemPrompt.

- README.md at the project root: title "Akka Sample: Inline Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of run cards; right = selected-run detail with agent definition summary, question,
  and structured answer).
  Browser title exactly: <title>Akka Sample: Inline Agent</title>. No subtitle on the
  Overview tab.

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
  <task-id>.json and picks one entry pseudo-randomly per call (seedFor(runId)).
- Per-task mock-response shapes for THIS blueprint:
    answer-question.json — 6 AgentResponse entries covering Q&A, keyword extraction, and
      tone classification. Each entry has a non-empty answer string, a plausible tokenCount
      (50–300), and an answeredAt timestamp. Answers vary across the three seeded definition
      types so the UI shows diverse output shapes.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. InlineAgentRunner extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion RunTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (validateStep
  5s, runStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the Run record is Optional<T>. The view table
  updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: RunTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(AgentResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9801 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides (state-label
  colour, edge-label foreignObject overflow:visible) AND the mermaid.initialize themeVariables
  block (nodeTextColor, stateLabelColor, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI.
- The single-agent invariant: there is exactly ONE AutonomousAgent (InlineAgentRunner).
- The question is passed as TaskDef.instructions(request.question); the agent definition's
  system prompt and output schema are injected into the AgentDefinition at construction time,
  NOT as inline instruction text appended to the question.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
