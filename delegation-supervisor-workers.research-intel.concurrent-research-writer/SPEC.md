# SPEC — concurrent-research-writer

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Concurrent Research Writer.
**One-line pitch:** Submit a topic; a coordinator delegates source research and narrative drafting to two workers in parallel, then assembles one polished written report.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to assemble a final written report. The blueprint also demonstrates an **output guardrail** that vets the assembled report for unsupported claims before it is returned, and an **eval-event** that samples the coordinator's assembly decision for a readability and accuracy score.

## 3. User-facing flows

The user opens the App UI tab and submits a writing topic via the form.

1. The system creates a `Report` record in `PLANNING` and starts a `ReportWorkflow`.
2. The Coordinator decomposes the topic into two parallel assignments: a source-gathering brief for the SourceResearcher, and a narrative-drafting brief for the DraftWriter.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the two payloads into an `AssembledReport { headline, body, sourceList, guardrailVerdict }`.
5. An output guardrail vets the assembled report; if it fails, the report moves to `BLOCKED`. Otherwise, the report moves to `ASSEMBLED`.
6. If either worker times out after 60 seconds, the workflow short-circuits: it asks the Coordinator to assemble from whichever side returned, and the report enters `DEGRADED`.

A `TopicSimulator` (TimedAction) drips a sample writing topic every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReportCoordinator` | `AutonomousAgent` | Decomposes the topic into parallel assignments, assembles the final report, runs the output guardrail. | `ReportWorkflow` | returns typed result to workflow |
| `SourceResearcher` | `AutonomousAgent` | Gathers factual source material for the assigned subtopic. Seeded "lookup tool" returns canned results. | `ReportWorkflow` | — |
| `DraftWriter` | `AutonomousAgent` | Writes a narrative section from the given writing brief. | `ReportWorkflow` | — |
| `ReportWorkflow` | `Workflow` | Coordinates the parallel fan-out, the assembly, the guardrail. | `ReportEndpoint`, `ReportRequestConsumer` | `ReportEntity` |
| `ReportEntity` | `EventSourcedEntity` | Holds the report's lifecycle (planning → in-progress → assembled / degraded / blocked). | `ReportWorkflow` | `ReportView` |
| `TopicQueue` | `EventSourcedEntity` | Logs each submitted topic for replay/audit. | `ReportEndpoint`, `TopicSimulator` | `ReportRequestConsumer` |
| `ReportView` | `View` | List-of-reports read model. | `ReportEntity` events | `ReportEndpoint` |
| `ReportRequestConsumer` | `Consumer` | Listens to `TopicQueue` events and starts a workflow per submission. | `TopicQueue` events | `ReportWorkflow` |
| `TopicSimulator` | `TimedAction` | Drips a sample topic every 60 s. | scheduler | `TopicQueue` |
| `QualitySampler` | `TimedAction` | Samples one assembled report every 5 minutes for quality scoring; emits a `ReportScored` event. | scheduler | `ReportEntity` |
| `ReportEndpoint` | `HttpEndpoint` | `/api/reports/*` — submit, get, list, SSE. | — | `ReportView`, `TopicQueue`, `ReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TopicRequest(String topic, String requestedBy) {}

record SourceBundle(List<Source> sources, Instant gatheredAt) {}
record Source(String title, String reference, String excerpt) {}

record DraftSection(String narrativeText, List<String> keyPoints, Instant draftedAt) {}

record WritingPlan(String sourceQuery, String narrativeBrief) {}

record AssembledReport(String headline, String body, SourceBundle sourceList,
                       String guardrailVerdict, Instant assembledAt) {}

record Report(
    String reportId,
    String topic,
    ReportStatus status,
    Optional<SourceBundle> sources,
    Optional<DraftSection> draft,
    Optional<AssembledReport> assembled,
    Optional<String> failureReason,
    Optional<Integer> qualityScore,
    Optional<String> qualityRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReportStatus { PLANNING, IN_PROGRESS, ASSEMBLED, DEGRADED, BLOCKED }
```

### Events (on `ReportEntity`)

`ReportCreated`, `SourcesAttached`, `DraftAttached`, `ReportAssembled`, `ReportDegraded`, `ReportBlocked`, `ReportScored`.

### Events (on `TopicQueue`)

`TopicSubmitted { reportId, topic, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/reports` — body `{ topic }` → `{ reportId }`. Starts a workflow.
- `GET /api/reports` — list all reports. Optional `?status=PLANNING|IN_PROGRESS|ASSEMBLED|DEGRADED|BLOCKED`.
- `GET /api/reports/{id}` — one report.
- `GET /api/reports/sse` — server-sent events stream of every report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Concurrent Research Writer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a topic, live list of reports with status pills, expand-row to see sources + draft section + assembled report + quality score.

Browser title: `<title>Akka Sample: Concurrent Research Writer</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `ReportCoordinator`): vets the assembled report for unsupported claims and fabricated references before it is returned. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `QualitySampler` (TimedAction) picks one assembled report every 5 minutes and emits a `ReportScored` event with a 1–5 readability-and-accuracy score and a short rationale.

## 9. Agent prompts

- `ReportCoordinator` → `prompts/report-coordinator.md`. Decomposes the topic into parallel work assignments; later assembles source material and draft text into the final report.
- `SourceResearcher` → `prompts/source-researcher.md`. Gathers source material; returns `SourceBundle`.
- `DraftWriter` → `prompts/draft-writer.md`. Writes the narrative section; returns `DraftSection`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a topic; report progresses PLANNING → IN_PROGRESS → ASSEMBLED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `SourceResearcher` timeout to 1 s); report enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a guardrail failure (Coordinator returns an unsupported claim); report enters BLOCKED.
4. **J4** — Wait after a successful assembly; the report's row in the UI shows a quality score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named concurrent-research-writer demonstrating the
delegation-supervisor-workers × research-intel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-research-intel-concurrent-research-writer.
Java package io.akka.samples.workflowconcurrentresearchwriter. Akka 3.6.0.
HTTP port 9432.

Components to wire (exactly):
- 3 AutonomousAgents:
  * ReportCoordinator — definition() with
    capability(TaskAcceptance.of(PLAN_REPORT).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(ASSEMBLE_REPORT).maxIterationsPerTask(3)).
    System prompt loaded from prompts/report-coordinator.md. Returns
    WritingPlan{sourceQuery, narrativeBrief} for PLAN_REPORT and
    AssembledReport{headline, body, sourceList, guardrailVerdict, assembledAt}
    for ASSEMBLE_REPORT.
  * SourceResearcher — capability(TaskAcceptance.of(GATHER_SOURCES).maxIterationsPerTask(3)).
    System prompt from prompts/source-researcher.md. Returns
    SourceBundle{sources: List<Source{title, reference, excerpt}>, gatheredAt}.
  * DraftWriter — capability(TaskAcceptance.of(WRITE_DRAFT).maxIterationsPerTask(2)).
    System prompt from prompts/draft-writer.md. Returns
    DraftSection{narrativeText, keyPoints: List<String>, draftedAt}.

- 1 Workflow ReportWorkflow with steps:
  planStep -> [parallel] researchStep, draftStep -> joinStep -> assembleStep
  -> guardrailStep -> emitStep.
  planStep calls forAutonomousAgent(ReportCoordinator.class, PLAN_REPORT).
  researchStep and draftStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(ReportWorkflow::researchStep, ofSeconds(60))
  and stepTimeout(ReportWorkflow::draftStep, ofSeconds(60)). On either timeout,
  transition to a degradeStep that calls assembleStep with whichever side returned,
  then ends with ReportDegraded.
  assembleStep calls forAutonomousAgent(ReportCoordinator.class, ASSEMBLE_REPORT) with
  the merged inputs; give assembleStep a 90s stepTimeout. guardrailStep runs the
  deterministic vetter + LLM judge on the assembled content; on failure, end with
  ReportBlocked. WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity ReportEntity holding state Report{reportId, topic,
  ReportStatus, Optional<SourceBundle> sources, Optional<DraftSection> draft,
  Optional<AssembledReport> assembled, Optional<String> failureReason,
  Optional<Integer> qualityScore, Optional<String> qualityRationale, Instant createdAt,
  Optional<Instant> finishedAt}. ReportStatus enum: PLANNING, IN_PROGRESS, ASSEMBLED,
  DEGRADED, BLOCKED. Events: ReportCreated, SourcesAttached, DraftAttached,
  ReportAssembled, ReportDegraded, ReportBlocked, ReportScored. Commands: createReport,
  attachSources, attachDraft, assemble, degrade, block, recordScore, getReport.
  emptyState() returns Report.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity TopicQueue with command enqueueTopic(topic, requestedBy)
  emitting TopicSubmitted{reportId, topic, requestedBy, submittedAt}.

- 1 View ReportView with row type ReportRow (mirrors Report minus heavy nested
  payloads; every nullable field is Optional<T>). Table updater consumes ReportEntity
  events. ONE query getAllReports SELECT * AS reports FROM report_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters
  client-side.

- 1 Consumer ReportRequestConsumer subscribed to TopicQueue events; on TopicSubmitted
  starts a ReportWorkflow with the reportId as the workflow id.

- 2 TimedActions:
  * TopicSimulator — every 60s, reads next line from
    src/main/resources/sample-events/writing-topics.jsonl and calls
    TopicQueue.enqueueTopic.
  * QualitySampler — every 5 minutes, queries ReportView.getAllReports, picks the
    oldest ASSEMBLED report without a qualityScore, runs a 1–5 readability-and-accuracy
    rubric judge over the assembled content, then calls
    ReportEntity.recordScore(score, rationale).

- 2 HttpEndpoints:
  * ReportEndpoint at /api with POST /reports, GET /reports, GET /reports/{id},
    GET /reports/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ReportTasks.java declaring four Task<R> constants: PLAN_REPORT (WritingPlan),
  GATHER_SOURCES (SourceBundle), WRITE_DRAFT (DraftSection), ASSEMBLE_REPORT
  (AssembledReport).
- Domain records WritingPlan, Source, SourceBundle, DraftSection, AssembledReport.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9432 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/writing-topics.jsonl with 8 canned topic lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = research and analysis /
  knowledge synthesis automation, decisions.authority_level = recommend-only,
  data.pii = false, capabilities.content_generation = true,
  capabilities.synthetic_media = true; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/report-coordinator.md, prompts/source-researcher.md,
  prompts/draft-writer.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Concurrent Research Writer",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams
  + click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix
  (5-column ID/Control/Mechanism/Implementation/Source table with click-to-expand rows),
  App UI (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: Concurrent Research Writer</title>. No subtitle on the
  Overview tab.

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
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
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
  src/main/resources/mock-responses/<agent-name>.json (report-coordinator.json,
  source-researcher.json, draft-writer.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    report-coordinator.json — list of either WritingPlan or AssembledReport objects.
      4–6 WritingPlan entries (sourceQuery + narrativeBrief pairs) and
      4–6 AssembledReport entries (each with a 80–150 word body, a SourceBundle
      with 3–5 sources, guardrailVerdict = "ok").
    source-researcher.json — 4–6 SourceBundle entries, each with 3–5 sources
      whose reference values are believable (e.g., "Nature 2024, vol. 630",
      "IPCC AR6 Chapter 4", "unsourced — knowledge").
    draft-writer.json — 4–6 DraftSection entries, each with a 100–200 word
      narrativeText and 3–5 keyPoints as short bullets.
- A MockModelProvider.seedFor(reportId) helper makes the selection deterministic
  per report id so the same report produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause
  matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout
  (60s workers, 90s assembly); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion ReportTasks.java declaring
  Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9432 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"),
  never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND
  theme variables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc). Without these, state names
  render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never
  NodeList index; delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, marketing tone.
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
