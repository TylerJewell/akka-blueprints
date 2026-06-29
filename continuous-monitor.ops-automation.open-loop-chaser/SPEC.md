# SPEC — open-loop-chaser

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Intelligent Reminders.
**One-line pitch:** A background worker monitors meeting notes, messages, and documents for open action items and nudges the responsible owner before the item goes stale.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of two AI primitives (`ActionItemExtractorAgent` and `NudgeComposerAgent`). Specifically:

- A **PII sanitizer** runs inside a Consumer between the raw source text and the first LLM call — the model never sees names, email addresses, or internal identifiers in the source content.
- A **before-tool-call guardrail** on the `sendNudge` tool enforces "nudges can only be dispatched when the item has a confirmed, non-redacted owner set by a human operator". This prevents the system from sending a reminder to a redacted or unknown recipient.

Together these two controls contain the two primary risks of a system that scans broadly and communicates outwardly: PII exposure through the model and unsolicited automated messages to unknown or incorrect recipients.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live action item board: every detected item, its source type, extracted owner, current status, and (if pending) the next nudge due time.
2. `ActionItemPoller` (TimedAction) ticks every 20 s and inserts new simulated source events into `SourceEventQueue`. (A simulator drips canned source documents covering meeting notes, Slack-style messages, and document extracts.)
3. For each new source event: `PiiSanitizer` (Consumer) redacts the payload, then `ActionItemExtractorAgent` parses it and emits one or more `ActionItemDetected` events.
4. Each detected action item enters `PENDING` state. `StaleItemScanner` (TimedAction) ticks every 10 minutes and marks items whose last nudge time (or detection time, if never nudged) exceeds the configured staleness threshold.
5. For each stale item, `NudgeWorkflow` calls `NudgeComposerAgent` to draft a reminder. The before-tool-call guardrail checks that the item's `confirmedOwner` field is non-null before dispatching via `sendNudge`.
6. The operator can mark an item `CLOSED` via the UI — no further nudges fire.
7. The operator can `SNOOZE` an item — nudges are suppressed until the snooze expiry.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ActionItemPoller` | `TimedAction` | Drips simulated source events into `SourceEventQueue` every 20 s. | scheduler | `SourceEventQueue` |
| `SourceEventQueue` | `EventSourcedEntity` | Append-only log of `SourceEventReceived` events from all source types. | `ActionItemPoller`, `ActionItemEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Reads `SourceEventReceived` events, redacts PII from source text, emits `SourceEventSanitized` via `ActionItemEntity`. | `SourceEventQueue` events | `ActionItemEntity` |
| `ActionItemExtractorAgent` | `Agent` (typed, not autonomous) | Parses sanitized source text and extracts a list of `ExtractedActionItem` records. | invoked by `ExtractionWorkflow` | returns `ExtractionResult` |
| `NudgeComposerAgent` | `AutonomousAgent` | Drafts a nudge message for a stale open item. | invoked by `NudgeWorkflow` | returns `ComposedNudge` |
| `ExtractionWorkflow` | `Workflow` | Per-source-event orchestration: sanitize → extract items → persist each as an `ActionItemEntity`. | `PiiSanitizer` (one workflow per `SourceEventSanitized`) | `ActionItemEntity` (one per extracted item) |
| `NudgeWorkflow` | `Workflow` | Per-item nudge orchestration: compose nudge → guardrail check → dispatch → update entity. | `StaleItemScanner` | `ActionItemEntity` |
| `ActionItemEntity` | `EventSourcedEntity` | Lifecycle per action item: detected → sanitized → pending → nudged / snoozed / closed. | `ExtractionWorkflow`, `NudgeWorkflow`, `ActionItemEndpoint` | `ActionItemView` |
| `ActionItemView` | `View` | Read-model row per item for the UI. | `ActionItemEntity` events | `ActionItemEndpoint` |
| `StaleItemScanner` | `TimedAction` | Every 10 min, queries `ActionItemView` for PENDING items past the staleness threshold; starts a `NudgeWorkflow` per item. | scheduler | `NudgeWorkflow` |
| `ActionItemEndpoint` | `HttpEndpoint` | `/api/items/*` — list, get, close, snooze, confirm-owner, SSE. | — | `ActionItemView`, `ActionItemEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record SourceEvent(String eventId, SourceType sourceType, String rawText, String rawAuthor, Instant receivedAt) {}
enum SourceType { MEETING_NOTES, MESSAGE, DOCUMENT }

record SanitizedSource(String redactedText, String redactedAuthor, List<String> piiCategoriesFound) {}

record ExtractedActionItem(String description, String suggestedOwner, Optional<LocalDate> dueDate, String sourceEventId) {}

record ExtractionResult(List<ExtractedActionItem> items, String extractionReason) {}

record ComposedNudge(String recipientHint, String subject, String body, Instant composedAt) {}

record SnoozeRequest(String snoozedBy, Duration snoozeDuration, Instant snoozedAt) {}

record CloseRequest(String closedBy, String closureNote, Instant closedAt) {}

record OwnerConfirmation(String confirmedOwner, String confirmedBy, Instant confirmedAt) {}

record ActionItem(
    String itemId,
    String sourceEventId,
    SourceType sourceType,
    SanitizedSource sanitized,
    ExtractedActionItem extracted,
    Optional<OwnerConfirmation> ownerConfirmation,
    Optional<ComposedNudge> lastNudge,
    Optional<SnoozeRequest> snooze,
    Optional<CloseRequest> closure,
    int nudgeCount,
    ActionItemStatus status,
    Instant detectedAt,
    Optional<Instant> finishedAt
) {}

enum ActionItemStatus {
    DETECTED, SANITIZED, PENDING, STALE, NUDGED, SNOOZED, CLOSED, FAILED
}
```

