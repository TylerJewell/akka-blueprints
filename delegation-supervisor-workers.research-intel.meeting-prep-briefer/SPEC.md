# Meeting Prep Briefer

Type a meeting topic and the people who will be there. A supervisor agent delegates research to worker agents, and a briefing document comes back with background on each participant, talking points, and questions to ask.

## 1. System name + one-line pitch

`Meeting Prep Briefer` — a user submits a meeting topic plus a participant list; the system researches each participant and the topic, then returns one briefing document.

## 2. What this blueprint demonstrates

The delegation-supervisor-workers coordination pattern: a supervisor agent decomposes a meeting request into research subtasks, fans those out to worker agents (one per participant plus a topic worker), and a composer agent assembles their results into a single briefing. The governance pattern wires a before-tool-call guardrail that gates every research lookup against a privacy allow-rule, and a PII sanitizer that redacts participant personal data before it is persisted on the entity.

## 3. User-facing flows

1. The user POSTs a meeting request `{ meetingTopic, participants: [...] }` to `/api/briefings`. The response carries the `briefingId`.
2. The UI subscribes to `/api/briefings/sse` and sees the briefing appear in `PLANNING`, then `RESEARCHING`, then `COMPOSING`, then `READY`.
3. When `READY`, the briefing document renders with three sections: participant background, talking points, and suggested questions.
4. Without any user action, the request simulator drips a canned meeting request every 30 seconds, so briefings keep arriving.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| BriefingSupervisor | AutonomousAgent | Turns a meeting request into a research plan (which participants, which topic angles) | BriefingWorkflow | ParticipantResearcher, TopicResearcher |
| ParticipantResearcher | AutonomousAgent | Researches one participant, returns a redacted profile | BriefingWorkflow | ResearchSource (tool) |
| TopicResearcher | AutonomousAgent | Researches the meeting topic, returns a topic brief | BriefingWorkflow | ResearchSource (tool) |
| BriefingComposer | Agent | Assembles profiles + topic brief into the final briefing document | BriefingWorkflow | — |
| BriefingWorkflow | Workflow | Orchestrates plan → research (fan-out) → compose | BriefingRequestConsumer | the four agents, BriefingEntity |
| BriefingEntity | EventSourcedEntity | Holds one briefing's lifecycle and content | BriefingWorkflow | BriefingsView |
| RequestQueue | EventSourcedEntity | Records each inbound meeting request | RequestSimulator, BriefingEndpoint | BriefingRequestConsumer |
| BriefingsView | View | Read model the UI lists and streams | BriefingEntity | BriefingEndpoint |
| BriefingRequestConsumer | Consumer | Starts a workflow per queued request | RequestQueue | BriefingWorkflow |
| RequestSimulator | TimedAction | Drips a canned request every 30s | sample-events file | RequestQueue |
| ResearchSource | HttpEndpoint | In-process research surface returning canned participant/topic data | researcher agents | — |
| BriefingEndpoint | HttpEndpoint | The /api surface — submit, list, single, SSE, metadata | UI, clients | RequestQueue, BriefingsView |
| AppEndpoint | HttpEndpoint | Serves the UI | browser | static-resources |

## 5. Data model

See `reference/data-model.md` for the full table. Authoritative records:

```java
record Briefing(
  String id,
  String meetingTopic,
  List<String> participants,
  BriefingStatus status,
  Optional<Instant> plannedAt,
  Optional<List<String>> researchPlan,
  Optional<List<ParticipantProfile>> profiles,
  Optional<TopicBrief> topicBrief,
  Optional<String> briefingDoc,
  Optional<List<String>> talkingPoints,
  Optional<List<String>> questions,
  Optional<Instant> readyAt,
  Optional<Instant> failedAt,
  Optional<String> failureReason) {}

record ParticipantProfile(String name, String role, String redactedBackground) {}
record TopicBrief(String summary, List<String> keyPoints) {}

enum BriefingStatus { RECEIVED, PLANNING, RESEARCHING, COMPOSING, READY, FAILED }
```

Every nullable lifecycle field is `Optional<T>` (Lesson 6). Events: `BriefingRequested`, `ResearchPlanned`, `ProfileAdded`, `TopicBriefAdded`, `BriefingComposed`, `BriefingFailed`.

## 6. API contract

Full schemas in `reference/api-contract.md`. Top-level surface:

```
POST /api/briefings                 -> { briefingId }
GET  /api/briefings                 -> { briefings: [Briefing, ...] }
GET  /api/briefings/{id}            -> Briefing
GET  /api/briefings/sse             -> Server-Sent Events of Briefing
GET  /api/metadata/eval-matrix      -> text/yaml
GET  /api/metadata/risk-survey      -> text/yaml
GET  /api/metadata/readme           -> text/markdown
GET  /research/participant?name=    -> canned profile JSON (in-process)
GET  /research/topic?q=             -> canned topic JSON (in-process)
GET  /                              -> 302 /app/index.html
GET  /app/{*path}                   -> static-resources/{*path}
```

