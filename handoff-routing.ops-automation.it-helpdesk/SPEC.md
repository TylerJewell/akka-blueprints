# SPEC — it-helpdesk

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** IT Help Desk.
**One-line pitch:** A classifier agent reads an incoming IT request and hands the resolution off to an access, infrastructure, or software specialist that owns it end-to-end, with credential redaction before any LLM call, a before-tool-call guardrail on ticket creation, and an inline eval scoring every routing decision.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides which specialist domain owns the task, then transfers the same task identity to a downstream specialist that produces the final resolution. The downstream specialist owns the entire ticket response; the classifier does not narrate or summarize. Two governance mechanisms are layered on top:

- A **secret sanitizer** runs inside a Consumer between the raw request event and the LLM call. Users frequently paste API keys, passwords, tokens, and certificate PEM blocks into IT requests. The sanitizer strips these before any agent sees the payload.
- A **before-tool-call guardrail** fires when a specialist proposes filing an IT ticket (a write action). It checks the proposed ticket against an IT policy rubric — no self-service permission grants above tier 1, no tickets that authorize open-ended admin access, no duplicate ticket creation for an already-open incident — and blocks the write when it violates.

An **on-decision eval** fires every time the classifier emits a routing decision. A `RoutingJudge` agent grades the decision against the sanitized payload on a 1–5 rubric. The score and rationale are written back to the request entity and surfaced in the UI.

The pattern is a fan-out-of-one: the workflow branches on the classifier's category, and only the matched specialist is invoked. The other specialists see no traffic for that request.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live request list. Every request displays its category chip, status pill, routing score, and (if resolved) the filed ticket summary.
2. `RequestSimulator` (TimedAction) ticks every 30 s and inserts a new canned IT request from `sample-events/it-requests.jsonl` into `RequestQueue`.
3. For each new request: `SecretSanitizer` (Consumer) redacts the payload, registers a `RequestEntity`, and starts a `HelpdeskWorkflow`.
4. The workflow calls `ClassifierAgent`, gets a `ClassificationDecision { category, confidence, reason }`, and emits `RequestClassified` on the entity.
5. Branch on `category`:
   - `ACCESS` → workflow calls `AccessSpecialist` with the `RESOLVE` task and waits for the typed `Resolution` result.
   - `INFRASTRUCTURE` → workflow calls `InfraSpecialist` with the same `RESOLVE` task.
   - `SOFTWARE` → workflow calls `SoftwareSpecialist` with the same `RESOLVE` task.
   - `UNCLEAR` → workflow emits `RequestEscalated`; ends.
