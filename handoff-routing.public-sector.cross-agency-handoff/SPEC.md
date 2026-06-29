# SPEC — cross-agency-case-handoff-mesh

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Cross-Agency Case Handoff Mesh.
**One-line pitch:** A constituent case is classified by a jurisdiction router, then transferred across agency boundaries via a handoff mesh in which each agency's autonomous agent owns its segment, a case-owner approves the handoff via a HITL step, and a before-agent-invocation guardrail verifies the receiving agency's scope before any model call.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern in a public-sector context — one router agent determines which agency segment should own the case, then the same case identity travels through an ordered mesh of agency-owned processing steps. Each agency's agent owns its segment end-to-end; the router does not narrate or aggregate. Three governance mechanisms are layered on top:

- A **PII scoping filter** runs inside a Consumer before every agency's agent sees the case payload. Each agency is authorised to see only its own fields; cross-boundary fields are dropped or redacted at handoff. The categories of dropped fields are recorded for audit.
- An **application-level HITL** step fires before each handoff is released to the next agency. The case-owner at the current agency reviews the outgoing case summary and either approves or rejects the transfer. A rejection puts the case in `HANDOFF_REJECTED` and no downstream agent is invoked.
- A **before-agent-invocation guardrail** runs immediately before invoking the receiving agency's agent. It verifies that the routed case falls within that agency's declared jurisdiction (program type, geographic region, eligibility tier). A jurisdiction mismatch blocks the invocation and transitions the case to `JURISDICTION_BLOCKED`.

The mesh is a sequential chain of at most two agency segments in this baseline. Each segment re-emits a `HandoffEmitted` event that records who owned the segment, what they produced, and who is receiving next.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live case list. Every case displays its segment badge, status pill, and (if closed) the final determination.
2. `CaseSimulator` (TimedAction) ticks every 30 s and inserts a new canned case from `sample-events/constituent-cases.jsonl` into `CaseInbox`.
3. For each new case: `PiiScopingFilter` (Consumer) scopes the payload for the intake segment, registers a `CaseEntity`, and starts a `HandoffWorkflow`.
4. The workflow calls `JurisdictionRouter`, gets a `RoutingDecision { segment, confidence, rationale }`, and emits `CaseRouted` on the entity.
5. Branch on `segment`:
   - `INTAKE` → workflow calls `JurisdictionGuardrail` to verify intake scope, then if cleared calls `IntakeAssessor` with the `ASSESS` task. On result, emits `SegmentCompleted`.
   - `BENEFITS` → guardrail verifies benefits scope, then calls `BenefitsReviewer` with the `REVIEW` task. On result, emits `SegmentCompleted`.
   - `UNROUTABLE` → workflow emits `CaseUnroutable`; ends in `UNROUTABLE`.
6. After `SegmentCompleted`, the workflow enters the HITL approval step: it records `HandoffPending` on the entity and waits for an operator call to `POST /api/cases/{id}/approve-handoff` or `POST /api/cases/{id}/reject-handoff`.
   - Approve → workflow emits `HandoffEmitted`; if a next segment exists, repeat from step 5. If the current segment is the last in the chain, emit `CaseClosed` (terminal `CLOSED`).
   - Reject → workflow emits `HandoffRejected`; terminal `HANDOFF_REJECTED`.
