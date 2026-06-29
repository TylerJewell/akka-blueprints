# SPEC — lats-tree-search

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** LATS Tree Search.
**One-line pitch:** Submit a problem; a search agent expands candidate next-actions at each tree node; a reflector agent scores each candidate and backpropagates the score; the workflow selects the highest-scoring branch and continues until a terminal node is reached or the node budget is exhausted.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between an expansion agent (`SearchAgent`) and a scoring agent (`ReflectorAgent`), backpropagating scores to navigate the tree toward a terminal solution. The blueprint also demonstrates one governance mechanism — an **eval-event** that records every node's reflection verdict for downstream quality measurement and search-efficiency analysis.

## 3. User-facing flows

The user opens the App UI tab and submits a problem statement (a task description plus a node budget).

1. The system creates a `SearchTree` record in `EXPANDING` and starts a `TreeSearchWorkflow`.
2. The Search Agent expands the root node by generating `expansionWidth` (default 3) candidate next-actions.
3. The Reflector scores each candidate on a 1–10 scale, provides a structured `ReflectionNote` (one rationale sentence plus an actionability assessment), and returns a `NodeScore`.
4. The workflow selects the candidate with the highest score, advances the best-path cursor to that node, and records a backpropagation delta on all sibling nodes.
5. If the selected node is terminal (the Reflector marks it `IS_TERMINAL = true`) the workflow transitions to `SOLVED` with the best path and the terminal node's content.
6. If the selected node is not terminal and the node budget remains, the workflow expands that node and repeats from step 2.
7. If the node budget is exhausted before a terminal node is found, the workflow ends with `BUDGET_EXHAUSTED`, preserving the best partial path and every node for audit.

A `ProblemSimulator` (TimedAction) drips a canned problem every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SearchAgent` | `AutonomousAgent` | Expands a tree node by generating `expansionWidth` candidate next-actions. | `TreeSearchWorkflow` | returns `NodeExpansion` to workflow |
| `ReflectorAgent` | `AutonomousAgent` | Scores a candidate node; returns a `NodeScore` with a `IS_TERMINAL` flag and a `ReflectionNote`. | `TreeSearchWorkflow` | returns `NodeScore` to workflow |
| `TreeSearchWorkflow` | `Workflow` | Orchestrates expand → reflect → select → expand cycle; halts on terminal or budget exhaustion. | `SearchEndpoint`, `ProblemConsumer` | `SearchTreeEntity` |
| `SearchTreeEntity` | `EventSourcedEntity` | Holds the full tree: every node, every expansion, every reflection, best-path cursor. | `TreeSearchWorkflow` | `TreeView` |
| `ProblemQueue` | `EventSourcedEntity` | Logs each submitted problem for replay and audit. | `SearchEndpoint`, `ProblemSimulator` | `ProblemConsumer` |
| `TreeView` | `View` | List-of-trees read model. | `SearchTreeEntity` events | `SearchEndpoint` |
| `ProblemConsumer` | `Consumer` | Subscribes to `ProblemQueue` events; starts a workflow per submission. | `ProblemQueue` events | `TreeSearchWorkflow` |
| `ProblemSimulator` | `TimedAction` | Drips a sample problem every 90 s from `sample-events/problems.jsonl`. | scheduler | `ProblemQueue` |
| `EvalSampler` | `TimedAction` | Every 45 s, scans `TreeView`, records an `EvalRecorded` event for any node that has been reflected but not yet sampled. | scheduler | `SearchTreeEntity` |
| `SearchEndpoint` | `HttpEndpoint` | `/api/trees/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `TreeView`, `ProblemQueue`, `SearchTreeEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Problem(String taskDescription, int nodeBudget, String submittedBy) {}

record NodeExpansion(String nodeId, List<CandidateNode> candidates, Instant expandedAt) {}

record CandidateNode(String candidateId, String actionDescription, String reasoning) {}

record ReflectionNote(String rationale, String actionability) {}

record NodeScore(
    String candidateId,
    int score,
    boolean isTerminal,
    ReflectionNote note,
    double backpropDelta,
    Instant reflectedAt
) {}

record TreeNode(
    String nodeId,
    String parentNodeId,
    int depth,
    String actionDescription,
    Optional<NodeScore> score,
    NodeStatus nodeStatus
) {}

record SearchTree(
    String treeId,
    String taskDescription,
    int nodeBudget,
    int expansionWidth,
    TreeStatus status,
    List<TreeNode> nodes,
    List<String> bestPath,
    Optional<String> terminalNodeId,
    Optional<String> terminalContent,
    Optional<String> exhaustionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TreeStatus { EXPANDING, REFLECTING, SOLVED, BUDGET_EXHAUSTED }

enum NodeStatus { PENDING, SELECTED, PRUNED, TERMINAL }
```

