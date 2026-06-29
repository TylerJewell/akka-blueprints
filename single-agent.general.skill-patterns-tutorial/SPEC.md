# SPEC — skill-patterns-tutorial

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SkillPatternsTutorial.
**One-line pitch:** A single agent demonstrates four skill-wiring patterns — inline, file-based, external, and meta skill-creator — in one runnable service, so a developer can see each pattern work before adopting it.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain, focused on skill composition. One `SkillDemoAgent` (AutonomousAgent) handles all four task types; the surrounding components track invocation history and surface results to the UI.

The four patterns are:

- **Inline skill** — the skill's full prompt and behaviour rules are declared inside `SkillDemoAgent.definition()` as a `Skill.inline(...)` entry. No external file. The developer can see the entire skill definition by reading the agent class.
- **File-based skill** — the skill's system prompt is loaded from `src/main/resources/skills/summarizer-skill.md` at startup via `Skill.fromResource(...)`. Swapping the resource file swaps the skill's behaviour without recompiling the agent.
- **External skill** — the skill calls a registered HTTP tool (`ToolDefinition.http(...)`) that targets a lightweight in-process stub endpoint (`SkillToolStub`). The tool response is incorporated into the agent's reply, demonstrating how external data (a lookup, a calculation, an API read) is woven into a skill result.
- **Meta skill-creator** — the agent receives a user-supplied skill description in the task payload and emits a new `SkillDefinition` record. That record can be posted back to the service to register it as a fifth (deployer-defined) skill on the running agent, demonstrating dynamic skill registration.

No governance controls are wired on this sample. The blueprint's purpose is skill-pattern pedagogy, not risk management. A deployer adding production use would layer controls from other samples on top.

## 3. User-facing flows

The user opens the App UI tab.

