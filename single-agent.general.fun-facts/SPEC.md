# SPEC — fun-facts

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** FunFacts.
**One-line pitch:** A user submits a topic string; one AI agent returns a structured `FactCollection` — a headline fact, a list of supporting facts, and a confidence tier per fact — all typed and stored for browsing.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `FactGeneratorAgent` (AutonomousAgent) carries the entire generation; the surrounding components prepare its input and project its output into the read model. No governance controls are wired in this baseline — the eval-matrix is empty (`controls: []`) and the risk-survey flags the deployment as low-stakes informational content. Deployers who need controls can add them without touching the core agent logic.

The blueprint shows the minimal viable single-agent skeleton: one entity, one workflow, one agent, one view, two endpoints, and the embedded UI — nothing more.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **topic** into the text field (e.g., "deep-sea bioluminescence") or clicks one of three seeded topics (Octopus Intelligence, Black Holes, Ancient Roman Engineering).
2. The user clicks **Generate facts**. The UI POSTs to `/api/fact-requests` and receives a `requestId`.
3. The card appears in the live list in `PENDING` state.
4. Within ~5–20 s the workflow's `generateStep` completes. The card transitions to `GENERATING` then `GENERATED`. The fact collection appears: a headline fact in bold, then a numbered list of supporting facts each showing topic area, statement, and a confidence chip (HIGH / MEDIUM / LOW).
5. The user can submit another topic; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `FactEndpoint` | `HttpEndpoint` | `/api/fact-requests/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `FactRequestEntity`, `FactRequestView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `FactRequestEntity` | `EventSourcedEntity` | Per-request lifecycle: pending → generating → generated → failed. Source of truth. | `FactEndpoint`, `FactRequestWorkflow` | `FactRequestView` |
| `FactRequestWorkflow` | `Workflow` | One workflow per requestId. Steps: `generateStep` → terminal. | started by `FactEndpoint` after entity creation | `FactGeneratorAgent`, `FactRequestEntity` |
| `FactGeneratorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives topic in task instructions; returns `FactCollection`. | invoked by `FactRequestWorkflow` | returns collection |
| `FactRequestView` | `View` | Read model: one row per fact request for the UI. | `FactRequestEntity` events | `FactEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record FactRequest(
    String requestId,
    String topic,
    String requestedBy,
    Instant requestedAt
) {}

record Fact(
    String area,             // e.g. "physiology", "behaviour", "history"
    String statement,
    Confidence confidence
) {}
enum Confidence { HIGH, MEDIUM, LOW }

record FactCollection(
    String headlineFact,
    List<Fact> facts,
    Instant generatedAt
) {}