### Events (on `SearchTreeEntity`)

`TreeCreated`, `NodeExpanded`, `NodeReflected`, `BestPathAdvanced`, `BackpropRecorded`, `TreeSolved`, `BudgetExhausted`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/trees` — body `{ taskDescription, nodeBudget?, submittedBy? }` → `{ treeId }`. Starts a workflow.
- `GET /api/trees` — list all trees. Optional `?status=EXPANDING|REFLECTING|SOLVED|BUDGET_EXHAUSTED`.
- `GET /api/trees/{id}` — one tree (including every node, every score, and the best path).
- `GET /api/trees/sse` — server-sent events stream of every tree change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "LATS Tree Search"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue).
- **App UI** — form to submit a problem, live list of trees with status pills, click-to-expand per-node timeline showing each expansion, each candidate scored, the selected node, and backpropagation deltas.

Browser title: `<title>Akka Sample: LATS Tree Search</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event** (`on-decision-eval`): every node's reflection score is recorded as an `EvalRecorded` event with `{ nodeId, depth, score, isTerminal, backpropDelta }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-node timeline and in `/api/trees/{id}`.

## 9. Agent prompts

- `SearchAgent` → `prompts/search-agent.md`. Expands a tree node by generating candidate next-actions, each with a description and a reasoning chain.
- `ReflectorAgent` → `prompts/reflector-agent.md`. Scores a candidate node on a 1–10 scale, determines whether it represents a terminal solution, provides a structured `ReflectionNote`, and computes a backpropagation delta.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — solve within budget** — Submit a problem; tree progresses `EXPANDING` → `REFLECTING` → `SOLVED` within the node budget; the App UI shows every node's expansion and score.
2. **J2 — budget exhaustion** — Submit a problem designed to never reach a terminal node; tree progresses through every node budget cycle and lands in `BUDGET_EXHAUSTED` with the best partial path preserved.
3. **J3 — backpropagation visible** — After a reflection cycle, sibling candidates that were not selected show a non-zero `backpropDelta`; the UI renders these deltas in the per-node expansion view.
4. **J4 — eval-event timeline** — The expanded view of any tree shows one `EvalRecorded` event per reflected node and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named lats-tree-search demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-lats-tree-search.
Java package io.akka.samples.latslanguageagenttreesearch. Akka 3.6.0. HTTP port 9913.

