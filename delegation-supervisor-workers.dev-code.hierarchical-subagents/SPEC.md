# SPEC — hierarchical-subagents

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Hierarchical Sub-Agent Delegation.
**One-line pitch:** Submit a coding task; an orchestrator validates each specialist's input, delegates design to an ArchitectSpecialist and implementation outlining to an ImplementationSpecialist in parallel, then synthesises one structured solution summary.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern in the dev-code domain, wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, validates each sub-agent's input schema before invocation (before-agent-invocation guardrail), gathers their results, and asks a third AutonomousAgent to synthesise a unified solution summary. The blueprint also demonstrates **eval-event** governance that scores each specialist contribution before synthesis so the orchestrator knows the quality of its inputs.

## 3. User-facing flows

The user opens the App UI tab and submits a coding task description via the form.

1. The system creates a `DelegationTask` record in `SCOPING` and starts a `DelegationWorkflow`.
2. The TaskOrchestrator decomposes the description into two parallel work items: a design query for the ArchitectSpecialist and an implementation query for the ImplementationSpecialist.
3. Before invoking each specialist, the workflow runs a before-agent-invocation guardrail that validates the query payload against each specialist's input schema. A malformed payload short-circuits to `REJECTED`.
4. The workflow forks: both specialists run concurrently. Each returns a typed payload.
5. Before synthesis, ContributionEvaluator scores the two contributions (1–5). Scores are stored on the task record.
6. The TaskOrchestrator merges the contributions into a `SynthesisedSolution { summary, design, implementation, validationVerdict }`.
7. A validation guardrail vets the synthesised output; if it fails, the task moves to `REJECTED`. Otherwise, the task moves to `SYNTHESISED`.
8. If either specialist times out after 60 seconds, the workflow short-circuits: it asks the TaskOrchestrator to synthesise from whichever side returned, and the task enters `DEGRADED`.

A `TaskSimulator` (TimedAction) drips a sample coding task every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TaskOrchestrator` | `AutonomousAgent` | Decomposes the task description, synthesises the merged contributions, runs the output validation guardrail. | `DelegationWorkflow` | returns typed result to workflow |
| `ArchitectSpecialist` | `AutonomousAgent` | Produces a design proposal: approach, component list, tradeoffs. | `DelegationWorkflow` | — |
| `ImplementationSpecialist` | `AutonomousAgent` | Produces an implementation outline: pseudo-code sketch and risk list. | `DelegationWorkflow` | — |
| `DelegationWorkflow` | `Workflow` | Validates sub-agent inputs before invocation, coordinates the parallel fan-out, the contribution eval, the synthesis, the output guardrail. | `TaskEndpoint`, `TaskQueueConsumer` | `TaskDelegationEntity` |
| `TaskDelegationEntity` | `EventSourcedEntity` | Holds the task's lifecycle (scoping → delegating → synthesised / degraded / rejected). | `DelegationWorkflow` | `DelegationView` |
| `TaskQueue` | `EventSourcedEntity` | Logs each submitted task for replay/audit. | `TaskEndpoint`, `TaskSimulator` | `TaskQueueConsumer` |
| `DelegationView` | `View` | List-of-tasks read model. | `TaskDelegationEntity` events | `TaskEndpoint` |
| `TaskQueueConsumer` | `Consumer` | Listens to `TaskQueue` events and starts a workflow per submission. | `TaskQueue` events | `DelegationWorkflow` |
| `TaskSimulator` | `TimedAction` | Drips a sample coding task every 60 s. | scheduler | `TaskQueue` |
| `ContributionEvaluator` | `TimedAction` | Scores specialist contributions every 5 minutes; emits a `ContributionScored` event. | scheduler | `TaskDelegationEntity` |
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE. | — | `DelegationView`, `TaskQueue`, `TaskDelegationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TaskSubmission(String description, String requestedBy) {}

record DelegationPlan(String designQuery, String implementationQuery) {}

record DesignProposal(String approach, List<String> components,
                     List<String> tradeoffs, Instant proposedAt) {}

record ImplementationOutline(String pseudoCode, List<String> risks,
                             Instant outlinedAt) {}

record SynthesisedSolution(String summary, DesignProposal design,
                           ImplementationOutline implementation,
                           String validationVerdict, Instant synthesisedAt) {}

