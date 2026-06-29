# SPEC — auto-insurance-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Auto Insurance Agent.
**One-line pitch:** A triage agent classifies an inbound insurance member request and routes it to the appropriate specialist — claims, policy, rewards, or roadside — that handles it end-to-end, with PII redaction before any LLM call and a before-response guardrail enforcing insurance-communications policy.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *which* specialist should own the member request, then transfers the same task identity to that downstream specialist. The specialist is responsible for the full response; the classifier does not narrate or summarise. Two governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw member request event and the LLM call. The classifier and the specialists never see raw member IDs, policy numbers, VIN numbers, driver's license numbers, phone numbers, or account numbers.
- A **before-agent-response guardrail** runs on the specialist's draft response. It checks the draft against an insurance-communications policy rubric (no invented claim settlement amounts, no specific coverage promises outside the member's declared policy tier, no legal or medical advice, no echoing of `[REDACTED]` tokens) and blocks the draft from being delivered when it violates.

The pattern is a fan-out-of-one across four specialist branches: the workflow routes the request to exactly one specialist per member interaction. The other three specialists see no traffic for that request.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live request list. Every request displays its category chip, status pill, triage score, and (if resolved) the published response.
2. `RequestSimulator` (TimedAction) ticks every 30 s and inserts a new canned member request from `sample-events/insurance-requests.jsonl` into `RequestQueue`.
3. For each new request: `PiiSanitizer` (Consumer) redacts the payload, registers a `RequestEntity`, and starts an `InsuranceWorkflow`.
4. The workflow calls `ClaimTriageAgent`, gets a `TriageDecision { category, confidence, reason }`, and emits `TriageDecided` on the request entity.
5. Branch on `category`:
   - `CLAIM` → workflow calls `ClaimsSpecialist` with the `HANDLE` task and waits for the typed `MemberResponse` result.
   - `POLICY` → workflow calls `PolicySpecialist` with the same `HANDLE` task.
   - `REWARDS` → workflow calls `RewardsSpecialist`.
   - `ROADSIDE` → workflow calls `RoadsideSpecialist`.
   - `UNCLEAR` → workflow emits `RequestEscalated`; ends.
