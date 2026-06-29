# SPEC — merchandiser

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Merchandiser.
**One-line pitch:** A merchant submits a merchandising objective and catalog context; one AI agent reads the catalog (passed as a task attachment, never as inline prompt text) and returns a structured `MerchandisingProposal` — recommended description updates, promotion rules, category reorderings, and pricing nudges — which is held in `PENDING_APPROVAL` until the merchant explicitly approves or rejects it.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the sales-marketing domain. One `MerchandiserAgent` (AutonomousAgent) carries the entire proposal decision; the surrounding components prepare its input, gate its tool calls, and route its output through an approval workflow. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** intercepts every tool invocation the agent attempts before it executes. Write-class tools (`updateProductDescription`, `setPromotion`, `reorderCategory`) are blocked unless the proposal has passed the approval gate. The guardrail logs the attempt, returns a structured rejection to the agent loop, and the agent is forced to use read-only alternatives (fetch, search) during the generation phase. No live storefront changes occur before merchant approval.
- An **application-level HITL** gate holds each `MerchandisingProposal` in `PENDING_APPROVAL` state after the agent returns. The workflow does not advance to the publish step until a merchant calls `POST /api/proposals/{id}/approve` or `POST /api/proposals/{id}/reject`. The approval gate has a configurable timeout; proposals that are neither approved nor rejected within the window transition to `EXPIRED`.

The blueprint shows that a single-agent system can still enforce a meaningful pre-action review boundary: the agent's tool vocabulary is split into read and write tiers, and the boundary is enforced by the guardrail — not by trust.

## 3. User-facing flows

The user opens the App UI tab.

1. The user enters a **merchandising objective** in the text area (or picks one of three seeded examples — a seasonal promotion push, a new-product launch, an inventory-clearance markdown).
2. The user picks a **product scope** from a dropdown (All Products / Category / SKU list) and optionally provides a list of target SKUs.
3. The user clicks **Submit objective**. The UI POSTs to `/api/proposals` and receives a `proposalId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `CONTEXT_LOADED` — the catalog context snapshot is visible in the card detail (product count, category names, current promotion slots).
5. Within ~10–30 s, the workflow's `generateStep` completes. The card transitions to `GENERATING` then `PENDING_APPROVAL`. The proposal appears: a summary of recommended changes, a per-change table (change type, target SKU or category, current value, proposed value, rationale), and a confidence score the agent assigned.
6. The merchant clicks **Approve** or **Reject** on the card. On approval the card transitions to `PUBLISHING` then `PUBLISHED`. On rejection it moves to `REJECTED` with an optional rejection note.
7. The user can submit another objective; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ProposalEndpoint` | `HttpEndpoint` | `/api/proposals/*` — submit, list, get, approve, reject, SSE; serves `/api/metadata/*`. | — | `ProposalEntity`, `ProposalView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ProposalEntity` | `EventSourcedEntity` | Per-proposal lifecycle: submitted → context-loaded → generating → pending-approval → published / rejected / expired. Source of truth. | `ProposalEndpoint`, `CatalogReader`, `ProposalWorkflow` | `ProposalView` |
| `CatalogReader` | `Consumer` | Subscribes to `ObjectiveSubmitted` events; fetches catalog context from seed data; calls `ProposalEntity.attachContext`. | `ProposalEntity` events | `ProposalEntity` |
| `ProposalWorkflow` | `Workflow` | One workflow per proposal. Steps: `awaitContextStep` → `generateStep` → `awaitApprovalStep` → `publishStep`. | started by `CatalogReader` once context event lands | `MerchandiserAgent`, `ProposalEntity` |
| `MerchandiserAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the merchant's objective in the task definition and the catalog snapshot as a task attachment; returns `MerchandisingProposal`. | invoked by `ProposalWorkflow` | returns proposal |
| `StorefrontGuardrail` | supporting class | `before-tool-call` hook on `MerchandiserAgent`. Blocks write-class tool calls during the generation phase. | agent hook | — |
| `ProposalView` | `View` | Read model: one row per proposal for the UI. | `ProposalEntity` events | `ProposalEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ProductScope(
    ScopeKind kind,               // ALL, CATEGORY, SKU_LIST
    @Nullable String categorySlug,
    @Nullable List<String> skuList
) {}
enum ScopeKind { ALL, CATEGORY, SKU_LIST }

record MerchandisingObjective(
    String proposalId,
    String objectiveText,
    ProductScope scope,
    String submittedBy,
    Instant submittedAt
) {}

record ProductSnapshot(
    String sku,
    String name,
    String currentDescription,
    Optional<String> currentPromotion,
    String categorySlug,
    double price
) {}

