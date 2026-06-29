# SPEC — support-multi-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Support Multi-Agent.
**One-line pitch:** A router agent classifies an incoming support request and hands the resolution off to a billing, technical, or account specialist that owns it end-to-end, with PII redaction and a before-response guardrail on every specialist draft.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *who* should own the task, then transfers the same task identity to a downstream specialist agent that produces the final output. The downstream agent is responsible for the whole resolution; the classifier does not narrate or summarize. Two governance mechanisms are layered on top:

- A **PII sanitizer** runs inside a Consumer between the raw ticket event and the LLM call. The router and all specialists never see raw emails, phone numbers, card numbers, or account ids.
- A **before-agent-response guardrail** runs on every specialist's draft resolution. It checks the draft against a policy rubric (no invented refund amounts, no specific timelines outside the published SLA, no medical/legal advice, no echoing of `[REDACTED]` tokens) and blocks the draft from reaching the customer when it violates.

The pattern is a fan-out-of-one: the workflow branches on the classifier's category, and only the matched specialist is invoked. The other specialists see no traffic for that case.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live case list. Every case displays its routing chip, status pill, and (if resolved) the published response.
2. `TicketSimulator` (TimedAction) ticks every 30 s and inserts a new canned request from `sample-events/support-cases.jsonl` into `TicketQueue`.
3. For each new request: `PiiSanitizer` (Consumer) redacts the payload, registers a `CaseEntity`, and starts a `SupportWorkflow`.
4. The workflow calls `RouterAgent`, gets a `RoutingDecision { category, confidence, reason }`, and emits `CaseRouted` on the case.
5. Branch on `category`:
   - `BILLING` → workflow calls `BillingSpecialist` with the `RESOLVE` task and waits for the typed `Resolution`.
   - `TECHNICAL` → workflow calls `TechnicalSpecialist` with the same `RESOLVE` task.
   - `ACCOUNT` → workflow calls `AccountSpecialist` with the same `RESOLVE` task.
   - `UNCLEAR` → workflow emits `CaseEscalated`; ends.
