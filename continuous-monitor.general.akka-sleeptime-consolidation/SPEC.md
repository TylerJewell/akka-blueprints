# SPEC — sleeptime-consolidation

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Sleeptime Memory-Consolidation Agent.
**One-line pitch:** A background consolidation agent shares memory blocks with a primary agent and rewrites them asynchronously every N steps to compact and refine accumulated context.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with three governance mechanisms layered around an asynchronous memory rewriting primitive (`SleeptimeConsolidatorAgent`). Specifically:

- A **before-tool-call guardrail** on the `writeMemoryBlock` tool enforces that only an authorised consolidation workflow can mutate a memory block — ad-hoc writes by uncontrolled callers are rejected.
- A **HOTL (human-on-the-loop) deployer-runtime-monitoring** mechanism streams every consolidation event to an operator dashboard so that unexpected rewrites are visible without blocking production throughput.
- A **eval-periodic drift-fairness-watch** sampler running every 30 minutes compares each newly consolidated block against the prior version, scores semantic drift, and surfaces blocks whose meaning has shifted materially so a human can review.

The result is a system where asynchronous memory mutation is observable, gated, and continuously monitored for drift — the three properties that make background rewriting safe in production.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live memory state: each `MemoryBlock` with its current content, version number, consolidation status, and (if scored) its drift score.
2. `ConsolidationTrigger` (TimedAction) ticks every 30 s. For each session whose `InteractionCounterEntity` has accumulated N or more unprocessed steps since the last consolidation, it starts a `ConsolidationWorkflow`.
3. `ConsolidationWorkflow` calls `SleeptimeConsolidatorAgent`, which rewrites the block into a consolidated form. The block transitions to CONSOLIDATED.
4. If the before-tool-call guardrail detects a `writeMemoryBlock` call that does not originate from an active `ConsolidationWorkflow`, it rejects the call and logs a `GuardrailTriggered` event.
5. `DriftEvalRunner` (TimedAction) ticks every 30 minutes, samples recently consolidated blocks without a `driftScore`, calls `DriftEvalAgent`, and writes a `DriftScored` event with a numeric score and rationale.
6. The operator opens the HOTL activity stream in the App UI to see a real-time feed of consolidation events and any guardrail trips.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConsolidationTrigger` | `TimedAction` | Every 30 s: finds sessions with ≥ N unprocessed steps; starts `ConsolidationWorkflow` per eligible block. | scheduler | `ConsolidationWorkflow` |
| `InteractionCounterEntity` | `EventSourcedEntity` | Per-session step counter. Commands: `incrementStep`, `resetCounter`. Events: `StepRecorded`, `CounterReset`. | `MemoryEndpoint` | `ConsolidationTrigger` (via `MemoryView`) |
| `MemoryBlockEntity` | `EventSourcedEntity` | Per-block lifecycle. Commands: `createBlock`, `beginConsolidation`, `applyConsolidation`, `markStale`, `recordDrift`. Events: `BlockCreated`, `BlockUpdated`, `ConsolidationStarted`, `BlockConsolidated`, `BlockMarkedStale`, `DriftScored`, `GuardrailTriggered`. | `ConsolidationWorkflow`, `MemoryEndpoint` | `MemoryView` |
| `ConsolidationWorkflow` | `Workflow` | Per-block consolidation run: loadStep → consolidateStep → commitStep. | `ConsolidationTrigger` | `MemoryBlockEntity`, `SleeptimeConsolidatorAgent` |
| `PrimaryAgent` | `Agent` (typed) | Handles user requests; reads current block content from `MemoryBlockEntity`. Never writes directly to blocks. | invoked by `MemoryEndpoint` | returns `AgentResponse` |
| `SleeptimeConsolidatorAgent` | `AutonomousAgent` | Rewrites memory block content to distil learned context into a compact, coherent form. | invoked by `ConsolidationWorkflow` | returns `ConsolidatedBlock` |
| `DriftEvalAgent` | `Agent` (typed) | Compares prior and current block content; returns a semantic drift score 0–100 and rationale. | invoked by `DriftEvalRunner` | returns `DriftResult` |
| `MemoryView` | `View` | Read model: one row per block with latest content, version, status, drift score. | `MemoryBlockEntity` events, `InteractionCounterEntity` events | `MemoryEndpoint` |
| `DriftEvalRunner` | `TimedAction` | Every 30 min: samples CONSOLIDATED blocks without `driftScore`; calls `DriftEvalAgent`; writes `DriftScored`. | scheduler | `MemoryBlockEntity` |
| `MemoryEndpoint` | `HttpEndpoint` | `/api/memory/*` — list blocks, get block, add content, increment counter, SSE stream. | — | `MemoryView`, `MemoryBlockEntity`, `InteractionCounterEntity`, `PrimaryAgent` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record MemoryBlockContent(String blockId, String sessionId, String rawContent, List<String> topicHints, Instant capturedAt) {}

record ConsolidatedContent(String consolidatedText, String changeRationale, int priorVersion, Instant consolidatedAt) {}

record DriftResult(int driftScore, String rationale, String priorSummary, String currentSummary) {}

record AgentResponse(String reply, String blockIdRead, int blockVersionRead, Instant respondedAt) {}

record GuardrailRecord(String callerId, String toolName, String rejectionReason, Instant triggeredAt) {}

record MemoryBlock(
    String blockId,
    String sessionId,
    Optional<MemoryBlockContent> rawContent,
    Optional<ConsolidatedContent> consolidated,
    int version,
    Optional<DriftResult> drift,
    Optional<GuardrailRecord> lastGuardrailTrip,
    BlockStatus status,
    Instant createdAt,
    Optional<Instant> lastConsolidatedAt
) {}

enum BlockStatus {
    ACTIVE, CONSOLIDATING, CONSOLIDATED, STALE
}

record SessionCounter(String sessionId, int totalSteps, int stepsSinceLastConsolidation, Instant lastConsolidatedAt) {}
```

Events on `MemoryBlockEntity`: `BlockCreated`, `BlockUpdated`, `ConsolidationStarted`, `BlockConsolidated`, `BlockMarkedStale`, `DriftScored`, `GuardrailTriggered`.

Events on `InteractionCounterEntity`: `StepRecorded`, `CounterReset`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/memory/blocks` — list all memory blocks. Optional `?sessionId=…` or `?status=…`.
- `GET /api/memory/blocks/{blockId}` — one block with full content history.
- `POST /api/memory/blocks` — body `{ sessionId, rawContent, topicHints }` → creates a new block (ACTIVE).
- `POST /api/memory/blocks/{blockId}/content` — body `{ rawContent, topicHints }` → appends content (triggers step counter increment).
- `GET /api/memory/sessions/{sessionId}/counter` — get step counter state for a session.
- `POST /api/memory/sessions/{sessionId}/step` — increment the step counter; `ConsolidationTrigger` will pick this up on its next tick.
- `POST /api/memory/agent/query` — body `{ sessionId, prompt }` → calls `PrimaryAgent` with current block as context; returns `AgentResponse`.
- `GET /api/memory/sse` — Server-Sent Events for every block state transition and guardrail trip.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,survey-template}` — UI metadata files.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Sleeptime Memory-Consolidation Agent</title>`.

App UI tab is the most distinctive: a two-column layout with the memory block list on the left and the selected block's detail + HOTL activity stream on the right. The before-tool-call guardrail trips are surfaced as red event rows in the activity stream.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** on `writeMemoryBlock`: rejects any call where the caller is not a `ConsolidationWorkflow` instance with an active `blockId` lease.
- **H1 — HOTL deployer-runtime-monitoring**: every `ConsolidationStarted`, `BlockConsolidated`, `GuardrailTriggered`, and `DriftScored` event is streamed to the operator via `GET /api/memory/sse`. The operator can observe but cannot block (non-blocking mechanism).
- **E1 — eval-periodic drift-fairness-watch**: `DriftEvalRunner` fires every 30 minutes; samples blocks and scores semantic drift 0–100; blocks with drift ≥ 70 are flagged for human review.

## 9. Agent prompts

- `PrimaryAgent` → `prompts/primary-agent.md`. Typed. Reads current block content; answers user questions.
- `SleeptimeConsolidatorAgent` → `prompts/sleeptime-consolidator.md`. AutonomousAgent. Rewrites block content into a consolidated form.
- `DriftEvalAgent` (used by `DriftEvalRunner`) → `prompts/drift-eval.md`. Typed. Scores semantic drift between block versions.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Increment counter to N; within 30 s, `ConsolidationTrigger` starts a `ConsolidationWorkflow`; block transitions to CONSOLIDATED.
2. **J2** — Attempt a direct `writeMemoryBlock` call outside a workflow; guardrail blocks it; `GuardrailTriggered` event appears in the SSE stream.
3. **J3** — HOTL stream shows every state transition within 1 s of the event being emitted.
4. **J4** — `DriftEvalRunner` scores a consolidated block within 30 minutes; drift score and rationale appear in the App UI.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sleeptime-memory-consolidation-agent demonstrating the
continuous-monitor × general cell. Runs out of the box (in-memory session
store; no external persistence). Maven group io.akka.samples, artifact
continuous-monitor-general-akka-sleeptime-consolidation. Java package
io.akka.samples.sleeptimememoryconsolidationagent. HTTP port 9574.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) PrimaryAgent — question-answering agent.
  System prompt loaded from prompts/primary-agent.md. Input:
  PrimaryAgentInput{sessionId: String, prompt: String, blockContent: Optional<String>,
  blockVersion: int}. Output: AgentResponse{reply: String, blockIdRead: String,
  blockVersionRead: int, respondedAt: Instant}. Defaults to a safe "I don't have
  enough context yet" reply when blockContent is empty.

- 1 AutonomousAgent SleeptimeConsolidatorAgent — memory rewriter. definition()
  with capability(TaskAcceptance.of(CONSOLIDATE).maxIterationsPerTask(5)).
  System prompt from prompts/sleeptime-consolidator.md. Input:
  ConsolidationInput{blockId: String, rawContent: String, topicHints: List<String>,
  priorConsolidated: Optional<String>}. Output: ConsolidatedContent{consolidatedText,
  changeRationale, priorVersion: int, consolidatedAt: Instant}. The agent ONLY
  writes via the workflow's commitStep — never directly.

- 1 Agent (typed) DriftEvalAgent — drift scorer. System prompt from
  prompts/drift-eval.md. Input: DriftInput{blockId: String, priorText: String,
  currentText: String}. Output: DriftResult{driftScore: int (0–100), rationale:
  String, priorSummary: String, currentSummary: String}.

- 1 Workflow ConsolidationWorkflow per block-consolidation run with steps:
  loadStep → consolidateStep → commitStep → doneStep.
  loadStep: reads current MemoryBlockEntity state; emits ConsolidationStarted.
  consolidateStep: calls SleeptimeConsolidatorAgent with stepTimeout(Duration.ofSeconds(60));
  on timeout, emits BlockMarkedStale and ends.
  commitStep: calls MemoryBlockEntity.applyConsolidation with the result; on
  success, emits BlockConsolidated. doneStep: terminates workflow.
  Workflow id = blockId. Duplicate starts for the same blockId fold into the
  running instance.

- 2 EventSourcedEntities:
  * InteractionCounterEntity (one per sessionId) — step counter.
    State SessionCounter{sessionId, totalSteps, stepsSinceLastConsolidation,
    lastConsolidatedAt: Instant}. Commands: incrementStep(sessionId), resetCounter(sessionId).
    Events: StepRecorded{sessionId, newTotal: int}, CounterReset{sessionId}.
    emptyState() returns SessionCounter.initial(sessionId, 0, 0, Instant.EPOCH).
  * MemoryBlockEntity (one per blockId) — full block lifecycle.
    State MemoryBlock{blockId, sessionId, rawContent: Optional<MemoryBlockContent>,
    consolidated: Optional<ConsolidatedContent>, version: int,
    drift: Optional<DriftResult>, lastGuardrailTrip: Optional<GuardrailRecord>,
    status: BlockStatus (ACTIVE/CONSOLIDATING/CONSOLIDATED/STALE),
    createdAt: Instant, lastConsolidatedAt: Optional<Instant>}.
    Commands: createBlock, beginConsolidation, applyConsolidation, markStale,
    recordDrift, recordGuardrailTrip, getBlock.
    Events: BlockCreated, BlockUpdated, ConsolidationStarted, BlockConsolidated,
    BlockMarkedStale, DriftScored, GuardrailTriggered.
    emptyState() returns MemoryBlock.initial("", "") without commandContext() reference.

- 1 View MemoryView with row type MemoryBlockRow (mirrors MemoryBlock minus raw
  MemoryBlockContent.rawContent text — content is fetched on-demand via getBlock).
  Table updaters consume MemoryBlockEntity events and InteractionCounterEntity
  events. ONE query getAllBlocks SELECT * AS blocks FROM memory_block_view.

- 2 TimedActions:
  * ConsolidationTrigger — every 30 s, queries MemoryView.getAllBlocks, finds
    sessions where stepsSinceLastConsolidation >= N (default N=5, configurable via
    application.conf akka.samples.consolidation-threshold), starts a
    ConsolidationWorkflow per eligible block (status == ACTIVE). At most 10 blocks
    per tick to avoid burst.
  * DriftEvalRunner — every 30 minutes, queries MemoryView.getAllBlocks, picks up
    to 5 CONSOLIDATED blocks without a driftScore (oldest-first), calls DriftEvalAgent
    per block with priorText and currentText, then calls
    MemoryBlockEntity.recordDrift(driftScore, rationale) per block.

- 2 HttpEndpoints:
  * MemoryEndpoint at /api/memory with:
    GET /blocks, GET /blocks/{blockId},
    POST /blocks (body {sessionId, rawContent, topicHints}),
    POST /blocks/{blockId}/content (body {rawContent, topicHints}),
    GET /sessions/{sessionId}/counter,
    POST /sessions/{sessionId}/step,
    POST /agent/query (body {sessionId, prompt}),
    GET /sse (Server-Sent Events for all MemoryBlockEntity events).
    POST /blocks/{blockId}/content also calls InteractionCounterEntity.incrementStep.
    Three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- MemoryTasks.java declaring two Task<R> constants: CONSOLIDATE (ConsolidatedContent),
  DRIFT_EVAL (DriftResult).
- Domain records MemoryBlockContent, ConsolidatedContent, DriftResult, AgentResponse,
  GuardrailRecord, SessionCounter.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9574,
  akka.samples.consolidation-threshold = 5, and the three model-provider blocks
  (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash)
  reading the canonical env vars.
- src/main/resources/sample-events/memory-blocks.jsonl with 6 canned block entries
  covering ACTIVE blocks with varying topic hints and raw content lengths.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls: G1 guardrail before-tool-call
  scope-enforcement, H1 hotl deployer-runtime-monitoring, E1 eval-periodic
  drift-fairness-watch. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with sector=general,
  decisions.authority_level=autonomous-with-oversight,
  oversight.human_on_loop=true, oversight.audit_frequency=continuous,
  failure.failure_modes including "context-drift-undetected" and
  "unauthorized-memory-mutation"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/primary-agent.md, prompts/sleeptime-consolidator.md, prompts/drift-eval.md
  loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Sleeptime Memory-Consolidation Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live block list with status chips, drift score badge, consolidation version;
  right = selected block detail showing raw content summary, consolidated text, drift
  rationale, and HOTL activity stream). Browser title exactly:
  <title>Akka Sample: Sleeptime Memory-Consolidation Agent</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    primary-agent.json — 6–8 AgentResponse entries spanning typical replies
      (brief acknowledgements, clarifying questions, context-informed answers).
      Each entry includes a blockIdRead and blockVersionRead. Replies stay
      under 150 words.
    sleeptime-consolidator.json — 4–6 ConsolidatedContent entries with
      compact consolidatedText (3–4 sentences), a one-sentence changeRationale,
      and a priorVersion of 0 or 1.
    drift-eval.json — 6–8 DriftResult entries spanning scores 5–85 with
      one-sentence rationales and short priorSummary / currentSummary pairs.
      Include at least two entries with driftScore >= 70 to exercise the
      high-drift flag.
- A MockModelProvider.seedFor(blockId) helper makes per-block selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- The SleeptimeConsolidatorAgent NEVER writes directly to MemoryBlockEntity outside
  the ConsolidationWorkflow's commitStep.
- The before-tool-call guardrail on writeMemoryBlock checks caller identity against
  the active ConsolidationWorkflow's blockId lease; unauthorised callers receive a
  structured rejection and a GuardrailTriggered event is emitted.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply in full.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
