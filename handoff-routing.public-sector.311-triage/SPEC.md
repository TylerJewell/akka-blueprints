# SPEC — 311-triage

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** 311 Constituent Triage Agent.
**One-line pitch:** A triage agent classifies an inbound 311 service request and hands it off to the correct city department specialist, with PII redaction before any LLM call, a before-agent-invocation guardrail that audits the routing decision, and an inline eval scoring every route.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *which department* should own the request, then a before-agent-invocation guardrail audits the route, and only then is the request transferred to the appropriate department specialist that produces the final response. The specialist owns the full reply; the classifier does not narrate or summarise. Two governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw request event and the LLM call. The classifier, the guardrail, and the specialists never see raw constituent names, street addresses, phone numbers, or email addresses.
- A **before-agent-invocation guardrail** runs after the triage agent emits its routing decision but **before** any specialist is invoked. It checks the proposed route against a policy rubric (category confidence threshold, request-content alignment, department jurisdiction) and flags the route when the decision does not meet the bar. A flagged route sends the request to `FLAGGED_FOR_REVIEW` for a human triage reviewer instead of proceeding to the specialist.

The on-decision eval fires independently: `TriageEvalScorer` (Consumer) listens for `TriageDecided` events, calls `TriageJudge`, and writes the score back to the request. The score is metadata, not a gate.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live request list. Every request displays its category chip, status pill, triage score, and (if resolved) the published department response.
2. `RequestSimulator` (TimedAction) ticks every 30 s and inserts a new canned 311 request from `sample-events/311-requests.jsonl` into `RequestQueue`.
3. For each new request: `PiiSanitizer` (Consumer) redacts the payload, registers a `ServiceRequestEntity`, and starts a `RequestWorkflow`.
4. The workflow calls `TriageAgent`, gets a `TriageDecision { category, confidence, reason }`, and emits `TriageDecided` on the entity.
5. The workflow calls `RouteGuardrail` with the sanitized request and the triage decision. The guardrail returns `RouteVerdict { approved, flags }`.
   - If `approved=false`: workflow emits `RequestFlagged`; terminates in `FLAGGED_FOR_REVIEW`.
6. Branch on `category`:
   - `PUBLIC_WORKS` → workflow calls `PublicWorksSpecialist` with the `HANDLE` task and waits for a typed `DepartmentResponse`.
   - `PERMITS_ZONING` → workflow calls `PermitsSpecialist` with the same `HANDLE` task.
   - `UNCLEAR` → workflow emits `RequestEscalated`; terminates in `ESCALATED`.