6. The specialist's draft `Resolution` passes through the before-agent-response guardrail. If accepted, `ResolutionPublished` is emitted (terminal `RESOLVED`). If rejected, `ResolutionBlocked` is emitted (terminal `BLOCKED`) with the violation list.
7. The user can click any case card and see the redacted request, the routing decision, the chosen specialist, the draft (or blocked draft + violations), and the published response.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TicketSimulator` | `TimedAction` | Drips simulated support cases into `TicketQueue` every 30 s. | scheduler | `TicketQueue` |
| `TicketQueue` | `EventSourcedEntity` | Append-only audit log of every inbound case (`InboundCaseReceived`). | `TicketSimulator`, `SupportEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Subscribes to `TicketQueue` events; redacts PII; registers `CaseEntity`; starts a `SupportWorkflow`. | `TicketQueue` events | `CaseEntity`, `SupportWorkflow` |
| `RouterAgent` | `Agent` (typed, not autonomous) | Classifies a `SanitizedCase` into `BILLING` / `TECHNICAL` / `ACCOUNT` / `UNCLEAR` with confidence + reason. | invoked by `SupportWorkflow` | returns `RoutingDecision` |
| `BillingSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for billing cases. Returns typed `Resolution`. | invoked by `SupportWorkflow` | returns `Resolution` |
| `TechnicalSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for technical cases. Returns typed `Resolution`. | invoked by `SupportWorkflow` | returns `Resolution` |
| `AccountSpecialist` | `AutonomousAgent` | Owns the `RESOLVE` task for account management cases. Returns typed `Resolution`. | invoked by `SupportWorkflow` | returns `Resolution` |
| `DraftGuardrail` | `Agent` (typed) | Before-agent-response guardrail: checks a draft `Resolution` against the policy rubric. Returns `GuardrailVerdict { allowed, violations }`. | invoked by `SupportWorkflow` | returns `GuardrailVerdict` |
| `SupportWorkflow` | `Workflow` | Per-case orchestration: sanitize-handoff → route → resolve → guardrail → publish. | `PiiSanitizer` (start) | `CaseEntity` |
| `CaseEntity` | `EventSourcedEntity` | Per-case lifecycle. | `SupportWorkflow` | `CaseView` |
| `CaseView` | `View` | Read-model row per case. | `CaseEntity` events | `SupportEndpoint` |
| `SupportEndpoint` | `HttpEndpoint` | `/api/cases/*` — list, get, manual submit, manual unblock, SSE; `/api/metadata/*`. | — | `CaseView`, `CaseEntity`, `TicketQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record InboundCase(
    String caseId,
    String customerId,
    String channel,            // "email" | "chat" | "web-form" | "phone"
    String subject,
    String body,
    Instant receivedAt
) {}

record SanitizedCase(
    String redactedSubject,
    String redactedBody,
    List<String> piiCategoriesFound
) {}

enum CaseCategory { BILLING, TECHNICAL, ACCOUNT, UNCLEAR }

record RoutingDecision(
    CaseCategory category,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

enum ResolutionAction { REFUND_INITIATED, ARTICLE_LINKED, FOLLOW_UP_SCHEDULED, INFO_PROVIDED, ACCESS_RESTORED, ESCALATED }

record Resolution(
    String responseSubject,
    String responseBody,
    ResolutionAction action,
    String specialistTag,      // "billing" | "technical" | "account"
    Instant resolvedAt
) {}

record GuardrailVerdict(
    boolean allowed,
    List<String> violations,   // empty when allowed
    String rubricVersion
) {}

record Case(
    String caseId,
    InboundCase incoming,
    Optional<SanitizedCase> sanitized,
    Optional<RoutingDecision> routing,
    Optional<Resolution> resolution,
    Optional<GuardrailVerdict> guardrail,
    Optional<String> escalationReason,
    CaseStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum CaseStatus {
    RECEIVED,
    SANITIZED,
    ROUTED_BILLING,
    ROUTED_TECHNICAL,
    ROUTED_ACCOUNT,
    RESOLUTION_DRAFTED,
    BLOCKED,
    RESOLVED,
    ESCALATED
}
```

Events on `CaseEntity`: `CaseRegistered`, `CaseSanitized`, `CaseRouted`, `ResolutionDrafted`, `GuardrailVerdictAttached`, `ResolutionPublished`, `ResolutionBlocked`, `CaseEscalated`.

