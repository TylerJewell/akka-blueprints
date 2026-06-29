# SPEC — a2a-interactions

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Agent-to-Agent Interactions Sample.
**One-line pitch:** A delegator agent receives a task over a peer interactions API, verifies the receiver's identity, and hands the task off to a receiver agent that owns the resolution end-to-end, with identity verification before any handoff and an inline eval scoring every delegation decision.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one delegator agent decides *who* should own the task, then transfers the same task identity to a downstream receiver agent that produces the final fulfillment. The downstream agent is responsible for the whole resolution; the delegator does not narrate or summarise. Two governance mechanisms are layered on top:

- A **before-agent-invocation guardrail** runs inside a Consumer between the raw interaction event and any LLM call. It verifies the sender's claimed agent identity and the receiver's registered capability before the handoff is initiated. An interaction whose sender cannot be verified is rejected immediately — no LLM ever sees it.
- An **on-handoff eval** fires every time the delegator agent emits a delegation decision. A `HandoffJudge` agent grades the decision against the incoming task on a 1–5 rubric. The score and rationale are written back to the interaction and surfaced in the UI.

The pattern is a textbook fan-out-of-one: the workflow branches on the delegator's routing decision, and only the named receiver is invoked. All other registered receivers see no traffic for that interaction.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live interaction list. Every interaction displays its task-type chip, status pill, handoff score, and (if fulfilled) the published fulfillment.
2. `InteractionSimulator` (TimedAction) ticks every 30 s and inserts a new canned interaction payload from `sample-events/a2a-interactions.jsonl` into `InteractionQueue`.
3. For each new interaction: `IdentityVerifier` (Consumer) checks the sender's identity, registers an `InteractionEntity`, and starts an `InteractionWorkflow`.
4. The workflow calls `DelegatorAgent`, gets a `DelegationDecision { receiverTag, confidence, reason }`, and emits `DelegationDecided` on the interaction.
5. Branch on `receiverTag`:
   - `RECEIVER` → workflow calls `ReceiverAgent` with the `FULFILL` task and waits for the typed `Fulfillment` result.
   - `UNROUTABLE` → workflow emits `InteractionUnroutable`; ends.
6. The receiver's `Fulfillment` is published: `FulfillmentPublished` is emitted (terminal FULFILLED). If identity verification failed earlier, `InteractionRejected` was already emitted (terminal REJECTED) and the workflow never starts.
7. Independent of the workflow, `HandoffEvalScorer` (Consumer) listens for `DelegationDecided` events, calls `HandoffJudge`, and writes `HandoffScored { score, rationale }` back to the interaction.
8. The user can click any interaction card and see the sender identity check, the delegation reason, the handoff score, the chosen receiver, and the published fulfillment (or rejection reason).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `InteractionSimulator` | `TimedAction` | Drips simulated A2A interactions into `InteractionQueue` every 30 s. | scheduler | `InteractionQueue` |
| `InteractionQueue` | `EventSourcedEntity` | Append-only audit log of every inbound interaction (`InboundInteractionReceived`). | `InteractionSimulator`, `InteractionEndpoint` | `IdentityVerifier` |
| `IdentityVerifier` | `Consumer` | Subscribes to `InteractionQueue` events; verifies sender identity; registers `InteractionEntity`; starts `InteractionWorkflow`. Rejects unverified senders immediately. | `InteractionQueue` events | `InteractionEntity`, `InteractionWorkflow` |
| `DelegatorAgent` | `Agent` (typed, not autonomous) | Routes an `IncomingInteraction` to the correct receiver tag (`RECEIVER` or `UNROUTABLE`) with confidence + reason. | invoked by `InteractionWorkflow` | returns `DelegationDecision` |
| `ReceiverAgent` | `AutonomousAgent` | Owns the `FULFILL` task after the handoff. Returns typed `Fulfillment`. | invoked by `InteractionWorkflow` | returns `Fulfillment` |
| `HandoffJudge` | `Agent` (typed) | Grades a delegation decision against the incoming interaction. Returns `HandoffScore { score 1–5, rationale }`. | invoked by `HandoffEvalScorer` | returns `HandoffScore` |
| `InteractionWorkflow` | `Workflow` | Per-interaction orchestration: verify → delegate → route → fulfill → publish. | `IdentityVerifier` (start) | `InteractionEntity` |
| `InteractionEntity` | `EventSourcedEntity` | Per-interaction lifecycle. | `InteractionWorkflow`, `HandoffEvalScorer` | `InteractionView` |
| `InteractionView` | `View` | Read-model row per interaction. | `InteractionEntity` events | `InteractionEndpoint` |
| `HandoffEvalScorer` | `Consumer` | Subscribes to `InteractionEntity` events; on `DelegationDecided` invokes `HandoffJudge` and writes `HandoffScored` back. | `InteractionEntity` events | `InteractionEntity` |
| `InteractionEndpoint` | `HttpEndpoint` | `/api/interactions/*` — list, get, manual submit, SSE; `/api/metadata/*`. | — | `InteractionView`, `InteractionEntity`, `InteractionQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record IncomingInteraction(
    String interactionId,
    String senderAgentId,
    String senderCapability,   // claimed capability of the sender
    String taskType,           // "FULFILL" | "QUERY" | "REPORT"
    String payload,
    Instant receivedAt
) {}

record VerifiedInteraction(
    String interactionId,
    String verifiedSenderAgentId,
    boolean identityConfirmed,
    List<String> verificationNotes
) {}

enum ReceiverTag { RECEIVER, UNROUTABLE }

record DelegationDecision(
    ReceiverTag receiverTag,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

enum FulfillmentOutcome { COMPLETED, PARTIAL, ESCALATED, DECLINED }

record Fulfillment(
    String responsePayload,
    FulfillmentOutcome outcome,
    String receiverTag,        // "receiver"
    Instant fulfilledAt
) {}

record HandoffScore(
    int score,                 // 1..5
    String rationale,
    Instant scoredAt
) {}

record Interaction(
    String interactionId,
    IncomingInteraction incoming,
    Optional<VerifiedInteraction> verified,
    Optional<DelegationDecision> delegation,
    Optional<Fulfillment> fulfillment,
    Optional<HandoffScore> handoffScore,
    Optional<String> rejectionReason,
    InteractionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum InteractionStatus {
    RECEIVED,
    VERIFIED,
    DELEGATED,
    ROUTED,
    FULFILLMENT_PRODUCED,
    FULFILLED,
    REJECTED,
    UNROUTABLE
}
```

