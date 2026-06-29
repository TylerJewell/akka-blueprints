# SPEC — mortgage-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Mortgage Assistant (Multi-Agent).
**One-line pitch:** Submit a mortgage application; a supervisor delegates eligibility, documentation, and underwriting in parallel, then waits for a human underwriter's sign-off before issuing a decision.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to three AutonomousAgents in parallel, gathers their assessments, and requests a fourth agent to draft applicant communications at each stage. The blueprint demonstrates a **PII sanitiser** that strips applicant identity before prompts reach any agent, a **sectoral compliance filter** that screens lending terms against applicable product rules, a **human-in-the-loop (HITL) gate** that halts the workflow at the underwriting recommendation step until a credentialled underwriter approves, and an **eval-event** sampler that scores the supervisor's recommendation packet for lending-decision quality.

## 3. User-facing flows

The user opens the App UI tab and submits an application via the form.

1. The system creates a `MortgageApplication` record in `RECEIVED` and starts a `MortgageWorkflow`.
2. The Supervisor decomposes the application into three parallel work items: an eligibility check, a documentation review, and an underwriting analysis.
3. The workflow forks: EligibilityAgent, DocumentationAgent, and UnderwritingAgent run concurrently. Each returns a typed assessment.
4. CustomerCommsAgent drafts an acknowledgement notification for the applicant.
5. The Supervisor merges the three assessments into a `RecommendationPacket { eligibilityVerdict, documentationVerdict, proposedTerms, complianceStatus, sanitisedSummary }`.
6. A sectoral compliance filter vets the proposed lending terms; if the terms violate product rules, the application enters `COMPLIANCE_HOLD`.
7. If terms pass, the workflow emits a `SignOffRequested` event and the application enters `PENDING_SIGNOFF`. It waits here until a human underwriter calls `POST /api/applications/{id}/decision`.
8. On approval, the application enters `APPROVED`. On rejection, it enters `DECLINED`. CustomerCommsAgent drafts the outcome notification.
9. If any worker times out after 90 seconds, the workflow short-circuits, enters `REVIEW_REQUIRED`, and notifies the underwriter team.