Events on `TicketQueue`: `InboundCaseReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/cases` — list all cases (newest-first), optional `?category=BILLING|TECHNICAL|ACCOUNT|UNCLEAR&status=…` filtered client-side.
- `GET /api/cases/{id}` — one case.
- `POST /api/cases` — manually submit a request (body `InboundCase` minus `caseId` and `receivedAt`); server assigns both.
- `POST /api/cases/{id}/unblock` — body `{ decidedBy, note }` — operator override; transitions `BLOCKED` to `RESOLVED` if the operator chooses to publish the blocked draft.
- `GET /api/cases/sse` — Server-Sent Events for every case change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Support Multi-Agent</title>`.

The App UI tab is a three-pane layout: **left** is the case list (status pill + category chip), **centre** is the selected case's redacted request + routing decision, **right** is the chosen specialist's draft + guardrail verdict + published response (or violations + Unblock button when `BLOCKED`).

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts emails, phone numbers, card numbers, and account ids from the request body before any LLM sees it. The categories found are kept for audit; the redacted text is what reaches the agents.
- **G1 — before-agent-response guardrail** on `BillingSpecialist`, `TechnicalSpecialist`, and `AccountSpecialist`: checks every draft `Resolution` against a rubric (no invented refund amounts, no specific timelines outside published SLA, no medical/legal/financial advice, no echoing of `[REDACTED]` tokens, no invented KB article ids). Blocking — a violation puts the case in `BLOCKED` for human review.

## 9. Agent prompts

- `RouterAgent` → `prompts/router-agent.md`. Typed classifier; returns one of `BILLING`, `TECHNICAL`, `ACCOUNT`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `BillingSpecialist` → `prompts/billing-specialist.md`. Owns the `RESOLVE` task for billing cases. Never invents refund amounts; routes outside its authority to `ESCALATED`.
- `TechnicalSpecialist` → `prompts/technical-specialist.md`. Owns the `RESOLVE` task for technical cases. Cites a help-centre article id when one applies; never invents one.
- `AccountSpecialist` → `prompts/account-specialist.md`. Owns the `RESOLVE` task for account management cases. Handles access recovery, profile updates, and subscription setting changes.
- `DraftGuardrail` → `prompts/draft-guardrail.md`. Returns a `GuardrailVerdict { allowed, violations }`. Conservative — borderline drafts are blocked.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a billing-flavoured case → routed `BILLING` → resolved by `BillingSpecialist` → guardrail passes → published.
2. **J2** — Simulator drips a technical case → routed `TECHNICAL` → resolved by `TechnicalSpecialist` → guardrail passes → published.
3. **J3** — Simulator drips an account management case → routed `ACCOUNT` → resolved by `AccountSpecialist` → guardrail passes → published.
4. **J4** — An ambiguous case routes as `UNCLEAR` and lands in `ESCALATED` without any specialist invocation.
5. **J5** — A draft that promises a 24-hour refund (outside policy) is blocked; the case lands in `BLOCKED` with the violation listed; the operator can either unblock or leave it.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named support-multi-agent demonstrating the handoff-routing × cx-support cell.
Runs out of the box (in-process simulated inbound stream; no real ticketing integration).
Maven group io.akka.samples. Maven artifact handoff-routing-cx-support-support-router-team.
Java package io.akka.samples.supportmultiagent. Akka 3.6.0. HTTP port 9427.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) RouterAgent — classifier. System prompt loaded from
  prompts/router-agent.md. Input: SanitizedCase{redactedSubject, redactedBody,
  piiCategoriesFound: List<String>}. Output: RoutingDecision{category: CaseCategory
  (BILLING/TECHNICAL/ACCOUNT/UNCLEAR), confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent BillingSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/billing-specialist.md. Input:
  SanitizedCase + RoutingDecision. Output: Resolution{responseSubject, responseBody,
  action: ResolutionAction, specialistTag = "billing", resolvedAt}. Never invents refund
  amounts; sets action=ESCALATED with a reason when outside authority.

- 1 AutonomousAgent TechnicalSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/technical-specialist.md. Same
  input shape; specialistTag = "technical".

- 1 AutonomousAgent AccountSpecialist — definition() with capability(TaskAcceptance.of(RESOLVE)
  .maxIterationsPerTask(3)). System prompt from prompts/account-specialist.md. Same
  input shape; specialistTag = "account". Handles access restoration, profile data updates,
  and subscription configuration changes.

- 1 Agent (typed) DraftGuardrail — typed rubric check. System prompt from
  prompts/draft-guardrail.md. Input: SanitizedCase + Resolution. Output:
  GuardrailVerdict{allowed: boolean, violations: List<String>, rubricVersion: String}.
  Used by SupportWorkflow before publishing; blocking.

- 1 Workflow SupportWorkflow per caseId. Steps:
    routeStep -> {billingStep | technicalStep | accountStep | escalateStep}
              -> guardrailStep -> publishStep
  routeStep calls componentClient.forAgent().inSession(caseId).method(RouterAgent::route)
    .invoke(sanitized). On success emits CaseRouted via CaseEntity.recordRouting.
  Branch on RoutingDecision.category:
    BILLING   -> billingStep  (emits CaseRouted{BILLING})
    TECHNICAL -> technicalStep (emits CaseRouted{TECHNICAL})
    ACCOUNT   -> accountStep  (emits CaseRouted{ACCOUNT})
    UNCLEAR   -> escalateStep (emits CaseEscalated; terminates).
  billingStep / technicalStep / accountStep call
    forAutonomousAgent(<Specialist>.class, caseId)
    .runSingleTask(TaskDef.instructions(buildPrompt(sanitized, routing))) returning a taskId,
    then forTask(taskId).result(SupportTasks.RESOLVE) to block on the typed Resolution.
    On success emits ResolutionDrafted.
  guardrailStep calls forAgent(...).method(DraftGuardrail::check).invoke(sanitized, draft).
    On verdict.allowed=true proceed to publishStep (emits ResolutionPublished, terminal
    RESOLVED). On verdict.allowed=false emit ResolutionBlocked (terminal BLOCKED) and end.
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on routeStep and
    guardrailStep, stepTimeout(Duration.ofSeconds(60)) on billingStep, technicalStep,
    accountStep, and publishStep. defaultStepRecovery(maxRetries(2).failoverTo(SupportWorkflow::error)).

- 2 EventSourcedEntities:
    * TicketQueue — append-only audit log. Command receive(InboundCase) emits
      InboundCaseReceived{incoming}. No mutable state beyond a counter; commands are
      idempotent on incoming.caseId.
    * CaseEntity (one per caseId) — full per-case lifecycle. State Case{caseId,
      incoming: InboundCase, Optional<SanitizedCase> sanitized,
      Optional<RoutingDecision> routing, Optional<Resolution> resolution,
      Optional<GuardrailVerdict> guardrail, Optional<String> escalationReason,
      CaseStatus status, Instant createdAt, Optional<Instant> finishedAt}.
      CaseStatus enum: RECEIVED, SANITIZED, ROUTED_BILLING, ROUTED_TECHNICAL,
      ROUTED_ACCOUNT, RESOLUTION_DRAFTED, BLOCKED, RESOLVED, ESCALATED.
      Events: CaseRegistered, CaseSanitized, CaseRouted, ResolutionDrafted,
      GuardrailVerdictAttached, ResolutionPublished, ResolutionBlocked, CaseEscalated.
      Commands: registerIncoming, attachSanitized, recordRouting, recordDraft,
      recordGuardrailVerdict, publish, block, escalate, unblock, getCase.
      emptyState() returns Case.initial("") with no commandContext() reference.

- 1 Consumer PiiSanitizer subscribed to TicketQueue events; for each InboundCaseReceived
    applies a regex+heuristic redaction pipeline (emails, phone numbers, payment-card
    numbers via Luhn-aware regex, account ids matching ACCT-\w+, customer ids matching
    CUST-\d+, plus heuristic name-redaction in greeting positions) to subject + body,
    builds SanitizedCase with piiCategoriesFound, and calls CaseEntity
    .registerIncoming then attachSanitized for the caseId; then starts a SupportWorkflow
    with caseId as the workflow id.

- 1 View CaseView with row type CaseRow (mirrors Case; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). Table updater consumes CaseEntity events.
  ONE query getAllCases SELECT * AS cases FROM case_view. No WHERE category or
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side in callers.

- 1 TimedAction TicketSimulator — every 30s, reads next line from
  src/main/resources/sample-events/support-cases.jsonl (loops at EOF) and calls
  TicketQueue.receive with a fresh caseId (UUID).

- 2 HttpEndpoints:
    * SupportEndpoint at /api with GET /cases (list from CaseView.getAllCases,
      filter client-side by ?category and ?status query params), GET /cases/{id},
      POST /cases (body InboundCase minus caseId/receivedAt — server assigns),
      POST /cases/{id}/unblock (body {decidedBy, note} — operator override:
      publishes the blocked draft as RESOLVED with an audit note),
      GET /cases/sse (serverSentEventsForView over getAllCases), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- SupportTasks.java declaring the task constants: RESOLVE (resultConformsTo Resolution.class,
  description "Resolve the support case end-to-end and return a typed Resolution").
- Domain records InboundCase, SanitizedCase, RoutingDecision, Resolution,
  GuardrailVerdict, and the Case entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9427 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/support-cases.jsonl with 12 canned lines (3 BILLING-
  flavoured, 3 TECHNICAL-flavoured, 3 ACCOUNT-flavoured, 2 UNCLEAR-flavoured, 1 designed
  to trip the guardrail with a specific-timeline refund promise).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, G1 guardrail
  before-agent-response. Matching simplified_view list. No regulation_anchors
  (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = customer-support,
  data.data_classes.pii = true, decisions.authority_level = autonomous-with-guardrail,
  oversight.human_in_loop = false (the system publishes without HITL by default — only
  blocked drafts wait for a human), data.pii_handled_by_sanitizer_before_llm = true,
  failure.failure_modes including "wrong-category-routing", "inappropriate-reply-content",
  "pii-leakage-via-llm", "refund-amount-fabrication"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/router-agent.md, prompts/billing-specialist.md, prompts/technical-specialist.md,
  prompts/account-specialist.md, prompts/draft-guardrail.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Support Multi-Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a three-column layout
  (left = case list with category chip + status pill; centre = redacted request + routing
  block; right = specialist draft + guardrail verdict + published response or violations
  + Unblock button). Browser title exactly:
  <title>Akka Sample: Support Multi-Agent</title>. No subtitle on the Overview tab.

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
  pseudo-randomly per call (seeded by caseId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    router-agent.json — 12 RoutingDecision entries spanning BILLING (charge
      disputes, refund requests, plan changes), TECHNICAL (API errors,
      integration questions, performance reports), ACCOUNT (password reset,
      MFA setup, profile update requests), and UNCLEAR (mixed content, very
      short messages, off-topic). Confidence + a one-sentence reason on each.
    billing-specialist.json — 8 Resolution entries: 5 with action
      INFO_PROVIDED or REFUND_INITIATED (well within authority), 1 with
      action ESCALATED (refund > authority), 1 with action ARTICLE_LINKED,
      1 designed to trip the guardrail (promises a "24-hour refund window" —
      outside published SLA) so guardrail tests have material.
    technical-specialist.json — 8 Resolution entries: 5 with INFO_PROVIDED
      or ARTICLE_LINKED, 1 with FOLLOW_UP_SCHEDULED, 1 with ESCALATED,
      1 designed to trip the guardrail (gives specific medical-adjacent
      advice — for the rubric's no-medical-advice rule).
    account-specialist.json — 8 Resolution entries: 5 with ACCESS_RESTORED
      or INFO_PROVIDED, 1 with FOLLOW_UP_SCHEDULED, 1 with ESCALATED,
      1 designed to trip the guardrail (echoes a [REDACTED] token in the
      response body).
    draft-guardrail.json — 10 GuardrailVerdict entries. 7 with
      allowed=true and empty violations. 3 with allowed=false and one
      violation each ("invented-refund-timeline", "out-of-scope-medical-advice",
      "echoes-redacted-token"). The mock should match the rubric-tripping
      entries above when called for the same caseId.
- A MockModelProvider.seedFor(caseId) helper makes per-case selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  BillingSpecialist, TechnicalSpecialist, and AccountSpecialist all extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): routeStep 20s,
  guardrailStep 20s, billingStep / technicalStep / accountStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Case is Optional<T>. The
  CaseView row type uses the same Optional wrapping.
- (Lesson 7) SupportTasks.java declares the RESOLVE Task<Resolution> constant.
  All three specialists' definition().capability(TaskAcceptance.of(RESOLVE)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9427 in application.conf; not 9000.
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
- The guardrail step happens BEFORE ResolutionPublished. A blocked draft
  never reaches the UI as published — only as a "blocked draft + violations"
  surface for the operator.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, use, use, marketing
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

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
