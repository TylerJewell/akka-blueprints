# SPEC — shared-memory-multi-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Shared-Memory Multi-Agent.
**One-line pitch:** Multiple agents share named memory blocks — a write by one agent is visible to all others in the next read cycle, with an optional sleeptime consolidation pass that merges concurrent fragments into a single coherent snapshot.

## 2. What this blueprint demonstrates

The **team-shared-list** coordination pattern applied to shared agent memory: a roster of writer agents each reads a named `MemoryBlockEntity`, appends a knowledge fragment, and writes it back; the entity's single-writer guarantee stops two writers from corrupting the same block simultaneously. A `ConsolidationWorkflow` runs a consolidator agent on a schedule (and on-demand) to merge all pending raw fragments into a clean snapshot. The blueprint also demonstrates two governance mechanisms that shared memory requires: a **before-tool-call guardrail** that prevents writes to protected block sections, and a **special-category sanitizer** that scrubs protected-attribute data before any fragment is committed to the shared block.

## 3. User-facing flows

The user opens the App UI tab and submits a memory block seed via the form (a block name plus initial content).

1. The system logs the creation on `MemoryEventLog` and a `Consumer` starts a `WriteWorkflow` for each registered writer agent.
2. Each `WriteWorkflow` reads the current `MemoryBlockView` snapshot, calls its `WriterAgent` instance to produce a `MemoryFragment` (a knowledge contribution), and writes the fragment to `FragmentEntity`.
3. Every write passes through a before-tool-call guardrail; a write targeting a protected block section is refused and the fragment is recorded `REJECTED`.
4. Every fragment body is passed through the special-category sanitizer before it is committed; any protected-attribute value is redacted and the fragment is flagged `SANITIZED`.
5. Once a block accumulates more than a threshold number of raw fragments (or the `ConsolidationScheduler` fires), a `ConsolidationWorkflow` starts. It calls `ConsolidatorAgent` to merge all `PENDING` fragments on the block into a single `ConsolidatedSnapshot` and writes the result back to `MemoryBlockEntity`.
6. The updated block (with its consolidated snapshot) is projected into `MemoryBlockView`; all writer agents read the new snapshot on their next cycle.
7. When every registered writer has contributed to a block and at least one successful consolidation has completed, the block reaches `CONSOLIDATED` status.