record FactRequestState(
    String requestId,
    Optional<FactRequest> request,
    Optional<FactCollection> collection,
    FactRequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum FactRequestStatus {
    PENDING, GENERATING, GENERATED, FAILED
}
```

Events on `FactRequestEntity`: `FactRequested`, `GenerationStarted`, `CollectionGenerated`, `RequestFailed`.

Every nullable lifecycle field on the `FactRequestState` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/fact-requests` — body `{ topic, requestedBy }` → `{ requestId }`.
- `GET /api/fact-requests` — list all requests, newest-first.
- `GET /api/fact-requests/{id}` — one request.
- `GET /api/fact-requests/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: FunFacts</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted requests (status pill + topic + age) and a right pane with the selected request's detail — topic label, status, headline fact, supporting facts list with confidence chips.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. This baseline declares **no controls** (`controls: []`). The risk-survey reflects a general-purpose, low-stakes informational use case: no PII in inputs, authority-level is informational-only, no human-on-loop requirement. Deployers adding topic-sensitive content or audience restrictions should add controls before production use.

## 9. Agent prompts

- `FactGeneratorAgent` → `prompts/fact-generator.md`. The single decision-making LLM. System prompt instructs it to read the topic from the task's instructions field and return one well-formed `FactCollection`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits "Octopus Intelligence"; within 30 s the card reaches `GENERATED` with a non-empty `headlineFact` and at least 3 `facts`, each with a non-empty `statement`.
2. **J2** — User submits the same topic twice; both cards reach `GENERATED`; each has its own `requestId` and independent `collection`; neither interferes with the other.
3. **J3** — The live list updates via SSE when a new card transitions to `GENERATED`; no page reload is required.
4. **J4** — User submits an empty string; the service returns `400` with a descriptive error message and no entity is created.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named fun-facts demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-fun-facts. Java package io.akka.samples.funfacts. Akka 3.6.0.
HTTP port 9195.

Components to wire (exactly):

- 1 AutonomousAgent FactGeneratorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/fact-generator.md>) and
  .capability(TaskAcceptance.of(FactTasks.GENERATE_FACTS).maxIterationsPerTask(2)).
  Output: FactCollection{headlineFact: String, facts: List<Fact>, generatedAt: Instant}.
  No guardrail. No sanitizer. The topic is passed in the task's instructions text.

- 1 Workflow FactRequestWorkflow per requestId with two steps:
  * generateStep — emits GenerationStarted on FactRequestEntity, then calls
    componentClient.forAutonomousAgent(FactGeneratorAgent.class, "generator-" + requestId)
    .runSingleTask(TaskDef.instructions("Generate interesting facts about: " + topic))
    — returns a taskId, then forTask(taskId).result(FactTasks.GENERATE_FACTS) to fetch
    the collection. On success calls FactRequestEntity.recordCollection(collection).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(FactRequestWorkflow::error).
  * error step — calls FactRequestEntity.fail(reason). WorkflowSettings.stepTimeout 5s.
    Transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity FactRequestEntity (one per requestId). State FactRequestState{
  requestId: String, request: Optional<FactRequest>, collection: Optional<FactCollection>,
  status: FactRequestStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  FactRequestStatus enum: PENDING, GENERATING, GENERATED, FAILED. Events:
  FactRequested{request}, GenerationStarted{}, CollectionGenerated{collection},
  RequestFailed{reason}. Commands: submit, markGenerating, recordCollection, fail,
  getRequest. emptyState() returns FactRequestState.initial("") with all Optional fields
  as Optional.empty() and status = PENDING. emptyState() never references commandContext()
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 View FactRequestView with row type FactRequestRow (mirrors FactRequestState). Table
  updater consumes FactRequestEntity events. ONE query getAllRequests:
  SELECT * AS requests FROM fact_request_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * FactEndpoint at /api with POST /fact-requests (body {topic, requestedBy}; validates
    topic is non-blank — returns 400 if blank; mints requestId; calls FactRequestEntity
    .submit; starts FactRequestWorkflow; returns {requestId}), GET /fact-requests (list
    from getAllRequests, sorted newest-first), GET /fact-requests/{id} (one row), GET
    /fact-requests/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- FactTasks.java declaring one Task<R> constant: GENERATE_FACTS = Task
  .name("Generate facts").description("Generate a structured collection of interesting
  facts about the given topic").resultConformsTo(FactCollection.class). DO NOT skip
  this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records FactRequest, Fact, Confidence, FactCollection, FactRequestState,
  FactRequestStatus.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9195 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The FactGeneratorAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or
  the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/topics.jsonl with 3 seeded topics:
  Octopus Intelligence, Black Holes, Ancient Roman Engineering.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 0 controls and an empty simplified_view list.

- risk-survey.yaml at the project root with sector = general-information,
  decisions.authority_level = informational-only, oversight.human_in_loop = false,
  data.data_classes.pii = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/fact-generator.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: FunFacts", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of fact-request cards; right = selected-request detail with topic
  label, headline fact, supporting facts list with confidence chips).
  Browser title exactly: <title>Akka Sample: FunFacts</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct FactCollection outputs (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM via the
        MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using
        the matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(requestId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-facts.json — 6 FactCollection entries covering all three seeded topics.
      Each entry has a non-empty headlineFact (1 sentence) and a facts array with 4–6
      Fact records. Each Fact has a non-empty area (e.g. "cognition", "anatomy",
      "history"), a non-empty statement (1–2 sentences), and a confidence value from the
      enum (HIGH/MEDIUM/LOW, varied realistically). The mock should produce deterministic
      output per requestId using seedFor so J2 is reproducible.
- A MockModelProvider.seedFor(requestId) helper makes per-request selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. FactGeneratorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion FactTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (generateStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the FactRequestState row record is
  Optional<T>. The view table updater wraps values with Optional.of(...); callers use
  .orElse(...) or .isPresent().
- Lesson 7: FactTasks.java with GENERATE_FACTS = Task.name(...).description(...)
  .resultConformsTo(FactCollection.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9195 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (FactGeneratorAgent).
  No guardrails, no sanitizers, no secondary agents in this baseline.
- The topic is passed in the task's instructions field (TaskDef.instructions(...)) — not
  as an attachment. There is no document attachment in this blueprint.
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