6. The specialist's draft `MemberResponse` passes through the before-agent-response guardrail. If accepted, `ResponsePublished` is emitted (terminal `RESOLVED`). If rejected, `ResponseBlocked` is emitted (terminal `BLOCKED`) with the violation list.
7. Independent of the workflow, `TriageEvalScorer` (Consumer) listens for `TriageDecided` events, calls `TriageJudge`, and writes `TriageScored { score, rationale }` back to the entity.
8. The user can click any request card and see the redacted request, the triage reason, the triage score, the chosen specialist, the draft (or blocked draft + violations), and the published response.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RequestSimulator` | `TimedAction` | Drips simulated member requests into `RequestQueue` every 30 s. | scheduler | `RequestQueue` |
| `RequestQueue` | `EventSourcedEntity` | Append-only audit log of every inbound request (`InboundRequestReceived`). | `RequestSimulator`, `InsuranceEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Subscribes to `RequestQueue` events; redacts PII; registers `RequestEntity`; starts an `InsuranceWorkflow`. | `RequestQueue` events | `RequestEntity`, `InsuranceWorkflow` |
| `ClaimTriageAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedRequest` into `CLAIM` / `POLICY` / `REWARDS` / `ROADSIDE` / `UNCLEAR` with confidence + reason. | invoked by `InsuranceWorkflow` | returns `TriageDecision` |
| `ClaimsSpecialist` | `AutonomousAgent` | Owns the `HANDLE` task for claim status and first-notice-of-loss requests. Returns typed `MemberResponse`. | invoked by `InsuranceWorkflow` | returns `MemberResponse` |
| `PolicySpecialist` | `AutonomousAgent` | Owns the `HANDLE` task for policy changes, coverage questions, and billing inquiries. Returns typed `MemberResponse`. | invoked by `InsuranceWorkflow` | returns `MemberResponse` |
| `RewardsSpecialist` | `AutonomousAgent` | Owns the `HANDLE` task for rewards point redemptions and program inquiries. Returns typed `MemberResponse`. | invoked by `InsuranceWorkflow` | returns `MemberResponse` |
| `RoadsideSpecialist` | `AutonomousAgent` | Dispatches roadside assistance and returns a typed `MemberResponse` with a confirmation reference. Returns typed `MemberResponse`. | invoked by `InsuranceWorkflow` | returns `MemberResponse` |
| `TriageJudge` | `Agent` (typed) | Grades a triage decision against the sanitized request. Returns `TriageScore { score 1–5, rationale }`. | invoked by `TriageEvalScorer` | returns `TriageScore` |
| `ResponseGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks a draft `MemberResponse` against the insurance-communications policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `InsuranceWorkflow` | returns `GuardrailVerdict` |
| `InsuranceWorkflow` | `Workflow` | Per-request orchestration: triage → branch → resolve → guardrail → publish. | `PiiSanitizer` (start) | `RequestEntity` |
| `RequestEntity` | `EventSourcedEntity` | Per-request lifecycle. | `InsuranceWorkflow`, `TriageEvalScorer` | `RequestView` |
| `RequestView` | `View` | Read-model row per request. | `RequestEntity` events | `InsuranceEndpoint` |
| `TriageEvalScorer` | `Consumer` | Subscribes to `RequestEntity` events; on `TriageDecided` invokes `TriageJudge` and writes `TriageScored` back. | `RequestEntity` events | `RequestEntity` |
| `InsuranceEndpoint` | `HttpEndpoint` | `/api/requests/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `RequestView`, `RequestEntity`, `RequestQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record IncomingRequest(
    String requestId,
    String memberId,
    String channel,          // "phone" | "mobile-app" | "web-portal" | "email"
    String subject,
    String body,
    Instant receivedAt
) {}

record SanitizedRequest(
    String redactedSubject,
    String redactedBody,
    List<String> piiCategoriesFound
) {}

enum RequestCategory { CLAIM, POLICY, REWARDS, ROADSIDE, UNCLEAR }

record TriageDecision(
    RequestCategory category,
    String confidence,       // "high" | "medium" | "low"
    String reason            // one short sentence
) {}

enum ResponseAction {
    CLAIM_ACKNOWLEDGED,
    CLAIM_STATUS_PROVIDED,
    POLICY_INFO_PROVIDED,
    COVERAGE_CONFIRMED,
    REWARDS_REDEEMED,
    REWARDS_INFO_PROVIDED,
    ROADSIDE_DISPATCHED,
    FOLLOW_UP_SCHEDULED,
    ESCALATED
}

record MemberResponse(
    String responseSubject,
    String responseBody,
    ResponseAction action,
    String specialistTag,    // "claims" | "policy" | "rewards" | "roadside"
    Optional<String> confirmationRef,   // roadside dispatch reference number
    Instant resolvedAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,
    String rubricVersion
) {}

record TriageScore(
    int score,               // 1..5
    String rationale,
    Instant scoredAt
) {}

record MemberRequest(
    String requestId,
    IncomingRequest incoming,
    Optional<SanitizedRequest> sanitized,
    Optional<TriageDecision> triage,
    Optional<MemberResponse> response,
    Optional<GuardrailVerdict> guardrail,
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
    ROUTED_CLAIM,
    ROUTED_POLICY,
    ROUTED_REWARDS,
    ROUTED_ROADSIDE,
    RESPONSE_DRAFTED,
    BLOCKED,
    RESOLVED,
    ESCALATED
}
```

Events on `RequestEntity`: `RequestRegistered`, `RequestSanitized`, `TriageDecided`, `RequestRouted`, `ResponseDrafted`, `GuardrailVerdictAttached`, `ResponsePublished`, `ResponseBlocked`, `RequestEscalated`, `TriageScored`.