7. When `JurisdictionGuardrail` returns `allowed=false`, the workflow emits `JurisdictionBlocked`; terminal `JURISDICTION_BLOCKED`. No agent is invoked.
8. The user can click any case card and see the scoped payload, the routing decision, each segment's determination, the approval history, and any jurisdiction violations.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CaseSimulator` | `TimedAction` | Drips simulated constituent cases every 30 s. | scheduler | `CaseInbox` |
| `CaseInbox` | `EventSourcedEntity` | Append-only audit log of every inbound case (`InboundCaseReceived`). | `CaseSimulator`, `CaseEndpoint` | `PiiScopingFilter` |
| `PiiScopingFilter` | `Consumer` | Subscribes to `CaseInbox` events; scopes PII per receiving segment; registers `CaseEntity`; starts `HandoffWorkflow`. | `CaseInbox` events | `CaseEntity`, `HandoffWorkflow` |
| `JurisdictionRouter` | `Agent` (typed, not autonomous) | Classifies a `ScopedCase` into `INTAKE` / `BENEFITS` / `UNROUTABLE` with confidence + rationale. | invoked by `HandoffWorkflow` | returns `RoutingDecision` |
| `IntakeAssessor` | `AutonomousAgent` | Owns the `ASSESS` task for intake-segment cases. Returns typed `SegmentOutcome`. | invoked by `HandoffWorkflow` | returns `SegmentOutcome` |
| `BenefitsReviewer` | `AutonomousAgent` | Owns the `REVIEW` task for benefits-segment cases. Returns typed `SegmentOutcome`. | invoked by `HandoffWorkflow` | returns `SegmentOutcome` |
| `JurisdictionGuardrail` | `Agent` (typed) | Before-agent-invocation guardrail: verifies the receiving agency's jurisdiction scope. Returns `JurisdictionVerdict { allowed, violations }`. | invoked by `HandoffWorkflow` before each agent | returns `JurisdictionVerdict` |
| `HandoffWorkflow` | `Workflow` | Per-case orchestration: scope → route → guardrail → agent segment → HITL approval → re-emit or close. | `PiiScopingFilter` (start) | `CaseEntity` |
| `CaseEntity` | `EventSourcedEntity` | Per-case cross-agency lifecycle. | `HandoffWorkflow`, `CaseEndpoint` | `CaseView` |
| `CaseView` | `View` | Read-model row per case. | `CaseEntity` events | `CaseEndpoint` |
| `CaseEndpoint` | `HttpEndpoint` | `/api/cases/*` — list, get, approve-handoff, reject-handoff, manual submit, SSE; metadata. | — | `CaseView`, `CaseEntity`, `CaseInbox` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record InboundCase(
    String caseId,
    String constituentId,
    String programType,       // "food-assistance" | "housing-aid" | "child-services"
    String geographicRegion,  // ISO 3166-2 subdivision code
    String summary,
    String fullPayload,       // raw; may contain PII
    Instant submittedAt
) {}

record ScopedCase(
    String caseId,
    String programType,
    String geographicRegion,
    String redactedSummary,
    String redactedPayload,
    List<String> droppedFieldCategories  // fields withheld for this agency
) {}

enum CaseSegment { INTAKE, BENEFITS, UNROUTABLE }

record RoutingDecision(
    CaseSegment segment,
    String confidence,         // "high" | "medium" | "low"
    String rationale           // one short sentence
) {}

record JurisdictionVerdict(
    boolean allowed,
    List<String> violations,   // empty when allowed
    String agencyCode,         // receiving agency identifier
    String policyVersion
) {}

enum DeterminationType {
    ELIGIBLE, INELIGIBLE, REFERRED_TO_BENEFITS, AWARD_ISSUED,
    AWARD_DENIED, PENDING_DOCUMENTS, ESCALATED
}

record SegmentOutcome(
    CaseSegment segment,
    DeterminationType determination,
    String summaryNote,        // 2–4 sentences
    String agencyTag,          // "intake" | "benefits"
    Instant completedAt
) {}

record HandoffApproval(
    String approvedBy,
    String note,
    Instant approvedAt
) {}

record HandoffRejection(
    String rejectedBy,
    String reason,
    Instant rejectedAt
) {}

record Case(
    String caseId,
    InboundCase inbound,
    Optional<ScopedCase> scoped,
    Optional<RoutingDecision> routing,
    Optional<JurisdictionVerdict> jurisdictionVerdict,
    Optional<SegmentOutcome> intakeOutcome,
    Optional<SegmentOutcome> benefitsOutcome,
    Optional<HandoffApproval> handoffApproval,
    Optional<HandoffRejection> handoffRejection,
    CaseStatus status,
    Instant createdAt,
    Optional<Instant> closedAt
) {}