7. The specialist's `DepartmentResponse` is published. `ResponsePublished` is emitted (terminal `RESOLVED`).
8. Independent of the workflow, `TriageEvalScorer` (Consumer) listens for `TriageDecided` events, calls `TriageJudge`, and writes `TriageScored { score, rationale }` back to the request.
9. The user can click any request card and see the redacted description, the triage decision, the route verdict, the triage score, the chosen specialist's response, and the published result.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RequestSimulator` | `TimedAction` | Drips simulated 311 requests into `RequestQueue` every 30 s. | scheduler | `RequestQueue` |
| `RequestQueue` | `EventSourcedEntity` | Append-only audit log of every inbound request (`InboundRequestReceived`). | `RequestSimulator`, `TriageEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Subscribes to `RequestQueue` events; redacts PII; registers `ServiceRequestEntity`; starts a `RequestWorkflow`. | `RequestQueue` events | `ServiceRequestEntity`, `RequestWorkflow` |
| `TriageAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedServiceRequest` into `PUBLIC_WORKS` / `PERMITS_ZONING` / `UNCLEAR` with confidence + reason. | invoked by `RequestWorkflow` | returns `TriageDecision` |
| `RouteGuardrail` | `Agent` (typed) | Before-agent-invocation guardrail: audits the proposed route against a policy rubric. Returns `RouteVerdict { approved, flags, rubricVersion }`. | invoked by `RequestWorkflow` | returns `RouteVerdict` |
| `PublicWorksSpecialist` | `AutonomousAgent` | Owns the `HANDLE` task for public-works and sanitation requests. Returns typed `DepartmentResponse`. | invoked by `RequestWorkflow` | returns `DepartmentResponse` |
| `PermitsSpecialist` | `AutonomousAgent` | Owns the `HANDLE` task for permits and zoning requests. Returns typed `DepartmentResponse`. | invoked by `RequestWorkflow` | returns `DepartmentResponse` |
| `TriageJudge` | `Agent` (typed) | Grades a triage decision against the sanitized request. Returns `TriageScore { score 1–5, rationale }`. | invoked by `TriageEvalScorer` | returns `TriageScore` |
| `RequestWorkflow` | `Workflow` | Per-request orchestration: triage → route-guardrail → branch → handle → publish. | `PiiSanitizer` (start) | `ServiceRequestEntity` |
| `ServiceRequestEntity` | `EventSourcedEntity` | Per-request lifecycle. | `RequestWorkflow`, `TriageEvalScorer` | `RequestView` |
| `RequestView` | `View` | Read-model row per request. | `ServiceRequestEntity` events | `TriageEndpoint` |
| `TriageEvalScorer` | `Consumer` | Subscribes to `ServiceRequestEntity` events; on `TriageDecided` invokes `TriageJudge` and writes `TriageScored` back. | `ServiceRequestEntity` events | `ServiceRequestEntity` |
| `TriageEndpoint` | `HttpEndpoint` | `/api/requests/*` — list, get, manual submit, manual review, SSE; `/api/metadata/*`. | — | `RequestView`, `ServiceRequestEntity`, `RequestQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record InboundServiceRequest(
    String requestId,
    String constituentId,
    String channel,           // "phone" | "web-form" | "mobile-app"
    String subject,
    String description,
    String locationHint,      // raw, may contain full address
    Instant receivedAt
) {}

record SanitizedServiceRequest(
    String redactedSubject,
    String redactedDescription,
    String redactedLocation,          // street name kept; unit/apartment redacted
    List<String> piiCategoriesFound   // e.g. ["name","phone","email","address"]
) {}

enum RequestCategory { PUBLIC_WORKS, PERMITS_ZONING, UNCLEAR }

record TriageDecision(
    RequestCategory category,
    String confidence,        // "high" | "medium" | "low"
    String reason             // one short sentence
) {}

record RouteVerdict(
    boolean approved,
    List<String> flags,       // empty when approved
    String rubricVersion      // "v1"
) {}

enum DepartmentAction {
    WORK_ORDER_CREATED,
    PERMIT_INFO_PROVIDED,
    INSPECTION_SCHEDULED,
    REFERRAL_ISSUED,
    INFO_PROVIDED,
    ESCALATED
}

record DepartmentResponse(
    String responseSubject,
    String responseBody,
    DepartmentAction action,
    String departmentTag,     // "public-works" | "permits-zoning"
    Instant respondedAt
) {}

record TriageScore(
    int score,                // 1..5
    String rationale,
    Instant scoredAt
) {}

record ServiceRequest(
    String requestId,
    InboundServiceRequest inbound,
    Optional<SanitizedServiceRequest> sanitized,
    Optional<TriageDecision> triage,
    Optional<RouteVerdict> routeVerdict,
    Optional<DepartmentResponse> response,
    Optional<TriageScore> triageScore,
    Optional<String> escalationReason,
    RequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus {
    RECEIVED,
    SANITIZED,
    TRIAGED,
    ROUTE_APPROVED,
    ROUTED_PUBLIC_WORKS,
    ROUTED_PERMITS_ZONING,
    RESPONSE_DRAFTED,
    FLAGGED_FOR_REVIEW,
    RESOLVED,
    ESCALATED
}
```

Events on `ServiceRequestEntity`: `ServiceRequestRegistered`, `ServiceRequestSanitized`, `TriageDecided`, `RouteVerdictAttached`, `ServiceRequestRouted`, `ResponseDrafted`, `ResponsePublished`, `RequestFlagged`, `RequestEscalated`, `TriageScored`.

Events on `RequestQueue`: `InboundRequestReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/requests` — list all requests (newest-first), optional `?category=PUBLIC_WORKS|PERMITS_ZONING|UNCLEAR&status=…` filtered client-side.
- `GET /api/requests/{id}` — one request.
- `POST /api/requests` — manually submit a 311 request (body `InboundServiceRequest` minus `requestId` and `receivedAt`); server assigns both.
- `POST /api/requests/{id}/review` — body `{ reviewedBy, decision, note }` — triage reviewer resolves a `FLAGGED_FOR_REVIEW` request: `decision` is `"approve"` (proceed to specialist) or `"escalate"` (terminal `ESCALATED`).
- `GET /api/requests/sse` — Server-Sent Events for every request change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: 311 Constituent Triage Agent</title>`.

The App UI tab is a three-pane layout: **left** is the request list (status pill + category chip + score chip), **centre** is the selected request's redacted description + triage decision + route verdict + score, **right** is the chosen specialist's response or (when `FLAGGED_FOR_REVIEW`) a flag detail block with a Review button.

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts constituent names, phone numbers, email addresses, and specific street addresses from the request before any LLM sees it. PII categories found are kept for audit; the redacted text is what reaches the agents.
- **G1 — before-agent-invocation guardrail** on the routing step in `RequestWorkflow`: `RouteGuardrail` (typed Agent) checks every proposed route against a rubric (minimum confidence threshold, request-content alignment, department jurisdiction boundary). Blocking — a flagged route puts the request in `FLAGGED_FOR_REVIEW` and the specialist is never invoked. A human triage reviewer resolves the flag via `POST /api/requests/{id}/review`.

The eval is wired separately:

- **E1 — on-decision eval** (`eval-event`, on the triage decision): `TriageEvalScorer` (Consumer) listens for `TriageDecided` events and calls `TriageJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate.

## 9. Agent prompts

- `TriageAgent` → `prompts/triage-agent.md`. Typed classifier; returns one of `PUBLIC_WORKS`, `PERMITS_ZONING`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `RouteGuardrail` → `prompts/route-guardrail.md`. Before-agent-invocation rubric check; returns `RouteVerdict { approved, flags }`. Conservative — borderline routes are flagged.
- `PublicWorksSpecialist` → `prompts/public-works-specialist.md`. Owns the `HANDLE` task for public-works requests. Issues work orders; never promises a specific completion date outside the published SLA.
- `PermitsSpecialist` → `prompts/permits-specialist.md`. Owns the `HANDLE` task for permits and zoning requests. Provides accurate code citations; never invents a permit number.
- `TriageJudge` → `prompts/triage-judge.md`. Grades a triage decision against a 1–5 rubric with one-sentence rationale.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a public-works request → triaged `PUBLIC_WORKS` → route guardrail approves → resolved by `PublicWorksSpecialist` → published.
2. **J2** — Simulator drips a permits request → triaged `PERMITS_ZONING` → route guardrail approves → resolved by `PermitsSpecialist` → published.
3. **J3** — An ambiguous request triages as `UNCLEAR` and lands in `ESCALATED` without any specialist invocation.
4. **J4** — A request whose route the guardrail cannot confidently approve is flagged; the request lands in `FLAGGED_FOR_REVIEW`; no specialist is invoked; a reviewer can approve or escalate via `POST /api/requests/{id}/review`.
5. **J5** — Every triaged request carries a `TriageScore` (1–5) and rationale within ~10 s of the triage decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named 311-triage demonstrating the handoff-routing × public-sector cell.
Runs out of the box (in-process simulated inbound stream; no real 311 feed integration).
Maven group io.akka.samples. Maven artifact handoff-routing-public-sector-311-triage.
Java package io.akka.samples.constituenttriage. Akka 3.6.0. HTTP port 9403.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) TriageAgent — classifier. System prompt loaded from
  prompts/triage-agent.md. Input: SanitizedServiceRequest{redactedSubject, redactedDescription,
  redactedLocation, piiCategoriesFound: List<String>}. Output: TriageDecision{category:
  RequestCategory (PUBLIC_WORKS/PERMITS_ZONING/UNCLEAR), confidence: "high"|"medium"|"low",
  reason: String}. Defaults to UNCLEAR under uncertainty.

- 1 Agent (typed) RouteGuardrail — before-agent-invocation guardrail. System prompt from
  prompts/route-guardrail.md. Input: SanitizedServiceRequest + TriageDecision. Output:
  RouteVerdict{approved: boolean, flags: List<String>, rubricVersion: String = "v1"}.
  Used by RequestWorkflow.routeGuardrailStep BEFORE any specialist is invoked. Blocking —
  approved=false sends the request to FLAGGED_FOR_REVIEW.

- 1 AutonomousAgent PublicWorksSpecialist — definition() with capability(TaskAcceptance
  .of(HANDLE).maxIterationsPerTask(3)). System prompt from prompts/public-works-specialist.md.
  Input: SanitizedServiceRequest + TriageDecision. Output: DepartmentResponse{responseSubject,
  responseBody, action: DepartmentAction, departmentTag = "public-works", respondedAt}.
  Never promises a specific completion date; uses WORK_ORDER_CREATED or INFO_PROVIDED;
  sets action=ESCALATED when outside department scope.

- 1 AutonomousAgent PermitsSpecialist — definition() with capability(TaskAcceptance
  .of(HANDLE).maxIterationsPerTask(3)). System prompt from prompts/permits-specialist.md.
  Same input shape; departmentTag = "permits-zoning". Uses PERMIT_INFO_PROVIDED,
  INSPECTION_SCHEDULED, INFO_PROVIDED; never invents a permit number or application id.

- 1 Agent (typed) TriageJudge — judge. System prompt from prompts/triage-judge.md. Input:
  SanitizedServiceRequest + TriageDecision. Output: TriageScore{score: int 1–5, rationale:
  String, scoredAt: Instant}.

- 1 Workflow RequestWorkflow per requestId. Steps:
    triageStep -> routeGuardrailStep -> routeStep ->
      {publicWorksStep | permitsStep | escalateStep} -> publishStep
  triageStep calls componentClient.forAgent().inSession(requestId).method(TriageAgent::triage)
    .invoke(sanitized). On success emits TriageDecided via ServiceRequestEntity.recordTriage.
  routeGuardrailStep calls componentClient.forAgent().inSession(requestId)
    .method(RouteGuardrail::check).invoke(sanitized, decision). On verdict.approved=false:
    emit RequestFlagged (terminal FLAGGED_FOR_REVIEW). On approved=true: emit
    RouteVerdictAttached and proceed to routeStep.
  routeStep branches on TriageDecision.category:
    PUBLIC_WORKS -> proceed to publicWorksStep (emits ServiceRequestRouted{PUBLIC_WORKS})
    PERMITS_ZONING -> proceed to permitsStep (emits ServiceRequestRouted{PERMITS_ZONING})
    UNCLEAR -> escalateStep (emits RequestEscalated; terminates).
  publicWorksStep / permitsStep call forAutonomousAgent(<Specialist>.class, requestId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, decision))) returning a taskId,
    then forTask(taskId).result(ServiceTasks.HANDLE) to block on the typed DepartmentResponse.
    On success emits ResponseDrafted.
  publishStep emits ResponsePublished (terminal RESOLVED).
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on triageStep and
    routeGuardrailStep, stepTimeout(Duration.ofSeconds(60)) on publicWorksStep, permitsStep,
    and publishStep. defaultStepRecovery(maxRetries(2).failoverTo(RequestWorkflow::error)).

- 2 EventSourcedEntities:
    * RequestQueue — append-only audit log. Command receive(InboundServiceRequest) emits
      InboundRequestReceived{inbound}. Idempotent on inbound.requestId.
    * ServiceRequestEntity (one per requestId) — full per-request lifecycle. State
      ServiceRequest{requestId, inbound: InboundServiceRequest, Optional<SanitizedServiceRequest>
      sanitized, Optional<TriageDecision> triage, Optional<RouteVerdict> routeVerdict,
      Optional<DepartmentResponse> response, Optional<TriageScore> triageScore,
      Optional<String> escalationReason, RequestStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. RequestStatus enum: RECEIVED, SANITIZED, TRIAGED,
      ROUTE_APPROVED, ROUTED_PUBLIC_WORKS, ROUTED_PERMITS_ZONING, RESPONSE_DRAFTED,
      FLAGGED_FOR_REVIEW, RESOLVED, ESCALATED. Events: ServiceRequestRegistered,
      ServiceRequestSanitized, TriageDecided, RouteVerdictAttached, ServiceRequestRouted,
      ResponseDrafted, ResponsePublished, RequestFlagged, RequestEscalated, TriageScored.
      Commands: registerInbound, attachSanitized, recordTriage, recordRouteVerdict,
      recordRouting, recordDraft, publish, flag, escalate, recordTriageScore, review,
      getRequest. emptyState() returns ServiceRequest.initial("") with no commandContext()
      reference.

- 2 Consumers:
    * PiiSanitizer subscribed to RequestQueue events; for each InboundRequestReceived
      applies a regex+heuristic redaction pipeline (emails, phone numbers, names in
      greeting/salutation positions, street-level address patterns matching \d+ [A-Z][a-z]+
      [St|Ave|Blvd|Rd|Dr|Ln|Way|Ct]+) to subject + description + locationHint; keeps the
      street-name segment but redacts unit/apartment numbers; builds SanitizedServiceRequest
      with piiCategoriesFound; calls ServiceRequestEntity.registerInbound then attachSanitized;
      then starts a RequestWorkflow with requestId as the workflow id.
    * TriageEvalScorer subscribed to ServiceRequestEntity events; on TriageDecided invokes
      TriageJudge.score(sanitized, decision) and calls ServiceRequestEntity.recordTriageScore(
      requestId, score). On any other event type, no-op. Use componentClient — do NOT call
      the agent from a TimedAction.

- 1 View RequestView with row type RequestRow (mirrors ServiceRequest; uses Optional<T> for
  every nullable lifecycle field per Lesson 6). Table updater consumes ServiceRequestEntity
  events. ONE query getAllRequests SELECT * AS requests FROM request_view. No WHERE category
  or WHERE status filter — filter client-side in callers.

- 1 TimedAction RequestSimulator — every 30s, reads next line from
  src/main/resources/sample-events/311-requests.jsonl (loops at EOF) and calls
  RequestQueue.receive with a fresh requestId (UUID).

- 2 HttpEndpoints:
    * TriageEndpoint at /api with GET /requests (list from RequestView.getAllRequests,
      filter client-side by ?category and ?status query params), GET /requests/{id},
      POST /requests (body InboundServiceRequest minus requestId/receivedAt — server assigns),
      POST /requests/{id}/review (body {reviewedBy, decision, note} — triage reviewer
      resolves FLAGGED_FOR_REVIEW: decision="approve" re-enters routeStep; decision="escalate"
      terminates ESCALATED with note as escalationReason), GET /requests/sse
      (serverSentEventsForView over getAllRequests), and three /api/metadata/{readme,
      risk-survey,eval-matrix} endpoints serving the YAML/MD files from
      src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ServiceTasks.java declaring the task constants: HANDLE (resultConformsTo
  DepartmentResponse.class, description "Handle the 311 service request end-to-end and
  return a typed DepartmentResponse").
- Domain records InboundServiceRequest, SanitizedServiceRequest, TriageDecision,
  RouteVerdict, DepartmentResponse, TriageScore, and the ServiceRequest entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9403 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/311-requests.jsonl with 9 canned lines (3
  PUBLIC_WORKS-flavoured: pothole, broken street light, abandoned vehicle; 3
  PERMITS_ZONING-flavoured: deck permit, zoning question, home addition; 2
  UNCLEAR-flavoured: too short, mixed category; 1 designed to trip the route guardrail
  with low-confidence triage on a borderline PUBLIC_WORKS/PERMITS_ZONING description).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, G1 guardrail
  before-agent-invocation route-rubric. Plus E1 eval-event on-decision-eval. Matching
  simplified_view list. No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = constituent-services,
  purpose.sector = public-sector, data.data_classes.pii = true,
  decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false (the system publishes without HITL by default — only
  flagged routes wait for a human reviewer), data.pii_handled_by_sanitizer_before_llm = true,
  failure.failure_modes including "wrong-department-routing", "incorrect-permit-guidance",
  "pii-leakage-via-llm", "invented-permit-number", "out-of-jurisdiction-advice";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/triage-agent.md, prompts/route-guardrail.md, prompts/public-works-specialist.md,
  prompts/permits-specialist.md, prompts/triage-judge.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: 311 Constituent Triage Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = request list with category chip + status pill + score chip; centre = redacted
  description + triage block + route verdict; right = specialist response or flag detail
  + Review button). Browser title exactly:
  <title>Akka Sample: 311 Constituent Triage Agent</title>. No subtitle on the Overview tab.

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
  pseudo-randomly per call (seeded by requestId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    triage-agent.json — 12 TriageDecision entries spanning PUBLIC_WORKS
      (pothole complaints, broken street lights, abandoned vehicles),
      PERMITS_ZONING (deck permits, addition projects, zoning inquiries),
      and UNCLEAR (too short, mixed content, off-topic). Confidence + a
      one-sentence reason on each.
    public-works-specialist.json — 8 DepartmentResponse entries: 5 with
      action WORK_ORDER_CREATED or INFO_PROVIDED (well within department
      scope), 1 with action ESCALATED (outside maintenance jurisdiction),
      1 with action REFERRAL_ISSUED, 1 designed to trip the route guardrail
      with a specific SLA promise not in policy.
    permits-specialist.json — 8 DepartmentResponse entries: 5 with
      PERMIT_INFO_PROVIDED or INSPECTION_SCHEDULED, 1 with INFO_PROVIDED,
      1 with ESCALATED (state-level jurisdiction), 1 designed to trip the
      route guardrail (invents a permit application number).
    triage-judge.json — 10 TriageScore entries, score 1–5, one-sentence
      rationale matching the rubric (category-correctness / confidence-
      calibration / reason-quality).
    route-guardrail.json — 10 RouteVerdict entries. 7 with approved=true
      and empty flags. 3 with approved=false and one flag each
      ("low-confidence-route", "content-category-mismatch",
      "out-of-jurisdiction"). The mock should match the rubric-tripping
      entries above when called for the same requestId.
- A MockModelProvider.seedFor(requestId) helper makes per-request selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  PublicWorksSpecialist and PermitsSpecialist both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): triageStep 20s,
  routeGuardrailStep 20s, publicWorksStep / permitsStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on ServiceRequest is Optional<T>. The
  RequestView row type uses the same Optional wrapping.
- (Lesson 7) ServiceTasks.java declares the HANDLE Task<DepartmentResponse> constant.
  Both specialists' definition().capability(TaskAcceptance.of(HANDLE)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9403 in application.conf; not 9000.
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
- The PiiSanitizer runs INSIDE a Consumer before any LLM call — not inside
  an Agent's prompt and not after the LLM has seen the raw payload.
- The RouteGuardrail step happens BEFORE any specialist is invoked — it
  audits the route, not the response.
- The TriageEvalScorer Consumer reacts to TriageDecided events and calls
  TriageJudge via componentClient.forAgent(). It does NOT modify the workflow
  flow — the eval is out-of-band metadata.
- No forbidden words in user-facing text: shape, simplest, simpler, complex,
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