1. The user sees four **pattern cards** in a grid — one per skill pattern. Each card shows the pattern name, a one-line description of how the skill is wired, and an **Invoke** button.
2. Clicking **Invoke** on the Inline card opens a small input panel: a `Topic` text field and a `Style` selector (concise / detailed / bullet-list). The user fills in values and clicks **Run**.
3. The UI POSTs to `/api/skill-runs` with `pattern = INLINE`, receives a `runId`, and a new row appears in the live list at the bottom of the card in `REQUESTED` state.
4. Within ~10–30 s the row transitions to `COMPLETED`. The result panel opens on the right: `patternName`, `output` text, and `wiringNote` (a one-sentence explanation of how this skill was loaded).
5. The File-based, External, and Meta skill-creator cards follow the same flow, each with a slightly different input panel. The External card shows a `Lookup key` field; the Meta card shows a `Skill description` textarea and a `Skill name` field.
6. After the meta skill-creator run completes, a **Register** button appears. Clicking it sends the returned `SkillDefinition` JSON back to `POST /api/skills` to register it on the running agent. A fifth card appears in the grid.
7. The live list at the bottom of each card shows all prior runs for that pattern, newest-first, with status pills and the first 80 characters of output.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SkillRunEndpoint` | `HttpEndpoint` | `/api/skill-runs/*` and `/api/skills/*` — invoke, list, get, register skill, SSE. `/api/metadata/*`. | — | `SkillRunEntity`, `SkillRunView`, `SkillDemoAgent` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SkillRunEntity` | `EventSourcedEntity` | Per-run lifecycle: requested → running → completed or failed. Source of truth. | `SkillRunEndpoint`, `SkillRunWorkflow` | `SkillRunView` |
| `SkillRunWorkflow` | `Workflow` | One workflow per runId. Steps: `runStep` → `recordStep`. On error: `errorStep`. | started by `SkillRunEndpoint` | `SkillDemoAgent`, `SkillRunEntity` |
| `SkillDemoAgent` | `AutonomousAgent` | The single LLM. Dispatches to the appropriate skill based on the incoming task's `pattern` field. Returns `SkillResult`. | invoked by `SkillRunWorkflow` | returns result |
| `SkillToolStub` | `HttpEndpoint` | In-process stub that responds to the external skill's tool HTTP call. Returns canned lookup data for tutorial purposes. | invoked by `SkillDemoAgent` via tool call | returns tool response |
| `SkillRunView` | `View` | Read model: one row per run for the UI list. | `SkillRunEntity` events | `SkillRunEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum SkillPattern { INLINE, FILE_BASED, EXTERNAL, META_CREATOR }

record InlineSkillInput(String topic, String style) {}        // style: concise|detailed|bullet-list
record FileBasedSkillInput(String textToSummarize) {}
record ExternalSkillInput(String lookupKey) {}
record MetaCreatorInput(String skillName, String skillDescription) {}

// Union carried as JSON + discriminator — one of the four above
record SkillRunRequest(
    String runId,
    SkillPattern pattern,
    String patternInputJson,   // serialised union; deserialised by workflow based on pattern
    String requestedBy,
    Instant requestedAt
) {}

record SkillResult(
    String patternName,        // display name matching SkillPattern
    String output,             // the skill's produced text
    String wiringNote,         // one sentence: how this skill was loaded/wired
    Instant completedAt
) {}

// Returned by META_CREATOR runs; also accepted by POST /api/skills
record SkillDefinition(
    String skillId,
    String displayName,
    String systemPrompt,
    String createdBy,
    Instant createdAt
) {}

record SkillRun(
    String runId,
    Optional<SkillRunRequest> request,
    Optional<SkillResult> result,
    Optional<String> failureReason,
    SkillRunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SkillRunStatus { REQUESTED, RUNNING, COMPLETED, FAILED }
```

Events on `SkillRunEntity`: `SkillRunRequested`, `SkillRunStarted`, `SkillRunCompleted`, `SkillRunFailed`.

Every nullable lifecycle field on the `SkillRun` state record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/skill-runs` — body `{ pattern, patternInputJson, requestedBy }` → `{ runId }`.
- `GET /api/skill-runs` — list all runs, newest-first.
- `GET /api/skill-runs/{id}` — one run.
- `GET /api/skill-runs/sse` — Server-Sent Events; one event per state transition.
- `POST /api/skills` — body `SkillDefinition` → `{ skillId }`. Registers a deployer-defined skill on the running agent.
- `GET /api/skills` — list registered deployer skills.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Skill Patterns Tutorial</title>`.

The App UI tab uses a two-column layout: a left column with the four pattern cards (Inline, File-based, External, Meta) each containing an input panel and a mini live list; and a right pane showing the selected run's full detail — pattern name, wiring note, output text, and (for meta runs) the returned `SkillDefinition` with a Register button.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

No governance controls are wired in this baseline. The eval-matrix has zero entries; the risk-survey's `compliance.capabilities` block is all-false. A deployer building on this pattern would add controls appropriate to their use case.

See `eval-matrix.yaml` and `risk-survey.yaml`.

## 9. Agent prompts

- `SkillDemoAgent` → `prompts/skill-demo-agent.md`. The single decision-making LLM. Routes to the inline, file-based, external, or meta-creator skill based on the task's `pattern` field. Each skill path has its own behaviour rules.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User invokes all four patterns via the App UI. Each produces a `SkillResult` with the correct `patternName` and a non-empty `output`.
2. **J2** — User invokes the meta skill-creator with a description of a "haiku generator" skill. The returned `SkillDefinition` is registered via the Register button and a fifth pattern card appears.
3. **J3** — User invokes the external skill pattern. The service log shows the outbound tool HTTP call to `SkillToolStub`; the result's `wiringNote` references the tool endpoint.
4. **J4** — User submits concurrent invocations of all four patterns. All four complete without interfering; each run's result appears on its card.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named skill-patterns-tutorial demonstrating the single-agent × general-domain
skill-pattern cell. Runs out of the box (no external services beyond the in-process stub).
Maven group io.akka.samples. Maven artifact single-agent-general-skill-patterns-tutorial.
Java package io.akka.samples.agentskillstutorial. Akka 3.6.0. HTTP port 9588.

Components to wire (exactly):

- 1 AutonomousAgent SkillDemoAgent — definition() returns AgentDefinition with:
    .instructions(<system prompt loaded from prompts/skill-demo-agent.md>)
    .skill(Skill.inline("inline-explainer",
        "You explain topics in the requested style. Topic and style are in the task text.",
        "Inline skill: prompt lives in SkillDemoAgent.definition() — no file load."))
    .skill(Skill.fromResource("file-based-summarizer", "skills/summarizer-skill.md"))
    .skill(Skill.withTool("external-lookup",
        ToolDefinition.http("lookup-tool", "http://localhost:9588/api/tool-stub/lookup",
            "Calls the in-process SkillToolStub to retrieve a canned value for the given key."),
        "Wrap the tool's return value in a sentence."))
    .capability(TaskAcceptance.of(INVOKE_SKILL).maxIterationsPerTask(2))
  The meta-creator path is handled inside the agent's reasoning, not as a registered Skill —
  it reads the task payload and returns a SkillDefinition JSON block.
  Output: SkillResult{patternName, output, wiringNote, completedAt}.
  The companion SkillDemoTasks.java MUST exist (Lesson 7).

- 1 Workflow SkillRunWorkflow per runId with steps:
  * runStep — emits SkillRunStarted on the entity, then calls
    componentClient.forAutonomousAgent(SkillDemoAgent.class, "skill-agent-" + runId)
      .runSingleTask(TaskDef.instructions(buildTaskText(request))) to run the agent.
    Fetches the result via forTask(taskId).result(INVOKE_SKILL).
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(1)
    .failoverTo(SkillRunWorkflow::errorStep).
  * recordStep — calls SkillRunEntity.recordResult(result). WorkflowSettings.stepTimeout 10s.
  * errorStep — calls SkillRunEntity.fail(reason). WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SkillRunEntity (one per runId). State SkillRun{runId: String,
  request: Optional<SkillRunRequest>, result: Optional<SkillResult>,
  failureReason: Optional<String>, status: SkillRunStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. SkillRunStatus: REQUESTED, RUNNING, COMPLETED, FAILED.
  Events: SkillRunRequested{request}, SkillRunStarted{}, SkillRunCompleted{result},
  SkillRunFailed{reason}. Commands: request, markRunning, recordResult, fail, getRun.
  emptyState() returns SkillRun.initial("") with all Optional fields as Optional.empty()
  and status = REQUESTED. emptyState() never references commandContext() (Lesson 3).

- 1 HttpEndpoint SkillToolStub at /api/tool-stub — GET /lookup?key={key} returns a JSON
  object { "key": "<key>", "value": "<canned-string>", "source": "in-process-stub" }.
  Five seeded keys: "alpha", "beta", "gamma", "delta", "epsilon", each returning a short
  canned phrase. Unknown keys return a 200 with value = "no-data-for-key:<key>".

- 1 View SkillRunView with row type SkillRunRow (mirrors SkillRun). ONE query:
  getAllRuns: SELECT * AS runs FROM skill_run_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * SkillRunEndpoint at /api with:
      POST /skill-runs (body {pattern, patternInputJson, requestedBy}; mints runId;
        calls SkillRunEntity.request; starts SkillRunWorkflow; returns {runId})
      GET /skill-runs (list from getAllRuns, newest-first)
      GET /skill-runs/{id} (one row)
      GET /skill-runs/sse (SSE from view stream-updates)
      POST /skills (body SkillDefinition; stores in a non-replicated in-memory registry
        on the endpoint; returns {skillId})
      GET /skills (list registered skills from in-memory registry)
      GET /api/metadata/{readme,risk-survey,eval-matrix} (serve from classpath metadata/)
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SkillDemoTasks.java declaring one Task<R> constant:
    INVOKE_SKILL = Task.name("Invoke skill")
      .description("Run the requested skill pattern and return a SkillResult")
      .resultConformsTo(SkillResult.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records: SkillRunRequest, SkillResult, SkillDefinition, SkillRun, SkillPattern,
  SkillRunStatus, InlineSkillInput, FileBasedSkillInput, ExternalSkillInput, MetaCreatorInput.

- src/main/resources/skills/summarizer-skill.md — loaded by Skill.fromResource(). Contains
  a focused summarization prompt: read the input text and return a 3-bullet summary, ending
  each bullet with the key takeaway. Includes the wiring note "File-based skill: prompt
  loaded from src/main/resources/skills/summarizer-skill.md at startup via Skill.fromResource."

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9588 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-inputs.jsonl with four seeded invocation payloads,
  one per pattern, that the UI's "Load example" button can fill in.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.

- eval-matrix.yaml at the project root — schema_version: 1, zero controls.

- risk-survey.yaml at the project root — pre-filled for general/tutorial use; deployer
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/skill-demo-agent.md loaded as the agent system prompt.

- README.md at the project root.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs. App UI tab: two-column layout (left = four pattern cards with input panels
  and mini live lists; right = selected-run detail with pattern name, wiring note, output,
  and optional SkillDefinition + Register button for meta runs).
  Browser title exactly: <title>Akka Sample: Skill Patterns Tutorial</title>.
  No subtitle on the Overview tab.

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
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with a per-task dispatch on INVOKE_SKILL. Reads
  src/main/resources/mock-responses/invoke-skill.json and picks an entry based on the
  pattern field in the task text. The file contains four entries — one per SkillPattern —
  each with a realistic output and wiringNote. Plus one additional entry per pattern for
  variety (8 total). Selection is deterministic per runId via MockModelProvider.seedFor(runId).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SkillDemoAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. SkillDemoTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (runStep 90s, recordStep 10s,
  errorStep 5s).
- Lesson 6: every nullable lifecycle field on SkillRun is Optional<T>. View updater wraps
  with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: SkillDemoTasks.java with INVOKE_SKILL is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9588 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: generated static-resources/index.html includes mermaid CSS overrides and
  themeVariables block (nodeTextColor, stateLabelColor, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute; exactly five <section
  class="tab-panel"> elements in the DOM.
- Single-agent invariant: exactly ONE AutonomousAgent (SkillDemoAgent). The in-process tool
  stub (SkillToolStub) is an HttpEndpoint, not an agent.
- The meta skill-creator output is a SkillDefinition record; it is NOT a new AutonomousAgent
  or a side-effect that registers itself — the user explicitly triggers registration via
  POST /api/skills. This keeps the single-agent invariant and teaches the deployer-controlled
  registration pattern.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