6. The specialist proposes a `Resolution` that includes an optional `ProposedTicket`. If a `ProposedTicket` is present, the before-tool-call guardrail inspects it. On `allowed=true`, the workflow files the ticket and emits `TicketFiled`. On `allowed=false`, the workflow emits `TicketBlocked` and the request transitions to `BLOCKED` with the violation list.
7. After the ticket phase, the workflow emits `ResolutionPublished` (terminal `RESOLVED`).
8. Independent of the workflow, `RoutingEvalScorer` (Consumer) listens for `RequestClassified` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the entity.
9. The user can click any request card and see the redacted body, the classification reason, the routing score, the chosen specialist, the filed ticket (or blocked ticket + violations), and the published response.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RequestSimulator` | `TimedAction` | Drips simulated IT requests into `RequestQueue` every 30 s. | scheduler | `RequestQueue` |
| `RequestQueue` | `EventSourcedEntity` | Append-only audit log of every inbound request (`InboundRequestReceived`). | `RequestSimulator`, `HelpdeskEndpoint` | `SecretSanitizer` |
| `SecretSanitizer` | `Consumer` | Subscribes to `RequestQueue` events; strips credentials; registers `RequestEntity`; starts a `HelpdeskWorkflow`. | `RequestQueue` events | `RequestEntity`, `HelpdeskWorkflow` |
| `ClassifierAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedRequest` into `ACCESS` / `INFRASTRUCTURE` / `SOFTWARE` / `UNCLEAR` with confidence + reason. | invoked by `HelpdeskWorkflow` | returns `ClassificationDecision` |
| `AccessSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for access and identity requests. Returns typed `Resolution`. | invoked by `HelpdeskWorkflow` | returns `Resolution` |
| `InfraSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for infrastructure requests. Returns typed `Resolution`. | invoked by `HelpdeskWorkflow` | returns `Resolution` |
| `SoftwareSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for software requests. Returns typed `Resolution`. | invoked by `HelpdeskWorkflow` | returns `Resolution` |
| `RoutingJudge` | `Agent` (typed) | Grades a classification decision against the sanitized payload. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RoutingEvalScorer` | returns `RoutingScore` |
| `TicketGuardrail` | `Agent` (typed) | Before-tool-call guardrail: checks a proposed ticket write against the IT policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `HelpdeskWorkflow` | returns `GuardrailVerdict` |
| `HelpdeskWorkflow` | `Workflow` | Per-request orchestration: classify → branch → resolve → guardrail → file ticket → publish. | `SecretSanitizer` (start) | `RequestEntity` |
| `RequestEntity` | `EventSourcedEntity` | Per-request lifecycle. | `HelpdeskWorkflow`, `RoutingEvalScorer` | `RequestView` |
| `RequestView` | `View` | Read-model row per request. | `RequestEntity` events | `HelpdeskEndpoint` |
| `RoutingEvalScorer` | `Consumer` | Subscribes to `RequestEntity` events; on `RequestClassified` invokes `RoutingJudge` and writes `RoutingScored` back. | `RequestEntity` events | `RequestEntity` |
| `HelpdeskEndpoint` | `HttpEndpoint` | `/api/requests/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `RequestView`, `RequestEntity`, `RequestQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI. | — | static resources |

## 5. Data model

```java
record IncomingRequest(
    String requestId,
    String submitterId,
    String channel,            // "email" | "chat" | "portal"
    String subject,
    String body,
    Instant receivedAt
) {}

record SanitizedRequest(
    String redactedSubject,
    String redactedBody,
    List<String> secretCategoriesFound  // e.g. "api-key", "password", "token", "pem-block"
) {}

enum RequestCategory { ACCESS, INFRASTRUCTURE, SOFTWARE, UNCLEAR }