enum CaseStatus {
    RECEIVED,
    SCOPED,
    ROUTED,
    JURISDICTION_CHECK,
    IN_SEGMENT,
    SEGMENT_COMPLETE,
    HANDOFF_PENDING,
    HANDOFF_EMITTED,
    HANDOFF_REJECTED,
    JURISDICTION_BLOCKED,
    CLOSED,
    UNROUTABLE
}
```

Events on `CaseEntity`: `CaseRegistered`, `CaseScoped`, `CaseRouted`, `JurisdictionChecked`, `SegmentStarted`, `SegmentCompleted`, `HandoffPending`, `HandoffEmitted`, `HandoffRejected`, `JurisdictionBlocked`, `CaseClosed`, `CaseUnroutable`.

Events on `CaseInbox`: `InboundCaseReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/cases` — list all cases (newest-first), optional `?segment=INTAKE|BENEFITS&status=…` filtered client-side.
- `GET /api/cases/{id}` — one case.
- `POST /api/cases` — manually submit a case (body `InboundCase` minus `caseId` and `submittedAt`); server assigns both.
- `POST /api/cases/{id}/approve-handoff` — body `{ approvedBy, note }` — case-owner approves the pending handoff; transitions `HANDOFF_PENDING` → `HANDOFF_EMITTED`.
- `POST /api/cases/{id}/reject-handoff` — body `{ rejectedBy, reason }` — case-owner rejects; transitions `HANDOFF_PENDING` → `HANDOFF_REJECTED`.
- `GET /api/cases/sse` — Server-Sent Events for every case change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Cross-Agency Case Handoff Mesh</title>`.

The App UI tab is a three-pane layout: **left** is the case list (status pill + segment badge + program-type chip), **centre** is the selected case's scoped payload + routing decision + jurisdiction verdict, **right** is the active segment's outcome + approval panel (Approve / Reject buttons when `HANDOFF_PENDING`) or the closure summary.

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII scoping filter** (`pii`, applied in `PiiScopingFilter` Consumer): each agency is authorised to see only its segment's required fields. At every cross-boundary handoff, out-of-scope fields are dropped and the dropped-field categories are logged. No agency's agent ever sees a field outside its declared scope.
- **H1 — application-level HITL** on every inter-agency handoff: `HandoffWorkflow.approvalStep` emits `HandoffPending` and suspends. It only resumes when an operator calls `POST /api/cases/{id}/approve-handoff` (proceeds) or `POST /api/cases/{id}/reject-handoff` (terminal `HANDOFF_REJECTED`). No downstream agency agent is invoked without explicit human sign-off.
- **G1 — before-agent-invocation guardrail** (`JurisdictionGuardrail`): verifies that the routed case is within the receiving agency's declared scope (program type, geographic region, eligibility tier) before invoking `IntakeAssessor` or `BenefitsReviewer`. A `JurisdictionVerdict { allowed=false }` terminates the workflow in `JURISDICTION_BLOCKED` — the agent is never called.

## 9. Agent prompts

- `JurisdictionRouter` → `prompts/jurisdiction-router.md`. Typed classifier; returns one of `INTAKE`, `BENEFITS`, `UNROUTABLE`; defaults to `UNROUTABLE` under ambiguity.
- `IntakeAssessor` → `prompts/intake-assessor.md`. Owns the `ASSESS` task for intake cases. Returns a `SegmentOutcome` with a determination and a 2–4 sentence note.
- `BenefitsReviewer` → `prompts/benefits-reviewer.md`. Owns the `REVIEW` task for benefits cases. Returns a `SegmentOutcome` with an entitlement determination.
- `JurisdictionGuardrail` → `prompts/jurisdiction-guardrail.md`. Returns a `JurisdictionVerdict { allowed, violations, agencyCode, policyVersion }`. Conservative — borderline cases are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an intake-eligible case → scoped, routed `INTAKE`, jurisdiction clears, `IntakeAssessor` returns `ELIGIBLE`, case-owner approves handoff → `HANDOFF_EMITTED`, case continues to benefits segment.
2. **J2** — A benefits-only case routes directly to `BenefitsReviewer` → determination `AWARD_ISSUED`, case-owner approves → `CLOSED`.
3. **J3** — A case outside both agencies' geographic region fails the jurisdiction guardrail → `JURISDICTION_BLOCKED`; no agent invoked.
4. **J4** — A case-owner rejects the intake-to-benefits handoff → `HANDOFF_REJECTED`; `BenefitsReviewer` is never invoked.
5. **J5** — A case with no recognisable program type routes `UNROUTABLE` → workflow terminates without invoking any agent.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named cross-agency-case-handoff-mesh demonstrating the
handoff-routing × public-sector cell.
Runs out of the box (in-process simulated inbound stream; no real agency
integration).
Maven group io.akka.samples. Maven artifact
handoff-routing-public-sector-cross-agency-handoff. Java package
io.akka.samples.crossagencycasehandoffmesh. Akka 3.6.0. HTTP port 9690.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) JurisdictionRouter — classifier. System
  prompt loaded from prompts/jurisdiction-router.md. Input:
  ScopedCase{caseId, programType, geographicRegion, redactedSummary,
  redactedPayload, droppedFieldCategories: List<String>}. Output:
  RoutingDecision{segment: CaseSegment (INTAKE/BENEFITS/UNROUTABLE),
  confidence: "high"|"medium"|"low", rationale: String}.
  Defaults to UNROUTABLE under uncertainty.

