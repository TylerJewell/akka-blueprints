# SPEC — research

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Multi-Agent Research Brief.
**One-line pitch:** Type a topic; a coordinator delegates findings to a Researcher and analysis to an Analyst in parallel, then synthesises a single research brief.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise a unified brief. The blueprint also demonstrates an **output guardrail** that vets the user-facing brief before it is returned, and **eval-event** governance that samples the coordinator's synthesis decision for a quality score.

## 3. User-facing flows

The user opens the App UI tab and submits a topic via the form.

1. The system creates a `ResearchBrief` record in `PLANNING` and starts a `ResearchWorkflow`.
2. The Coordinator decomposes the topic into two parallel work items: a findings query for the Researcher, an analytical question for the Analyst.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the two payloads into a `SynthesisedBrief { summary, findings, analysis, guardrailVerdict }`.
5. An output guardrail vets the synthesised brief; if it fails, the brief moves to `BLOCKED`. Otherwise, the brief moves to `SYNTHESISED`.
6. If either worker times out after 60 seconds, the workflow short-circuits: it asks the Coordinator to synthesise from whichever side returned, and the brief enters `DEGRADED`.

A `RequestSimulator` (TimedAction) drips a sample topic every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchCoordinator` | `AutonomousAgent` | Decomposes the topic, synthesises the merged brief, runs the output guardrail. | `ResearchWorkflow` | returns typed result to workflow |
| `Researcher` | `AutonomousAgent` | Gathers findings for the topic. Seeded "search tool" returns canned results. | `ResearchWorkflow` | — |
| `Analyst` | `AutonomousAgent` | Interprets the question; proposes implications. | `ResearchWorkflow` | — |
| `ResearchWorkflow` | `Workflow` | Coordinates the parallel fan-out, the synthesis, the guardrail. | `ResearchEndpoint`, `ResearchRequestConsumer` | `ResearchBriefEntity` |
| `ResearchBriefEntity` | `EventSourcedEntity` | Holds the brief's lifecycle (planning → in-progress → synthesised / degraded / blocked). | `ResearchWorkflow` | `ResearchView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted topic for replay/audit. | `ResearchEndpoint`, `RequestSimulator` | `ResearchRequestConsumer` |
| `ResearchView` | `View` | List-of-briefs read model. | `ResearchBriefEntity` events | `ResearchEndpoint` |
| `ResearchRequestConsumer` | `Consumer` | Listens to `RequestQueue` events and starts a workflow per submission. | `RequestQueue` events | `ResearchWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample topic every 60 s. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one synthesised brief every 5 minutes for eval scoring; emits an `EvalScored` event. | scheduler | `ResearchBriefEntity` |
| `ResearchEndpoint` | `HttpEndpoint` | `/api/research/*` — submit, get, list, SSE. | — | `ResearchView`, `RequestQueue`, `ResearchBriefEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TopicRequest(String topic, String requestedBy) {}

record FindingsBundle(List<Finding> findings, Instant gatheredAt) {}
record Finding(String headline, String source, String content) {}

record AnalyticalReport(String thesis, List<String> implications, Instant analysedAt) {}

record WorkPlan(String researchQuery, String analyticalQuestion) {}

record SynthesisedBrief(String summary, FindingsBundle findings, AnalyticalReport analysis,
                       String guardrailVerdict, Instant synthesisedAt) {}

record ResearchBrief(
    String briefId,
    String topic,
    BriefStatus status,
    Optional<FindingsBundle> findings,
    Optional<AnalyticalReport> analysis,
    Optional<SynthesisedBrief> synthesised,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BriefStatus { PLANNING, IN_PROGRESS, SYNTHESISED, DEGRADED, BLOCKED }
```

### Events (on `ResearchBriefEntity`)

`BriefCreated`, `FindingsAttached`, `AnalysisAttached`, `BriefSynthesised`, `BriefDegraded`, `BriefBlocked`, `EvalScored`.

### Events (on `RequestQueue`)

`TopicSubmitted { briefId, topic, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/research` — body `{ topic }` → `{ briefId }`. Starts a workflow.
- `GET /api/research` — list all briefs. Optional `?status=PLANNING|IN_PROGRESS|SYNTHESISED|DEGRADED|BLOCKED`.
- `GET /api/research/{id}` — one brief.
- `GET /api/research/sse` — server-sent events stream of every brief change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Multi-Agent Research Brief"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a topic, live list of briefs with status pills, expand-row to see findings + analysis + synthesised summary + eval score.

Browser title: `<title>Akka Sample: Multi-Agent Research Brief</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `ResearchCoordinator`): vets the synthesised brief for hallucinated citations and explicit policy violations. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one synthesised brief every 5 minutes and emits an `EvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `ResearchCoordinator` → `prompts/coordinator.md`. Decomposes the brief into work items; later synthesises results into the final brief.
- `Researcher` → `prompts/researcher.md`. Gathers findings; returns `FindingsBundle`.
- `Analyst` → `prompts/analyst.md`. Interprets the question; returns `AnalyticalReport`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a topic; brief progresses PLANNING → IN_PROGRESS → SYNTHESISED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `Researcher` timeout to 1 s); brief enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a guardrail failure (Coordinator returns flagged content); brief enters BLOCKED.
4. **J4** — Wait after a successful synthesis; the brief's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named research demonstrating the
delegation-supervisor-workers × research-intel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact research.
Java package io.akka.samples.research. Akka 3.6.0. HTTP port 9762.

Components to wire (exactly):
- 3 AutonomousAgents:
  * ResearchCoordinator — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/coordinator.md. Returns WorkPlan{researchQuery, analyticalQuestion} for DECOMPOSE
    and SynthesisedBrief{summary, findings, analysis, guardrailVerdict, synthesisedAt} for SYNTHESISE.
  * Researcher — capability(TaskAcceptance.of(GATHER).maxIterationsPerTask(3)). System prompt
    from prompts/researcher.md. Returns FindingsBundle{findings: List<Finding{headline, source,
    content}>, gatheredAt}.
  * Analyst — capability(TaskAcceptance.of(ANALYSE).maxIterationsPerTask(2)). System prompt from
    prompts/analyst.md. Returns AnalyticalReport{thesis, implications: List<String>, analysedAt}.

- 1 Workflow ResearchWorkflow with steps:
  decomposeStep -> [parallel] researchStep, analyseStep -> joinStep -> synthesiseStep -> guardrailStep -> emitStep.
  decomposeStep calls forAutonomousAgent(ResearchCoordinator.class, DECOMPOSE).
  researchStep and analyseStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(ResearchWorkflow::researchStep, ofSeconds(60)) and
  stepTimeout(ResearchWorkflow::analyseStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls synthesiseStep with whichever side returned, then ends with BriefDegraded.
  synthesiseStep calls forAutonomousAgent(ResearchCoordinator.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. guardrailStep runs the deterministic vetter +
  LLM judge on the synthesised content; on failure, end with BriefBlocked. WorkflowSettings is
  nested inside Workflow — no import.

- 1 EventSourcedEntity ResearchBriefEntity holding state ResearchBrief{briefId, topic,
  BriefStatus, Optional<FindingsBundle> findings, Optional<AnalyticalReport> analysis,
  Optional<SynthesisedBrief> synthesised, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. BriefStatus enum: PLANNING, IN_PROGRESS, SYNTHESISED,
  DEGRADED, BLOCKED. Events: BriefCreated, FindingsAttached, AnalysisAttached,
  BriefSynthesised, BriefDegraded, BriefBlocked, EvalScored. Commands: createBrief,
  attachFindings, attachAnalysis, synthesise, degrade, block, recordEval, getBrief.
  emptyState() returns ResearchBrief.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity RequestQueue with command enqueueTopic(topic, requestedBy) emitting
  TopicSubmitted{briefId, topic, requestedBy, submittedAt}.

- 1 View ResearchView with row type ResearchBriefRow (mirrors ResearchBrief minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  ResearchBriefEntity events. ONE query getAllBriefs SELECT * AS briefs FROM brief_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer ResearchRequestConsumer subscribed to RequestQueue events; on TopicSubmitted
  starts a ResearchWorkflow with the briefId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/research-topics.jsonl and calls RequestQueue.enqueueTopic.
  * EvalSampler — every 5 minutes, queries ResearchView.getAllBriefs, picks the oldest
    SYNTHESISED brief without an evalScore, runs a 1–5 rubric judge over the synthesised
    content, then calls ResearchBriefEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with POST /research, GET /research, GET /research/{id},
    GET /research/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ResearchTasks.java declaring four Task<R> constants: DECOMPOSE (WorkPlan), GATHER
  (FindingsBundle), ANALYSE (AnalyticalReport), SYNTHESISE (SynthesisedBrief).
- Domain records WorkPlan, Finding, FindingsBundle, AnalyticalReport, SynthesisedBrief.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9762 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/research-topics.jsonl with 8 canned topic lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = research-
  synthesis, decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md, prompts/researcher.md, prompts/analyst.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Multi-Agent Research Brief", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Multi-Agent Research Brief</title>. No
  subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (coordinator.json,
  researcher.json, analyst.json), picks one entry pseudo-randomly per call,
  and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    coordinator.json — list of either WorkPlan or SynthesisedBrief objects.
      4–6 WorkPlan entries (researchQuery + analyticalQuestion pairs) and
      4–6 SynthesisedBrief entries (each with an 80–150 word summary, a 3–6
      finding bundle, a 3–6 implication analysis, guardrailVerdict = "ok").
    researcher.json — 4–6 FindingsBundle entries, each with 3–6 findings
      whose source values are believable (e.g., "EUR-Lex 2024/1689",
      "EU AI Act Annex III", "unsourced — knowledge").
    analyst.json — 4–6 AnalyticalReport entries, each with a one-sentence
      thesis taking a position and 3–6 short implication bullets.
- A MockModelProvider.seedFor(briefId) helper makes the selection
  deterministic per brief id so the same brief produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion ResearchTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9762 in application.conf.
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

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