record ClassificationDecision(
    RequestCategory category,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

record ProposedTicket(
    String title,
    String assigneeGroup,      // "access-team" | "infra-team" | "software-team"
    String priority,           // "P1" | "P2" | "P3"
    String body
) {}

enum ResolutionAction { TICKET_FILED, SELF_SERVICE_RESOLVED, RUNBOOK_LINKED, FOLLOW_UP_SCHEDULED, ESCALATED }

record Resolution(
    String responseSubject,
    String responseBody,
    ResolutionAction action,
    Optional<ProposedTicket> proposedTicket,
    String specialistTag,      // "access" | "infra" | "software"
    Instant resolvedAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,   // empty when allowed
    String rubricVersion
) {}

record RoutingScore(
    int score,                 // 1..5
    String rationale,
    Instant scoredAt
) {}

record Request(
    String requestId,
    IncomingRequest incoming,
    Optional<SanitizedRequest> sanitized,
    Optional<ClassificationDecision> classification,
    Optional<Resolution> resolution,
    Optional<GuardrailVerdict> guardrail,
    Optional<RoutingScore> routingScore,
    Optional<String> escalationReason,
    RequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RequestStatus {
    RECEIVED,
    SANITIZED,
    CLASSIFIED,
    ROUTED_ACCESS,
    ROUTED_INFRASTRUCTURE,
    ROUTED_SOFTWARE,
    RESOLUTION_DRAFTED,
    TICKET_BLOCKED,
    RESOLVED,
    ESCALATED
}
```

Events on `RequestEntity`: `RequestRegistered`, `RequestSanitized`, `RequestClassified`, `RequestRouted`, `ResolutionDrafted`, `GuardrailVerdictAttached`, `TicketFiled`, `TicketBlocked`, `ResolutionPublished`, `RequestEscalated`, `RoutingScored`.

Events on `RequestQueue`: `InboundRequestReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/requests` — list all requests (newest-first), optional `?category=ACCESS|INFRASTRUCTURE|SOFTWARE|UNCLEAR&status=…` filtered client-side.
- `GET /api/requests/{id}` — one request.
- `POST /api/requests` — manually submit a request (body `IncomingRequest` minus `requestId` and `receivedAt`); server assigns both.
- `POST /api/requests/{id}/unblock` — body `{ decidedBy, note }` — technician override; transitions `TICKET_BLOCKED` to `RESOLVED` if the technician chooses to file the blocked ticket.
- `GET /api/requests/sse` — Server-Sent Events for every request change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: IT Help Desk</title>`.

The App UI tab is a three-pane layout: **left** is the request list (status pill + category chip + score chip), **centre** is the selected request's redacted body + classification decision + score, **right** is the chosen specialist's resolution + guardrail verdict + filed ticket summary (or violations + Unblock button when `TICKET_BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** on `AccessSpecialist`, `InfraSpecialist`, and `SoftwareSpecialist`: checks every proposed `ProposedTicket` against an IT policy rubric (no self-service permission grants above tier 1, no open-ended admin access requests, no duplicate creation for an existing open incident, no tickets assigning priority P1 without explicit approver). Blocking — a violation puts the request in `TICKET_BLOCKED` for technician review.
- **S1 — secret sanitizer** (`secret`, applied in `SecretSanitizer` Consumer): strips API keys (patterns like `sk-`, `ghp_`, `AKIA`), passwords, bearer tokens, and PEM certificate blocks from the request body and subject before any LLM sees them. The secret categories found are kept for audit; the redacted text is what reaches the agents.

## 9. Agent prompts

- `ClassifierAgent` → `prompts/classifier-agent.md`. Typed classifier; returns one of `ACCESS`, `INFRASTRUCTURE`, `SOFTWARE`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `AccessSpecialist` → `prompts/access-specialist.md`. Owns the `RESOLVE` task for access and identity requests. Never grants permissions above tier 1 self-service; routes above-authority requests to `ESCALATED`.
- `InfraSpecialist` → `prompts/infra-specialist.md`. Owns the `RESOLVE` task for infrastructure requests. References a runbook id when one applies; never invents one.
- `SoftwareSpecialist` → `prompts/software-specialist.md`. Owns the `RESOLVE` task for software requests. Never invents download links or license keys.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a classification decision against a 1–5 rubric with one-sentence rationale.
- `TicketGuardrail` → `prompts/ticket-guardrail.md`. Returns a `GuardrailVerdict { allowed, violations }`. Conservative — borderline proposed tickets are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an access-flavoured request → classified `ACCESS` → resolved by `AccessSpecialist` → guardrail passes → ticket filed → published.
2. **J2** — Simulator drips an infrastructure request → classified `INFRASTRUCTURE` → resolved by `InfraSpecialist` → ticket filed → published.
3. **J3** — Simulator drips a software request → classified `SOFTWARE` → resolved by `SoftwareSpecialist` → ticket filed → published.
4. **J4** — An ambiguous request classifies as `UNCLEAR` and lands in `ESCALATED` without any specialist invocation.
5. **J5** — A request body containing a password string has `[REDACTED-SECRET]` substituted; `secretCategoriesFound` lists `"password"`.
6. **J6** — A specialist draft that proposes an open-ended admin ticket (outside policy) is blocked; the request lands in `TICKET_BLOCKED` with the violation listed; the technician can unblock or leave it.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named it-helpdesk demonstrating the handoff-routing × ops-automation cell.
Runs out of the box (in-process simulated inbound stream; no real ticketing integration).
Maven group io.akka.samples. Maven artifact handoff-routing-ops-automation-it-helpdesk. Java
package io.akka.samples.ithelpdesk. Akka 3.6.0. HTTP port 9335.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ClassifierAgent — classifier. System prompt loaded from
  prompts/classifier-agent.md. Input: SanitizedRequest{redactedSubject, redactedBody,
  secretCategoriesFound: List<String>}. Output: ClassificationDecision{category:
  RequestCategory (ACCESS/INFRASTRUCTURE/SOFTWARE/UNCLEAR), confidence:
  "high"|"medium"|"low", reason: String}. Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent AccessSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/access-specialist.md. Input:
  SanitizedRequest + ClassificationDecision. Output: Resolution{responseSubject,
  responseBody, action: ResolutionAction, proposedTicket: Optional<ProposedTicket>,
  specialistTag = "access", resolvedAt}. Never grants permissions above tier 1 self-service;
  sets action=ESCALATED when above authority.

- 1 AutonomousAgent InfraSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/infra-specialist.md. Same input
  shape; specialistTag = "infra".

- 1 AutonomousAgent SoftwareSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/software-specialist.md. Same input
  shape; specialistTag = "software".

- 1 Agent (typed) RoutingJudge — judge. System prompt from prompts/routing-judge.md. Input:
  SanitizedRequest + ClassificationDecision. Output: RoutingScore{score: int 1–5,
  rationale: String, scoredAt: Instant}.

- 1 Agent (typed) TicketGuardrail — typed rubric check. System prompt from
  prompts/ticket-guardrail.md. Input: SanitizedRequest + ProposedTicket. Output:
  GuardrailVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by HelpdeskWorkflow before filing; blocking. Only invoked when resolution
  contains a non-empty proposedTicket.

- 1 Workflow HelpdeskWorkflow per requestId. Steps:
    classifyStep -> routeStep ->
        {accessStep | infraStep | softwareStep | escalateStep}
        -> guardrailStep -> fileStep -> publishStep
  classifyStep calls componentClient.forAgent().inSession(requestId)
    .method(ClassifierAgent::classify).invoke(sanitized). On success emits
    RequestClassified via RequestEntity.recordClassification.
  routeStep branches on ClassificationDecision.category:
    ACCESS -> proceed to accessStep (emits RequestRouted{ACCESS})
    INFRASTRUCTURE -> proceed to infraStep (emits RequestRouted{INFRASTRUCTURE})
    SOFTWARE -> proceed to softwareStep (emits RequestRouted{SOFTWARE})
    UNCLEAR -> escalateStep (emits RequestEscalated; terminates).
  accessStep / infraStep / softwareStep call
    forAutonomousAgent(<Specialist>.class, requestId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, classification)))
    returning a taskId, then forTask(taskId).result(HelpdeskTasks.RESOLVE) to block on
    the typed Resolution. On success emits ResolutionDrafted.
  guardrailStep: if resolution.proposedTicket is empty, skip to fileStep directly.
    Otherwise calls forAgent(...).method(TicketGuardrail::check)
    .invoke(sanitized, proposedTicket). On verdict.allowed=true proceed to fileStep
    (emits GuardrailVerdictAttached). On verdict.allowed=false emit TicketBlocked
    (terminal TICKET_BLOCKED) and end.
  fileStep emits TicketFiled with the proposedTicket (or no-op if absent).
  publishStep emits ResolutionPublished (terminal RESOLVED).
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on classifyStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on accessStep, infraStep,
    softwareStep, fileStep, and publishStep. defaultStepRecovery(
    maxRetries(2).failoverTo(HelpdeskWorkflow::error)).

- 2 EventSourcedEntities:
    * RequestQueue — append-only audit log. Command receive(IncomingRequest) emits
      InboundRequestReceived{incoming}. Idempotent on incoming.requestId.
    * RequestEntity (one per requestId) — full per-request lifecycle. State
      Request{requestId, incoming: IncomingRequest, Optional<SanitizedRequest> sanitized,
      Optional<ClassificationDecision> classification, Optional<Resolution> resolution,
      Optional<GuardrailVerdict> guardrail, Optional<RoutingScore> routingScore,
      Optional<String> escalationReason, RequestStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. RequestStatus enum: RECEIVED, SANITIZED,
      CLASSIFIED, ROUTED_ACCESS, ROUTED_INFRASTRUCTURE, ROUTED_SOFTWARE,
      RESOLUTION_DRAFTED, TICKET_BLOCKED, RESOLVED, ESCALATED.
      Events: RequestRegistered, RequestSanitized, RequestClassified, RequestRouted,
      ResolutionDrafted, GuardrailVerdictAttached, TicketFiled, TicketBlocked,
      ResolutionPublished, RequestEscalated, RoutingScored.
      Commands: registerIncoming, attachSanitized, recordClassification, recordRouting,
      recordDraft, recordGuardrailVerdict, fileTicket, blockTicket, publish, escalate,
      recordRoutingScore, unblock, getRequest.

- 2 Consumers:
    * SecretSanitizer subscribed to RequestQueue events; for each InboundRequestReceived
      applies a regex pipeline: API key patterns (sk-[A-Za-z0-9]{20,}, ghp_[A-Za-z0-9]{36},
      AKIA[A-Z0-9]{16}), generic bearer token patterns (Bearer [A-Za-z0-9._-]{20,}),
      password-field patterns (password[:=]\s*\S+), PEM blocks (-----BEGIN [A-Z ]+-----
      through -----END [A-Z ]+-----), plus heuristic detection for hex strings > 32 chars
      and base64 blobs > 40 chars likely to be secrets. Substitutes each match with
      [REDACTED-SECRET]. Builds SanitizedRequest with secretCategoriesFound. Then calls
      RequestEntity.registerIncoming, attachSanitized, and starts a HelpdeskWorkflow.
    * RoutingEvalScorer subscribed to RequestEntity events; on RequestClassified invokes
      RoutingJudge.score(sanitized, decision) and calls
      RequestEntity.recordRoutingScore(requestId, score). No-op on other events.
      Use componentClient.

- 1 View RequestView with row type RequestRow (mirrors Request; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). ONE query getAllRequests SELECT * AS requests
  FROM request_view. No WHERE filters — filter client-side.

- 1 TimedAction RequestSimulator — every 30s, reads next line from
  src/main/resources/sample-events/it-requests.jsonl (loops at EOF) and calls
  RequestQueue.receive with a fresh requestId (UUID).

- 2 HttpEndpoints:
    * HelpdeskEndpoint at /api with GET /requests (list from RequestView.getAllRequests,
      filter client-side by ?category and ?status query params), GET /requests/{id},
      POST /requests (body IncomingRequest minus requestId/receivedAt — server assigns),
      POST /requests/{id}/unblock (body {decidedBy, note} — technician override: files
      the blocked proposed ticket as RESOLVED with an audit note),
      GET /requests/sse (serverSentEventsForView over getAllRequests), and
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- HelpdeskTasks.java declaring the task constants: RESOLVE (resultConformsTo Resolution.class,
  description "Resolve the IT request end-to-end and return a typed Resolution").
- Domain records IncomingRequest, SanitizedRequest, ClassificationDecision, ProposedTicket,
  Resolution, GuardrailVerdict, RoutingScore, and the Request entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9335 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/it-requests.jsonl with 10 canned lines (3 ACCESS-
  flavoured including one with a password in the body, 3 INFRASTRUCTURE-flavoured, 2
  SOFTWARE-flavoured, 1 UNCLEAR, 1 designed to trip the guardrail with an open-ended
  admin-access ticket proposal).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.
- eval-matrix.yaml at the project root with 2 controls: G1 guardrail before-tool-call,
  S1 sanitizer secret. Matching simplified_view list.
- risk-survey.yaml at the project root with purpose.primary_function = it-helpdesk,
  data.data_classes.credentials = true, decisions.authority_level = draft-only,
  oversight.human_in_loop = false, data.secrets_handled_by_sanitizer_before_llm = true,
  failure.failure_modes including "wrong-domain-routing", "credential-leakage-via-llm",
  "unauthorized-permission-grant", "open-ended-admin-ticket-creation".
- prompts/ files loaded as agent system prompts.
- README.md: title "Akka Sample: IT Help Desk".
- src/main/resources/static-resources/index.html — single self-contained file. Five tabs.
  App UI: three-column (left = request list with category chip + status pill + score chip;
  centre = redacted request + classification block; right = specialist draft + guardrail
  verdict + filed ticket or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: IT Help Desk</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects ANTHROPIC_API_KEY / OPENAI_API_KEY /
  GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default application.conf's
  model-provider to match and proceed silently.
- If none is set, ask the user:
    (a) Mock LLM — no real key; MockModelProvider with per-agent mock-response files.
    (b) Existing env var — record the NAME in application.conf.
    (c) Env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session.
- NEVER write the key value to any file.

Mock LLM provider — required when option (a) is selected:
- MockModelProvider.java with per-agent dispatch on agent class.
- Per-agent mock-response shapes:
    classifier-agent.json — 12 ClassificationDecision entries spanning ACCESS (password
      reset, account lockout, permission request), INFRASTRUCTURE (VPN down, server
      unreachable, network latency), SOFTWARE (install failure, config error, application
      crash), and UNCLEAR (very short, off-topic, ambiguous). Confidence + reason.
    access-specialist.json — 8 Resolution entries: 4 SELF_SERVICE_RESOLVED (password
      resets, MFA setup), 2 TICKET_FILED (permission grants within authority), 1
      ESCALATED (above tier-1 authority), 1 designed to trip the guardrail (proposes
      an open-ended admin ticket without approver).
    infra-specialist.json — 8 Resolution entries: 4 TICKET_FILED or RUNBOOK_LINKED,
      2 FOLLOW_UP_SCHEDULED, 1 ESCALATED, 1 designed to trip the guardrail (P1 without
      approver).
    software-specialist.json — 8 Resolution entries: 4 RUNBOOK_LINKED or
      SELF_SERVICE_RESOLVED, 2 TICKET_FILED, 1 ESCALATED, 1 FOLLOW_UP_SCHEDULED.
    routing-judge.json — 10 RoutingScore entries, score 1–5, one-sentence rationale.
    ticket-guardrail.json — 10 GuardrailVerdict entries. 7 allowed=true. 3 allowed=false
      with violations "open-ended-admin-access", "p1-without-approver",
      "duplicate-open-incident". Match the guardrail-tripping entries above by requestId.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- (Lesson 1) AccessSpecialist, InfraSpecialist, SoftwareSpecialist all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) stepTimeout: classifyStep 20s, guardrailStep 20s, accessStep / infraStep /
  softwareStep / fileStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Request is Optional<T>.
- (Lesson 7) HelpdeskTasks.java declares the RESOLVE Task<Resolution> constant.
- (Lesson 8) Model names: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build".
- (Lesson 10) Port 9335 in application.conf.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column.
- (Lesson 13) Integration tier label is "Runs out of the box".
- (Lesson 23) No competitor brand names.
- (Lesson 24) Mermaid CSS overrides in index.html.
- (Lesson 25) API key sourcing five-option protocol.
- (Lesson 26) Tab switching by data-tab / data-panel attribute.
- SecretSanitizer runs INSIDE a Consumer before any LLM call.
- RoutingEvalScorer is out-of-band metadata; does not gate the workflow.
- The guardrail step happens BEFORE TicketFiled (and is skipped when proposedTicket is
  absent). A blocked proposed ticket never gets filed.
- TicketGuardrail is only invoked when the specialist's Resolution contains a non-empty
  proposedTicket; resolutions without a ticket proposal pass through unconditionally.
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

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
