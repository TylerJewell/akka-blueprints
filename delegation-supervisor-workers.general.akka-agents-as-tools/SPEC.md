# SPEC — akka-agents-as-tools

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** OpenAI Agents-as-Tools Composition.
**One-line pitch:** Submit a task; a supervisor agent routes it to the right specialist sub-agent (summarizer, classifier, or translator), all running inside a durable workflow with a before-tool-call guardrail.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives, realising the OpenAI agents-as-tools topology: a Supervisor AutonomousAgent treats each specialist sub-agent as a named tool. The Workflow drives the tool-call loop with durable step tracking so no call is lost on restart. The blueprint also demonstrates a **before-tool-call guardrail** that inspects every sub-agent invocation before it is dispatched, which is the defining governance control in this topology.

## 3. User-facing flows

The user opens the App UI tab and submits a task request (text + desired operation) via the form.

1. The system creates a `TaskRequest` record in `QUEUED` and starts a `CompositionWorkflow`.
2. The Supervisor inspects the request and decides which sub-agent tool to call (SummarizerAgent, ClassifierAgent, or TranslatorAgent).
3. Before the sub-agent is dispatched, the before-tool-call guardrail inspects the tool name and its arguments. If the call is prohibited, the request moves to `BLOCKED`.
4. If the guardrail passes, the selected sub-agent runs and returns a typed `ToolResult`.
5. The Supervisor assembles a `TaskResult { operation, toolUsed, output, guardrailVerdict, completedAt }` and the request moves to `COMPLETED`.
6. If the selected sub-agent does not respond within 60 seconds, the workflow short-circuits to `TIMED_OUT` and records a `failureReason`.

A `RequestSimulator` (TimedAction) drips a sample task every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `Supervisor` | `AutonomousAgent` | Decides which sub-agent tool to invoke and assembles the final `TaskResult`. | `CompositionWorkflow` | returns typed result to workflow |
| `SummarizerAgent` | `AutonomousAgent` | Produces a concise text summary of the input. Exposed as a tool named `summarize`. | `CompositionWorkflow` | — |
| `ClassifierAgent` | `AutonomousAgent` | Assigns one or more category labels to the input text. Exposed as a tool named `classify`. | `CompositionWorkflow` | — |
| `TranslatorAgent` | `AutonomousAgent` | Translates input text to the requested target language. Exposed as a tool named `translate`. | `CompositionWorkflow` | — |
| `CompositionWorkflow` | `Workflow` | Drives the supervisor's routing decision, runs the guardrail, dispatches the selected sub-agent, persists the result. | `CompositionEndpoint`, `TaskRequestConsumer` | `TaskRequestEntity` |
| `TaskRequestEntity` | `EventSourcedEntity` | Holds the task request lifecycle (queued → in-progress → completed / blocked / timed-out). | `CompositionWorkflow` | `CompositionView` |
| `TaskRequestQueue` | `EventSourcedEntity` | Logs each inbound request for replay and audit. | `CompositionEndpoint`, `RequestSimulator` | `TaskRequestConsumer` |
| `CompositionView` | `View` | List-of-requests read model. | `TaskRequestEntity` events | `CompositionEndpoint` |
| `TaskRequestConsumer` | `Consumer` | Listens to `TaskRequestQueue` events and starts one `CompositionWorkflow` per request. | `TaskRequestQueue` events | `CompositionWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample task request every 60 s. | scheduler | `TaskRequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one completed request every 5 minutes for eval scoring; emits a `TaskEvalScored` event. | scheduler | `TaskRequestEntity` |
| `CompositionEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE. | — | `CompositionView`, `TaskRequestQueue`, `TaskRequestEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TaskSubmission(String inputText, String operation, String requestedBy) {}

record ToolResult(String toolName, String output, Instant returnedAt) {}

record RoutingDecision(String selectedTool, String reasoning) {}

record TaskResult(
    String operation,
    String toolUsed,
    String output,
    String guardrailVerdict,
    Instant completedAt
) {}