Events on `InteractionEntity`: `InteractionRegistered`, `InteractionVerified`, `DelegationDecided`, `InteractionRouted`, `FulfillmentProduced`, `FulfillmentPublished`, `InteractionRejected`, `InteractionUnroutable`, `HandoffScored`.

Events on `InteractionQueue`: `InboundInteractionReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/interactions` — list all interactions (newest-first), optional `?status=…` filtered client-side.
- `GET /api/interactions/{id}` — one interaction.
- `POST /api/interactions` — manually submit an interaction (body `IncomingInteraction` minus `interactionId` and `receivedAt`); server assigns both.
- `GET /api/interactions/sse` — Server-Sent Events for every interaction change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Agent-to-Agent Interactions Sample</title>`.

The App UI tab is a three-pane layout: **left** is the interaction list (status pill + task-type chip + score chip), **centre** is the selected interaction's incoming payload + identity check + delegation decision + handoff score, **right** is the receiver's fulfillment (or rejection reason, or unroutable block).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-invocation guardrail** (`IdentityVerifier` Consumer): verifies the sender's claimed `senderAgentId` against a registered identity list before any LLM call. An unverified sender causes immediate `REJECTED` — the workflow is never started and no agent sees the payload. Blocking.
- **E1 — on-handoff eval** (`eval-event`, on the delegation decision): `HandoffEvalScorer` (Consumer) listens for `DelegationDecided` events and calls `HandoffJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate; persistent low scores would be surfaced as a system-level alert in a deployed setting.

## 9. Agent prompts