- 1 AutonomousAgent IntakeAssessor — definition() with
  capability(TaskAcceptance.of(ASSESS).maxIterationsPerTask(3)). System
  prompt from prompts/intake-assessor.md. Input: ScopedCase +
  RoutingDecision. Output: SegmentOutcome{segment=INTAKE,
  determination: DeterminationType, summaryNote, agencyTag="intake",
  completedAt}. Sets determination=PENDING_DOCUMENTS when supporting
  documents are missing; sets determination=ESCALATED when outside
  intake authority.

- 1 AutonomousAgent BenefitsReviewer — definition() with
  capability(TaskAcceptance.of(REVIEW).maxIterationsPerTask(3)). System
  prompt from prompts/benefits-reviewer.md. Same input shape;
  agencyTag="benefits". Determination is one of AWARD_ISSUED,
  AWARD_DENIED, PENDING_DOCUMENTS, ESCALATED.

- 1 Agent (typed) JurisdictionGuardrail — before-agent-invocation check.
  System prompt from prompts/jurisdiction-guardrail.md. Input:
  ScopedCase + RoutingDecision. Output: JurisdictionVerdict{allowed:
  boolean, violations: List<String>, agencyCode: String,
  policyVersion: String}. Used by HandoffWorkflow before invoking any
  agency agent; blocking.

- 1 Workflow HandoffWorkflow per caseId. Steps:
    routeStep -> guardStep -> segmentStep -> approvalStep -> emitStep
    (or unroutableStep / blockedStep if guardrail denies)
  routeStep calls componentClient.forAgent().inSession(caseId)
    .method(JurisdictionRouter::route).invoke(scoped). On success emits
    CaseRouted via CaseEntity.recordRouting.
  guardStep calls forAgent(...).method(JurisdictionGuardrail::verify)
    .invoke(scoped, routing). On verdict.allowed=true proceed to
    segmentStep (emits JurisdictionChecked{allowed=true}). On
    verdict.allowed=false emit JurisdictionBlocked; terminate.
  segmentStep branches on routing.segment:
    INTAKE -> forAutonomousAgent(IntakeAssessor.class, caseId)
      .runSingleTask(TaskDef.instructions(buildPrompt(scoped,routing)))
      returning a taskId, then forTask(taskId).result(CaseTasks.ASSESS)
      to block on the typed SegmentOutcome. On success emits
      SegmentCompleted.
    BENEFITS -> same pattern with BenefitsReviewer and CaseTasks.REVIEW.
    UNROUTABLE -> emit CaseUnroutable; terminate.
  approvalStep emits HandoffPending and suspends. Resumed by operator
    calling approve-handoff or reject-handoff endpoint on CaseEntity.
    On approve emits HandoffEmitted; if no further segment emits
    CaseClosed (terminal CLOSED). On reject emits HandoffRejected
    (terminal HANDOFF_REJECTED).
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on
    routeStep and guardStep, stepTimeout(Duration.ofSeconds(60)) on
    segmentStep and approvalStep, stepTimeout(Duration.ofSeconds(20))
    on emitStep. defaultStepRecovery(maxRetries(2).failoverTo(
    HandoffWorkflow::error)).