Events on `ActionItemEntity`: `ActionItemDetected`, `ActionItemSanitized`, `OwnerConfirmed`, `ItemMarkedStale`, `NudgeComposed`, `NudgeDispatched`, `ItemSnoozed`, `SnoozeLapsed`, `ItemClosed`, `ExtractionFailed`.

Events on `SourceEventQueue`: `SourceEventReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/items` — list all action items. Optional `?status=…&sourceType=…`.
- `GET /api/items/{id}` — one item.
- `POST /api/items/{id}/confirm-owner` — body `{ confirmedOwner, confirmedBy }` → sets the confirmed owner field.
- `POST /api/items/{id}/close` — body `{ closedBy, closureNote }` → transitions to CLOSED.
- `POST /api/items/{id}/snooze` — body `{ snoozedBy, durationMinutes }` → transitions to SNOOZED until expiry.
- `GET /api/items/sse` — Server-Sent Events for every item change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Intelligent Reminders</title>`.

App UI tab is the most distinctive: it shows the **live action item board** with status lanes and a detail panel. Operators confirm owners, snooze items, and mark items closed directly from the detail panel.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts names, email addresses, phone numbers, internal identifiers, and personal references from the raw source text before any LLM sees it. Records the categories found.
- **G1 — before-tool-call guardrail** on the `sendNudge` tool: blocks the call unless the item's `ownerConfirmation` field is populated (i.e., a human operator has confirmed the recipient). Prevents automated messages from going to a redacted, inferred, or incorrect owner.

## 9. Agent prompts

- `ActionItemExtractorAgent` → `prompts/action-item-extractor.md`. Typed extractor. Returns a list of `ExtractedActionItem` values; returns an empty list when none are found — never fabricates items.
- `NudgeComposerAgent` → `prompts/nudge-composer.md`. Drafts a reminder staying within the constraints of `SanitizedSource` (it never sees raw names or emails).

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a source event; within 20 s an action item appears in the UI with status PENDING and a sanitized description.
2. **J2** — Operator confirms an owner; `StaleItemScanner` marks the item STALE; `NudgeWorkflow` composes and dispatches a nudge (simulated; no real outbound).
3. **J3** — Operator snoozes an item; it transitions to SNOOZED and does not appear in the STALE query until the snooze window expires.
4. **J4** — Operator marks an item CLOSED; subsequent `StaleItemScanner` ticks skip it.
5. **J5** — An item whose `confirmedOwner` is null cannot have a nudge dispatched — the guardrail rejects the `sendNudge` call and logs the block.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named open-loop-chaser demonstrating the continuous-monitor × ops-automation
cell. Runs out of the box (in-memory source simulator; no real calendar, messaging, or document
integration). Maven group io.akka.samples. Maven artifact
continuous-monitor-ops-automation-open-loop-chaser. Java package
io.akka.samples.intelligentreminders. HTTP port 9408.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ActionItemExtractorAgent — extractor. System prompt loaded from
  prompts/action-item-extractor.md. Input: SanitizedSource{redactedText, redactedAuthor,
  piiCategoriesFound: List<String>, sourceType: SourceType}. Output:
  ExtractionResult{items: List<ExtractedActionItem{description, suggestedOwner,
  dueDate: Optional<LocalDate>, sourceEventId}>, extractionReason: String}. Returns empty
  items list when no action items are found. Never fabricates items.

- 1 AutonomousAgent NudgeComposerAgent — definition() with capability(TaskAcceptance.of(COMPOSE)
  .maxIterationsPerTask(3)). System prompt from prompts/nudge-composer.md. Input:
  ActionItem state + OwnerConfirmation confirmedOwner. Output:
  ComposedNudge{recipientHint, subject, body, composedAt}. The agent NEVER calls sendNudge
  directly — only the workflow calls sendNudge after the guardrail passes.

- 2 Workflows:
  * ExtractionWorkflow per source event with steps: extractStep → persistItemsStep.
    extractStep wraps ActionItemExtractorAgent with WorkflowSettings.builder()
    .stepTimeout(Duration.ofSeconds(15)). persistItemsStep iterates ExtractionResult.items
    and creates one ActionItemEntity per item via ActionItemDetected command.
    If items list is empty, the workflow ends normally (no items to persist).
  * NudgeWorkflow per item with steps: composeStep → guardStep → dispatchStep → finaliseStep.
    composeStep wraps NudgeComposerAgent with stepTimeout 30s.
    guardStep calls the sendNudge guardrail: reads ActionItemEntity.getItem(itemId),
    requires ownerConfirmation.isPresent(). If not, ends workflow with ExtractionFailed event
    (reason="guardrail:no-confirmed-owner"). dispatchStep calls the simulated sendNudge stub
    (logs the nudge; no real network call). finaliseStep emits NudgeDispatched.

- 2 EventSourcedEntities:
  * SourceEventQueue — append-only audit log. Command receive(SourceEvent) emits
    SourceEventReceived{sourceEvent}.
  * ActionItemEntity (one per itemId) — full lifecycle. State ActionItem{itemId, sourceEventId,
    sourceType, sanitized: SanitizedSource{redactedText, redactedAuthor, piiCategoriesFound},
    extracted: ExtractedActionItem{description, suggestedOwner,
    dueDate: Optional<LocalDate>, sourceEventId},
    ownerConfirmation: Optional<OwnerConfirmation{confirmedOwner, confirmedBy, confirmedAt}>,
    lastNudge: Optional<ComposedNudge>, snooze: Optional<SnoozeRequest>,
    closure: Optional<CloseRequest>, nudgeCount: int, status: ActionItemStatus,
    detectedAt: Instant, finishedAt: Optional<Instant>}.
    ActionItemStatus enum: DETECTED, SANITIZED, PENDING, STALE, NUDGED, SNOOZED, CLOSED, FAILED.
    Events: ActionItemDetected, ActionItemSanitized, OwnerConfirmed, ItemMarkedStale,
    NudgeComposed, NudgeDispatched, ItemSnoozed, SnoozeLapsed, ItemClosed, ExtractionFailed.
    Commands: createItem, attachSanitized, confirmOwner, markStale, attachNudge, recordDispatch,
    snoozeItem, lapseSnoze, closeItem, markFailed, getItem. emptyState() returns
    ActionItem.initial("", null) without commandContext() reference.

- 1 Consumer PiiSanitizer subscribed to SourceEventQueue events; for each SourceEventReceived,
  applies a regex+heuristic redaction pipeline (email addresses, phone numbers, personal names
  via NLP heuristics, internal IDs, Slack-style @mentions) to rawText + rawAuthor, builds
  SanitizedSource with piiCategoriesFound, calls ActionItemEntity.attachSanitized for any
  pre-existing item or creates a new one, then starts an ExtractionWorkflow with eventId as
  workflow id.

- 1 View ActionItemView with row type ActionItemRow (mirrors ActionItem minus raw source text).
  Table updater consumes ActionItemEntity events. ONE query getAllItems
  SELECT * AS items FROM action_item_view. No WHERE status filter — caller filters client-side.

- 2 TimedActions:
  * ActionItemPoller — every 20s, reads next line from
    src/main/resources/sample-events/source-events.jsonl and calls SourceEventQueue.receive.
  * StaleItemScanner — every 10 minutes, queries ActionItemView.getAllItems, picks items in
    PENDING or NUDGED status whose last nudge (or detectedAt) exceeds the staleness threshold
    (default 30 minutes), calls ActionItemEntity.markStale per item, then starts a
    NudgeWorkflow per item (using itemId as workflow id, idempotent).

- 2 HttpEndpoints:
  * ActionItemEndpoint at /api with GET /items, GET /items/{id},
    POST /items/{id}/confirm-owner (body {confirmedOwner, confirmedBy}),
    POST /items/{id}/close (body {closedBy, closureNote}),
    POST /items/{id}/snooze (body {snoozedBy, durationMinutes}),
    GET /items/sse, and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. confirm-owner writes OwnerConfirmed; close writes ItemClosed;
    snooze writes ItemSnoozed.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ActionItemTasks.java declaring two Task<R> constants: COMPOSE (ComposedNudge) and
  EXTRACT (ExtractionResult).
- Domain records SourceEvent, SanitizedSource, ExtractedActionItem, ExtractionResult,
  ComposedNudge, SnoozeRequest, CloseRequest, OwnerConfirmation.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9408 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/source-events.jsonl with 10 canned source events spanning
  all three SourceType values (MEETING_NOTES, MESSAGE, DOCUMENT) and producing a mix of items
  that will become PENDING (triggering nudges) and items that will be closed by the user.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, G1 guardrail
  before-tool-call owner-verification. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = notify-only,
  oversight.human_in_loop = true, oversight.owner_must_be_confirmed_before_nudge = true,
  failure.failure_modes including "pii-leakage-via-llm" and "nudge-sent-to-wrong-owner";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/action-item-extractor.md, prompts/nudge-composer.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Intelligent Reminders", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live action item list with status pills, source type chips, staleness indicator;
  right = selected item detail with confirmed owner field + Close / Snooze buttons and
  nudge history). Browser title exactly:
  <title>Akka Sample: Intelligent Reminders</title>. No subtitle on the Overview tab.

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
    action-item-extractor.json — 8–10 ExtractionResult entries. Each entry
      has 1–3 ExtractedActionItem records with realistic descriptions
      (schedule follow-up, send doc, review PR, confirm budget) and
      suggestedOwner values like "owner-A", "owner-B". About half have a
      dueDate set. Two entries return an empty items list (source had no
      action items). extractionReason is one sentence.
    nudge-composer.json — 5–7 ComposedNudge entries with subject lines like
      "Reminder: [action description]", 2–3 paragraph bodies, signed
      "— Team Automation". recipientHint must NOT contain any email address
      or real name. No invented deadlines.
- A `MockModelProvider.seedFor(eventId)` helper makes per-event selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- PiiSanitizer runs INSIDE a Consumer before any LLM call — not inside an Agent's prompt and
  not after the LLM has seen the raw payload.
- The agent NEVER calls sendNudge; only the NudgeWorkflow calls sendNudge after the guardrail
  guard check passes.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during
  generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
- Lessons referenced: 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26.
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
