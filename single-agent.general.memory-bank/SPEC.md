# SPEC — memory-bank

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** MemoryBank.
**One-line pitch:** A user or calling agent submits a fact to remember or a query to recall; one AI agent stores structured memory entries and returns a ranked list of matching results, with PII stripped before the model ever indexes or reasons over the content.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `MemoryAgent` (AutonomousAgent) handles both storage decisions and recall ranking; the surrounding components only prepare its input and audit its output. One governance mechanism is wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw memory submission and the agent call — so the model never sees identifiers embedded in stored facts.

The blueprint shows a foundational persistence capability pattern: the agent maintains continuity of context across interactions without storing raw personal data in a form the model can see.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a fact into the **Remember** input (e.g., "My preferred meeting time is 9 AM") and optionally assigns it to a **namespace** (personal-assistant, project-notes, or custom).
2. The user clicks **Store**. The UI POSTs to `/api/memories` and receives a `memoryId`.
3. The entry card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `SANITIZED` — the redacted content is visible in the card detail, with a small list of PII categories the sanitizer found.
4. Within ~5–15 s, the workflow's `storeStep` completes. The card transitions to `STORING` then `STORED`. The agent's confirmation appears: a one-line acknowledgement and a set of `tags` the agent extracted.
5. The user types a recall query into the **Recall** input (e.g., "What time do I prefer for meetings?") and clicks **Recall**. The UI POSTs to `/api/memories/recall`.
6. Within ~5–15 s, a `RecallResult` appears: a ranked list of matching memory entries, each with a relevance score and the matching sanitized content.
7. The user can add more entries; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MemoryEndpoint` | `HttpEndpoint` | `/api/memories/*` — store, recall, list, get, SSE; serves `/api/metadata/*`. | — | `MemoryEntity`, `MemoryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `MemoryEntity` | `EventSourcedEntity` | Per-entry lifecycle: submitted → sanitized → storing → stored. Source of truth. | `MemoryEndpoint`, `MemorySanitizer`, `MemoryWorkflow` | `MemoryView` |
| `MemorySanitizer` | `Consumer` | Subscribes to `MemoryEntrySubmitted` events; redacts PII; calls `MemoryEntity.attachSanitized`. | `MemoryEntity` events | `MemoryEntity` |
| `MemoryWorkflow` | `Workflow` | One workflow per memory operation. Steps: `awaitSanitizedStep` → `storeStep`. | started by `MemorySanitizer` once sanitized event lands | `MemoryAgent`, `MemoryEntity` |
| `MemoryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the operation type (remember/recall) and sanitized content; returns `StoreConfirmation` or `RecallResult`. | invoked by `MemoryWorkflow` | returns result |
| `MemoryView` | `View` | Read model: one row per memory entry for the UI. | `MemoryEntity` events | `MemoryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum MemoryOperation { REMEMBER, RECALL }
enum MemoryNamespace { PERSONAL_ASSISTANT, PROJECT_NOTES, CUSTOM }
enum MemoryStatus { SUBMITTED, SANITIZED, STORING, STORED, FAILED }

record MemoryRequest(
    String memoryId,
    MemoryOperation operation,
    MemoryNamespace namespace,
    String rawContent,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedContent(
    String redactedContent,
    List<String> piiCategoriesFound
) {}

record MemoryEntry(
    String entryId,
    String sanitizedContent,
    List<String> tags,
    MemoryNamespace namespace,
    Instant storedAt
) {}

record StoreConfirmation(
    String memoryId,
    String acknowledgement,
    List<String> tags,
    Instant confirmedAt
) {}

record RecallMatch(
    String memoryId,
    String sanitizedContent,
    List<String> tags,
    float relevanceScore,
    Instant storedAt
) {}

record RecallResult(
    String queryContent,
    List<RecallMatch> matches,
    Instant recalledAt
) {}

record Memory(
    String memoryId,
    Optional<MemoryRequest> request,
    Optional<SanitizedContent> sanitized,
    Optional<StoreConfirmation> confirmation,
    Optional<RecallResult> recallResult,
    MemoryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}
```

Events on `MemoryEntity`: `MemoryEntrySubmitted`, `MemoryEntrySanitized`, `MemoryStoringStarted`, `MemoryEntryStored`, `MemoryFailed`.

Every nullable lifecycle field on the `Memory` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/memories` — body `{ operation, namespace, rawContent, submittedBy }` → `{ memoryId }`.
- `POST /api/memories/recall` — body `{ queryContent, namespace, submittedBy }` → `{ memoryId }` (recall is itself a memory operation).
- `GET /api/memories` — list all memory entries, newest-first.
- `GET /api/memories/{id}` — one memory entry.
- `GET /api/memories/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: MemoryBank</title>`.

The App UI tab is a two-column layout: a left rail with the submission panel (operation toggle, namespace selector, content input, submit button) and the live list of memory entries (status pill + operation badge + age); a right pane with the selected entry's detail — sanitized content, PII category chips, tags, and recall result list (for RECALL operations).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MemorySanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from the raw memory content before any LLM call. Records which categories were found. The raw content is preserved on the entity for audit purposes only.

## 9. Agent prompts

- `MemoryAgent` → `prompts/memory-agent.md`. The single decision-making LLM. System prompt instructs it to interpret the operation type and sanitized content: for REMEMBER operations, extract tags and return a `StoreConfirmation`; for RECALL operations, rank stored entries by relevance and return a `RecallResult`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User stores a fact; within 15 s the entry appears with agent-extracted tags and `STORED` status.
2. **J2** — User submits a recall query; within 15 s a ranked `RecallResult` appears with at least one match from previously stored entries.
3. **J3** — A memory entry containing `john.smith@example.com` and `+1-800-555-0199` is submitted; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`; the entity's `request.rawContent` retains the raw text for audit.
4. **J4** — A recall query against an empty namespace returns an empty `matches` list and status `STORED` with the agent's confirmation that no relevant entries were found.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named memory-bank demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-memory-bank. Java package io.akka.samples.memorybank. Akka 3.6.0.
HTTP port 9876.

Components to wire (exactly):

- 1 AutonomousAgent MemoryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/memory-agent.md>) and
  .capability(TaskAcceptance.of(REMEMBER_CONTENT, RECALL_CONTENT).maxIterationsPerTask(3)).
  For REMEMBER tasks: receives sanitized content and namespace as task instructions; returns
  StoreConfirmation{memoryId, acknowledgement, tags, confirmedAt}. For RECALL tasks: receives
  query content and namespace; returns RecallResult{queryContent, matches: List<RecallMatch>,
  recalledAt}. Each RecallMatch has {memoryId, sanitizedContent, tags, relevanceScore, storedAt}.
  The agent is NOT configured with a before-agent-response guardrail — this baseline keeps
  governance minimal to focus on the sanitizer pattern.

- 1 Workflow MemoryWorkflow per memoryId with two steps:
  * awaitSanitizedStep — polls MemoryEntity.getMemory every 1s; on memory.sanitized().isPresent()
    advances to storeStep. WorkflowSettings.stepTimeout 15s (sanitizer is in-process and fast).
  * storeStep — emits MemoryStoringStarted, then calls componentClient.forAutonomousAgent(
    MemoryAgent.class, "memory-" + memoryId).runSingleTask(taskDef) where taskDef is built
    from the operation type: REMEMBER → Task REMEMBER_CONTENT with instructions containing
    namespace + sanitized content; RECALL → Task RECALL_CONTENT with instructions containing
    namespace + query text + list of stored entries from MemoryView. On success calls
    MemoryEntity.recordResult(result). WorkflowSettings.stepTimeout 30s with
    defaultStepRecovery maxRetries(2).failoverTo(MemoryWorkflow::error). error step
    transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity MemoryEntity (one per memoryId). State Memory{memoryId: String,
  request: Optional<MemoryRequest>, sanitized: Optional<SanitizedContent>,
  confirmation: Optional<StoreConfirmation>, recallResult: Optional<RecallResult>,
  status: MemoryStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  MemoryStatus enum: SUBMITTED, SANITIZED, STORING, STORED, FAILED. Events:
  MemoryEntrySubmitted{request}, MemoryEntrySanitized{sanitized}, MemoryStoringStarted{},
  MemoryEntryStored{confirmation | recallResult}, MemoryFailed{reason}. Commands: submit,
  attachSanitized, markStoring, recordResult, fail, getMemory. emptyState() returns
  Memory.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer MemorySanitizer subscribed to MemoryEntity events; on MemoryEntrySubmitted runs
  a regex+heuristic redaction pipeline (emails, phone numbers, SSN-like, payment-card-like,
  postal addresses, person-name heuristic, account-id-like tokens) over rawContent, computes
  the list of categories found, builds SanitizedContent, then calls
  MemoryEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer
  starts a MemoryWorkflow with id = "memory-" + memoryId.

- 1 View MemoryView with row type MemoryRow (mirrors Memory minus request.rawContent — the
  audit log keeps the raw; the view holds the sanitized form for the UI and for RECALL
  context injection). Table updater consumes MemoryEntity events. ONE query getAllMemories:
  SELECT * AS memories FROM memory_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * MemoryEndpoint at /api with:
    POST /memories (body {operation: REMEMBER|RECALL, namespace, rawContent, submittedBy};
      mints memoryId; calls MemoryEntity.submit; returns {memoryId}),
    POST /memories/recall (shorthand — same as POST /memories with operation=RECALL),
    GET /memories (list from getAllMemories, sorted newest-first),
    GET /memories/{id} (one row),
    GET /memories/sse (Server-Sent Events forwarded from the view's stream-updates), and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- MemoryTasks.java declaring two Task<R> constants:
    REMEMBER_CONTENT = Task.name("Remember content")
      .description("Store the sanitized content with extracted tags and return a StoreConfirmation")
      .resultConformsTo(StoreConfirmation.class);
    RECALL_CONTENT = Task.name("Recall content")
      .description("Search stored memory entries for the query and return a ranked RecallResult")
      .resultConformsTo(RecallResult.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records: MemoryOperation, MemoryNamespace, MemoryStatus, MemoryRequest,
  SanitizedContent, MemoryEntry, StoreConfirmation, RecallMatch, RecallResult, Memory.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9876 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The MemoryAgent.definition() binds
  the configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-memories.jsonl with 6 seeded memory entries across
  the three namespaces: 2 in personal-assistant (meeting time preference, dietary restriction),
  2 in project-notes (sprint goal, tech stack decision), 2 in custom (a reminder and a
  reference fact). Each contains 1–2 plausible PII strings so S1 has work to do.

- src/main/resources/sample-events/seed-queries.jsonl with 3 recall queries paired with their
  expected namespace and at least one matching seed entry.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (S1) matching the mechanism in Section 8
  of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = autonomous
  (the agent stores and retrieves without human approval), oversight.human_in_loop = false
  (memory operations are automatic), failure.failure_modes including "pii-leakage-via-llm",
  "stale-memory-returned", "irrelevant-recall-ranking", "over-redaction-losing-meaning";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/memory-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: MemoryBank", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  submission panel + live list of memory cards; right = selected-entry detail with operation
  badge, sanitized content, PII category chips, extracted tags, and recall result list for
  RECALL operations).
  Browser title exactly: <title>Akka Sample: MemoryBank</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent task type. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id.
- Per-task mock-response shapes for THIS blueprint:
    remember-content.json — 6 StoreConfirmation entries with varied acknowledgement strings
      and tags arrays (2–4 tags per entry). Each entry has namespace-appropriate tags.
    recall-content.json — 4 RecallResult entries with varied matches lists (1–3 matches
      each). Each match has a non-empty sanitizedContent snippet, 1–2 tags, and a
      relevanceScore between 0.5 and 1.0.
- MockModelProvider.seedFor(memoryId) makes per-entry selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. MemoryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion MemoryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (awaitSanitizedStep 15s,
  storeStep 30s, error 5s).
- Lesson 6: every nullable lifecycle field on the Memory row record is Optional<T>.
- Lesson 7: MemoryTasks.java with REMEMBER_CONTENT and RECALL_CONTENT constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9876 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (MemoryAgent). Context
  injection for recall (passing stored entries to the agent) is done by constructing the
  task instructions string in MemoryWorkflow, NOT by wiring a second agent.
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