- 2 EventSourcedEntities:
    * CaseInbox — append-only audit log. Command receive(InboundCase)
      emits InboundCaseReceived{inbound}. Idempotent on inbound.caseId.
    * CaseEntity (one per caseId) — full cross-agency lifecycle. State
      Case{caseId, inbound: InboundCase, Optional<ScopedCase> scoped,
      Optional<RoutingDecision> routing,
      Optional<JurisdictionVerdict> jurisdictionVerdict,
      Optional<SegmentOutcome> intakeOutcome,
      Optional<SegmentOutcome> benefitsOutcome,
      Optional<HandoffApproval> handoffApproval,
      Optional<HandoffRejection> handoffRejection,
      CaseStatus status, Instant createdAt, Optional<Instant> closedAt}.
      CaseStatus enum: RECEIVED, SCOPED, ROUTED, JURISDICTION_CHECK,
      IN_SEGMENT, SEGMENT_COMPLETE, HANDOFF_PENDING, HANDOFF_EMITTED,
      HANDOFF_REJECTED, JURISDICTION_BLOCKED, CLOSED, UNROUTABLE.
      Events: CaseRegistered, CaseScoped, CaseRouted,
      JurisdictionChecked, SegmentStarted, SegmentCompleted,
      HandoffPending, HandoffEmitted, HandoffRejected,
      JurisdictionBlocked, CaseClosed, CaseUnroutable.
      Commands: registerInbound, attachScoped, recordRouting,
      recordJurisdictionVerdict, startSegment, completeSegment,
      pendHandoff, emitHandoff, rejectHandoff, blockJurisdiction,
      closeCase, markUnroutable, approveHandoff, rejectHandoffOp,
      getCase.

- 1 Consumer PiiScopingFilter subscribed to CaseInbox events; for each
  InboundCaseReceived applies a per-segment field-scoping policy:
  drops constituentId (never passed cross-boundary), redacts
  fullPayload via regex pipeline (SSN patterns \d{3}-\d{2}-\d{4},
  date-of-birth heuristic, address-shaped lines, phone numbers,
  emails), and records droppedFieldCategories. Calls
  CaseEntity.registerInbound then attachScoped; then starts a
  HandoffWorkflow with caseId as the workflow id.

- 1 View CaseView with row type CaseRow (mirrors Case; uses Optional<T>
  for every nullable lifecycle field per Lesson 6). ONE query
  getAllCases SELECT * AS cases FROM case_view. No WHERE segment or
  WHERE status filter — filter client-side in callers.

- 1 TimedAction CaseSimulator — every 30s, reads next line from
  src/main/resources/sample-events/constituent-cases.jsonl (loops at
  EOF) and calls CaseInbox.receive with a fresh caseId (UUID).