## 7. UI

Five tabs, single self-contained `src/main/resources/static-resources/index.html` (no npm, no `ui/`). Browser title `<title>Akka Sample: Meeting Prep Briefer</title>`. Detail in `reference/ui-mockup.md`.

- **Overview** — eyebrow + headline + the four cards (Try it / How it works / Components / API contract).
- **Architecture** — the four mermaid diagrams with the Lesson 24 state-label CSS overrides.
- **Risk Survey** — renders `/api/metadata/risk-survey` in `matrix-card` / `matrix-row` style; deployer placeholders muted.
- **Eval Matrix** — renders `/api/metadata/eval-matrix` with a colored mechanism pill per control row.
- **App UI** — submit a meeting request (topic + participant list), live SSE list of briefings, expand a `READY` briefing to read its three sections.

Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index; no zombie panels (Lesson 26).

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Two mechanisms the generated system wires:

- **G1 — before-tool-call guardrail.** Fires before each `ResearchSource` lookup from `ParticipantResearcher` and `TopicResearcher`. Blocks a lookup unless the requested name/topic is part of the active briefing request, preventing the workers from scraping arbitrary people.
- **S1 — PII sanitizer.** Runs on each researched profile before `ProfileAdded` is emitted on `BriefingEntity`, redacting emails, phone numbers, and home addresses so stored profiles carry only role-relevant background.

## 9. Agent prompts

- `prompts/briefing-supervisor.md` — plans the research subtasks for a meeting request.
- `prompts/participant-researcher.md` — researches one participant and returns a redacted profile.
- `prompts/topic-researcher.md` — researches the meeting topic and returns a topic brief.
- `prompts/briefing-composer.md` — assembles the briefing document.

## 10. Acceptance

Full journeys in `reference/user-journeys.md`. The three that must pass:

1. **Submit and complete** — POST a request; the briefing moves RECEIVED → PLANNING → RESEARCHING → COMPOSING → READY within ~60s; `briefingDoc` is non-empty.
2. **Briefing has all three sections** — a READY briefing exposes non-empty `talkingPoints` and `questions` plus per-participant background.
3. **PII is redacted** — stored `ParticipantProfile.redactedBackground` contains no email, phone, or street-address tokens even though the canned research source includes them.

## 11. Implementation directives