record CatalogContext(
    List<ProductSnapshot> products,
    List<String> categoryNames,
    int activePromotionCount,
    Instant fetchedAt
) {}

record ChangeRecommendation(
    ChangeType type,
    String targetRef,              // sku or categorySlug
    String currentValue,
    String proposedValue,
    String rationale,
    Confidence confidence
) {}
enum ChangeType { DESCRIPTION_UPDATE, PROMOTION_CREATE, PROMOTION_REMOVE, CATEGORY_REORDER, PRICE_NUDGE }
enum Confidence { LOW, MEDIUM, HIGH }

record MerchandisingProposal(
    String summary,
    List<ChangeRecommendation> changes,
    Confidence overallConfidence,
    Instant proposedAt
) {}

record ApprovalDecision(
    ApprovalStatus status,
    String decidedBy,
    Optional<String> note,
    Instant decidedAt
) {}
enum ApprovalStatus { APPROVED, REJECTED }

record Proposal(
    String proposalId,
    Optional<MerchandisingObjective> objective,
    Optional<CatalogContext> context,
    Optional<MerchandisingProposal> proposal,
    Optional<ApprovalDecision> approval,
    ProposalStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ProposalStatus {
    SUBMITTED, CONTEXT_LOADED, GENERATING, PENDING_APPROVAL, PUBLISHING, PUBLISHED, REJECTED, EXPIRED, FAILED
}
```

Events on `ProposalEntity`: `ObjectiveSubmitted`, `CatalogContextLoaded`, `GenerationStarted`, `ProposalGenerated`, `ProposalApproved`, `ProposalRejected`, `ProposalPublished`, `ProposalExpired`, `ProposalFailed`.

Every nullable lifecycle field on the `Proposal` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/proposals` — body `{ objectiveText, scope: { kind, categorySlug?, skuList? }, submittedBy }` → `{ proposalId }`.
- `GET /api/proposals` — list all proposals, newest-first.
- `GET /api/proposals/{id}` — one proposal.
- `POST /api/proposals/{id}/approve` — body `{ decidedBy, note? }` → `{ proposalId, status }`.
- `POST /api/proposals/{id}/reject` — body `{ decidedBy, note? }` → `{ proposalId, status }`.
- `GET /api/proposals/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Merchandiser</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted proposals (status pill + confidence badge + age) and a right pane with the selected proposal's detail — objective text, product scope, catalog context summary, proposal summary, change-recommendation table, and approval/rejection controls.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call attempted by `MerchandiserAgent`. A tool-call is classified as a write operation if its name is `updateProductDescription`, `setPromotion`, `removePromotion`, or `reorderCategory`. During the generation phase (before the proposal is approved), write-class tool calls are unconditionally blocked. The guardrail returns a structured `blocked-write-tool` error to the agent loop naming the tool, the blocked action, and the instruction to use read-only alternatives. The agent may retry using `fetchProduct` or `searchCatalog` to gather the data it needs and surface the intended changes as part of the `MerchandisingProposal` instead.
- **H1 — application-level HITL approval gate**: `ProposalWorkflow.awaitApprovalStep` halts execution after `ProposalGenerated` and waits for an external signal from `ProposalEndpoint.approve` or `ProposalEndpoint.reject`. The step has a timeout of 72 hours (configurable via `application.conf`); expiry emits `ProposalExpired` and the entity transitions to `EXPIRED`. Only on an `APPROVED` signal does the workflow advance to `publishStep`, which executes the write-class tool calls for real. The human decision — who approved, when, and any note — is persisted as an `ApprovalDecision` on the entity for audit.

## 9. Agent prompts

- `MerchandiserAgent` → `prompts/merchandiser-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached catalog snapshot, evaluate the merchant's objective, and return one `ChangeRecommendation` per proposed change with a rationale and confidence score.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Merchant submits the seasonal-promotion seed objective; within 30 s the proposal appears in `PENDING_APPROVAL` with at least one `ChangeRecommendation` per product in scope and an overall confidence badge.
2. **J2** — The agent attempts to call `setPromotion` during generation (mock LLM path) — the `before-tool-call` guardrail blocks it; the agent retries with `fetchProduct` and includes the intended promotion in the `MerchandisingProposal` instead; the final proposal lands without any write tool having executed.
3. **J3** — A merchant approves a proposal; the entity transitions through `PUBLISHING` to `PUBLISHED`; the approval decision (approver, timestamp, note) is preserved on the entity and visible in the UI.
4. **J4** — A merchant rejects a proposal with a note; the entity transitions to `REJECTED`; the original agent output and the rejection note are both preserved for audit; the UI shows both the proposal changes and the rejection reason side-by-side.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named merchandiser demonstrating the single-agent × sales-marketing cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-sales-marketing-ecommerce-merchandiser. Java package io.akka.samples.merchandiser.
Akka 3.6.0. HTTP port 9967.

Components to wire (exactly):

- 1 AutonomousAgent MerchandiserAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/merchandiser-agent.md>) and
  .capability(TaskAcceptance.of(GENERATE_PROPOSAL).maxIterationsPerTask(4)). The task receives
  the merchant's objective as its instruction text and the catalog snapshot as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). The agent's tool vocabulary during the generation phase includes only read-class calls:
  fetchProduct(sku: String) → ProductSnapshot and searchCatalog(query: String) → List<ProductSnapshot>.
  Write-class tool calls are blocked by the before-tool-call guardrail (see G1 in
  eval-matrix.yaml). Output: MerchandisingProposal{summary: String, changes: List<ChangeRecommendation>,
  overallConfidence: Confidence, proposedAt: Instant}. The agent is configured with a
  before-tool-call guardrail (see G1) registered via the agent's guardrail-configuration block.