A `BlockSimulator` (TimedAction) seeds a sample block every 90 seconds so the App UI is not empty on first load. A `FragmentAgeMonitor` (TimedAction) re-queues fragments that have been `PENDING` for more than three minutes, so a stalled consolidation does not leave a block permanently dirty.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `WriterAgent` | `AutonomousAgent` | Reads the current block snapshot and produces a `MemoryFragment` knowledge contribution. Run as several instances (`writer-1`, `writer-2`, `writer-3`). Subject to before-tool-call guardrail (G1) and sanitizer (S1). | `WriteWorkflow` | returns typed `MemoryFragment` to workflow |
| `ConsolidatorAgent` | `AutonomousAgent` | Receives all `PENDING` fragments on a block and returns a `ConsolidatedSnapshot` merging them into coherent content. | `ConsolidationWorkflow` | returns typed `ConsolidatedSnapshot` to workflow |
| `WriteWorkflow` | `Workflow` | Per-writer durable loop: read snapshot → call WriterAgent → guardrail/sanitizer → write fragment → schedule next cycle. | `Bootstrap` (one instance per writer id) | `WriterAgent`, `FragmentEntity`, `MemoryBlockView`, `AgentRegistry` |
| `ConsolidationWorkflow` | `Workflow` | Runs `ConsolidatorAgent` on a named block, commits the merged snapshot to `MemoryBlockEntity`, marks fragments `MERGED`. | `FragmentConsumer`, `ConsolidationScheduler` | `ConsolidatorAgent`, `MemoryBlockEntity`, `FragmentEntity` |
| `MemoryBlockEntity` | `EventSourcedEntity` | One per named block. Holds the current snapshot and lifecycle status. Atomic write prevents concurrent corruption. | `ConsolidationWorkflow`, `MemoryEndpoint` | `MemoryBlockView` |
| `FragmentEntity` | `EventSourcedEntity` | One per submitted fragment. Holds content, provenance, and merge status. | `WriteWorkflow`, `ConsolidationWorkflow`, `FragmentAgeMonitor` | `MemoryBlockView` |
| `AgentRegistry` | `KeyValueEntity` | Lists the registered writer roster with their ids and last-active timestamps. | `MemoryEndpoint`, `Bootstrap` | `WriteWorkflow` |
| `MemoryEventLog` | `EventSourcedEntity` | Single instance; audit log of every block creation and consolidation event. | `MemoryEndpoint`, `BlockSimulator` | `FragmentConsumer` |
| `MemoryBlockView` | `View` | Read model combining the current block snapshot with its pending fragment list; agents and UI query this. | `MemoryBlockEntity` and `FragmentEntity` events | `WriteWorkflow`, `ConsolidationWorkflow`, `MemoryEndpoint` |
| `FragmentConsumer` | `Consumer` | Listens to `FragmentEntity` events; starts a `ConsolidationWorkflow` when pending fragment count exceeds the threshold. | `FragmentEntity` events | `ConsolidationWorkflow` |
| `BlockSimulator` | `TimedAction` | Seeds a sample memory block every 90 s. | scheduler | `MemoryEventLog` |
| `ConsolidationScheduler` | `TimedAction` | Every 120 s, queries `MemoryBlockView` for blocks with `PENDING` fragments older than 2 min; triggers `ConsolidationWorkflow` for each. | scheduler | `ConsolidationWorkflow` |
| `FragmentAgeMonitor` | `TimedAction` | Every 60 s, re-queues fragments `PENDING` for more than 3 min to eligible status so consolidation does not stall. | scheduler | `FragmentEntity` |
| `MemoryEndpoint` | `HttpEndpoint` | `/api/*` — create block, read block, list fragments, trigger consolidation, register/deregister writer, SSE. | — | `MemoryEventLog`, `MemoryBlockEntity`, `FragmentEntity`, `MemoryBlockView`, `AgentRegistry` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the three TimedActions; starts one `WriteWorkflow` per writer id. | — | `WriteWorkflow`, scheduler |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record BlockSeed(String blockId, String blockName, String initialContent, String seededBy) {}

record MemoryFragment(String fragmentId, String blockId, String authorId,
                      String content, FragmentStatus status, Instant createdAt,
                      Optional<String> sanitizedNote) {}

record ConsolidatedSnapshot(String blockId, String content, String mergeSummary,
                             List<String> mergedFragmentIds, Instant consolidatedAt) {}

record WriterContribution(String blockId, String snapshotContent, String authorId) {}
```

### Entity state — `MemoryBlock` (state of `MemoryBlockEntity`, basis of the view row)

```java
record MemoryBlock(
    String blockId,
    String blockName,
    String currentContent,
    BlockStatus status,
    int pendingFragmentCount,
    Optional<String> lastConsolidationSummary,
    Optional<Instant> lastConsolidatedAt,
    Optional<String> lastWriterAgent,
    Instant createdAt,
    Optional<Instant> updatedAt
) {}

enum BlockStatus { SEEDED, ACTIVE, CONSOLIDATING, CONSOLIDATED, ARCHIVED }
```

### Entity state — `Fragment` (state of `FragmentEntity`)

```java
record Fragment(
    String fragmentId,
    String blockId,
    String authorId,
    String content,
    FragmentStatus status,
    boolean sanitized,
    Optional<String> sanitizedNote,
    Instant createdAt,
    Optional<Instant> mergedAt
) {}

