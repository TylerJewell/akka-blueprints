# SPEC — job-posting-eeo

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Job Posting Team.
**One-line pitch:** Type a company and a role; a posting lead delegates culture analysis and role analysis to two specialist agents in parallel, then synthesises a polished job posting that is screened for discriminatory language, sanitized of protected-class preferences, and cleared against a hiring-policy documentation gate.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their typed outputs, and asks a supervisor AutonomousAgent to synthesise the final posting. Because a job posting is regulated speech, the blueprint embeds three governance mechanisms: a **before-agent-response guardrail** that screens the draft for discriminatory language, a **sanitizer** that strips encoded protected-class preferences, and a **documentation gate** (ci-gate) that verifies the required hiring-policy statements are present before a posting is cleared for publication.

## 3. User-facing flows

The user opens the App UI tab and submits a request via the form (company name + role title).

1. The system creates a `JobPosting` record in `PLANNING` and starts a `JobPostingWorkflow`.
2. `PostingLead` decomposes the request into two parallel work items: a culture-analysis brief for `CultureAnalyst`, a role-and-market brief for `RoleAnalyst`.
3. The workflow forks: both analysts run concurrently. Each returns a typed payload.
4. `PostingLead` merges the two payloads into a `DraftPosting { title, body, eeoStatement }`. A before-agent-response guardrail screens the draft for discriminatory language; on failure the posting enters `BLOCKED`. Otherwise it enters `DRAFTED`.
5. A sanitizer scans the draft for protected-class preference terms, removes them, records the removed terms, and the posting enters `SANITIZED`.
6. A documentation gate verifies the sanitized posting carries the required EEO statement; the posting enters `CLEARED`, ready for publication.
7. If either analyst times out after 60 seconds, the workflow short-circuits: `PostingLead` synthesises from whichever side returned and the posting enters `DEGRADED`.