Events on `RequestQueue`: `InboundRequestReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/requests` — list all requests (newest-first), optional `?category=CLAIM|POLICY|REWARDS|ROADSIDE|UNCLEAR&status=…` filtered client-side.
- `GET /api/requests/{id}` — one request.
- `POST /api/requests` — manually submit a member request (body `IncomingRequest` minus `requestId` and `receivedAt`); server assigns both.
- `POST /api/requests/{id}/unblock` — body `{ decidedBy, note }` — supervisor override; transitions `BLOCKED` to `RESOLVED` if the supervisor elects to publish the blocked draft.
- `GET /api/requests/sse` — Server-Sent Events for every request change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Auto Insurance Agent</title>`.

The App UI tab is a three-pane layout: **left** is the request list (status pill + category chip + score chip), **centre** is the selected request's redacted content + triage decision + score, **right** is the chosen specialist's draft + guardrail verdict + published response (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts member IDs, policy numbers, VIN numbers, driver's license numbers, phone numbers, and account numbers from the request body before any LLM sees it. The categories found are kept for audit; the redacted text is what reaches the agents.
- **G1 — before-agent-response guardrail** on all four specialist agents: checks every draft `MemberResponse` against a rubric (no invented claim settlement amounts, no specific coverage promises outside published tier, no legal or medical advice, no `[REDACTED]` token echoes, no invented roadside dispatch reference numbers). Blocking — a violation puts the request in `BLOCKED` for supervisor review.

## 9. Agent prompts

- `ClaimTriageAgent` → `prompts/claim-triage-agent.md`. Typed classifier; returns one of `CLAIM`, `POLICY`, `REWARDS`, `ROADSIDE`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `ClaimsSpecialist` → `prompts/claims-specialist.md`. Owns the `HANDLE` task for claim requests. Never invents settlement amounts; escalates outside authority.
- `PolicySpecialist` → `prompts/policy-specialist.md`. Owns the `HANDLE` task for policy and coverage requests. Never confirms coverage tiers it cannot verify.
- `RewardsSpecialist` → `prompts/rewards-specialist.md`. Owns the `HANDLE` task for rewards program requests. Never invents point balances.
- `RoadsideSpecialist` → `prompts/roadside-specialist.md`. Dispatches assistance and returns a confirmation reference. Never invents an ETA.
- `TriageJudge` → `prompts/triage-judge.md`. Grades a triage decision against a 1–5 rubric with one-sentence rationale.
- `ResponseGuardrail` → `prompts/response-guardrail.md`. Returns a `GuardrailVerdict { allowed, violations }`. Conservative — borderline drafts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a claim-status request → triaged `CLAIM` → resolved by `ClaimsSpecialist` → guardrail passes → published.
2. **J2** — Simulator drips a roadside request → triaged `ROADSIDE` → dispatched by `RoadsideSpecialist` → published with a confirmation reference.
3. **J3** — An ambiguous or off-topic request triages as `UNCLEAR` and lands in `ESCALATED` without any specialist being invoked.
4. **J4** — A draft that states a specific claim settlement dollar amount not provided by the member is blocked; the request lands in `BLOCKED` with the violation listed.
5. **J5** — Every triaged request carries a `TriageScore` (1–5) and rationale within ~10 s of the triage decision.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named auto-insurance-agent demonstrating the handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real insurance-portal integration).
Maven group io.akka.samples. Maven artifact handoff-routing-cx-support-insurance-triage-router.
Java package io.akka.samples.autoinsuranceagent. Akka 3.6.0. HTTP port 9615.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ClaimTriageAgent — classifier. System prompt loaded from
  prompts/claim-triage-agent.md. Input: SanitizedRequest{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}. Output: TriageDecision{category: RequestCategory
  (CLAIM/POLICY/REWARDS/ROADSIDE/UNCLEAR), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent ClaimsSpecialist — definition() with capability(TaskAcceptance.of(HANDLE)
  .maxIterationsPerTask(3)). System prompt from prompts/claims-specialist.md. Input:
  SanitizedRequest + TriageDecision. Output: MemberResponse{responseSubject, responseBody,
  action: ResponseAction, specialistTag = "claims", confirmationRef: Optional<String>,
  resolvedAt}. Never invents settlement amounts; sets action=ESCALATED when outside authority.

- 1 AutonomousAgent PolicySpecialist — definition() with capability(TaskAcceptance.of(HANDLE)
  .maxIterationsPerTask(3)). System prompt from prompts/policy-specialist.md. Same
  input shape; specialistTag = "policy".

- 1 AutonomousAgent RewardsSpecialist — definition() with capability(TaskAcceptance.of(HANDLE)
  .maxIterationsPerTask(3)). System prompt from prompts/rewards-specialist.md. Same
  input shape; specialistTag = "rewards".

- 1 AutonomousAgent RoadsideSpecialist — definition() with capability(TaskAcceptance.of(HANDLE)
  .maxIterationsPerTask(3)). System prompt from prompts/roadside-specialist.md. Same
  input shape; specialistTag = "roadside". Populates confirmationRef with a generated
  dispatch reference (format "RSA-XXXXXXXX"); never invents an ETA.

- 1 Agent (typed) TriageJudge — judge. System prompt from prompts/triage-judge.md. Input:
  SanitizedRequest + TriageDecision. Output: TriageScore{score: int 1–5, rationale: String,
  scoredAt: Instant}.

- 1 Agent (typed) ResponseGuardrail — typed rubric check. System prompt from
  prompts/response-guardrail.md. Input: SanitizedRequest + MemberResponse. Output:
  GuardrailVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by InsuranceWorkflow before publishing; blocking.

- 1 Workflow InsuranceWorkflow per requestId. Steps:
    triageStep -> routeStep ->
      {claimsStep | policyStep | rewardsStep | roadsideStep | escalateStep}
      -> guardrailStep -> publishStep
  triageStep calls componentClient.forAgent().inSession(requestId).method(ClaimTriageAgent::triage)
    .invoke(sanitized). On success emits TriageDecided via RequestEntity.recordTriage.
  routeStep branches on TriageDecision.category:
    CLAIM    -> proceed to claimsStep   (emits RequestRouted{CLAIM})
    POLICY   -> proceed to policyStep   (emits RequestRouted{POLICY})
    REWARDS  -> proceed to rewardsStep  (emits RequestRouted{REWARDS})
    ROADSIDE -> proceed to roadsideStep (emits RequestRouted{ROADSIDE})
    UNCLEAR  -> escalateStep (emits RequestEscalated; terminates).
  claimsStep / policyStep / rewardsStep / roadsideStep call
    forAutonomousAgent(<Specialist>.class, requestId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, triage))) returning a taskId,
    then forTask(taskId).result(InsuranceTasks.HANDLE) to block on the typed MemberResponse.
    On success emits ResponseDrafted.
  guardrailStep calls forAgent(...).method(ResponseGuardrail::check).invoke(sanitized, draft).
    On verdict.allowed=true proceed to publishStep (emits ResponsePublished, terminal RESOLVED).
    On verdict.allowed=false emit ResponseBlocked (terminal BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on triageStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on claimsStep, policyStep, rewardsStep,
    roadsideStep, and publishStep. defaultStepRecovery(maxRetries(2).failoverTo(InsuranceWorkflow::error)).

- 2 EventSourcedEntities:
    * RequestQueue — append-only audit log. Command receive(IncomingRequest) emits
      InboundRequestReceived{incoming}. No mutable state beyond a counter; commands are
      idempotent on incoming.requestId.
    * RequestEntity (one per requestId) — full per-request lifecycle. State
      MemberRequest{requestId, incoming: IncomingRequest, Optional<SanitizedRequest> sanitized,
      Optional<TriageDecision> triage, Optional<MemberResponse> response,
      Optional<GuardrailVerdict> guardrail, Optional<TriageScore> triageScore,
      Optional<String> escalationReason, RequestStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. RequestStatus enum: RECEIVED, SANITIZED, TRIAGED,
      ROUTED_CLAIM, ROUTED_POLICY, ROUTED_REWARDS, ROUTED_ROADSIDE, RESPONSE_DRAFTED,
      BLOCKED, RESOLVED, ESCALATED. Events: RequestRegistered, RequestSanitized,
      TriageDecided, RequestRouted, ResponseDrafted, GuardrailVerdictAttached,
      ResponsePublished, ResponseBlocked, RequestEscalated, TriageScored. Commands:
      registerIncoming, attachSanitized, recordTriage, recordRouting, recordDraft,
      recordGuardrailVerdict, publish, block, escalate, recordTriageScore, unblock, getRequest.
      emptyState() returns MemberRequest.initial("") with no commandContext() reference.

- 2 Consumers:
    * PiiSanitizer subscribed to RequestQueue events; for each InboundRequestReceived
      applies a regex+heuristic redaction pipeline (member IDs matching MBR-\w+, policy
      numbers matching POL-\d+, VINs matching a 17-char alphanumeric pattern, driver's
      license numbers, phone numbers, account numbers matching ACCT-\w+, plus heuristic
      name-redaction in greeting positions) to subject + body, builds SanitizedRequest with
      piiCategoriesFound, and calls RequestEntity.registerIncoming then attachSanitized for
      the requestId; then starts an InsuranceWorkflow with requestId as the workflow id.
    * TriageEvalScorer subscribed to RequestEntity events; on TriageDecided invokes
      TriageJudge.score(sanitized, decision) and calls RequestEntity.recordTriageScore(
      requestId, score). On any other event type, no-op. Use componentClient — do NOT
      call the agent from a TimedAction.

- 1 View RequestView with row type RequestRow (mirrors MemberRequest; uses Optional<T> for
  every nullable lifecycle field per Lesson 6). Table updater consumes RequestEntity events.
  ONE query getAllRequests SELECT * AS requests FROM request_view. No WHERE category or
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side in callers.

- 1 TimedAction RequestSimulator — every 30s, reads next line from
  src/main/resources/sample-events/insurance-requests.jsonl (loops at EOF) and calls
  RequestQueue.receive with a fresh requestId (UUID).

- 2 HttpEndpoints:
    * InsuranceEndpoint at /api with GET /requests (list from RequestView.getAllRequests,
      filter client-side by ?category and ?status query params), GET /requests/{id},
      POST /requests (body IncomingRequest minus requestId/receivedAt — server assigns),
      POST /requests/{id}/unblock (body {decidedBy, note} — supervisor override:
      publishes the blocked draft as RESOLVED with an audit note),
      GET /requests/sse (serverSentEventsForView over getAllRequests), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- InsuranceTasks.java declaring the task constants: HANDLE (resultConformsTo
  MemberResponse.class, description "Handle the insurance member request end-to-end and
  return a typed MemberResponse").
- Domain records IncomingRequest, SanitizedRequest, TriageDecision, MemberResponse,
  GuardrailVerdict, TriageScore, and the MemberRequest entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9615 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/insurance-requests.jsonl with 10 canned lines (2 CLAIM-
  flavoured, 2 POLICY-flavoured, 2 REWARDS-flavoured, 2 ROADSIDE-flavoured, 1 UNCLEAR, 1
  designed to trip the guardrail with an invented settlement amount claim).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, G1 guardrail
  before-agent-response. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root pre-filled for insurance cx-support domain.
- prompts/claim-triage-agent.md, prompts/claims-specialist.md, prompts/policy-specialist.md,
  prompts/rewards-specialist.md, prompts/roadside-specialist.md, prompts/triage-judge.md,
  prompts/response-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Auto Insurance Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = request list with category chip + status pill + score chip; centre = redacted
  request + triage block; right = specialist draft + guardrail verdict + published response
  or violations + Unblock button). Browser title exactly:
  <title>Akka Sample: Auto Insurance Agent</title>. No subtitle on the Overview tab.

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
    claim-triage-agent.json — 12 TriageDecision entries spanning CLAIM (accident
      FNOL, total-loss inquiry, glass claim), POLICY (coverage question, deductible
      inquiry, endorsement change), REWARDS (point redemption, tier status), ROADSIDE
      (flat tyre, battery jump, lockout), and UNCLEAR (very short message, off-topic).
      Confidence + a one-sentence reason on each.
    claims-specialist.json — 8 MemberResponse entries: 5 with action
      CLAIM_STATUS_PROVIDED or CLAIM_ACKNOWLEDGED (well within authority), 1 with
      action ESCALATED (settlement amount above authority), 1 with action
      FOLLOW_UP_SCHEDULED, 1 designed to trip the guardrail (states a specific
      "$4,200 settlement" amount not provided by member).
    policy-specialist.json — 8 MemberResponse entries: 5 with POLICY_INFO_PROVIDED
      or COVERAGE_CONFIRMED, 1 with FOLLOW_UP_SCHEDULED, 1 with ESCALATED, 1
      designed to trip the guardrail (promises a specific coverage discount not
      in published tier).
    rewards-specialist.json — 6 MemberResponse entries: 4 with REWARDS_REDEEMED or
      REWARDS_INFO_PROVIDED, 1 with ESCALATED, 1 designed to trip the guardrail
      (echoes a [REDACTED] token back in the body).
    roadside-specialist.json — 6 MemberResponse entries: 4 with ROADSIDE_DISPATCHED
      (confirmationRef populated), 1 with ESCALATED, 1 designed to trip the guardrail
      (invents a "within 20 minutes" ETA not grounded in any policy).
    triage-judge.json — 10 TriageScore entries, score 1–5, one-sentence rationale
      matching the rubric (category-correctness / confidence-calibration / reason-quality).
    response-guardrail.json — 10 GuardrailVerdict entries: 7 with allowed=true and
      empty violations; 3 with allowed=false and one violation each
      ("invented-settlement-amount", "invented-coverage-discount", "echoes-redacted-token").
- A MockModelProvider.seedFor(requestId) helper makes per-request selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent. ClaimsSpecialist,
  PolicySpecialist, RewardsSpecialist, and RoadsideSpecialist all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): triageStep 20s,
  guardrailStep 20s, claimsStep / policyStep / rewardsStep / roadsideStep / publishStep 60s.
- (Lesson 6) Every nullable lifecycle field on MemberRequest is Optional<T>. The
  RequestView row type uses the same Optional wrapping.
- (Lesson 7) InsuranceTasks.java declares the HANDLE Task<MemberResponse> constant.
  All four specialists' definition().capability(TaskAcceptance.of(HANDLE)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9615 in application.conf; not 9000.
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
- The TriageEvalScorer Consumer reacts to TriageDecided events and calls
  TriageJudge via componentClient.forAgent(). It does NOT modify the workflow
  flow — the eval is out-of-band metadata.
- The guardrail step happens BEFORE ResponsePublished. A blocked draft
  never reaches the UI as published — only as a "blocked draft + violations"
  surface for the supervisor.
- No forbidden words in user-facing text.
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
