# SPEC — slack-lead-qualifier

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Slack Lead Qualifier.
**One-line pitch:** A background worker watches for new Slack workspace members, enriches and scores each lead, and posts a qualification summary to a designated channel.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms protecting an externally-facing write operation. Specifically:

- A **PII sanitizer** runs inside a Consumer between the raw enriched-lead event and the scoring LLM call — so the model never sees personal identifiers from the enrichment step.
- A **before-tool-call guardrail** on the `postToSlack` tool checks tone, absence of residual PII, and minimum-score threshold before the message leaves the process.

The result is a system where every qualification post has been enriched by one agent, scored by another, and independently cleared by two governance checks before any external write occurs.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live lead queue: every new-member event received, its enrichment status, its score, and (if posted) the Slack summary.
2. `SlackEventPoller` (TimedAction) ticks every 20 s and inserts new simulated `member_joined_channel` events into `SlackEventQueue`. (A request-simulator style — drips canned events.)
3. For each new event: `LeadWorkflow` starts. `LeadEnrichmentAgent` searches the web for company and role context. The raw enriched data lands in `LeadEntity` as `LeadEnriched`.
4. `PiiSanitizer` (Consumer) subscribes to `LeadEnriched` events, strips email addresses, phone numbers, and personal URLs from the enriched profile, emits `LeadSanitized`.
5. `LeadScoringAgent` receives the sanitized profile and returns a `ScoringResult` (score 0–100, tier, rationale).
6. If the score meets the configured threshold, `LeadWorkflow` calls the `postToSlack` tool. The `SlackPostGuardrail` (before-tool-call) inspects the draft post; if it passes, the call fires and `LeadPosted` is emitted.
7. If the score is below the threshold, the lead is marked `SUPPRESSED`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SlackEventPoller` | `TimedAction` | Drips simulated `member_joined_channel` events every 20 s. | scheduler | `SlackEventQueue` |
| `SlackEventQueue` | `EventSourcedEntity` | Append-only log of `MemberJoinedReceived` events. | `SlackEventPoller`, `LeadEndpoint` | `PiiSanitizer` (after enrichment) |
| `LeadEnrichmentAgent` | `Agent` (typed) | Web-search based enrichment: company name, role, LinkedIn URL, estimated headcount, industry. | invoked by `LeadWorkflow` | returns `EnrichmentResult` |
| `LeadScoringAgent` | `Agent` (typed) | Scores the sanitized enriched profile against ICP criteria (0–100) and assigns a tier. | invoked by `LeadWorkflow` | returns `ScoringResult` |
| `PiiSanitizer` | `Consumer` | Subscribes to `LeadEnriched` events; strips PII from the enriched profile; emits `LeadSanitized` via `LeadEntity`. | `LeadEntity` events | `LeadEntity` |
| `LeadWorkflow` | `Workflow` | Per-lead orchestration: enrich → wait for sanitize → score → conditional post/suppress. | one workflow per `MemberJoinedReceived` | `LeadEntity` |
| `LeadEntity` | `EventSourcedEntity` | Lifecycle per lead: received → enriched → sanitized → scored → posted/suppressed. | `LeadWorkflow`, `LeadEndpoint` | `LeadView` |
| `LeadView` | `View` | Read-model row per lead for the UI. | `LeadEntity` events | `LeadEndpoint` |
| `LeadEndpoint` | `HttpEndpoint` | `/api/leads/*` — list, get, SSE. | — | `LeadView`, `LeadEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record MemberJoinedEvent(String eventId, String userId, String displayName,
    String email, String channelId, Instant joinedAt) {}

record EnrichmentResult(String company, String jobTitle, String industry,
    Optional<String> linkedInUrl, Optional<Integer> estimatedHeadcount,
    Optional<String> location, String rawSearchSummary) {}

record SanitizedEnrichment(String company, String jobTitle, String industry,
    Optional<Integer> estimatedHeadcount, List<String> piiCategoriesStripped) {}

record ScoringResult(int score, LeadTier tier, String rationale,
    List<String> matchedCriteria, List<String> missedCriteria) {}
enum LeadTier { HOT, WARM, COLD, DISQUALIFIED }

record SlackPostDraft(String channel, String headline, String body,
    String scoreBadge, Instant draftedAt) {}

record Lead(
    String leadId,
    MemberJoinedEvent event,
    Optional<EnrichmentResult> enrichment,
    Optional<SanitizedEnrichment> sanitized,
    Optional<ScoringResult> scoring,
    Optional<SlackPostDraft> draft,
    Optional<String> slackTimestamp,
    LeadStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum LeadStatus {
    RECEIVED, ENRICHED, SANITIZED, SCORED, POSTING, POSTED, SUPPRESSED, FAILED
}
```

Events on `LeadEntity`: `MemberJoinedReceived`, `LeadEnriched`, `LeadSanitized`, `LeadScored`, `PostDrafted`, `LeadPosted`, `LeadSuppressed`, `LeadFailed`.

Events on `SlackEventQueue`: `MemberJoinedReceived` (the audit-log copy).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/leads` — list all leads. Optional `?status=…&tier=…`.
- `GET /api/leads/{id}` — one lead.
- `GET /api/leads/sse` — Server-Sent Events for every lead state change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Slack Lead Qualifier</title>`.

App UI tab shows the **live lead queue** sorted newest-first, with tier chips (HOT/WARM/COLD/DISQUALIFIED) and score badges. The right detail pane shows enrichment data, the sanitized profile diff (fields stripped), scoring rationale, and the Slack post draft or suppression reason.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): strips email addresses, phone numbers, personal URLs, and location tokens from the enriched lead profile before the scoring agent sees it. Records the categories stripped.
- **G1 — before-tool-call guardrail** on the `postToSlack` tool: checks that (a) score meets the configured threshold, (b) the draft post body contains no residual PII patterns, and (c) the tone classifier does not flag the post as aggressive or misleading. Rejects the call with a structured error on any violation.

## 9. Agent prompts

- `LeadEnrichmentAgent` → `prompts/lead-enrichment.md`. Typed enricher. Given a display name and channel context, returns structured company/role data via web search.
- `LeadScoringAgent` → `prompts/lead-scoring.md`. Typed scorer. Given a sanitized enrichment profile, returns a 0–100 score, a tier, a rationale, and matched/missed criteria.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a member event; within 20 s the lead appears in the UI; passes enrich → sanitize → score → POSTED for a high-scoring lead.
2. **J2** — A low-score lead is marked SUPPRESSED; no Slack post fires.
3. **J3** — PII from the enrichment phase does not appear in the scoring LLM call (audit check via debug log).
4. **J4** — A guardrail violation (residual PII injected into draft post body) blocks the Slack post and emits `LeadFailed` with the violation reason.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named slack-lead-qualifier demonstrating the continuous-monitor × sales-marketing
cell. Runs out of the box (in-memory Slack event source; no real Slack integration). Maven group
io.akka.samples, artifact continuous-monitor-sales-marketing-slack-lead-qualifier.
Java package io.akka.samples.slackleadqualifier. HTTP port 9931.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) LeadEnrichmentAgent — enricher. System prompt loaded from
  prompts/lead-enrichment.md. Input: EnrichmentRequest{userId: String, displayName: String,
  channelId: String}. Output: EnrichmentResult{company, jobTitle, industry,
  linkedInUrl: Optional<String>, estimatedHeadcount: Optional<Integer>,
  location: Optional<String>, rawSearchSummary: String}. Defaults to
  EnrichmentResult.unknown() (all fields populated with sentinel strings) on any
  model error — do NOT throw.

- 1 Agent (typed, NOT autonomous) LeadScoringAgent — scorer. System prompt loaded from
  prompts/lead-scoring.md. Input: SanitizedEnrichment{company, jobTitle, industry,
  estimatedHeadcount: Optional<Integer>, piiCategoriesStripped: List<String>}.
  Output: ScoringResult{score: Integer 0–100, tier: LeadTier (HOT/WARM/COLD/DISQUALIFIED),
  rationale: String, matchedCriteria: List<String>, missedCriteria: List<String>}.
  Defaults to tier COLD, score 0, rationale "enrichment-unavailable" when input is empty.

- 1 Workflow LeadWorkflow per lead with steps:
    enrichStep → waitSanitizedStep → scoreStep → conditionalPostStep.
  enrichStep wraps LeadEnrichmentAgent call with stepTimeout(Duration.ofSeconds(30)).
  waitSanitizedStep polls LeadEntity.getLead every 3 s until status == SANITIZED or
    FAILED; on FAILED, ends with LeadFailed.
  scoreStep wraps LeadScoringAgent call with stepTimeout(Duration.ofSeconds(20)).
  conditionalPostStep: if score >= POSTING_THRESHOLD (default 60), builds a SlackPostDraft
    and calls postToSlack tool (simulated). The SlackPostGuardrail before-tool-call hook
    runs first; on violation, emits LeadFailed. On success, emits LeadPosted.
    If score < POSTING_THRESHOLD, emits LeadSuppressed with reason
    "score-below-threshold:${score}".

- 2 EventSourcedEntities:
  * SlackEventQueue — append-only audit log. Command receive(MemberJoinedEvent) emits
    MemberJoinedReceived{event}. No state beyond the journal.
  * LeadEntity (one per leadId) — full per-lead lifecycle. State Lead{leadId,
    event: MemberJoinedEvent, Optional<EnrichmentResult> enrichment,
    Optional<SanitizedEnrichment> sanitized, Optional<ScoringResult> scoring,
    Optional<SlackPostDraft> draft, Optional<String> slackTimestamp,
    LeadStatus status, Instant createdAt, Optional<Instant> finishedAt}.
    LeadStatus enum: RECEIVED, ENRICHED, SANITIZED, SCORED, POSTING, POSTED, SUPPRESSED, FAILED.
    Events: MemberJoinedReceived, LeadEnriched, LeadSanitized, LeadScored, PostDrafted,
    LeadPosted, LeadSuppressed, LeadFailed.
    Commands: registerMember, attachEnrichment, attachSanitized, attachScoring,
    attachDraft, markPosted, markSuppressed, markFailed, getLead.
    emptyState() returns Lead.initial("") without commandContext() reference.

- 1 Consumer PiiSanitizer subscribed to LeadEntity events; for each LeadEnriched event,
  strips email addresses (RFC 5322 pattern), phone numbers, personal URL tokens, and
  geographic sub-city location strings from the EnrichmentResult, builds
  SanitizedEnrichment with piiCategoriesStripped list, and calls
  LeadEntity.attachSanitized. If EnrichmentResult.company is the sentinel "unknown",
  emits LeadFailed instead.

- 1 View LeadView with row type LeadRow (mirrors Lead minus the raw MemberJoinedEvent
  email field). Table updater consumes LeadEntity events. ONE query getAllLeads
  SELECT * AS leads FROM lead_view. No WHERE filter — caller filters client-side.

- 1 TimedAction SlackEventPoller — every 20 s, reads next line from
  src/main/resources/sample-events/member-joined-events.jsonl and calls
  SlackEventQueue.receive.

- 2 HttpEndpoints:
  * LeadEndpoint at /api with GET /leads (optional ?status=…&tier=…),
    GET /leads/{id}, GET /leads/sse, and three /api/metadata/* endpoints serving
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- Domain records MemberJoinedEvent, EnrichmentResult, SanitizedEnrichment, ScoringResult,
  SlackPostDraft, Lead.
- LeadTasks.java declaring two Task<R> constants: ENRICH (EnrichmentResult),
  SCORE (ScoringResult).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9931 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading canonical env vars.
  Add akka.javasdk.lead-qualifier.posting-threshold = 60 as a configurable value.
- src/main/resources/sample-events/member-joined-events.jsonl with 8 canned events:
  4 that should score HOT/WARM (engineering managers, VPs at mid-size companies),
  2 COLD (students, freelancers), 1 DISQUALIFIED (clearly non-ICP), 1 with
  PII-rich enrichment data to exercise the sanitizer.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii,
  G1 guardrail before-tool-call. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with sector=sales-marketing, data.data_classes.pii=true,
  pii_handled_by_sanitizer_before_llm=true; deployer fields TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/lead-enrichment.md, prompts/lead-scoring.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Slack Lead Qualifier", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column
  layout (left = live lead list with tier chips and score badges; right = selected lead
  detail showing enrichment, sanitized diff, scoring rationale, and Slack post draft or
  suppression reason). Browser title exactly:
  <title>Akka Sample: Slack Lead Qualifier</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java per-agent dispatch. Per-agent mock-response shapes:
    lead-enrichment.json — 8 EnrichmentResult entries: 3 with full data (VP Eng at
      Series B SaaS, Director of IT at enterprise, CTO at startup), 2 partial
      (student at university, freelancer), 2 unknown-sentinel entries, 1 with PII-rich
      rawSearchSummary (email address + phone in the summary text).
    lead-scoring.json — 6 ScoringResult entries spanning HOT (score 85), WARM (score 65),
      COLD (score 35), DISQUALIFIED (score 5), with rationale and matched/missed criteria.
- MockModelProvider.seedFor(leadId) makes selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
Lesson numbers referenced: 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26.
Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- Both agents are typed Agents (NOT AutonomousAgents) — never silently upgrade them.
- PiiSanitizer runs INSIDE a Consumer after LeadEnriched event, BEFORE scoring LLM call.
- The postToSlack tool is a simulated in-process stub; no real Slack HTTP call in dev.
- The SlackPostGuardrail runs as a before-tool-call hook on the postToSlack tool; it
  checks: score >= POSTING_THRESHOLD (reads from config); draft body passes a regex
  scan for email/phone patterns; tone classifier (simple keyword list: "guaranteed",
  "immediately buy", "don't miss") is negative. Rejects with structured error on violation.
- The generated static-resources/index.html must include the mermaid CSS overrides AND
  theme variables from Lesson 24 (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Tab switching in static-resources/index.html MUST match by data-tab / data-panel
  attribute, NEVER by NodeList index (Lesson 26).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export block.
- No forbidden words in user-facing text.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