- 1 Workflow ProposalWorkflow per proposalId with four steps:
  * awaitContextStep — polls ProposalEntity.getProposal every 1s; on proposal.context().isPresent()
    advances to generateStep. WorkflowSettings.stepTimeout 15s.
  * generateStep — emits GenerationStarted, then calls componentClient.forAutonomousAgent(
    MerchandiserAgent.class, "merchandiser-" + proposalId).runSingleTask(
      TaskDef.instructions(formatObjective(proposal.objective))
        .attachment("catalog.json", serializeCatalogContext(proposal.context).getBytes())
    ) — returns a taskId, then forTask(taskId).result(GENERATE_PROPOSAL) to fetch the proposal.
    On success calls ProposalEntity.recordProposal(merchandisingProposal). WorkflowSettings.stepTimeout 90s
    with defaultStepRecovery maxRetries(2).failoverTo(ProposalWorkflow::error).
  * awaitApprovalStep — polls ProposalEntity.getProposal every 5s; advances to publishStep on
    APPROVED, transitions to error on REJECTED or EXPIRED. WorkflowSettings.stepTimeout 259200s
    (72 hours — see approval gate timeout in application.conf). A ProposalExpired event is emitted
    by the workflow's error step if the approval timeout fires.
  * publishStep — executes write-class tool calls (now permitted because the approval decision is
    present on the entity): calls ProposalEntity.markPublishing(), applies each ChangeRecommendation
    via the appropriate storefront tool (updateProductDescription / setPromotion / removePromotion /
    reorderCategory), then calls ProposalEntity.markPublished(). WorkflowSettings.stepTimeout 60s.
    error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ProposalEntity (one per proposalId). State Proposal{proposalId: String,
  objective: Optional<MerchandisingObjective>, context: Optional<CatalogContext>,
  proposal: Optional<MerchandisingProposal>, approval: Optional<ApprovalDecision>,
  status: ProposalStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  ProposalStatus enum: SUBMITTED, CONTEXT_LOADED, GENERATING, PENDING_APPROVAL, PUBLISHING,
  PUBLISHED, REJECTED, EXPIRED, FAILED.
  Events: ObjectiveSubmitted{objective}, CatalogContextLoaded{context}, GenerationStarted{},
  ProposalGenerated{proposal}, ProposalApproved{decision}, ProposalRejected{decision},
  ProposalPublished{}, ProposalExpired{}, ProposalFailed{reason}.
  Commands: submit, attachContext, markGenerating, recordProposal, approve, reject,
  markPublishing, markPublished, expire, fail, getProposal.
  emptyState() returns Proposal.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer CatalogReader subscribed to ProposalEntity events; on ObjectiveSubmitted reads the
  catalog seed data from src/main/resources/sample-events/catalog-seed.jsonl, filters to the
  products matching the submitted ProductScope (ALL: take all; CATEGORY: filter by categorySlug;
  SKU_LIST: filter by sku), builds CatalogContext, then calls ProposalEntity.attachContext(context).
  After attachContext lands, the same Consumer starts a ProposalWorkflow with id = "proposal-" + proposalId.

- 1 View ProposalView with row type ProposalRow (mirrors Proposal). Table updater consumes
  ProposalEntity events. ONE query getAllProposals: SELECT * AS proposals FROM proposal_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * ProposalEndpoint at /api with:
    POST /proposals (body {objectiveText, scope: {kind, categorySlug?, skuList?}, submittedBy};
      mints proposalId; calls ProposalEntity.submit; returns {proposalId}),
    GET /proposals (list from getAllProposals, sorted newest-first),
    GET /proposals/{id} (one row),
    POST /proposals/{id}/approve (body {decidedBy, note?}; calls ProposalEntity.approve; returns {proposalId, status}),
    POST /proposals/{id}/reject (body {decidedBy, note?}; calls ProposalEntity.reject; returns {proposalId, status}),
    GET /proposals/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ProposalTasks.java declaring one Task<R> constant: GENERATE_PROPOSAL = Task.name("Generate
  merchandising proposal").description("Read the attached catalog snapshot and produce a
  MerchandisingProposal with ChangeRecommendations for the merchant's objective")
  .resultConformsTo(MerchandisingProposal.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records ProductScope, ScopeKind, MerchandisingObjective, ProductSnapshot, CatalogContext,
  ChangeRecommendation, ChangeType, Confidence, MerchandisingProposal, ApprovalDecision,
  ApprovalStatus, Proposal, ProposalStatus.

- StorefrontGuardrail.java implementing the before-tool-call hook. Inspects the tool-call name
  from the agent loop; if the name is updateProductDescription, setPromotion, removePromotion,
  or reorderCategory and the proposal entity is in any pre-approval status, returns
  Guardrail.reject(<structured-error>) with the blocked tool name and the instruction to surface
  the change as a ChangeRecommendation instead. Passes all read-class tool calls through
  unconditionally.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9967 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also include
  akka.javasdk.agent.approval-timeout-seconds = 259200 (72 h default). The
  MerchandiserAgent.definition() binds the configured provider via .modelProvider("${akka.javasdk.agent.default}")
  or the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/catalog-seed.jsonl with 3 seeded catalogs:
  a 12-SKU general retail catalog with 3 categories (Home, Apparel, Electronics),
  a 8-SKU seasonal outdoor catalog with 2 categories (Garden, Leisure),
  and a 6-SKU beauty catalog with 2 categories (Skincare, Haircare).
  Each product entry has sku, name, currentDescription, currentPromotion (null or string),
  categorySlug, and price.

- src/main/resources/sample-events/objectives-seed.jsonl with 3 seeded objectives:
  (1) "Refresh product descriptions for the Home category ahead of the summer season",
  (2) "Create a buy-two-get-one promotion across the Leisure catalog for the July sale",
  (3) "Clearance: cut pricing-nudge recommendations for all Skincare items with inventory > 500".
  Each entry pairs with a ProductScope so the UI can load both with one click.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (product data is
  not PII), decisions.authority_level = recommend-only (the agent's proposal is advisory until
  approved), oversight.human_in_loop = true (merchant must approve before any storefront change),
  failure.failure_modes including "hallucinated-sku", "incorrect-price-nudge",
  "promotion-conflict", "stale-catalog-context", "approval-timeout-expiry";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/merchandiser-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Merchandiser", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of proposal cards with status pill, confidence badge, age; right = selected-proposal
  detail with objective text, scope, catalog context summary, proposal summary, change-
  recommendation table, and approve/reject controls with note input).
  Browser title exactly: <title>Akka Sample: Merchandiser</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(proposalId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-merchandising-proposal.json — 8 MerchandisingProposal entries covering the
      three Confidence values. Each entry has a summary paragraph and a `changes` array with
      2–5 ChangeRecommendation entries. Each ChangeRecommendation has a targetRef matching
      a SKU or category from the seed catalogs, non-empty currentValue and proposedValue,
      an actionable rationale, and a Confidence enum value. ChangeType values vary across
      DESCRIPTION_UPDATE, PROMOTION_CREATE, CATEGORY_REORDER, PRICE_NUDGE. Plus 2
      deliberately MALFORMED entries (one with a ChangeType outside the enum; one that
      attempts to include a direct write action as a tool-call name in the changes list) —
      these exercise the guardrail and the retry path. The mock should select a malformed
      entry on the FIRST iteration of every 3rd proposal (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(proposalId) helper makes per-proposal selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. MerchandiserAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ProposalTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (awaitContextStep 15s, generateStep 90s,
  awaitApprovalStep 259200s, publishStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the Proposal row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ProposalTasks.java with GENERATE_PROPOSAL = Task.name(...).description(...)
  .resultConformsTo(MerchandisingProposal.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9967 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (MerchandiserAgent).
  The approval gate (awaitApprovalStep) is a workflow pause, not a second agent.
- The catalog is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  Verify the generated generateStep uses TaskDef.attachment(...) and not string interpolation
  into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent returns. Lesson 1's AutonomousAgent
  contract is the authoritative reference for how the hook is registered.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