- 2 HttpEndpoints:
    * CaseEndpoint at /api with GET /cases (list from
      CaseView.getAllCases, filter client-side by ?segment and ?status),
      GET /cases/{id},
      POST /cases (body InboundCase minus caseId/submittedAt — server
        assigns),
      POST /cases/{id}/approve-handoff (body {approvedBy, note} —
        resumes HandoffWorkflow approval step; transitions
        HANDOFF_PENDING → HANDOFF_EMITTED / CLOSED),
      POST /cases/{id}/reject-handoff (body {rejectedBy, reason} —
        transitions HANDOFF_PENDING → HANDOFF_REJECTED),
      GET /cases/sse (serverSentEventsForView over getAllCases), and
      three /api/metadata/{readme,risk-survey,eval-matrix} endpoints.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
      static-resources/*.

Companion files:
- CaseTasks.java declaring the task constants: ASSESS
  (resultConformsTo SegmentOutcome.class, description "Assess the
  constituent case for intake eligibility and return a typed
  SegmentOutcome") and REVIEW (resultConformsTo SegmentOutcome.class,
  description "Review the constituent case for benefits entitlement
  and return a typed SegmentOutcome").
- Domain records InboundCase, ScopedCase, RoutingDecision,
  JurisdictionVerdict, SegmentOutcome, HandoffApproval,
  HandoffRejection, and the Case entity state.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9690 and the three model-provider
  blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/constituent-cases.jsonl with
  10 canned lines (3 intake-eligible, 3 benefits-eligible, 2 mixed
  intake-then-benefits chains, 1 out-of-jurisdiction to trip the
  guardrail, 1 unrecognised program to route UNROUTABLE).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of root-level files for the classpath endpoints).
- eval-matrix.yaml at project root with 3 controls: S1 sanitizer pii,
  H1 hitl application, G1 guardrail before-agent-invocation. Matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at project root with purpose.primary_function =
  public-benefit-case-management, data.data_classes.pii = true,
  decisions.authority_level = human-approved-handoff,
  oversight.human_in_loop = true (handoff requires explicit approval),
  failure.failure_modes including "wrong-jurisdiction-routing",
  "out-of-scope-agent-invocation", "pii-cross-boundary-leak",
  "incorrect-entitlement-determination",
  "handoff-without-approval"; deployer fields
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/jurisdiction-router.md, prompts/intake-assessor.md,
  prompts/benefits-reviewer.md, prompts/jurisdiction-guardrail.md
  loaded as agent system prompts.
- README.md: title "Akka Sample: Cross-Agency Case Handoff Mesh".
- src/main/resources/static-resources/index.html — single
  self-contained file (no ui/, no npm). Five tabs. App UI tab uses a
  three-column layout (left = case list with segment badge + status
  pill + program-type chip; centre = scoped payload + routing block +
  jurisdiction verdict; right = segment outcome + approval panel or
  closure summary). Browser title exactly:
  <title>Akka Sample: Cross-Agency Case Handoff Mesh</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment
  for ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY.
  If exactly one is set, default application.conf's model-provider
  to match and proceed silently.
- If none is set, ask the user how to source the key, offering five
  options:
    (a) Mock LLM — no real key; generate a MockModelProvider that
        returns random-but-shape-correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch.
  Per-agent mock-response shapes:
    jurisdiction-router.json — 12 RoutingDecision entries: 4 INTAKE
      (food-assistance and child-services programs), 4 BENEFITS
      (housing-aid and income-support), 2 UNROUTABLE (unknown program
      codes), 2 with confidence=medium straddling intake/benefits.
    intake-assessor.json — 8 SegmentOutcome entries: 4 ELIGIBLE,
      2 PENDING_DOCUMENTS, 1 REFERRED_TO_BENEFITS, 1 ESCALATED.
    benefits-reviewer.json — 8 SegmentOutcome entries: 3 AWARD_ISSUED,
      2 AWARD_DENIED, 2 PENDING_DOCUMENTS, 1 ESCALATED.
    jurisdiction-guardrail.json — 10 JurisdictionVerdict entries: 7
      allowed=true, 3 allowed=false with violations
      ("geographic-region-out-of-scope", "program-type-not-delegated",
      "eligibility-tier-mismatch").

Constraints — see AKKA-EXEMPLAR-LESSONS.md for the full list:
- (Lesson 1) IntakeAssessor and BenefitsReviewer both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare
  definition(). Never silently downgraded to Agent.
- (Lesson 4) Workflow step timeouts overridden via settings():
  routeStep 20s, guardStep 20s, segmentStep 60s, approvalStep 60s,
  emitStep 20s.
- (Lesson 6) Every nullable lifecycle field on Case is Optional<T>.
  The CaseView row type uses the same Optional wrapping.
- (Lesson 7) CaseTasks.java declares ASSESS Task<SegmentOutcome> and
  REVIEW Task<SegmentOutcome>. Both specialists' definition().capability
  references these constants.
- (Lesson 8) Model names: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere.
- (Lesson 10) Port 9690 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column.
- (Lesson 13) Integration tier label is "Runs out of the box".
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS
  overrides and theme variables.
- (Lesson 25) API key sourcing follows the five-option protocol.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index.
- PiiScopingFilter runs INSIDE a Consumer before any LLM call.
- JurisdictionGuardrail fires BEFORE the agency agent is invoked — if
  it returns allowed=false, the agent method is never called.
- The HITL approvalStep suspends the workflow; no downstream processing
  occurs until the operator explicitly approves or rejects via the
  endpoint.
- No forbidden words in user-facing text.
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

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