enum FragmentStatus { PENDING, SANITIZED, MERGED, REJECTED, REQUEUED }
```

### Events

`MemoryBlockEntity`: `BlockSeeded`, `BlockActivated`, `ConsolidationStarted`, `BlockConsolidated`, `BlockArchived`.
`FragmentEntity`: `FragmentSubmitted`, `FragmentSanitized`, `FragmentMerged`, `FragmentRejected`, `FragmentRequeued`.
`MemoryEventLog`: `BlockCreated`, `ConsolidationTriggered`.
`AgentRegistry` is a `KeyValueEntity` — state transitions are command-driven, no events.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/blocks` — body `{ blockName, initialContent, seededBy? }` → `{ blockId }`. Creates and seeds the block.
- `GET /api/blocks/{id}` — one block with its current snapshot, status, and pending fragment count.
- `GET /api/blocks` — list all blocks. Optional `?status=...` filtered client-side.
- `GET /api/blocks/sse` — server-sent events stream of every block and fragment change.
- `GET /api/blocks/{id}/fragments` — list fragments for a block with status and provenance.
- `POST /api/blocks/{id}/consolidate` — trigger an on-demand consolidation pass.
- `GET /api/agents` — current writer roster from `AgentRegistry`.
- `POST /api/agents` — body `{ agentId, displayName }`. Registers a writer.
- `DELETE /api/agents/{agentId}` — deregisters a writer.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Shared-Memory Multi-Agent</title>`.

- **Overview** — eyebrow "Overview" + headline "Shared-Memory Multi-Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, block state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a block creation form, a live board grouped by block status (Seeded / Active / Consolidating / Consolidated / Archived), per-block cards showing the last writer, pending fragment count, and consolidated snapshot summary, a fragment list panel, and an on-demand Consolidate button.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`tool-permission` flavor, on `WriterAgent`): every write to a memory block passes through a guardrail that refuses writes targeting protected sections (sections named `_system`, `_immutable`, or any section the block owner has locked), attempts to overwrite the consolidated snapshot directly, or any write while the block is in `CONSOLIDATING` status. A refused write records the fragment `REJECTED`. Blocking.
- **S1 — special-category sanitizer** (`special-category` flavor): every fragment body is inspected before commit; any value matching a known protected-attribute pattern (personal identifier, health indicator, financial identifier) is redacted and replaced with a typed placeholder. The fragment is flagged `SANITIZED` and the sanitized fields are listed in `sanitizedNote`. Non-blocking — the fragment is committed but with redacted content.

## 9. Agent prompts

- `WriterAgent` → `prompts/writer-agent.md`. Reads the current block snapshot and contributes a knowledge fragment without overwriting existing content.
- `ConsolidatorAgent` → `prompts/consolidator-agent.md`. Merges a list of raw fragments on a block into a single coherent snapshot.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Seed a block; writer agents each produce fragments; a consolidation pass merges them; the block reaches `CONSOLIDATED`.
2. **J2** — Two writers write the same block concurrently; each fragment is recorded exactly once; no fragment content is lost or duplicated.
3. **J3** — A writer targets a protected section; the before-tool-call guardrail refuses the write; the fragment is recorded `REJECTED` with the guardrail reason.
4. **J4** — A fragment body contains a special-category attribute; the sanitizer redacts it before commit; the committed fragment is flagged `SANITIZED`.
5. **J5** — An on-demand consolidation pass is triggered via `POST /api/blocks/{id}/consolidate`; the consolidated snapshot replaces the previous one and all merged fragments move to `MERGED` status.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named shared-memory-multi-agent demonstrating the
team-shared-list × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact team-shared-list-general-akka-shared-memory-team.
Java package io.akka.samples.sharedmemorymultiagent. Akka 3.6.0. HTTP port 9748.

Components to wire (exactly):
- 2 AutonomousAgents:
  * WriterAgent — definition() with capability(MemoryTasks.of(CONTRIBUTE)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/writer-agent.md.
    Returns MemoryFragment{fragmentId, blockId, authorId, content, status, createdAt,
    sanitizedNote: Optional<String>}.
    Register a before-tool-call guardrail on this agent (control G1) and a
    special-category sanitizer on the outbound content field (control S1).
    Run as several runtime instances addressed by instanceId (writer-1, writer-2,
    writer-3) — ONE agent class, several instance ids.
  * ConsolidatorAgent — definition() with capability(MemoryTasks.of(CONSOLIDATE)
    .maxIterationsPerTask(3)). System prompt from prompts/consolidator-agent.md.
    Returns ConsolidatedSnapshot{blockId, content, mergeSummary,
    mergedFragmentIds: List<String>, consolidatedAt}.

- 1 Workflow WriteWorkflow with steps readSnapshotStep -> contributeStep ->
  sanitizeStep -> writeFragmentStep -> scheduleStep.
  readSnapshotStep: query MemoryBlockView.getBlock(blockId); if block is
  CONSOLIDATING, schedule a 10s resume timer and pause.
  contributeStep (stepTimeout 90s): call forAutonomousAgent(WriterAgent.class,
  writerId).runSingleTask(CONTRIBUTE.instructions(snapshot)). If guardrail refusal
  comes back, call FragmentEntity.reject(reason) and go to scheduleStep.
  sanitizeStep: apply special-category sanitizer to the fragment content; if any
  field is redacted, call FragmentEntity.sanitize(sanitizedNote).
  writeFragmentStep: call FragmentEntity.submit(fragment) and
  MemoryBlockEntity.incrementPending().
  scheduleStep: schedule a 30s resume timer and loop to readSnapshotStep.
  Override settings() with stepTimeout(contributeStep, 90s).

- 1 Workflow ConsolidationWorkflow with steps loadFragmentsStep ->
  consolidateStep -> commitStep -> markMergedStep.
  loadFragmentsStep: query MemoryBlockView.getPendingFragments(blockId); call
  MemoryBlockEntity.startConsolidation().
  consolidateStep (stepTimeout 120s): call
  forAutonomousAgent(ConsolidatorAgent.class, blockId)
  .runSingleTask(CONSOLIDATE.instructions(fragments)).
  commitStep: call MemoryBlockEntity.consolidate(snapshot) (-> CONSOLIDATED).
  markMergedStep: for each fragmentId in snapshot.mergedFragmentIds call
  FragmentEntity.merge(fragmentId, consolidatedAt).
  Override settings() with stepTimeout(consolidateStep, 120s).

- 3 EventSourcedEntities:
  * MemoryBlockEntity holding MemoryBlock state. Commands: seedBlock, activate,
    startConsolidation, consolidate, incrementPending, decrementPending, archive,
    getBlock. consolidate emits BlockConsolidated ONLY when status is CONSOLIDATING.
    BlockStatus enum: SEEDED, ACTIVE, CONSOLIDATING, CONSOLIDATED, ARCHIVED.
    Events: BlockSeeded, BlockActivated, ConsolidationStarted, BlockConsolidated,
    BlockArchived. emptyState() returns MemoryBlock.initial("", "") with NO
    commandContext() reference.
  * FragmentEntity holding Fragment state. Commands: submit, sanitize, merge,
    reject, requeue, getFragment. FragmentStatus enum: PENDING, SANITIZED, MERGED,
    REJECTED, REQUEUED. Events: FragmentSubmitted, FragmentSanitized, FragmentMerged,
    FragmentRejected, FragmentRequeued. emptyState() returns Fragment.initial("", "")
    with NO commandContext() reference.
  * MemoryEventLog, single instance "default". Commands: logBlockCreated,
    logConsolidationTriggered, getLog.
    Events: BlockCreated{blockId, blockName, seededBy, createdAt},
    ConsolidationTriggered{blockId, triggeredBy, triggeredAt}.

- 1 KeyValueEntity AgentRegistry, single instance "default", holding
  {agents: Map<String, AgentEntry{agentId, displayName, registeredAt,
  Optional<Instant> lastActiveAt}>}. Commands: register(agentId, displayName),
  deregister(agentId), touch(agentId), getRegistry.

- 1 View MemoryBlockView with two row types:
  BlockRow (mirrors MemoryBlock minus large content field — keep blockName,
  status, pendingFragmentCount, lastConsolidationSummary, lastConsolidatedAt,
  lastWriterAgent, createdAt).
  FragmentRow (mirrors Fragment — all fields).
  Table updaters consume MemoryBlockEntity and FragmentEntity events.
  TWO queries:
    getAllBlocks SELECT * AS blocks FROM memory_block_view.
    getFragmentsForBlock SELECT * AS fragments FROM fragment_view
      WHERE fragmentId = :blockId.  (Lesson 2: no enum-column WHERE; callers
      filter status client-side.)

- 1 Consumer FragmentConsumer subscribed to FragmentEntity events; on
  FragmentSubmitted or FragmentSanitized, queries MemoryBlockView for the block's
  pending count; if count >= 5 and no ConsolidationWorkflow is already running,
  starts a ConsolidationWorkflow with blockId as workflow id.

- 3 TimedActions:
  * BlockSimulator — every 90s, reads the next line from
    src/main/resources/sample-events/block-seeds.jsonl and calls
    MemoryEventLog.logBlockCreated then MemoryBlockEntity.seedBlock.
    Wraps when file is exhausted.
  * ConsolidationScheduler — every 120s, queries MemoryBlockView.getAllBlocks,
    finds ACTIVE blocks with pendingFragmentCount > 0 and lastConsolidatedAt
    older than 2 minutes (or never consolidated); starts a ConsolidationWorkflow
    per block if not already running.
  * FragmentAgeMonitor — every 60s, queries all fragment rows in PENDING status
    with createdAt older than 3 minutes; calls FragmentEntity.requeue for each
    (-> REQUEUED, eligible for next consolidation pass).

- 2 HttpEndpoints:
  * MemoryEndpoint at /api with POST /blocks, GET /blocks, GET /blocks/{id},
    GET /blocks/sse, GET /blocks/{id}/fragments, POST /blocks/{id}/consolidate,
    GET /agents, POST /agents, DELETE /agents/{agentId}, and three
    /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 service-setup Bootstrap: schedule BlockSimulator, ConsolidationScheduler,
  FragmentAgeMonitor; register default writer roster in AgentRegistry
  (writer-1, writer-2, writer-3); start one WriteWorkflow instance per
  writer id.

Companion files:
- MemoryTasks.java declaring two Task<R> constants: CONTRIBUTE (resultConformsTo
  MemoryFragment) and CONSOLIDATE (resultConformsTo ConsolidatedSnapshot).
- Domain records BlockSeed, MemoryFragment, ConsolidatedSnapshot, WriterContribution
  in domain/.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9748 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also a writer-roster setting
  shared-memory.writers = ["writer-1","writer-2","writer-3"] read by Bootstrap.
- src/main/resources/sample-events/block-seeds.jsonl with 6 canned block seeds
  (each a small, self-contained knowledge domain — e.g., "project-glossary",
  "team-conventions", "architecture-decisions", "known-issues", "api-changelog",
  "test-coverage-notes").
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 2 controls (G1 before-tool-call
  guardrail, S1 special-category sanitizer) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/writer-agent.md and prompts/consolidator-agent.md loaded at agent
  startup as system prompts.
- README.md at the project root: title "Akka Sample: Shared-Memory Multi-Agent",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section. NO "Visual" prefix
  on tab names.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs matching the formal exemplar:
  Overview, Architecture (4 mermaid diagrams + click-to-expand component table
  with syntax-highlighted Java snippets), Risk Survey (7 sections from
  governance.html with answers from risk-survey.yaml; unanswered .qb opacity
  0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table
  with click-to-expand rows and a colored mechanism pill in the ID column), App
  UI (block form + a board with one column per BlockStatus, a fragment list panel,
  and an on-demand Consolidate button per block). Browser title exactly:
  <title>Akka Sample: Shared-Memory Multi-Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via
  the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider block
        below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the
        session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing the
  ModelProvider interface with a per-agent dispatch on the agent class name.
- Per-agent mock-response shapes for THIS blueprint:
    writer-agent.json — 5-6 MemoryFragment entries. Each has a non-blank blockId,
      authorId matching a writer id, content that is a short factual sentence
      about the block topic, status PENDING, and empty sanitizedNote.
      Include 1 entry whose content contains a synthetic personal identifier
      (e.g., "contact: jane.doe@example.com") to exercise the sanitizer (S1).
      Include 1 entry whose content references a protected section name
      "_system" to exercise the guardrail refusal (G1).
    consolidator-agent.json — 3-4 ConsolidatedSnapshot entries. Each merges
      3-5 fictional fragmentIds into a coherent content paragraph plus a
      one-sentence mergeSummary.
- A MockModelProvider.seedFor(blockId) helper makes the selection deterministic
  per block id so the same block produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion MemoryTasks Task<R> constants (Lesson 7).
- View has no WHERE filter on enum status columns; filter client-side (Lesson 2).
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9748 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26).
- UI is a single self-contained static-resources/index.html — no ui/ folder,
  no package.json, no npm build (Lesson 17).
- The Overview tab's Try-it card shows just "/akka:build". Per Lesson 25,
  /akka:specify handles the key during generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, use, use, marketing
  tone, competitor brand names (Lessons 21, 22, 23).
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars plus the mock-LLM option from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