```
Create a sample named meeting-prep-briefer demonstrating the
delegation-supervisor-workers x research-intel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
meeting-prep-briefer. Java package io.akka.samples.meetingprepbriefer.
Akka 3.6.0. HTTP port 9382.

Components to wire (exactly):
- 3 AutonomousAgents:
  - BriefingSupervisor: definition() with a capability accepting a PLAN task,
    returns a typed ResearchPlan{ participantNames:List<String>,
    topicAngles:List<String> }. maxIterationsPerTask(3).
  - ParticipantResearcher: accepts a RESEARCH_PARTICIPANT task, calls the
    ResearchSource tool (GET /research/participant), returns a typed
    ParticipantProfile{name, role, redactedBackground}. The before-tool-call
    guardrail (G1) gates the tool call. maxIterationsPerTask(3).
  - TopicResearcher: accepts a RESEARCH_TOPIC task, calls ResearchSource
    (GET /research/topic), returns a typed TopicBrief{summary, keyPoints}.
    maxIterationsPerTask(3).
- 1 Agent BriefingComposer with a method compose(BriefingInputs) returning a
  typed ComposedBriefing{ briefingDoc, talkingPoints:List<String>,
  questions:List<String> } via effects().systemMessage(...).userMessage(...)
  .responseAs(ComposedBriefing.class).thenReply().
- 1 Workflow BriefingWorkflow with steps planStep -> researchStep ->
  composeStep -> done. planStep calls forAutonomousAgent(BriefingSupervisor)
  .runSingleTask then forTask(taskId).result. researchStep runs one
  RESEARCH_PARTICIPANT task per planned participant and one RESEARCH_TOPIC
  task, gathers results, and after the PII sanitizer (S1) records each via
  BriefingEntity.addProfile / setTopicBrief. composeStep calls
  BriefingComposer.compose and records BriefingComposed. Override settings()
  with stepTimeout(planStep, 60s), stepTimeout(researchStep, 120s),
  stepTimeout(composeStep, 60s) and
  defaultStepRecovery(maxRetries(2).failoverTo(BriefingWorkflow::error)).
- 1 EventSourcedEntity BriefingEntity holding the Briefing record (id,
  meetingTopic, participants:List<String>, BriefingStatus enum, and Optional
  lifecycle fields plannedAt, researchPlan, profiles, topicBrief, briefingDoc,
  talkingPoints, questions, readyAt, failedAt, failureReason). Events:
  BriefingRequested, ResearchPlanned, ProfileAdded, TopicBriefAdded,
  BriefingComposed, BriefingFailed. Commands: requestBriefing, recordPlan,
  addProfile, setTopicBrief, recordComposition, markFailed, getBriefing.
  emptyState() returns Briefing.initial with no commandContext() reference.
- 1 EventSourcedEntity RequestQueue with a single command enqueue(request)
  emitting RequestQueued.
- 1 View BriefingsView with row type Briefing, table updater consuming
  BriefingEntity events. ONE query: getAllBriefings SELECT * AS briefings FROM
  briefings_view. No WHERE-on-enum (Akka cannot auto-index enum columns);
  filter client-side in callers.
- 1 Consumer BriefingRequestConsumer subscribed to RequestQueue events; on
  each event starts a BriefingWorkflow with a fresh UUID.
- 1 TimedAction RequestSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/meeting-requests.jsonl and calls
  RequestQueue.enqueue).
- 3 HttpEndpoints:
  - BriefingEndpoint at /api: briefings POST, list (filter client-side from
    getAllBriefings), single, SSE stream, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  - ResearchSource at /research: participant and topic GETs returning canned
    JSON from src/main/resources/sample-data/{participants,topics}.json. The
    canned participant payloads deliberately include email/phone/address so
    the S1 sanitizer has something to redact.
  - AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- BriefingTasks.java declaring Task<R> constants: PLAN (resultConformsTo
  ResearchPlan), RESEARCH_PARTICIPANT (ParticipantProfile), RESEARCH_TOPIC
  (TopicBrief).
- Records: ResearchPlan(List<String> participantNames, List<String>
  topicAngles), ParticipantProfile(String name, String role, String
  redactedBackground), TopicBrief(String summary, List<String> keyPoints),
  ComposedBriefing(String briefingDoc, List<String> talkingPoints,
  List<String> questions), BriefingInputs(String meetingTopic,
  List<ParticipantProfile> profiles, TopicBrief topicBrief).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9382 and agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/meeting-requests.jsonl with 8 canned
  requests (each a meetingTopic + 2-3 participant names).
- src/main/resources/sample-data/participants.json and topics.json with
  canned research payloads.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.
- eval-matrix.yaml at the project root with the 2 controls (G1, S1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking deployer
  fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: see the authored README.
- src/main/resources/static-resources/index.html — one self-contained file
  (no ui/ folder, no npm). Inline CSS + JS. Runtime CDN imports for markdown
  and YAML are acceptable. Five tabs: Overview, Architecture, Risk Survey,
  Eval Matrix, App UI. Match the governance.html visual style (dark / yellow
  accent / Instrument Sans / dot-grid).

Generation workflow - see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM - generate a MockModelProvider with per-agent canned/random
  outputs (BriefingSupervisor -> ResearchPlan, ParticipantResearcher ->
  ParticipantProfile, TopicResearcher -> TopicBrief, BriefingComposer ->
  ComposedBriefing; see src/main/resources/mock-responses/{briefing-supervisor,
  participant-researcher,topic-researcher,briefing-composer}.json with 4-6
  entries each). Sets model-provider = mock.
  (b) Name an existing env var - record the NAME in application.conf.
  (c) Point to an existing env file path - record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI - recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session - value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE - env-var name, file path, secrets URI - never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes captured key
  material.

Mock LLM provider (option a) per-agent shapes:
- BriefingSupervisor -> { participantNames: [<each requested participant>],
  topicAngles: ["background", "recent news", "shared context"] }.
- ParticipantResearcher -> { name, role: "<plausible title>",
  redactedBackground: "<2-3 sentence background, no PII>" }.
- TopicResearcher -> { summary: "<2 sentences>", keyPoints: [3-4 bullets] }.
- BriefingComposer -> { briefingDoc: "<markdown with three sections>",
  talkingPoints: [3-5], questions: [3-5] }.

Constraints - see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: every agent-calling workflow step sets an explicit stepTimeout.
- Lesson 6: Optional<T> for every nullable field on the Briefing row record.
- Lesson 7: AutonomousAgents require the companion BriefingTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: run command is "/akka:build", never "mvn akka:run".
- Lesson 10: explicit http-port 9382 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px column, no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box", never T1/deferred.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: mermaid state-label CSS overrides + edge-label overflow:visible
  + transitionLabelColor #cccccc in static-resources/index.html.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute, never NodeList
  index; delete removed panels, never display:none zombies.
```

## 12. Post-scaffolding workflow

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue:

1. Run `/akka:plan` — produce the architectural plan. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and the PLAN.
2. Run `/akka:tasks` — break the plan into tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9382`) and a one-line summary of any step-3 failures. Do not narrate intermediate progress unless a step runs longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without the user — a missing/unresolvable API key (offer the five sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
