# SPEC — agents-as-tools

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Agents as Tools.
**One-line pitch:** Submit a task; a supervisor agent selects from a registry of specialist agents exposed as tools, delegates sub-tasks, and assembles the final response.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern where each worker agent is registered as a named, callable tool rather than a peer invoked via a shared workflow fan-out. The supervisor's `AutonomousAgent` definition exposes `WriterAgent` and `DataAgent` as tools in its capability block. When the supervisor decides which tool to call, that call becomes an agent-to-agent invocation tracked through the workflow. The blueprint shows: how to register specialist agents as tools, how the supervisor selects and chains tool calls, and how to persist the assembled result in an `EventSourcedEntity`.

## 3. User-facing flows

The user opens the App UI tab and submits a task request via the form.

1. The system creates a `TaskRecord` in `QUEUED` and starts a `TaskWorkflow`.
2. The supervisor receives the task description and tool registry. It selects one or both specialist tools.
3. For each tool call:
   - **writer_tool** → `WriterAgent` drafts text and returns a `DraftOutput { text, wordCount, draftedAt }`.
   - **data_tool** → `DataAgent` extracts key entities and returns a `DataOutput { entities: List<NamedEntity>, summarisedAt }`.
4. The supervisor assembles the individual tool outputs into an `AssembledResult { answer, toolsUsed: List<String>, assembledAt }`.
5. The workflow writes the result to `TaskRecordEntity` and the record enters `COMPLETED`.
6. If the supervisor cannot route the request to any tool, it returns a `REJECTED` outcome with a plain-language explanation.
7. If any tool call exceeds its step timeout (45 s), the workflow writes a partial result and the record enters `FAILED`.

A `TaskSimulator` (TimedAction) drips a sample task request every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskSupervisor` | `AutonomousAgent` | Receives task; selects and calls writer_tool and/or data_tool; assembles result. | `TaskWorkflow` | returns typed result to workflow |
| `WriterAgent` | `AutonomousAgent` | Drafts text for a writing task. Exposed as writer_tool. | `TaskSupervisor` (via tool call) | — |
| `DataAgent` | `AutonomousAgent` | Extracts named entities and summarises structured data. Exposed as data_tool. | `TaskSupervisor` (via tool call) | — |
| `TaskWorkflow` | `Workflow` | Starts a supervised session, invokes the supervisor, persists the result. | `TaskEndpoint`, `TaskRequestConsumer` | `TaskRecordEntity` |
| `TaskRecordEntity` | `EventSourcedEntity` | Holds the task's lifecycle (queued → processing → completed / failed / rejected). | `TaskWorkflow` | `TaskView` |
| `TaskQueue` | `EventSourcedEntity` | Logs each submitted task for replay/audit. | `TaskEndpoint`, `TaskSimulator` | `TaskRequestConsumer` |
| `TaskView` | `View` | List-of-tasks read model. | `TaskRecordEntity` events | `TaskEndpoint` |
| `TaskRequestConsumer` | `Consumer` | Subscribes to `TaskQueue` events and starts a `TaskWorkflow` per submission. | `TaskQueue` events | `TaskWorkflow` |
| `TaskSimulator` | `TimedAction` | Drips a sample task every 60 s. | scheduler | `TaskQueue` |
| `QualitySampler` | `TimedAction` | Every 5 minutes, scores one completed task and emits a `QualityScored` event. | scheduler | `TaskRecordEntity` |
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE. | — | `TaskView`, `TaskQueue`, `TaskRecordEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TaskRequest(String description, String requestedBy) {}

record DraftOutput(String text, int wordCount, Instant draftedAt) {}

record NamedEntity(String name, String kind, String context) {}

record DataOutput(List<NamedEntity> entities, Instant summarisedAt) {}

record AssembledResult(String answer, List<String> toolsUsed, Instant assembledAt) {}