record DelegationTask(
    String taskId,
    String description,
    TaskStatus status,
    Optional<DesignProposal> design,
    Optional<ImplementationOutline> implementation,
    Optional<SynthesisedSolution> synthesised,
    Optional<String> failureReason,
    Optional<Integer> architectScore,
    Optional<Integer> implementationScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TaskStatus { SCOPING, DELEGATING, SYNTHESISED, DEGRADED, REJECTED }
```

### Events (on `TaskDelegationEntity`)

`TaskCreated`, `DesignAttached`, `ImplementationAttached`, `TaskSynthesised`, `TaskDegraded`, `TaskRejected`, `ContributionScored`.

### Events (on `TaskQueue`)

`TaskSubmitted { taskId, description, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ description }` → `{ taskId }`. Starts a workflow.
- `GET /api/tasks` — list all tasks. Optional `?status=SCOPING|DELEGATING|SYNTHESISED|DEGRADED|REJECTED`.
- `GET /api/tasks/{id}` — one task.
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Hierarchical Sub-Agent Delegation"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task description, live list of tasks with status pills, expand-row to see design proposal, implementation outline, synthesised summary, and contribution scores.

Browser title: `<title>Akka Sample: Hierarchical Sub-Agent Delegation</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — input guardrail** (`before-agent-invocation` on each specialist invocation in `DelegationWorkflow`): validates the DelegationPlan's query fields against each specialist's expected input schema before the call is made. Blocking. Failure → `REJECTED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `ContributionEvaluator` (TimedAction) picks tasks in `DELEGATING` or `SYNTHESISED` state every 5 minutes, runs a 1–5 rubric judge over each specialist's contribution, and emits a `ContributionScored` event.

## 9. Agent prompts

- `TaskOrchestrator` → `prompts/task-orchestrator.md`. Decomposes the coding task into specialist queries; later synthesises both contributions into the final solution summary.
- `ArchitectSpecialist` → `prompts/architect-specialist.md`. Produces a design proposal; returns `DesignProposal`.
- `ImplementationSpecialist` → `prompts/implementation-specialist.md`. Produces an implementation outline; returns `ImplementationOutline`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a task; task progresses SCOPING → DELEGATING → SYNTHESISED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a specialist timeout (set `ArchitectSpecialist` timeout to 1 s); task enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a malformed DelegationPlan (missing implementationQuery); before-agent-invocation guardrail fires and task enters REJECTED before either specialist is called.
4. **J4** — Wait for ContributionEvaluator; task shows architectScore and implementationScore on the App UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named hierarchical-subagents demonstrating the
delegation-supervisor-workers × dev-code cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-dev-code-hierarchical-subagents.
Java package io.akka.samples.hierarchicalsubagentdelegation. Akka 3.6.0.
HTTP port 9882.

Components to wire (exactly):
- 3 AutonomousAgents:
  * TaskOrchestrator — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/task-orchestrator.md. Returns DelegationPlan{designQuery, implementationQuery}
    for DECOMPOSE and SynthesisedSolution{summary, design, implementation, validationVerdict,
    synthesisedAt} for SYNTHESISE.
  * ArchitectSpecialist — capability(TaskAcceptance.of(DESIGN).maxIterationsPerTask(3)). System
    prompt from prompts/architect-specialist.md. Returns DesignProposal{approach,
    components: List<String>, tradeoffs: List<String>, proposedAt}.
  * ImplementationSpecialist — capability(TaskAcceptance.of(OUTLINE).maxIterationsPerTask(2)).
    System prompt from prompts/implementation-specialist.md. Returns
    ImplementationOutline{pseudoCode, risks: List<String>, outlinedAt}.

- 1 Workflow DelegationWorkflow with steps:
  decomposeStep -> guardrailStep -> [parallel] designStep, outlineStep -> joinStep ->
  evaluateStep -> synthesiseStep -> validateStep -> emitStep.
  decomposeStep calls forAutonomousAgent(TaskOrchestrator.class, DECOMPOSE).
  guardrailStep validates the returned DelegationPlan: both fields non-null and non-blank;
  on failure, transition to rejectStep (emits TaskRejected) and end.
  designStep and outlineStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(DelegationWorkflow::designStep, ofSeconds(60)) and
  stepTimeout(DelegationWorkflow::outlineStep, ofSeconds(60)). On either timeout, transition to
  degradeStep that calls synthesiseStep with whichever side returned, then ends with TaskDegraded.
  evaluateStep runs a 1-5 rubric synchronously over both contributions and stores the scores
  via TaskDelegationEntity.recordContributionScores before proceeding.
  synthesiseStep calls forAutonomousAgent(TaskOrchestrator.class, SYNTHESISE) with the merged
  contributions; give synthesiseStep a 90s stepTimeout. validateStep runs the deterministic
  vetter + LLM judge on the synthesised output; on failure, end with TaskRejected.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity TaskDelegationEntity holding state DelegationTask{taskId, description,
  TaskStatus, Optional<DesignProposal> design, Optional<ImplementationOutline> implementation,
  Optional<SynthesisedSolution> synthesised, Optional<String> failureReason,
  Optional<Integer> architectScore, Optional<Integer> implementationScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  TaskStatus enum: SCOPING, DELEGATING, SYNTHESISED, DEGRADED, REJECTED. Events: TaskCreated,
  DesignAttached, ImplementationAttached, TaskSynthesised, TaskDegraded, TaskRejected,
  ContributionScored. Commands: createTask, attachDesign, attachImplementation, synthesise,
  degrade, reject, recordContributionScores, recordEval, getTask.
  emptyState() returns DelegationTask.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity TaskQueue with command enqueueTask(description, requestedBy) emitting
  TaskSubmitted{taskId, description, requestedBy, submittedAt}.

- 1 View DelegationView with row type DelegationTaskRow (mirrors DelegationTask minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  TaskDelegationEntity events. ONE query getAllTasks SELECT * AS tasks FROM delegation_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer TaskQueueConsumer subscribed to TaskQueue events; on TaskSubmitted starts a
  DelegationWorkflow with the taskId as the workflow id.

- 2 TimedActions:
  * TaskSimulator — every 60s, reads next line from
    src/main/resources/sample-events/coding-tasks.jsonl and calls TaskQueue.enqueueTask.
  * ContributionEvaluator — every 5 minutes, queries DelegationView.getAllTasks, picks any
    SYNTHESISED task without architectScore, runs a 1-5 rubric judge over each contribution,
    then calls TaskDelegationEntity.recordContributionScores(architectScore, implementationScore,
    rationale).

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks, GET /tasks, GET /tasks/{id},
    GET /tasks/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- DelegationTasks.java declaring four Task<R> constants: DECOMPOSE (DelegationPlan), DESIGN
  (DesignProposal), OUTLINE (ImplementationOutline), SYNTHESISE (SynthesisedSolution).
- Domain records DelegationPlan, DesignProposal, ImplementationOutline, SynthesisedSolution.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9882 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/coding-tasks.jsonl with 8 canned coding task lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 input guardrail
  before-agent-invocation, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = code-task-
  delegation, decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/task-orchestrator.md, prompts/architect-specialist.md,
  prompts/implementation-specialist.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Hierarchical Sub-Agent Delegation",
  one-line pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Hierarchical Sub-Agent Delegation</title>.
  No subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (task-orchestrator.json,
  architect-specialist.json, implementation-specialist.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return
  shape.
- Per-agent mock-response shapes for THIS blueprint:
    task-orchestrator.json — list of either DelegationPlan or SynthesisedSolution objects.
      4–6 DelegationPlan entries (designQuery + implementationQuery pairs for realistic
      coding scenarios) and 4–6 SynthesisedSolution entries (each with an 80–150 word
      summary, a DesignProposal, an ImplementationOutline, validationVerdict = "ok").
    architect-specialist.json — 4–6 DesignProposal entries, each with a one-sentence
      approach, 3–5 component names, and 2–4 tradeoff statements.
    implementation-specialist.json — 4–6 ImplementationOutline entries, each with a
      10–20 line pseudoCode sketch and 2–4 risk bullets.
- A MockModelProvider.seedFor(taskId) helper makes the selection deterministic per task
  id so the same task produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion DelegationTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9882 in application.conf.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
