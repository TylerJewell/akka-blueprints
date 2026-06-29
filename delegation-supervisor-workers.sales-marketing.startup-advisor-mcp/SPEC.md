# SPEC — startup-advisor-mcp

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Startup Advisor (MCP).
**One-line pitch:** Describe your startup; an advisor coordinator delegates market research and channel strategy to two specialist agents via MCP-exposed tools, then synthesises actionable GTM guidance.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise go-to-market guidance. The blueprint also demonstrates a **before-tool-call guardrail** that allow-lists the MCP tool surface before any worker can invoke a tool — the primary governance control for MCP-connected agents.

## 3. User-facing flows

The user opens the App UI tab and submits a startup brief (a few sentences describing the product, target market, and stage).

1. The system creates an `AdvisorySession` record in `SCOPING` and starts an `AdvisoryWorkflow`.
2. The AdvisorCoordinator decomposes the brief into a market research query and a growth channel question.
3. The workflow forks: MarketResearcher and GrowthAnalyst run concurrently. Each calls MCP-exposed tools (modelled in-process). Each returns a typed payload.
4. The AdvisorCoordinator merges the two payloads into `GtmGuidance { summary, marketSnapshot, channelPlan, guardrailVerdict }`.
5. If either worker times out after 60 seconds, the workflow short-circuits: it asks the AdvisorCoordinator to synthesise from whichever side returned, and the session enters `DEGRADED`.
6. The session moves to `ADVISED` on success.