record TaskRecord(
    String taskId,
    String description,
    TaskStatus status,
    Optional<DraftOutput> draft,
    Optional<DataOutput> data,
    Optional<AssembledResult> result,
    Optional<String> rejectionReason,
    Optional<Integer> qualityScore,
    Optional<String> qualityRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus { QUEUED, PROCESSING, COMPLETED, FAILED, REJECTED }
```

### Events (on `TaskRecordEntity`)

`TaskCreated`, `TaskProcessing`, `TaskCompleted`, `TaskFailed`, `TaskRejected`, `QualityScored`.

### Events (on `TaskQueue`)

`TaskSubmitted { taskId, description, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ description }` → `{ taskId }`. Starts a workflow.
- `GET /api/tasks` — list all task records. Optional `?status=QUEUED|PROCESSING|COMPLETED|FAILED|REJECTED`.
- `GET /api/tasks/{id}` — one task record.
- `GET /api/tasks/sse` — server-sent events stream of every task record change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Agents as Tools"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task description, live list of task records with status pills, expand-row to see draft output, data output, the assembled answer, and quality score.

Browser title: `<title>Akka Sample: Agents as Tools</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. This baseline has no wired controls — the `controls` list in the corpus entry is empty. The eval matrix file is minimal: no `controls` array entries, only a `simplified_view` placeholder that a deployer can expand.

## 9. Agent prompts

- `TaskSupervisor` → `prompts/supervisor.md`. Selects tools and assembles the final result.
- `WriterAgent` → `prompts/writer.md`. Drafts text; returns `DraftOutput`.
- `DataAgent` → `prompts/data-agent.md`. Extracts entities; returns `DataOutput`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a writing task; record progresses QUEUED → PROCESSING → COMPLETED within 60 s; assembled result contains a `DraftOutput`; UI reflects each transition via SSE.
2. **J2** — Submit a task that needs both tools; supervisor calls writer_tool then data_tool; both outputs appear in the assembled result.
3. **J3** — Submit a task outside both tools' scope; supervisor returns REJECTED with an explanation; no partial state written.
4. **J4** — Wait after a successful completion; `QualitySampler` scores the record; the quality score appears in the UI row.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named agents-as-tools demonstrating the
delegation-supervisor-workers × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-general-agents-as-tools.
Java package io.akka.samples.agentsastools. Akka 3.6.0. HTTP port 9624.

Components to wire (exactly):
- 3 AutonomousAgents:
  * TaskSupervisor — definition() with capability(TaskAcceptance.of(SUPERVISE).maxIterationsPerTask(5))
    and tool registrations for WriterAgent.class (tool name "writer_tool") and DataAgent.class
    (tool name "data_tool"). System prompt loaded from prompts/supervisor.md. Returns
    AssembledResult{answer, toolsUsed: List<String>, assembledAt} for SUPERVISE.
  * WriterAgent — capability(TaskAcceptance.of(DRAFT).maxIterationsPerTask(3)). System prompt
    from prompts/writer.md. Returns DraftOutput{text, wordCount, draftedAt}.
  * DataAgent — capability(TaskAcceptance.of(EXTRACT).maxIterationsPerTask(3)). System prompt
    from prompts/data-agent.md. Returns DataOutput{entities: List<NamedEntity{name, kind,
    context}>, summarisedAt}.

- 1 Workflow TaskWorkflow with steps:
  createStep -> superviseStep -> persistStep.
  createStep writes TaskCreated and sets status QUEUED.
  superviseStep calls forAutonomousAgent(TaskSupervisor.class, SUPERVISE) with the task
  description; give superviseStep a 120s stepTimeout (supervisor may call both tools
  sequentially). Each tool call inside the supervisor uses a 45s budget enforced via the
  supervisor's iteration limit. On superviseStep timeout, transition to failStep that writes
  TaskFailed and ends. On supervisor returning a rejection flag, transition to rejectStep.
  persistStep calls TaskRecordEntity.complete with the AssembledResult. WorkflowSettings is
  nested inside Workflow — no import.

- 1 EventSourcedEntity TaskRecordEntity holding state TaskRecord{taskId, description,
  TaskStatus, Optional<DraftOutput> draft, Optional<DataOutput> data,
  Optional<AssembledResult> result, Optional<String> rejectionReason,
  Optional<Integer> qualityScore, Optional<String> qualityRationale,
  Instant createdAt, Optional<Instant> finishedAt}. TaskStatus enum: QUEUED, PROCESSING,
  COMPLETED, FAILED, REJECTED. Events: TaskCreated, TaskProcessing, TaskCompleted,
  TaskFailed, TaskRejected, QualityScored. Commands: createTask, markProcessing,
  completeTask, failTask, rejectTask, recordQuality, getTask.
  emptyState() returns TaskRecord.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity TaskQueue with command submitTask(description, requestedBy) emitting
  TaskSubmitted{taskId, description, requestedBy, submittedAt}.

- 1 View TaskView with row type TaskRecordRow (mirrors TaskRecord minus heavy nested payloads;
  every nullable field is Optional<T>). Table updater consumes TaskRecordEntity events.
  ONE query getAllTasks SELECT * AS tasks FROM task_view. No WHERE status filter (Akka cannot
  auto-index enum columns) — caller filters client-side.

- 1 Consumer TaskRequestConsumer subscribed to TaskQueue events; on TaskSubmitted starts a
  TaskWorkflow with the taskId as the workflow id.

- 2 TimedActions:
  * TaskSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-tasks.jsonl and calls TaskQueue.submitTask.
  * QualitySampler — every 5 minutes, queries TaskView.getAllTasks, picks the oldest
    COMPLETED task without a qualityScore, runs a 1–5 rubric judge over the assembled
    result, then calls TaskRecordEntity.recordQuality(score, rationale).

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks, GET /tasks, GET /tasks/{id},
    GET /tasks/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- AgentTasks.java declaring three Task<R> constants: SUPERVISE (AssembledResult), DRAFT
  (DraftOutput), EXTRACT (DataOutput).
- Domain records TaskRequest, DraftOutput, NamedEntity, DataOutput, AssembledResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9624 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-tasks.jsonl with 8 canned task description lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with no controls (this baseline has controls: []) and
  a simplified_view placeholder. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = task-delegation,
  decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/supervisor.md, prompts/writer.md, prompts/data-agent.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Agents as Tools", one-line pitch,
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows; no controls to display — show an
  empty-state message "No controls defined for this baseline. Add controls to eval-matrix.yaml
  to enable governance."), App UI (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: Agents as Tools</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class name.
  Each agent's branch reads a JSON file from src/main/resources/mock-responses/
  (supervisor.json, writer.json, data-agent.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes:
    supervisor.json — 4–6 AssembledResult entries, each with a 60–120 word answer,
      toolsUsed listing one or both of ["writer_tool", "data_tool"].
    writer.json — 4–6 DraftOutput entries, each with a 60–120 word text and a
      realistic wordCount.
    data-agent.json — 4–6 DataOutput entries, each with 3–5 NamedEntity objects
      whose kind values are one of: "person", "organisation", "location", "concept".
- A MockModelProvider.seedFor(taskId) helper makes the selection deterministic
  per task id so the same task produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout; superviseStep
  gets 120s, any tool-delegating step gets 45s. Default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion AgentTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9624 in application.conf.
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