- `DelegatorAgent` → `prompts/delegator-agent.md`. Typed router; returns `RECEIVER` or `UNROUTABLE`; defaults to `UNROUTABLE` under ambiguity.
- `ReceiverAgent` → `prompts/receiver-agent.md`. Owns the `FULFILL` task after the handoff. Returns a typed `Fulfillment`. Never invents capabilities; escalates when outside its declared scope.
- `HandoffJudge` → `prompts/handoff-judge.md`. Grades a delegation decision against a 1–5 rubric with one-sentence rationale.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a routable interaction → verified → delegated to `ReceiverAgent` → fulfilled → published.
2. **J2** — An interaction whose sender fails identity verification is rejected before any LLM call; status `REJECTED`.
3. **J3** — An interaction whose task type has no registered receiver routes to `UNROUTABLE` without invoking `ReceiverAgent`.
4. **J4** — Every delegated interaction carries a `HandoffScore` (1–5) and rationale within ~10 s of the delegation decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named a2a-interactions demonstrating the handoff-routing × general cell.
Runs out of the box (in-process simulated inbound A2A stream; no real external agent registry).
Maven group io.akka.samples. Maven artifact handoff-routing-general-a2a-interactions. Java package
io.akka.samples.agenttoagentinteractionssample. Akka 3.6.0. HTTP port 9725.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) DelegatorAgent — router. System prompt loaded from
  prompts/delegator-agent.md. Input: IncomingInteraction{interactionId, senderAgentId,
  senderCapability, taskType, payload, receivedAt}. Output: DelegationDecision{receiverTag:
  ReceiverTag (RECEIVER/UNROUTABLE), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNROUTABLE under uncertainty.

- 1 AutonomousAgent ReceiverAgent — definition() with capability(TaskAcceptance.of(FULFILL)
  .maxIterationsPerTask(3)). System prompt from prompts/receiver-agent.md. Input:
  IncomingInteraction + DelegationDecision. Output: Fulfillment{responsePayload, outcome:
  FulfillmentOutcome, receiverTag = "receiver", fulfilledAt}. Escalates when outside
  declared capability scope; never invents capabilities.

- 1 Agent (typed) HandoffJudge — judge. System prompt from prompts/handoff-judge.md. Input:
  IncomingInteraction + DelegationDecision. Output: HandoffScore{score: int 1–5, rationale:
  String, scoredAt: Instant}.

- 1 Workflow InteractionWorkflow per interactionId. Steps:
    delegateStep -> routeStep -> {receiverStep | unroutableStep}
                -> publishStep
  delegateStep calls componentClient.forAgent().inSession(interactionId)
    .method(DelegatorAgent::route).invoke(interaction). On success emits DelegationDecided
    via InteractionEntity.recordDelegation.
  routeStep branches on DelegationDecision.receiverTag:
    RECEIVER -> proceed to receiverStep (emits InteractionRouted{RECEIVER})
    UNROUTABLE -> unroutableStep (emits InteractionUnroutable; terminates).
  receiverStep calls forAutonomousAgent(ReceiverAgent.class, interactionId)
    .runSingleTask(TaskDef.instructions(buildPrompt(interaction, delegation))) returning a
    taskId, then forTask(taskId).result(InteractionTasks.FULFILL) to block on the typed
    Fulfillment. On success emits FulfillmentProduced.
  publishStep emits FulfillmentPublished (terminal FULFILLED) and ends.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on delegateStep, and
    stepTimeout(Duration.ofSeconds(60)) on receiverStep and publishStep.
    defaultStepRecovery(maxRetries(2).failoverTo(InteractionWorkflow::error)).

- 2 EventSourcedEntities:
    * InteractionQueue — append-only audit log. Command receive(IncomingInteraction) emits
      InboundInteractionReceived{incoming}. No mutable state beyond a counter; commands are
      idempotent on incoming.interactionId.
    * InteractionEntity (one per interactionId) — full per-interaction lifecycle. State
      Interaction{interactionId, incoming: IncomingInteraction,
      Optional<VerifiedInteraction> verified, Optional<DelegationDecision> delegation,
      Optional<Fulfillment> fulfillment, Optional<HandoffScore> handoffScore,
      Optional<String> rejectionReason, InteractionStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. InteractionStatus enum: RECEIVED, VERIFIED, DELEGATED,
      ROUTED, FULFILLMENT_PRODUCED, FULFILLED, REJECTED, UNROUTABLE.
      Events: InteractionRegistered, InteractionVerified, DelegationDecided,
      InteractionRouted, FulfillmentProduced, FulfillmentPublished, InteractionRejected,
      InteractionUnroutable, HandoffScored. Commands: registerIncoming, attachVerified,
      recordDelegation, recordRouting, recordFulfillment, publish, reject, markUnroutable,
      recordHandoffScore, getInteraction. emptyState() returns Interaction.initial("") with
      no commandContext() reference.

- 2 Consumers:
    * IdentityVerifier subscribed to InteractionQueue events; for each
      InboundInteractionReceived verifies senderAgentId against a built-in registry
      (hard-coded list of known agent ids in a static set, e.g. "agent-alpha",
      "agent-beta", "agent-gamma") and checks that senderCapability is non-empty. If
      verification passes, calls InteractionEntity.registerIncoming then attachVerified
      (identityConfirmed=true) and starts an InteractionWorkflow with interactionId as
      workflow id. If verification fails, calls InteractionEntity.registerIncoming then
      reject(interactionId, "unknown sender agent: " + senderAgentId); workflow is NOT
      started.
    * HandoffEvalScorer subscribed to InteractionEntity events; on DelegationDecided
      invokes HandoffJudge.score(interaction, decision) and calls
      InteractionEntity.recordHandoffScore(interactionId, score). On any other event type,
      no-op. Use componentClient — do NOT call the agent from a TimedAction.

- 1 View InteractionView with row type InteractionRow (mirrors Interaction; uses
  Optional<T> for every nullable lifecycle field per Lesson 6). Table updater consumes
  InteractionEntity events. ONE query getAllInteractions SELECT * AS interactions FROM
  interaction_view. No WHERE status filter (Akka cannot auto-index enum columns) — filter
  client-side in callers.

- 1 TimedAction InteractionSimulator — every 30s, reads next line from
  src/main/resources/sample-events/a2a-interactions.jsonl (loops at EOF) and calls
  InteractionQueue.receive with a fresh interactionId (UUID).

- 2 HttpEndpoints:
    * InteractionEndpoint at /api with GET /interactions (list from
      InteractionView.getAllInteractions, filter client-side by ?status query param),
      GET /interactions/{id}, POST /interactions (body IncomingInteraction minus
      interactionId/receivedAt — server assigns), GET /interactions/sse
      (serverSentEventsForView over getAllInteractions), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- InteractionTasks.java declaring the task constants: FULFILL (resultConformsTo
  Fulfillment.class, description "Fulfill the interaction task end-to-end and return a
  typed Fulfillment").
- Domain records IncomingInteraction, VerifiedInteraction, DelegationDecision, Fulfillment,
  HandoffScore, and the Interaction entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9725 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/a2a-interactions.jsonl with 9 canned lines (4 from known
  sender agents with routable task types, 3 from known sender agents with UNROUTABLE task
  types, 1 from an unknown sender agent to trigger REJECTED, 1 designed to exercise a
  low-confidence delegation decision).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: G1 guardrail
  before-agent-invocation, E1 eval-event on-handoff-eval. Matching simplified_view list.
  No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = agent-coordination,
  data.data_classes.pii = false, decisions.authority_level = autonomous-routing,
  oversight.human_in_loop = false, failure.failure_modes including "wrong-receiver-routing",
  "unverified-sender-accepted", "capability-mismatch", "fulfillment-fabrication";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/delegator-agent.md, prompts/receiver-agent.md, prompts/handoff-judge.md loaded as
  agent system prompts.
- README.md at the project root: title "Akka Sample: Agent-to-Agent Interactions Sample",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = interaction list with task-type chip + status pill + score chip; centre =
  incoming payload + identity check + delegation block + handoff score; right = receiver
  fulfillment or rejection reason or unroutable block). Browser title exactly:
  <title>Akka Sample: Agent-to-Agent Interactions Sample</title>. No subtitle on the
  Overview tab.

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
  pseudo-randomly per call (seeded by interactionId so reruns are
  deterministic), and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    delegator-agent.json — 10 DelegationDecision entries: 5 with
      receiverTag=RECEIVER (high/medium confidence, clear routing reasons),
      3 with receiverTag=UNROUTABLE (task type not in registry), 2 with
      receiverTag=RECEIVER but low confidence (exercises the judge's
      calibration rubric).
    receiver-agent.json — 8 Fulfillment entries: 5 with outcome COMPLETED
      (within declared capability), 1 with outcome PARTIAL (task partially
      fulfilled, one sub-step escalated), 1 with outcome ESCALATED (outside
      declared scope), 1 with outcome DECLINED (payload insufficient to
      fulfill task).
    handoff-judge.json — 10 HandoffScore entries, score 1–5, one-sentence
      rationale matching the rubric (receiver-correctness / confidence-
      calibration / reason-quality).
- A MockModelProvider.seedFor(interactionId) helper makes per-interaction
  selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  ReceiverAgent extends akka.javasdk.agent.autonomous.AutonomousAgent and
  declares definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): delegateStep 20s,
  receiverStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Interaction is Optional<T>. The
  InteractionView row type uses the same Optional wrapping.
- (Lesson 7) InteractionTasks.java declares the FULFILL Task<Fulfillment> constant.
  ReceiverAgent's definition().capability(TaskAcceptance.of(FULFILL)...)
  references it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9725 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier label is "Runs out of the box" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour white, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels.
- The IdentityVerifier runs INSIDE a Consumer before any LLM call — not inside
  an Agent's prompt and not after the LLM has seen the raw payload.
- The HandoffEvalScorer Consumer reacts to DelegationDecided events and calls
  HandoffJudge via componentClient.forAgent(). It does NOT modify the workflow
  flow — the eval is out-of-band metadata.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, leverage, utilize, marketing
  tone, competitor brand names.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local key-source reference written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