record TaskRequest(
    String requestId,
    String inputText,
    String operation,
    RequestStatus status,
    Optional<RoutingDecision> routing,
    Optional<ToolResult> toolResult,
    Optional<TaskResult> result,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus { QUEUED, IN_PROGRESS, COMPLETED, BLOCKED, TIMED_OUT }
```

### Events (on `TaskRequestEntity`)

`RequestCreated`, `RequestRouted`, `ToolResultAttached`, `RequestCompleted`, `RequestBlocked`, `RequestTimedOut`, `TaskEvalScored`.

### Events (on `TaskRequestQueue`)

`TaskSubmitted { requestId, inputText, operation, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ inputText, operation }` → `{ requestId }`. Starts a workflow.
- `GET /api/tasks` — list all requests. Optional `?status=QUEUED|IN_PROGRESS|COMPLETED|BLOCKED|TIMED_OUT`.
- `GET /api/tasks/{id}` — one request.
- `GET /api/tasks/sse` — server-sent events stream of every request change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "OpenAI Agents-as-Tools Composition"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task request (inputText + operation dropdown), live list of requests with status pills, expand-row to see routing decision + tool result + assembled output + eval score.

Browser title: `<title>Akka Sample: OpenAI Agents-as-Tools Composition</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — before-tool-call guardrail** (`before-tool-call` on `Supervisor`): inspects the tool name and its arguments before the sub-agent is dispatched. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one completed request every 5 minutes and emits a `TaskEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `Supervisor` → `prompts/supervisor.md`. Routes the request to the correct tool; assembles the final result.
- `SummarizerAgent` → `prompts/summarizer-agent.md`. Produces a concise summary; returns `ToolResult`.
- `ClassifierAgent` → `prompts/classifier-agent.md`. Assigns category labels; returns `ToolResult`.
- `TranslatorAgent` → `prompts/translator-agent.md`. Translates text to the target language; returns `ToolResult`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a summarize task; request progresses QUEUED → IN_PROGRESS → COMPLETED within 60 s; UI reflects each transition via SSE.
2. **J2** — Submit a request with a tool name flagged as prohibited (test fixture); request enters BLOCKED before the sub-agent runs.
3. **J3** — Inject a sub-agent timeout (set sub-agent step timeout to 1 s); request enters TIMED_OUT.
4. **J4** — Wait after a successful completion; the request's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-agents-as-tools demonstrating the
delegation-supervisor-workers × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-general-akka-agents-as-tools.
Java package io.akka.samples.openaiagentsastoolscomposition. Akka 3.6.0. HTTP port 9545.

Components to wire (exactly):
- 4 AutonomousAgents:
  * Supervisor — definition() with capability(TaskAcceptance.of(ROUTE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/supervisor.md. Returns RoutingDecision{selectedTool,
    reasoning} for ROUTE and TaskResult{operation, toolUsed, output, guardrailVerdict,
    completedAt} for ASSEMBLE.
  * SummarizerAgent — capability(TaskAcceptance.of(SUMMARIZE).maxIterationsPerTask(2)).
    System prompt from prompts/summarizer-agent.md. Returns ToolResult{toolName="summarize",
    output, returnedAt}.
  * ClassifierAgent — capability(TaskAcceptance.of(CLASSIFY).maxIterationsPerTask(2)).
    System prompt from prompts/classifier-agent.md. Returns ToolResult{toolName="classify",
    output, returnedAt}.
  * TranslatorAgent — capability(TaskAcceptance.of(TRANSLATE).maxIterationsPerTask(2)).
    System prompt from prompts/translator-agent.md. Returns ToolResult{toolName="translate",
    output, returnedAt}.

- 1 Workflow CompositionWorkflow with steps:
  routeStep -> guardrailStep -> dispatchStep -> assembleStep -> emitStep.
  routeStep calls forAutonomousAgent(Supervisor.class, ROUTE).
  guardrailStep is a deterministic before-tool-call check on RoutingDecision.selectedTool
  and the task arguments; on failure, end with RequestBlocked.
  dispatchStep branches on selectedTool: SUMMARIZE -> SummarizerAgent, CLASSIFY ->
  ClassifierAgent, TRANSLATE -> TranslatorAgent. Each branch uses
  WorkflowSettings.builder().stepTimeout(CompositionWorkflow::dispatchStep, ofSeconds(60)).
  On timeout, transition to timeoutStep that ends with RequestTimedOut.
  assembleStep calls forAutonomousAgent(Supervisor.class, ASSEMBLE) with RoutingDecision
  and ToolResult; give assembleStep a 60s stepTimeout.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity TaskRequestEntity holding state TaskRequest{requestId, inputText,
  operation, RequestStatus, Optional<RoutingDecision> routing,
  Optional<ToolResult> toolResult, Optional<TaskResult> result,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  RequestStatus enum: QUEUED, IN_PROGRESS, COMPLETED, BLOCKED, TIMED_OUT.
  Events: RequestCreated, RequestRouted, ToolResultAttached, RequestCompleted,
  RequestBlocked, RequestTimedOut, TaskEvalScored.
  Commands: createRequest, attachRouting, attachToolResult, complete, block, timeOut,
  recordEval, getRequest.
  emptyState() returns TaskRequest.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity TaskRequestQueue with command enqueueTask(inputText, operation,
  requestedBy) emitting TaskSubmitted{requestId, inputText, operation, requestedBy, submittedAt}.

- 1 View CompositionView with row type TaskRequestRow (mirrors TaskRequest minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  TaskRequestEntity events. ONE query getAllRequests SELECT * AS requests FROM task_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer TaskRequestConsumer subscribed to TaskRequestQueue events; on TaskSubmitted
  starts a CompositionWorkflow with the requestId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-tasks.jsonl and calls TaskRequestQueue.enqueueTask.
  * EvalSampler — every 5 minutes, queries CompositionView.getAllRequests, picks the oldest
    COMPLETED request without an evalScore, runs a 1-5 rubric judge over the TaskResult
    output, then calls TaskRequestEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * CompositionEndpoint at /api with POST /tasks, GET /tasks, GET /tasks/{id},
    GET /tasks/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- CompositionTasks.java declaring five Task<R> constants: ROUTE (RoutingDecision),
  ASSEMBLE (TaskResult), SUMMARIZE (ToolResult), CLASSIFY (ToolResult), TRANSLATE (ToolResult).
- Domain records TaskSubmission, ToolResult, RoutingDecision, TaskResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9545 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-tasks.jsonl with 8 canned task lines covering
  summarize, classify, and translate operations.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 before-tool-call guardrail,
  E1 eval-event on-decision-eval) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  task-routing-composition, decisions.authority_level = automated-action,
  data.data_classes.pii = false, capabilities.* = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/supervisor.md, prompts/summarizer-agent.md, prompts/classifier-agent.md,
  prompts/translator-agent.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: OpenAI Agents-as-Tools Composition",
  one-line pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form with inputText textarea
  and operation dropdown + live list with status pills). Browser title exactly:
  <title>Akka Sample: OpenAI Agents-as-Tools Composition</title>. No subtitle on Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (supervisor.json,
  summarizer-agent.json, classifier-agent.json, translator-agent.json),
  picks one entry pseudo-randomly per call, and deserialises it into the
  agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    supervisor.json — list of either RoutingDecision or TaskResult objects.
      4–6 RoutingDecision entries (selectedTool one of "summarize"/"classify"/"translate"
      plus a one-sentence reasoning) and 4–6 TaskResult entries (each with operation,
      toolUsed, 30–80 word output, guardrailVerdict = "ok", completedAt).
    summarizer-agent.json — 4–6 ToolResult entries with toolName="summarize" and
      a 30–80 word summary output.
    classifier-agent.json — 4–6 ToolResult entries with toolName="classify" and
      output listing 2–4 category labels (e.g., "technology, AI governance, policy").
    translator-agent.json — 4–6 ToolResult entries with toolName="translate" and
      output containing translated text into Spanish, French, or German.
- A MockModelProvider.seedFor(requestId) helper makes the selection
  deterministic per request id so the same request produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s for
  dispatch and assemble); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion CompositionTasks.java declaring Task<R>
  constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9545 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- The guardrail step is BEFORE the sub-agent is dispatched (before-tool-call), not after.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