A `ApplicationSimulator` (TimedAction) drips a sample application every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MortgageSupervisor` | `AutonomousAgent` | Decomposes the application into a work plan; merges worker assessments into a `RecommendationPacket`. | `MortgageWorkflow` | returns typed result to workflow |
| `EligibilityAgent` | `AutonomousAgent` | Evaluates income, credit score, and loan-to-value ratio against product criteria. | `MortgageWorkflow` | — |
| `DocumentationAgent` | `AutonomousAgent` | Reviews document completeness and flags authenticity issues. | `MortgageWorkflow` | — |
| `UnderwritingAgent` | `AutonomousAgent` | Models risk exposure; proposes a term structure (rate, amortisation, conditions). | `MortgageWorkflow` | — |
| `CustomerCommsAgent` | `AutonomousAgent` | Drafts applicant-facing notifications at key lifecycle events. | `MortgageWorkflow` | — |
| `MortgageWorkflow` | `Workflow` | Coordinates the parallel fan-out, the compliance filter, the HITL gate, and the final decision. | `MortgageEndpoint`, `ApplicationConsumer` | `ApplicationEntity` |
| `ApplicationEntity` | `EventSourcedEntity` | Holds the application's lifecycle. | `MortgageWorkflow` | `ApplicationView` |
| `ApplicationQueue` | `EventSourcedEntity` | Logs each submitted application for replay/audit. | `MortgageEndpoint`, `ApplicationSimulator` | `ApplicationConsumer` |
| `ApplicationView` | `View` | List-of-applications read model. | `ApplicationEntity` events | `MortgageEndpoint` |
| `ApplicationConsumer` | `Consumer` | Listens to `ApplicationQueue` events and starts a workflow per submission. | `ApplicationQueue` events | `MortgageWorkflow` |
| `ApplicationSimulator` | `TimedAction` | Drips a sample application every 90 s. | scheduler | `ApplicationQueue` |
| `DecisionEvalSampler` | `TimedAction` | Samples one decided application every 5 minutes; emits `DecisionEvalScored`. | scheduler | `ApplicationEntity` |
| `MortgageEndpoint` | `HttpEndpoint` | `/api/applications/*` — submit, get, list, decision, SSE. | — | `ApplicationView`, `ApplicationQueue`, `ApplicationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record ApplicationRequest(
    String applicantRef,       // opaque reference; PII sanitiser maps to real identity
    String loanProduct,        // e.g. "30-year-fixed", "5-1-ARM"
    int    requestedAmountGBP,
    int    propertyValueGBP,
    String employmentStatus,
    int    annualIncomeGBP,
    int    creditScore,
    List<String> documentIds
) {}

record EligibilityAssessment(
    boolean eligible,
    double  ltv,
    double  dti,
    String  verdict,
    Instant assessedAt
) {}

record DocumentationAssessment(
    boolean complete,
    List<String> missingDocuments,
    List<String> authenticityFlags,
    Instant reviewedAt
) {}

record TermStructure(
    double  annualRatePercent,
    int     amortisationMonths,
    String  conditions,
    Instant proposedAt
) {}

record UnderwritingAssessment(
    String       riskBand,      // LOW / MEDIUM / HIGH
    TermStructure proposedTerms,
    String        rationale,
    Instant       assessedAt
) {}

record NotificationDraft(
    String subject,
    String bodyText,
    String channel,             // "email" | "sms"
    Instant draftedAt
) {}

record RecommendationPacket(
    EligibilityAssessment    eligibilityVerdict,
    DocumentationAssessment  documentationVerdict,
    UnderwritingAssessment   underwritingAssessment,
    String                   complianceStatus,  // "PASS" | "HOLD"
    String                   sanitisedSummary,
    Instant                  packetAt
) {}

record MortgageApplication(
    String                        applicationId,
    String                        applicantRef,
    String                        loanProduct,
    int                           requestedAmountGBP,
    ApplicationStatus             status,
    Optional<RecommendationPacket> packet,
    Optional<String>              complianceHoldReason,
    Optional<String>              decisionBy,
    Optional<String>              decisionNotes,
    Optional<Integer>             evalScore,
    Optional<String>              evalRationale,
    Instant                       receivedAt,
    Optional<Instant>             decidedAt
) {}

enum ApplicationStatus {
    RECEIVED, UNDER_REVIEW, PENDING_SIGNOFF,
    APPROVED, DECLINED, COMPLIANCE_HOLD, REVIEW_REQUIRED
}
```

### Events (on `ApplicationEntity`)

`ApplicationReceived`, `ReviewStarted`, `PacketAssembled`, `ComplianceHoldApplied`, `SignOffRequested`, `DecisionRecorded`, `WorkerTimedOut`, `DecisionEvalScored`.

### Events (on `ApplicationQueue`)

`ApplicationSubmitted { applicationId, applicantRef, loanProduct, requestedAmountGBP, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/applications` — body `{ applicantRef, loanProduct, requestedAmountGBP, propertyValueGBP, employmentStatus, annualIncomeGBP, creditScore, documentIds }` → `{ applicationId }`. Starts a workflow.
- `GET /api/applications` — list all applications. Optional `?status=RECEIVED|UNDER_REVIEW|PENDING_SIGNOFF|APPROVED|DECLINED|COMPLIANCE_HOLD|REVIEW_REQUIRED`.
- `GET /api/applications/{id}` — one application row.
- `POST /api/applications/{id}/decision` — body `{ approved: boolean, decidedBy, notes }` → `{ applicationId, status }`. Unblocks the HITL gate.
- `GET /api/applications/sse` — server-sent events stream of every application change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Mortgage Assistant (Multi-Agent)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit an application, live list with status pills, expand-row to see eligibility verdict, documentation assessment, proposed terms, compliance status, and eval score. Approved applications also show a "Make decision" panel for the HITL gate.

Browser title: `<title>Akka Sample: Mortgage Assistant (Multi-Agent)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitiser** (`system-level`, PII flavor): strips applicant identity fields (name, address, NI/SSN) from all agent inputs. The workflow passes only the opaque `applicantRef` to agents. Raw identity maps are held in `ApplicationEntity` and never passed to any LLM call.
- **S2 — sectoral compliance filter** (`before-agent-response` on `MortgageWorkflow`): vets proposed lending terms against a pre-loaded product-rule table (rate ceilings, LTV caps, affordability thresholds). Blocking. Failure → `COMPLIANCE_HOLD`.
- **H1 — HITL application gate** (`human-in-the-loop`, application flavor): the workflow halts at `PENDING_SIGNOFF` and does not advance until `POST /api/applications/{id}/decision` is received. The paused workflow step holds state in `ApplicationEntity`; no polling.
- **E1 — eval-event sampler** (`on-decision-eval`): `DecisionEvalSampler` (TimedAction) picks one decided application every 5 minutes and emits a `DecisionEvalScored` event with a 1–5 quality score and a short rationale.

## 9. Agent prompts

- `MortgageSupervisor` → `prompts/supervisor.md`. Decomposes the application into parallel work items; later merges assessments into a `RecommendationPacket`.
- `EligibilityAgent` → `prompts/eligibility.md`. Evaluates income, credit, and LTV criteria.
- `DocumentationAgent` → `prompts/documentation.md`. Reviews document completeness and authenticity.
- `UnderwritingAgent` → `prompts/underwriting.md`. Models risk and proposes a term structure.
- `CustomerCommsAgent` → `prompts/customer-comms.md`. Drafts applicant notifications.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit an application; it progresses `RECEIVED → UNDER_REVIEW → PENDING_SIGNOFF` within 90 s; UI reflects each transition via SSE.
2. **J2** — Call `POST /api/applications/{id}/decision` with `approved: true`; application enters `APPROVED` and CustomerCommsAgent draft appears.
3. **J3** — Inject a compliance rule violation in the term structure; application enters `COMPLIANCE_HOLD` without reaching the HITL gate.
4. **J4** — Inject a worker timeout (set UnderwritingAgent step to 1 s); application enters `REVIEW_REQUIRED`.
5. **J5** — Wait; a decided application gains an eval score without delivery being blocked.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named mortgage-team demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-mortgage-team.
Java package io.akka.samples.mortgageassistantmultiagent. Akka 3.6.0. HTTP port 9824.

Components to wire (exactly):
- 5 AutonomousAgents:
  * MortgageSupervisor — definition() with
      capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
      AND capability(TaskAcceptance.of(MERGE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/supervisor.md.
    Returns WorkPlan{eligibilityQuery, documentationQuery, underwritingQuery}
    for DECOMPOSE and RecommendationPacket{eligibilityVerdict, documentationVerdict,
    underwritingAssessment, complianceStatus, sanitisedSummary, packetAt} for MERGE.
  * EligibilityAgent — capability(TaskAcceptance.of(CHECK_ELIGIBILITY).maxIterationsPerTask(3)).
    System prompt from prompts/eligibility.md.
    Returns EligibilityAssessment{eligible, ltv, dti, verdict, assessedAt}.
  * DocumentationAgent — capability(TaskAcceptance.of(REVIEW_DOCUMENTS).maxIterationsPerTask(3)).
    System prompt from prompts/documentation.md.
    Returns DocumentationAssessment{complete, missingDocuments, authenticityFlags, reviewedAt}.
  * UnderwritingAgent — capability(TaskAcceptance.of(ASSESS_RISK).maxIterationsPerTask(3)).
    System prompt from prompts/underwriting.md.
    Returns UnderwritingAssessment{riskBand, proposedTerms:TermStructure, rationale, assessedAt}.
  * CustomerCommsAgent — capability(TaskAcceptance.of(DRAFT_COMMS).maxIterationsPerTask(2)).
    System prompt from prompts/customer-comms.md.
    Returns NotificationDraft{subject, bodyText, channel, draftedAt}.

- 1 Workflow MortgageWorkflow with steps:
  decomposeStep -> [parallel] eligibilityStep, documentationStep, underwritingStep ->
  joinStep -> commsAckStep -> mergeStep -> complianceStep ->
  signOffStep (HITL wait) -> outcomeCommsStep -> emitStep.
  decomposeStep calls forAutonomousAgent(MortgageSupervisor.class, DECOMPOSE).
  eligibilityStep, documentationStep, underwritingStep run in parallel (CompletionStage zip);
  governed by WorkflowSettings.builder()
    .stepTimeout(MortgageWorkflow::eligibilityStep, ofSeconds(90))
    .stepTimeout(MortgageWorkflow::documentationStep, ofSeconds(90))
    .stepTimeout(MortgageWorkflow::underwritingStep, ofSeconds(90)).
  On any timeout, transition to reviewRequiredStep that calls ApplicationEntity.workerTimedOut
  and ends with WorkerTimedOut. mergeStep calls forAutonomousAgent(MortgageSupervisor.class,
  MERGE) with the three assessments; give mergeStep a 120s stepTimeout.
  complianceStep is deterministic — checks proposed terms against rule table; on violation,
  calls ApplicationEntity.applyComplianceHold and ends. signOffStep emits SignOffRequested and
  PAUSES; resumes only when ApplicationEntity.recordDecision is called externally. WorkflowSettings
  is nested inside Workflow — no import.

- 1 EventSourcedEntity ApplicationEntity holding state MortgageApplication{applicationId,
  applicantRef, loanProduct, requestedAmountGBP, ApplicationStatus,
  Optional<RecommendationPacket> packet, Optional<String> complianceHoldReason,
  Optional<String> decisionBy, Optional<String> decisionNotes,
  Optional<Integer> evalScore, Optional<String> evalRationale,
  Instant receivedAt, Optional<Instant> decidedAt}.
  ApplicationStatus enum: RECEIVED, UNDER_REVIEW, PENDING_SIGNOFF,
  APPROVED, DECLINED, COMPLIANCE_HOLD, REVIEW_REQUIRED.
  Events: ApplicationReceived, ReviewStarted, PacketAssembled, ComplianceHoldApplied,
  SignOffRequested, DecisionRecorded, WorkerTimedOut, DecisionEvalScored.
  Commands: receiveApplication, startReview, assemblePacket, applyComplianceHold,
  requestSignOff, recordDecision, workerTimedOut, recordEval, getApplication.
  emptyState() returns MortgageApplication.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity ApplicationQueue with command enqueueApplication(applicantRef,
  loanProduct, requestedAmountGBP, ...) emitting
  ApplicationSubmitted{applicationId, applicantRef, loanProduct, requestedAmountGBP, submittedAt}.

- 1 View ApplicationView with row type ApplicationRow (mirrors MortgageApplication minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  ApplicationEntity events. ONE query getAllApplications SELECT * AS applications FROM application_view.
  No WHERE status filter — caller filters client-side.

- 1 Consumer ApplicationConsumer subscribed to ApplicationQueue events; on ApplicationSubmitted
  starts a MortgageWorkflow with the applicationId as the workflow id.

- 2 TimedActions:
  * ApplicationSimulator — every 90s, reads next line from
    src/main/resources/sample-events/mortgage-applications.jsonl and calls
    ApplicationQueue.enqueueApplication.
  * DecisionEvalSampler — every 5 minutes, queries ApplicationView.getAllApplications, picks the
    oldest APPROVED or DECLINED application without an evalScore, runs a 1-5 rubric judge over
    the packet, then calls ApplicationEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * MortgageEndpoint at /api with POST /applications, GET /applications, GET /applications/{id},
    POST /applications/{id}/decision, GET /applications/sse, and the /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- MortgageTasks.java declaring five Task<R> constants: DECOMPOSE (WorkPlan), CHECK_ELIGIBILITY
  (EligibilityAssessment), REVIEW_DOCUMENTS (DocumentationAssessment), ASSESS_RISK
  (UnderwritingAssessment), MERGE (RecommendationPacket), DRAFT_COMMS (NotificationDraft).
- Domain records ApplicationRequest, WorkPlan, EligibilityAssessment, DocumentationAssessment,
  TermStructure, UnderwritingAssessment, NotificationDraft, RecommendationPacket.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9824 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/mortgage-applications.jsonl with 8 canned application lines
  (various loan products, amounts, and credit profiles).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 4 controls (S1 PII sanitiser system-level,
  S2 sector sanitiser before-agent-response, H1 HITL human-in-the-loop application,
  E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector.primary = mortgage-lending,
  decisions.authority_level = final-with-human-gate, data.data_classes.pii = true,
  capabilities.* per the finance-analysis domain; deployer fields TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/supervisor.md, prompts/eligibility.md, prompts/documentation.md,
  prompts/underwriting.md, prompts/customer-comms.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Mortgage Assistant (Multi-Agent)",
  one-line pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills, HITL decision panel, eval score). Browser title exactly:
  <title>Akka Sample: Mortgage Assistant (Multi-Agent)</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via the
  conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in session memory only.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on agent class name.
  Each branch reads a JSON file from src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    supervisor.json — list of WorkPlan entries (eligibilityQuery + documentationQuery +
      underwritingQuery triples, 4-6 entries) AND RecommendationPacket entries (4-6 entries
      with realistic UK mortgage verdicts and complianceStatus = "PASS").
    eligibility.json — 4-6 EligibilityAssessment entries covering eligible and ineligible
      cases with realistic ltv (0.65-0.95) and dti (0.25-0.50) values.
    documentation.json — 4-6 DocumentationAssessment entries; some complete, some with
      missingDocuments lists (["P60", "Bank statement"]).
    underwriting.json — 4-6 UnderwritingAssessment entries with riskBand LOW/MEDIUM/HIGH,
      realistic annualRatePercent (4.5-7.5), amortisationMonths (120-360).
    customer-comms.json — 4-6 NotificationDraft entries for acknowledgement and outcome
      notifications via email channel.
- A MockModelProvider.seedFor(applicationId) helper makes selection deterministic per
  application id across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step calling an agent gets an explicit stepTimeout (90s workers,
  120s merge); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion MortgageTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9824 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- PII sanitiser must operate at the workflow level, before any agent invocation, not inside
  agent prompts.
- HITL signOffStep must use a paused Workflow step pattern, not a polling TimedAction.
- No forbidden words in user-facing text.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