A `RequestSimulator` (TimedAction) drips a sample request every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PostingLead` | `AutonomousAgent` | Decomposes the request, synthesises the merged posting, runs the EEO guardrail. | `JobPostingWorkflow` | returns typed result to workflow |
| `CultureAnalyst` | `AutonomousAgent` | Analyzes the hiring company's values and tone. | `JobPostingWorkflow` | — |
| `RoleAnalyst` | `AutonomousAgent` | Analyzes role requirements; backed by a seeded market-research tool. | `JobPostingWorkflow` | — |
| `JobPostingWorkflow` | `Workflow` | Coordinates the parallel fan-out, the synthesis, the guardrail, the sanitizer, the documentation gate. | `JobPostingEndpoint`, `PostingRequestConsumer` | `JobPostingEntity` |
| `JobPostingEntity` | `EventSourcedEntity` | Holds the posting lifecycle (planning → analyzing → drafted → sanitized → cleared / blocked / degraded). | `JobPostingWorkflow` | `JobPostingView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted request for replay/audit. | `JobPostingEndpoint`, `RequestSimulator` | `PostingRequestConsumer` |
| `JobPostingView` | `View` | List-of-postings read model. | `JobPostingEntity` events | `JobPostingEndpoint` |
| `PostingRequestConsumer` | `Consumer` | Listens to `RequestQueue` events and starts a workflow per submission. | `RequestQueue` events | `JobPostingWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample request every 60 s. | scheduler | `RequestQueue` |
| `JobPostingEndpoint` | `HttpEndpoint` | `/api/postings/*` — submit, get, list, SSE. | — | `JobPostingView`, `RequestQueue`, `JobPostingEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record PostingRequest(String company, String roleTitle, String requestedBy) {}

record CultureProfile(List<String> values, String tone, String summary, Instant analyzedAt) {}

record RoleSpec(List<String> responsibilities, List<String> qualifications,
                String marketSalaryRange, Instant analyzedAt) {}

record DraftPosting(String title, String body, String eeoStatement,
                    String guardrailVerdict, Instant draftedAt) {}

record SanitizedPosting(String title, String body, List<String> removedTerms,
                        Instant sanitizedAt) {}

record JobPosting(
    String postingId,
    String company,
    String roleTitle,
    PostingStatus status,
    Optional<CultureProfile> culture,
    Optional<RoleSpec> role,
    Optional<DraftPosting> draft,
    Optional<SanitizedPosting> sanitized,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum PostingStatus { PLANNING, ANALYZING, DRAFTED, SANITIZED, CLEARED, BLOCKED, DEGRADED }
```

### Events (on `JobPostingEntity`)

`PostingCreated`, `CultureAttached`, `RoleAttached`, `PostingDrafted`, `PostingBlocked`, `PostingDegraded`, `PostingSanitized`, `PostingCleared`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/postings` — body `{ company, roleTitle, requestedBy? }` → `{ postingId }`. Starts a workflow.
- `GET /api/postings` — list all postings. Optional `?status=PLANNING|ANALYZING|DRAFTED|SANITIZED|CLEARED|BLOCKED|DEGRADED`.
- `GET /api/postings/{id}` — one posting.
- `GET /api/postings/sse` — server-sent events stream of every posting change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Job Posting Team"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets.
- **Risk Survey** — sections mirroring the questionnaire, with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; each ID badge carries a colored mechanism pill (guardrail red, sanitizer green, ci-gate pale yellow).
- **App UI** — form to submit company + role, live list of postings with status pills, expand-row to see culture profile + role spec + draft body + sanitizer removed-terms + EEO statement.

Browser title: `<title>Akka Sample: Job Posting Team</title>`. Tab switching is attribute-based per Lesson 26; mermaid diagrams carry the Lesson 24 CSS overrides.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — EEO output guardrail** (`before-agent-response` on `PostingLead`): screens the synthesised draft for discriminatory language and protected-class targeting. Blocking. Failure → `BLOCKED`, `PostingBlocked` event.
- **S1 — protected-class sanitizer** (`special-category`): scans the draft body for encoded protected-class preference terms, removes them, and records the removed terms. Blocking. Runs in `sanitizeStep` → `SANITIZED`.
- **C1 — hiring-policy documentation gate** (`ci-gate`, `documentation-gate`): verifies the posting carries the required EEO statement and policy attestation before it is cleared for publication; the CI test-gate also fails the build if the EEO-statement template or sanitizer term-list is missing. Build-gate; runtime analog in `complianceStep` → `CLEARED`.

## 9. Agent prompts

- `PostingLead` → `prompts/posting-lead.md`. Decomposes the request into work items; later synthesises the draft posting and emits the EEO guardrail verdict.
- `CultureAnalyst` → `prompts/culture-analyst.md`. Analyzes company culture; returns `CultureProfile`.
- `RoleAnalyst` → `prompts/role-analyst.md`. Analyzes role requirements with a seeded market tool; returns `RoleSpec`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a request; posting progresses `PLANNING` → `ANALYZING` → `DRAFTED` → `SANITIZED` → `CLEARED` within 60 s; UI reflects each transition via SSE.
2. **J2** — A draft containing discriminatory language is caught by the guardrail; posting enters `BLOCKED` with a verdict reason.
3. **J3** — A draft carrying a protected-class preference term has it stripped by the sanitizer; `removedTerms` is non-empty and the posting reaches `SANITIZED`.
4. **J4** — A worker timeout drives the posting to `DEGRADED` with whichever partial output came back.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named job-posting-eeo demonstrating the
delegation-supervisor-workers × hr-recruiting cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact job-posting-eeo.
Java package io.akka.samples.jobpostingeeo. Akka 3.6.0. HTTP port 9795.

Components to wire (exactly):
- 3 AutonomousAgents:
  * PostingLead — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(DRAFT).maxIterationsPerTask(3)). System prompt loaded from
    prompts/posting-lead.md. Returns WorkPlan{cultureBrief, roleBrief} for DECOMPOSE and
    DraftPosting{title, body, eeoStatement, guardrailVerdict, draftedAt} for DRAFT.
  * CultureAnalyst — capability(TaskAcceptance.of(ANALYZE_CULTURE).maxIterationsPerTask(2)). System
    prompt from prompts/culture-analyst.md. Returns CultureProfile{values: List<String>, tone,
    summary, analyzedAt}.
  * RoleAnalyst — capability(TaskAcceptance.of(ANALYZE_ROLE).maxIterationsPerTask(3)). System prompt
    from prompts/role-analyst.md. Backed by a seeded in-process market-research tool returning canned
    salary ranges. Returns RoleSpec{responsibilities: List<String>, qualifications: List<String>,
    marketSalaryRange, analyzedAt}.

- 1 Workflow JobPostingWorkflow with steps:
  planStep -> [parallel] cultureStep, roleStep -> joinStep -> draftStep -> sanitizeStep ->
  complianceStep -> emitStep.
  planStep calls forAutonomousAgent(PostingLead.class, DECOMPOSE). cultureStep and roleStep run in
  parallel (CompletionStage zip); each wrapped in WorkflowSettings.builder().stepTimeout(
  Duration.ofSeconds(60)). On either timeout, transition to degradeStep that calls draftStep with
  whichever side returned, then ends with PostingDegraded. draftStep calls
  forAutonomousAgent(PostingLead.class, DRAFT) with the merged inputs, then runs the before-agent-
  response EEO guardrail (deterministic discriminatory-term matcher + LLM judge, 5s timeout); on
  failure ends with PostingBlocked. sanitizeStep runs the protected-class sanitizer over the draft
  body, records removedTerms, emits PostingSanitized. complianceStep verifies the EEO statement is
  present and emits PostingCleared. Override settings() with stepTimeout(60s) on cultureStep,
  roleStep, and draftStep, plus defaultStepRecovery(maxRetries(2).failoverTo(error)).

- 1 EventSourcedEntity JobPostingEntity holding state JobPosting{postingId, company, roleTitle,
  PostingStatus, Optional<CultureProfile> culture, Optional<RoleSpec> role,
  Optional<DraftPosting> draft, Optional<SanitizedPosting> sanitized, Optional<String> failureReason,
  Instant createdAt, Optional<Instant> finishedAt}. PostingStatus enum: PLANNING, ANALYZING, DRAFTED,
  SANITIZED, CLEARED, BLOCKED, DEGRADED. Events: PostingCreated, CultureAttached, RoleAttached,
  PostingDrafted, PostingBlocked, PostingDegraded, PostingSanitized, PostingCleared. Commands:
  createPosting, attachCulture, attachRole, recordDraft, block, degrade, recordSanitized, clear,
  getPosting. emptyState() returns JobPosting.initial("", "", "") with no commandContext() reference.

- 1 EventSourcedEntity RequestQueue with command enqueueRequest(company, roleTitle, requestedBy)
  emitting RequestSubmitted{postingId, company, roleTitle, requestedBy, submittedAt}.

- 1 View JobPostingView with row type JobPostingRow (mirrors JobPosting minus heavy nested payloads;
  every nullable lifecycle field declared Optional<T>). Table updater consumes JobPostingEntity
  events. ONE query getAllPostings SELECT * AS postings FROM posting_view. No WHERE status filter
  (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer PostingRequestConsumer subscribed to RequestQueue events; on RequestSubmitted starts a
  JobPostingWorkflow with the postingId as the workflow id.

- 1 TimedAction RequestSimulator — every 60s, reads next line from
  src/main/resources/sample-events/job-requests.jsonl and calls RequestQueue.enqueueRequest.

- 2 HttpEndpoints:
  * JobPostingEndpoint at /api with POST /postings, GET /postings (filter client-side from
    getAllPostings), GET /postings/{id}, GET /postings/sse, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- JobPostingTasks.java declaring four Task<R> constants: DECOMPOSE (WorkPlan), ANALYZE_CULTURE
  (CultureProfile), ANALYZE_ROLE (RoleSpec), DRAFT (DraftPosting).
- Domain records WorkPlan{cultureBrief, roleBrief}, CultureProfile, RoleSpec, DraftPosting,
  SanitizedPosting.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9795 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/job-requests.jsonl with 8 canned {company, roleTitle} lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the root-level
  files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 guardrail before-agent-response, S1
  sanitizer special-category, C1 ci-gate documentation-gate) and a matching simplified_view list.
- risk-survey.yaml at the project root, pre-filling purpose, data, decisions, failure, oversight,
  operations, compliance; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/posting-lead.md, prompts/culture-analyst.md, prompts/role-analyst.md loaded at agent
  startup as system prompts.
- README.md at the project root: title "Akka Sample: Job Posting Team", one-line pitch,
  prerequisites, generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/ folder,
  no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand component
  table), Risk Survey (sections from the questionnaire with answers from risk-survey.yaml; unanswered
  .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table with
  click-to-expand rows; colored mechanism pill in the ID column), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Job Posting Team</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default application.conf's
  model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via the conversational
  surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-correct
        outputs per agent (see Mock LLM provider block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build forwards
        the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://; recorded in
        .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed to the JVM via the
        MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in application.conf, no
  secrets.yaml, no .akka/ file with key material. Akka records only the REFERENCE (env-var name, file
  path, secrets URI); the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key reference if
  it does not resolve at runtime. The error message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing the ModelProvider
  interface with a per-agent dispatch on the agent class name (or the Task<R> id). Each agent's branch
  reads a JSON file from src/main/resources/mock-responses/<agent-name>.json (posting-lead.json,
  culture-analyst.json, role-analyst.json), picks one entry pseudo-randomly per call, and deserialises
  it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    posting-lead.json — list of either WorkPlan or DraftPosting objects. 4–6 WorkPlan entries
      (cultureBrief + roleBrief pairs across plausible companies/roles) and 4–6 DraftPosting entries
      (each with a title, a 120–200 word body, an eeoStatement equal to the standard equal-opportunity
      clause, guardrailVerdict = "ok").
    culture-analyst.json — 4–6 CultureProfile entries, each with 3–5 values, a tone word, and a
      2–3 sentence summary.
    role-analyst.json — 4–6 RoleSpec entries, each with 4–6 responsibilities, 4–6 qualifications, and
      a marketSalaryRange string (e.g., "$110k–$140k").
- A MockModelProvider.seedFor(postingId) helper makes the selection deterministic per posting id so
  the same posting in dev produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md for the
full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- View has no WHERE filter on the enum status column; filter client-side (Lesson 2).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- Workflow steps calling agents set explicit stepTimeout (Lesson 4).
- emptyState() never calls commandContext() (Lesson 7 / exemplar §3).
- AutonomousAgent never silently downgraded to Agent; each requires a companion Tasks file (Lessons
  1, 7).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit dev-mode http-port 9795 in application.conf (Lesson 10).
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- Source-platform metadata is corpus-internal — NEVER user-facing (Lesson 11).
- UI fits the 1080px content column with no horizontal scroll (Lesson 12).
- Integration label is "Runs out of the box" — never T1/T2/T3/T4 or "deferred" (Lessons 13, 23).
- API keys never written to disk; only the reference recorded (Lesson 25).
- The generated static-resources/index.html includes the mermaid CSS overrides AND theme variables
  from Lesson 24 (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- Tab switching matches by data-tab / data-panel attribute, NEVER by NodeList index (Lesson 26). No
  hidden zombie panels in the DOM — delete removed tabs, do not display:none them.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, "Akka SDK" in narrative,
  T1/T2/T3/T4, deferred, use, use, marketing tone, competitor brand names.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