Components to wire (exactly):
- 2 AutonomousAgents:
  * SearchAgent — definition() with
    capability(TaskAcceptance.of(EXPAND_NODE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/search-agent.md. Returns
    NodeExpansion{nodeId, candidates: List<CandidateNode>, expandedAt} for
    EXPAND_NODE. The EXPAND_NODE task takes (taskDescription, currentNodeId,
    currentActionDescription, depth) as inputs.
  * ReflectorAgent — definition() with
    capability(TaskAcceptance.of(REFLECT_NODE).maxIterationsPerTask(2)). System
    prompt from prompts/reflector-agent.md. Returns NodeScore{candidateId,
    score, isTerminal, note: ReflectionNote, backpropDelta, reflectedAt}
    where score is a 1–10 integer and backpropDelta is a double in [0.0, 1.0].

- 1 Workflow TreeSearchWorkflow with steps:
    startStep -> expandStep -> reflectStep -> selectStep ->
    [selected.isTerminal? solveStep : (nodeCount < nodeBudget ?
       expandStep with selected node : exhaustStep)] -> END.
  expandStep calls forAutonomousAgent(SearchAgent.class, treeId).runSingleTask(
    EXPAND_NODE) then forTask(taskId).result(EXPAND_NODE). reflectStep
  iterates over each CandidateNode and calls
  forAutonomousAgent(ReflectorAgent.class, candidateId).runSingleTask(
    REFLECT_NODE) for each. selectStep picks the candidate with the highest
  score, records backpropDelta on siblings, advances bestPath. solveStep
  emits TreeSolved. exhaustStep emits BudgetExhausted with the best partial
  path and an exhaustionReason. Override settings() with stepTimeout(90s) on
  expandStep and reflectStep, and defaultStepRecovery(maxRetries(2).
  failoverTo(exhaustStep)).

- 1 EventSourcedEntity SearchTreeEntity holding state SearchTree{treeId,
  taskDescription, nodeBudget, expansionWidth, TreeStatus status,
  List<TreeNode> nodes, List<String> bestPath,
  Optional<String> terminalNodeId, Optional<String> terminalContent,
  Optional<String> exhaustionReason, Instant createdAt,
  Optional<Instant> finishedAt}. TreeStatus enum: EXPANDING, REFLECTING,
  SOLVED, BUDGET_EXHAUSTED. NodeStatus enum: PENDING, SELECTED, PRUNED,
  TERMINAL. Events: TreeCreated, NodeExpanded, NodeReflected,
  BestPathAdvanced, BackpropRecorded, TreeSolved, BudgetExhausted,
  EvalRecorded. Commands: createTree, recordExpansion, recordReflection,
  advanceBestPath, recordBackprop, solveTree, exhaustBudget, recordEval,
  getTree. emptyState() returns SearchTree.initial("", "", 20, 3) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity ProblemQueue with command submitProblem(taskDescription,
  nodeBudget, submittedBy) emitting ProblemSubmitted{treeId, taskDescription,
  nodeBudget, submittedBy, submittedAt}.

- 1 View TreeView with row type TreeRow (mirrors SearchTree; the nodes list is
  preserved as-is — the list is bounded at nodeBudget so size stays
  reasonable). Table updater consumes SearchTreeEntity events. ONE query
  getAllTrees SELECT * AS trees FROM tree_view. No WHERE status filter —
  caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer ProblemConsumer subscribed to ProblemQueue events; on
  ProblemSubmitted starts a TreeSearchWorkflow with the treeId as the
  workflow id.

- 2 TimedActions:
  * ProblemSimulator — every 90s, reads next line from
    src/main/resources/sample-events/problems.jsonl and calls
    ProblemQueue.submitProblem.
  * EvalSampler — every 45s, queries TreeView.getAllTrees, finds nodes
    with a reflection score that has not yet been recorded as an
    EvalRecorded event, and calls SearchTreeEntity.recordEval(nodeId,
    depth, score, isTerminal, backpropDelta). Idempotent per (treeId, nodeId).

- 2 HttpEndpoints:
  * SearchEndpoint at /api with POST /trees, GET /trees, GET /trees/{id},
    GET /trees/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /trees body
    is {taskDescription, nodeBudget?, submittedBy?}; missing nodeBudget
    defaults to 20, missing submittedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- SearchTasks.java declaring two Task<R> constants: EXPAND_NODE (resultConformsTo
  NodeExpansion), REFLECT_NODE (NodeScore).
- Domain records NodeExpansion, CandidateNode, ReflectionNote, NodeScore,
  TreeNode, SearchTree; enums TreeStatus, NodeStatus.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9913 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  lats-tree-search.search.node-budget = 20,
  lats-tree-search.search.expansion-width = 3, and
  lats-tree-search.search.score-threshold = 8, overridable by env var.
- src/main/resources/sample-events/problems.jsonl with 8 canned problem
  lines, each shaped {"taskDescription":"...", "nodeBudget":20}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control (E1 eval-event
  on-decision-eval) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = tree-search-reasoning,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/search-agent.md, prompts/reflector-agent.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: LATS Tree Search",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (form + live list with status pills, click-to-expand
  per-node timeline). Browser title exactly:
  <title>Akka Sample: LATS Tree Search</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning
        the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: search-agent.json, reflector-agent.json), picks one
  entry pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    search-agent.json — 6 NodeExpansion entries. Each contains 3
      CandidateNode entries with distinct actionDescriptions and reasoning
      chains covering different problem-solving directions (e.g., decompose,
      analogise, reframe). At least one expansion per 6 covers a leaf
      situation likely to be scored terminal by the reflector.
    reflector-agent.json — 6 NodeScore entries. Three return isTerminal=false
      with score in [4, 7] and a ReflectionNote with a rationale and
      actionability. Two return isTerminal=false with score in [1, 3]
      (dead-end branch). One returns isTerminal=true with score=9 and a
      ReflectionNote explaining why the node constitutes a complete solution.
- A MockModelProvider.seedFor(treeId, nodeId) helper makes the selection
  deterministic per (treeId, nodeId) so the same tree in dev produces the
  same trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. SearchAgent
  and ReflectorAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a SearchTasks companion declaring the two Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(90s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the SearchTree row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: SearchTasks.java is mandatory; generating SearchAgent or
  ReflectorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9913, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK
  in narrative, marketing tone, competitor brand names) do not appear in
  any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the
  reference does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements; removed panels are deleted from
  the HTML, not hidden with display:none.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
