# SPEC — mac-hello-world

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Multi-Agent Hello World.
**One-line pitch:** Submit a name and context; a coordinator delegates message drafting to a GreetingWriter and action suggestions to an ActionAdvisor in parallel, then composes one response.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives at minimal scale: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to compose a final greeting response. No governance controls are layered on; this baseline is the pattern in its purest form.

## 3. User-facing flows

The user opens the App UI tab and submits a name and optional context string via the form.

1. The system creates a `GreetingRequest` record in `PENDING` and starts a `GreetingWorkflow`.
2. The Coordinator decides how to split the work: it emits a `WritingSpec { name, context, tone }` for the GreetingWriter and an `AdvisorySpec { name, context }` for the ActionAdvisor.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the two payloads into a `GreetingResponse { message, followUpAction, composedAt }`.
5. The request moves to `COMPLETED` and the response is visible in the App UI.
6. If either worker times out after 30 seconds, the workflow short-circuits: the Coordinator composes from whichever side returned and the request enters `DEGRADED`.

A `RequestSimulator` (TimedAction) drips a sample request every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GreetingCoordinator` | `AutonomousAgent` | Decomposes the request into work specs; later merges worker outputs into the final response. | `GreetingWorkflow` | returns typed result to workflow |
| `GreetingWriter` | `AutonomousAgent` | Drafts a greeting message for the given name and context. | `GreetingWorkflow` | — |
| `ActionAdvisor` | `AutonomousAgent` | Suggests a short follow-up action relevant to the context. | `GreetingWorkflow` | — |
| `GreetingWorkflow` | `Workflow` | Coordinates parallel fan-out, joins results, asks Coordinator to compose. | `GreetingEndpoint`, `GreetingRequestConsumer` | `GreetingEntity` |
| `GreetingEntity` | `EventSourcedEntity` | Holds the request lifecycle (pending → in-progress → completed / degraded). | `GreetingWorkflow` | `GreetingView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted request for replay. | `GreetingEndpoint`, `RequestSimulator` | `GreetingRequestConsumer` |
| `GreetingView` | `View` | List-of-requests read model. | `GreetingEntity` events | `GreetingEndpoint` |
| `GreetingRequestConsumer` | `Consumer` | Listens to `RequestQueue` events and starts a workflow per submission. | `RequestQueue` events | `GreetingWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample request every 60 s. | scheduler | `RequestQueue` |
| `GreetingEndpoint` | `HttpEndpoint` | `/api/greetings/*` — submit, get, list, SSE. | — | `GreetingView`, `RequestQueue`, `GreetingEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record GreetingSubmission(String name, String context, String requestedBy) {}

record WritingSpec(String name, String context, String tone) {}
record AdvisorySpec(String name, String context) {}

record DraftMessage(String message, Instant draftedAt) {}
record FollowUpAction(String action, Instant advisedAt) {}

record GreetingResponse(
    String message,
    String followUpAction,
    Instant composedAt
) {}

record GreetingRequest(
    String requestId,
    String name,
    String context,
    RequestStatus status,
    Optional<DraftMessage> draft,
    Optional<FollowUpAction> followUp,
    Optional<GreetingResponse> response,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus { PENDING, IN_PROGRESS, COMPLETED, DEGRADED }
```

### Events (on `GreetingEntity`)

`RequestCreated`, `DraftAttached`, `FollowUpAttached`, `RequestCompleted`, `RequestDegraded`.

### Events (on `RequestQueue`)

`GreetingSubmitted { requestId, name, context, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/greetings` — body `{ "name": "string", "context": "string" }` → `{ "requestId": "uuid" }`. Starts a workflow.
- `GET /api/greetings` — list all requests. Optional `?status=PENDING|IN_PROGRESS|COMPLETED|DEGRADED`.
- `GET /api/greetings/{id}` — one request.
- `GET /api/greetings/sse` — server-sent events stream of every request change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Multi-Agent Hello World"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows. Zero controls; renders an empty-state message.
- **App UI** — form to submit a name + context, live list of requests with status pills, expand-row to see draft message + follow-up action + composed response.

Browser title: `<title>Akka Sample: Multi-Agent Hello World</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. This baseline carries zero governance controls — it is the minimal pattern demonstration. `eval-matrix.yaml` has an empty `controls` list and a matching empty `simplified_view`. A deployer adding a production model provider should layer in at minimum a `before-agent-response` guardrail via their own eval-matrix entry.

## 9. Agent prompts

- `GreetingCoordinator` → `prompts/greeting-coordinator.md`. Decomposes the submission into work specs; later merges worker outputs into a final response.
- `GreetingWriter` → `prompts/greeting-writer.md`. Drafts a greeting message; returns `DraftMessage`.
- `ActionAdvisor` → `prompts/action-advisor.md`. Suggests a follow-up action; returns `FollowUpAction`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a name and context; request progresses PENDING → IN_PROGRESS → COMPLETED within 30 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `GreetingWriter` timeout to 1 s); request enters DEGRADED with whatever ActionAdvisor returned.
3. **J3** — Wait for the simulator; the App UI fills with requests automatically without any user input.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named mac-hello-world demonstrating the
delegation-supervisor-workers × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-general-mac-hello-world.
Java package io.akka.samples.multiagenthelloworld. Akka 3.6.0. HTTP port 9954.

Components to wire (exactly):
- 3 AutonomousAgents:
  * GreetingCoordinator — definition() with
    capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(COMPOSE).maxIterationsPerTask(2)).
    System prompt loaded from prompts/greeting-coordinator.md.
    Returns WritingSpec{name, context, tone} + AdvisorySpec{name, context}
    as a DecomposedWork{writingSpec, advisorySpec} for DECOMPOSE, and
    GreetingResponse{message, followUpAction, composedAt} for COMPOSE.
  * GreetingWriter — capability(TaskAcceptance.of(WRITE).maxIterationsPerTask(2)).
    System prompt from prompts/greeting-writer.md.
    Returns DraftMessage{message, draftedAt}.
  * ActionAdvisor — capability(TaskAcceptance.of(ADVISE).maxIterationsPerTask(2)).
    System prompt from prompts/action-advisor.md.
    Returns FollowUpAction{action, advisedAt}.

- 1 Workflow GreetingWorkflow with steps:
  decomposeStep -> [parallel] writeStep, adviseStep -> joinStep -> composeStep -> emitStep.
  decomposeStep calls forAutonomousAgent(GreetingCoordinator.class, DECOMPOSE).
  writeStep and adviseStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(GreetingWorkflow::writeStep, ofSeconds(30)) and
  stepTimeout(GreetingWorkflow::adviseStep, ofSeconds(30)). On either timeout, transition
  to a degradeStep that calls composeStep with whichever side returned, then ends with
  RequestDegraded. composeStep calls forAutonomousAgent(GreetingCoordinator.class, COMPOSE)
  with the merged inputs; give composeStep a 45s stepTimeout. WorkflowSettings is nested
  inside Workflow — no import.

- 1 EventSourcedEntity GreetingEntity holding state GreetingRequest{requestId, name,
  context, RequestStatus, Optional<DraftMessage> draft, Optional<FollowUpAction> followUp,
  Optional<GreetingResponse> response, Optional<String> failureReason, Instant createdAt,
  Optional<Instant> finishedAt}. RequestStatus enum: PENDING, IN_PROGRESS, COMPLETED,
  DEGRADED. Events: RequestCreated, DraftAttached, FollowUpAttached, RequestCompleted,
  RequestDegraded. Commands: createRequest, attachDraft, attachFollowUp, complete, degrade,
  getRequest. emptyState() returns GreetingRequest.initial("", "", null) with no
  commandContext() reference.

- 1 EventSourcedEntity RequestQueue with command enqueueGreeting(name, context, requestedBy)
  emitting GreetingSubmitted{requestId, name, context, requestedBy, submittedAt}.

- 1 View GreetingView with row type GreetingRequestRow (mirrors GreetingRequest minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes GreetingEntity
  events. ONE query getAllRequests SELECT * AS requests FROM greeting_view. No WHERE status
  filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer GreetingRequestConsumer subscribed to RequestQueue events; on GreetingSubmitted
  starts a GreetingWorkflow with the requestId as the workflow id.

- 1 TimedAction:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/greeting-requests.jsonl and calls
    RequestQueue.enqueueGreeting.

- 2 HttpEndpoints:
  * GreetingEndpoint at /api with POST /greetings, GET /greetings, GET /greetings/{id},
    GET /greetings/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- GreetingTasks.java declaring four Task<R> constants: DECOMPOSE (DecomposedWork), WRITE
  (DraftMessage), ADVISE (FollowUpAction), COMPOSE (GreetingResponse).
- Domain records WritingSpec, AdvisorySpec, DecomposedWork, DraftMessage, FollowUpAction,
  GreetingResponse, GreetingSubmission.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9954 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/greeting-requests.jsonl with 8 canned request lines
  covering varied names and contexts.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 0 controls and an empty simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  greeting-generation, decisions.authority_level = recommend-only, data.data_classes.pii =
  false, capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/greeting-coordinator.md, prompts/greeting-writer.md, prompts/action-advisor.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Multi-Agent Hello World", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no
  ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey (7
  sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows; zero controls
  renders empty-state message), App UI (form + live list with status pills). Browser title
  exactly: <title>Akka Sample: Multi-Agent Hello World</title>. No subtitle on the Overview
  tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
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
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (greeting-coordinator.json,
  greeting-writer.json, action-advisor.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    greeting-coordinator.json — list of either DecomposedWork or GreetingResponse
      objects. 4–6 DecomposedWork entries (writingSpec + advisorySpec pairs with
      varied tones: formal, friendly, playful) and 4–6 GreetingResponse entries
      (each with a 30–60 word message and a concrete followUpAction).
    greeting-writer.json — 4–6 DraftMessage entries, each with a complete
      greeting sentence addressed to a placeholder name.
    action-advisor.json — 4–6 FollowUpAction entries, each a one-sentence
      concrete suggestion.
- A MockModelProvider.seedFor(requestId) helper makes the selection
  deterministic per request id so the same request produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (30s workers,
  45s compose); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion GreetingTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9954 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and arrow
  labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
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