A `BriefSimulator` (TimedAction) drips a sample startup brief every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AdvisorCoordinator` | `AutonomousAgent` | Decomposes the startup brief, synthesises the merged GTM guidance. Guardrail runs before any tool call. | `AdvisoryWorkflow` | returns typed result to workflow |
| `MarketResearcher` | `AutonomousAgent` | Calls MCP market-data tools; returns `MarketSnapshot`. | `AdvisoryWorkflow` | — |
| `GrowthAnalyst` | `AutonomousAgent` | Calls MCP channel-scoring tools; returns `ChannelPlan`. | `AdvisoryWorkflow` | — |
| `AdvisoryWorkflow` | `Workflow` | Coordinates the parallel fan-out, the synthesis, and persists results. | `AdvisoryEndpoint`, `BriefConsumer` | `AdvisorySessionEntity` |
| `AdvisorySessionEntity` | `EventSourcedEntity` | Holds the session lifecycle (scoping → in-progress → advised / degraded). | `AdvisoryWorkflow` | `AdvisoryView` |
| `BriefQueue` | `EventSourcedEntity` | Logs each submitted startup brief for replay/audit. | `AdvisoryEndpoint`, `BriefSimulator` | `BriefConsumer` |
| `AdvisoryView` | `View` | List-of-sessions read model. | `AdvisorySessionEntity` events | `AdvisoryEndpoint` |
| `BriefConsumer` | `Consumer` | Listens to `BriefQueue` events and starts a workflow per submission. | `BriefQueue` events | `AdvisoryWorkflow` |
| `BriefSimulator` | `TimedAction` | Drips a sample startup brief every 60 s. | scheduler | `BriefQueue` |
| `EvalSampler` | `TimedAction` | Samples one advised session every 5 minutes; emits a `GuidanceEvalScored` event. | scheduler | `AdvisorySessionEntity` |
| `AdvisoryEndpoint` | `HttpEndpoint` | `/api/advisory/*` — submit, get, list, SSE. | — | `AdvisoryView`, `BriefQueue`, `AdvisorySessionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record StartupBrief(String description, String submittedBy) {}

record MarketSnapshot(
    String tam,
    List<String> competitors,
    List<BuyerSegment> buyerSegments,
    Instant researchedAt
) {}
record BuyerSegment(String name, String painPoint) {}

record ChannelPlan(
    List<ChannelRecommendation> channels,
    String primaryChannel,
    Instant analysedAt
) {}
record ChannelRecommendation(String channel, String rationale, String effort) {}

record BriefDecomposition(String marketResearchQuery, String channelStrategyQuestion) {}

record GtmGuidance(
    String summary,
    MarketSnapshot marketSnapshot,
    ChannelPlan channelPlan,
    String guardrailVerdict,
    Instant synthesisedAt
) {}

record AdvisorySession(
    String sessionId,
    String description,
    SessionStatus status,
    Optional<MarketSnapshot> marketSnapshot,
    Optional<ChannelPlan> channelPlan,
    Optional<GtmGuidance> guidance,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus { SCOPING, IN_PROGRESS, ADVISED, DEGRADED }
```

### Events (on `AdvisorySessionEntity`)

`SessionCreated`, `MarketSnapshotAttached`, `ChannelPlanAttached`, `GuidanceSynthesised`, `SessionDegraded`, `GuidanceEvalScored`.

### Events (on `BriefQueue`)

`BriefSubmitted { sessionId, description, submittedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/advisory` — body `{ "description": "..." }` → `{ "sessionId": "uuid" }`. Starts a workflow.
- `GET /api/advisory` — list all sessions. Optional `?status=SCOPING|IN_PROGRESS|ADVISED|DEGRADED`.
- `GET /api/advisory/{id}` — one session.
- `GET /api/advisory/sse` — server-sent events stream of every session change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Startup Advisor (MCP)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a startup brief, live list of sessions with status pills, expand-row to see market snapshot + channel plan + GTM guidance summary + eval score.

Browser title: `<title>Akka Sample: Startup Advisor (MCP)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — MCP tool allow-list guardrail** (`before-tool-call` on `MarketResearcher` and `GrowthAnalyst`): checks every tool call against a configured allow-list before it executes. Blocking. Disallowed calls are rejected and surfaced as a `failureReason` on the session.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one advised session every 5 minutes and emits a `GuidanceEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `AdvisorCoordinator` → `prompts/coordinator.md`. Decomposes the startup brief; later synthesises market and channel outputs into GTM guidance.
- `MarketResearcher` → `prompts/market-researcher.md`. Calls MCP market-data tools; returns `MarketSnapshot`.
- `GrowthAnalyst` → `prompts/growth-analyst.md`. Evaluates channel options; returns `ChannelPlan`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a startup brief; session progresses SCOPING → IN_PROGRESS → ADVISED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `MarketResearcher` timeout to 1 s); session enters DEGRADED with whichever partial output came back.
3. **J3** — A worker attempts to call a tool not on the allow-list; the guardrail blocks it before execution; the session records the failure reason.
4. **J4** — Wait after a successful synthesis; the session's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named startup-advisor-mcp demonstrating the
delegation-supervisor-workers × sales-marketing cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-sales-marketing-startup-advisor-mcp.
Java package io.akka.samples.startupadvisormcp. Akka 3.6.0. HTTP port 9603.

Components to wire (exactly):
- 3 AutonomousAgents:
  * AdvisorCoordinator — definition() with
    capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/coordinator.md. Returns
    BriefDecomposition{marketResearchQuery, channelStrategyQuestion} for DECOMPOSE
    and GtmGuidance{summary, marketSnapshot, channelPlan, guardrailVerdict, synthesisedAt}
    for SYNTHESISE.
  * MarketResearcher — capability(TaskAcceptance.of(RESEARCH).maxIterationsPerTask(3)).
    System prompt from prompts/market-researcher.md. Returns
    MarketSnapshot{tam, competitors: List<String>, buyerSegments: List<BuyerSegment{name,
    painPoint}>, researchedAt}. Guardrail: before-tool-call allow-list on every MCP tool call.
  * GrowthAnalyst — capability(TaskAcceptance.of(ANALYSE_CHANNELS).maxIterationsPerTask(3)).
    System prompt from prompts/growth-analyst.md. Returns
    ChannelPlan{channels: List<ChannelRecommendation{channel, rationale, effort}>,
    primaryChannel, analysedAt}. Guardrail: before-tool-call allow-list on every MCP tool call.

- 1 Workflow AdvisoryWorkflow with steps:
  decomposeStep -> [parallel] researchStep, analyseChannelsStep -> joinStep
  -> synthesiseStep -> emitStep.
  decomposeStep calls forAutonomousAgent(AdvisorCoordinator.class, DECOMPOSE).
  researchStep and analyseChannelsStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(AdvisoryWorkflow::researchStep, ofSeconds(60)) and
  stepTimeout(AdvisoryWorkflow::analyseChannelsStep, ofSeconds(60)). On either timeout,
  transition to a degradeStep that calls synthesiseStep with whichever side returned, then
  ends with SessionDegraded.
  synthesiseStep calls forAutonomousAgent(AdvisorCoordinator.class, SYNTHESISE) with the
  merged inputs; give synthesiseStep a 90s stepTimeout.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity AdvisorySessionEntity holding state AdvisorySession{sessionId,
  description, SessionStatus, Optional<MarketSnapshot> marketSnapshot,
  Optional<ChannelPlan> channelPlan, Optional<GtmGuidance> guidance,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  SessionStatus enum: SCOPING, IN_PROGRESS, ADVISED, DEGRADED.
  Events: SessionCreated, MarketSnapshotAttached, ChannelPlanAttached,
  GuidanceSynthesised, SessionDegraded, GuidanceEvalScored.
  Commands: createSession, attachMarketSnapshot, attachChannelPlan, synthesiseGuidance,
  degrade, recordEval, getSession.
  emptyState() returns AdvisorySession.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity BriefQueue with command enqueueBrief(description, submittedBy)
  emitting BriefSubmitted{sessionId, description, submittedBy, submittedAt}.

- 1 View AdvisoryView with row type AdvisorySessionRow (mirrors AdvisorySession minus
  heavy nested payloads; every nullable field is Optional<T>). Table updater consumes
  AdvisorySessionEntity events. ONE query getAllSessions SELECT * AS sessions FROM
  advisory_view. No WHERE status filter (Akka cannot auto-index enum columns) —
  caller filters client-side.

- 1 Consumer BriefConsumer subscribed to BriefQueue events; on BriefSubmitted
  starts an AdvisoryWorkflow with the sessionId as the workflow id.

- 2 TimedActions:
  * BriefSimulator — every 60s, reads next line from
    src/main/resources/sample-events/startup-briefs.jsonl and calls
    BriefQueue.enqueueBrief.
  * EvalSampler — every 5 minutes, queries AdvisoryView.getAllSessions, picks the oldest
    ADVISED session without an evalScore, runs a 1-5 rubric judge over the guidance
    content, then calls AdvisorySessionEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * AdvisoryEndpoint at /api with POST /advisory, GET /advisory, GET /advisory/{id},
    GET /advisory/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- AdvisoryTasks.java declaring four Task<R> constants: DECOMPOSE (BriefDecomposition),
  RESEARCH (MarketSnapshot), ANALYSE_CHANNELS (ChannelPlan), SYNTHESISE (GtmGuidance).
- Domain records BriefDecomposition, BuyerSegment, MarketSnapshot, ChannelRecommendation,
  ChannelPlan, GtmGuidance.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9603 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/startup-briefs.jsonl with 8 canned startup brief lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 before-tool-call guardrail,
  E1 eval-event on-decision-eval) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling appropriate sales-marketing answers;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md, prompts/market-researcher.md, prompts/growth-analyst.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Startup Advisor (MCP)", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no
  ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: Startup Advisor (MCP)</title>. No subtitle on the Overview tab.

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
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml.
    (e) Type once in this session — value lives in Claude session memory only.
- NEVER write the key value to any file. No .env, no entry in application.conf,
  no secrets.yaml, no .akka/ file with key material.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class name
  (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (coordinator.json,
  market-researcher.json, growth-analyst.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    coordinator.json — 4-6 BriefDecomposition entries (marketResearchQuery +
      channelStrategyQuestion pairs) and 4-6 GtmGuidance entries (each with an
      80-150 word summary, a MarketSnapshot, a ChannelPlan, guardrailVerdict="ok").
    market-researcher.json — 4-6 MarketSnapshot entries, each with a TAM string,
      3-5 competitor names, and 2-4 BuyerSegment entries.
    growth-analyst.json — 4-6 ChannelPlan entries, each with 3-5
      ChannelRecommendation entries and a designated primaryChannel.
- A MockModelProvider.seedFor(sessionId) helper makes the selection deterministic
  per session id so the same session produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion AdvisoryTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9603 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme
  variables (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index.
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
