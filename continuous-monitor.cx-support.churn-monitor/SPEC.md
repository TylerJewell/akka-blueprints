# SPEC — churn-monitor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Customer Churn Prediction Agent (Partner).
**One-line pitch:** A background worker continuously scores customer accounts for churn risk, produces AI-driven retention plans for high-risk accounts, and monitors its own predictions for drift and demographic bias.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with two governance mechanisms layered on top of two AI primitives (`ChurnScoringAgent` and `RetentionAdvisorAgent`). Specifically:

- A **PII sanitizer** runs inside a Consumer between the raw account snapshot event and any LLM call — so the model never sees customer names, emails, phone numbers, or account IDs.
- A **drift + fairness eval** sampler running every 6 hours scores a batch of recently-scored accounts for prediction drift (score distribution shift vs. a calibration window) and demographic bias (score disparity across industry segments or account sizes). The eval runs non-blocking — the live scoring path is not gated on it.

The result is a system where predictions are continuously produced without human approval overhead, while model quality and fairness are independently audited on a regular cadence.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live account risk board: every scored account, its risk level, and (for HIGH-risk accounts) the retention plan produced.
2. `AccountPoller` (TimedAction) ticks every 60 s and inserts new simulated account snapshots into `AccountSnapshotQueue`. (A simulator drips canned account records.)
3. For each new snapshot: `PiiSanitizer` (Consumer) redacts the payload, then `ChurnScoringAgent` produces a `ChurnScore`.
4. If the risk level is `HIGH`, `RetentionAdvisorAgent` produces a `RetentionPlan`. The account lands in `AWAITING_ACTION`.
5. A customer-success manager clicks **Mark Actioned** in the UI — the account transitions to ACTIONED.
6. A manager clicks **Dismiss** with a reason — the account transitions to DISMISSED.
7. `DriftFairnessEvalRunner` (TimedAction) ticks every 6 hours, picks accounts scored since the last eval run, computes drift metrics and fairness metrics, and writes an `EvalCompleted` event.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AccountPoller` | `TimedAction` | Drips simulated account snapshots every 60 s. | scheduler | `AccountSnapshotQueue` |
| `AccountSnapshotQueue` | `EventSourcedEntity` | Append-only audit log of `SnapshotReceived` events. | `AccountPoller`, `ChurnEndpoint` | `PiiSanitizer` |
| `PiiSanitizer` | `Consumer` | Reads `SnapshotReceived` events, applies PII redaction, emits `SnapshotSanitized` via `AccountChurnEntity`. | `AccountSnapshotQueue` events | `AccountChurnEntity` |
| `ChurnScoringAgent` | `Agent` (typed, not autonomous) | Scores a sanitized account snapshot into `ChurnScore` (riskLevel, probability, topSignals). | invoked by Workflow | returns ChurnScore |
| `RetentionAdvisorAgent` | `AutonomousAgent` | Generates a ranked `RetentionPlan` for HIGH-risk accounts. | invoked by Workflow | returns RetentionPlan |
| `ChurnWorkflow` | `Workflow` | Per-account orchestration: score → (if HIGH) advise → await action / auto-close LOW/MEDIUM. | `PiiSanitizer` (one workflow per `SnapshotSanitized`) | `AccountChurnEntity` |
| `AccountChurnEntity` | `EventSourcedEntity` | Lifecycle per account scoring cycle: received → sanitized → scored → (advised) → awaiting action / closed. | `ChurnWorkflow` | `ChurnView` |
| `ChurnView` | `View` | Read-model row per account for the UI. | `AccountChurnEntity` events | `ChurnEndpoint` |
| `DriftFairnessEvalRunner` | `TimedAction` | Every 6 h, samples scored accounts; computes drift + fairness metrics; emits `EvalCompleted`. | scheduler | `AccountChurnEntity` |
| `ChurnEndpoint` | `HttpEndpoint` | `/api/churn/*` — list, get, action, dismiss, SSE. | — | `ChurnView`, `AccountChurnEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record AccountSnapshot(
    String accountId,
    String accountName,
    String contactEmail,
    String contactPhone,
    String industry,
    String accountTier,
    int monthsActive,
    int supportTicketsLast90d,
    double usageRatioLast30d,
    Instant snapshotAt
) {}

record SanitizedSnapshot(
    String redactedAccountName,
    String industry,
    String accountTier,
    int monthsActive,
    int supportTicketsLast90d,
    double usageRatioLast30d,
    List<String> piiCategoriesFound
) {}

record ChurnScore(RiskLevel riskLevel, double probability, List<String> topSignals) {}
enum RiskLevel { HIGH, MEDIUM, LOW }

record RetentionAction(String actionType, String description, int priorityRank) {}
record RetentionPlan(List<RetentionAction> actions, String summary, Instant advisedAt) {}

record ActionDecision(String decidedBy, Optional<String> reason, Instant decidedAt) {}

record AccountChurnState(
    String accountId,
    AccountSnapshot snapshot,
    Optional<SanitizedSnapshot> sanitized,
    Optional<ChurnScore> score,
    Optional<RetentionPlan> retentionPlan,
    Optional<ActionDecision> decision,
    Optional<EvalSummary> latestEval,
    AccountChurnStatus status,
    Instant createdAt,
    Optional<Instant> closedAt
) {}

enum AccountChurnStatus {
    RECEIVED, SANITIZED, SCORED, ADVISED, AWAITING_ACTION, ACTIONED, DISMISSED, LOW_RISK_CLOSED, MEDIUM_RISK_CLOSED
}

record EvalSummary(
    double driftScore,
    String driftVerdict,
    double fairnessScore,
    String fairnessVerdict,
    Instant evaluatedAt
) {}
```

Events on `AccountChurnEntity`: `SnapshotReceived`, `SnapshotSanitized`, `AccountScored`, `RetentionAdvised`, `ActionRecorded`, `AccountDismissed`, `AccountClosed`, `EvalCompleted`.

Events on `AccountSnapshotQueue`: `SnapshotReceived` (the raw pre-sanitization audit record).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/churn` — list all account churn records. Optional `?status=…&riskLevel=…`.
- `GET /api/churn/{accountId}` — one account's current churn state.
- `POST /api/churn/{accountId}/action` — body `{ decidedBy }` → transitions AWAITING_ACTION to ACTIONED.
- `POST /api/churn/{accountId}/dismiss` — body `{ decidedBy, reason }` → transitions to DISMISSED.
- `GET /api/churn/sse` — Server-Sent Events for every account state change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Customer Churn Prediction Agent</title>`.

App UI tab is the most distinctive: it shows the **live account risk board** with a left column listing accounts sorted by risk level (HIGH first), and a right column showing the selected account's score, top signals, retention plan actions, and action/dismiss controls.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied in `PiiSanitizer` Consumer): redacts account names, email addresses, phone numbers, and account IDs from the snapshot before any LLM sees it. Records the categories found for audit.
- **E1 — eval-periodic** (`drift-fairness-watch`, every 6 hours): scores a batch of recently-scored accounts for prediction drift (comparing current score distribution to a calibration window) and demographic fairness (score disparity across industry segments and account size tiers).

## 9. Agent prompts

- `ChurnScoringAgent` → `prompts/churn-scoring.md`. Typed scorer. Always returns one of the three `RiskLevel` values with a probability and a list of up to five signals.
- `RetentionAdvisorAgent` → `prompts/retention-advisor.md`. Produces a `RetentionPlan` with ranked actions. Never invents offers the deployer has not listed as available.
- `DriftFairnessEvalJudge` (used by `DriftFairnessEvalRunner`) → `prompts/drift-fairness-eval.md`. Evaluates a batch of scored accounts for drift and fairness, returns an `EvalSummary`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips an account snapshot; it appears in the UI within 60 s; passes sanitize → score → advise (if HIGH) → AWAITING_ACTION.
2. **J2** — Click Mark Actioned; account transitions to ACTIONED; the decision is logged with the manager's ID.
3. **J3** — Click Dismiss with reason; account transitions to DISMISSED with the reason visible.
4. **J4** — A LOW or MEDIUM risk account auto-closes after scoring (no human action required).
5. **J5** — DriftFairnessEvalRunner runs and an `EvalCompleted` event is visible on the account; the Eval Matrix tab shows the latest drift and fairness verdicts.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named churn-monitor demonstrating the continuous-monitor × cx-support
cell. Runs out of the box (in-memory account source; no real CRM integration). Maven
group io.akka.samples, artifact continuous-monitor-cx-support-churn-monitor. Java package
io.akka.samples.customerchurnpredictionagentpartner. HTTP port 9608.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) ChurnScoringAgent — scorer. System prompt loaded from
  prompts/churn-scoring.md. Input: SanitizedSnapshot{redactedAccountName, industry,
  accountTier, monthsActive, supportTicketsLast90d, usageRatioLast30d,
  piiCategoriesFound: List<String>}. Output: ChurnScore{riskLevel: RiskLevel (enum
  HIGH/MEDIUM/LOW), probability: double 0.0–1.0, topSignals: List<String> ≤ 5}. Defaults
  to HIGH under uncertainty.

- 1 AutonomousAgent RetentionAdvisorAgent — definition() with capability(TaskAcceptance
  .of(ADVISE).maxIterationsPerTask(3)). System prompt from prompts/retention-advisor.md.
  Input: SanitizedSnapshot + ChurnScore. Output: RetentionPlan{actions:
  List<RetentionAction{actionType, description, priorityRank}>, summary: String,
  advisedAt: Instant}. The agent NEVER commits to offers outside those listed in the
  deployer-supplied playbook.

- 1 AutonomousAgent DriftFairnessEvalJudge — definition() with capability(TaskAcceptance
  .of(EVAL).maxIterationsPerTask(2)). System prompt from prompts/drift-fairness-eval.md.
  Input: List<ChurnScoreSample{accountId, riskLevel, probability, industry, accountTier}>.
  Output: EvalSummary{driftScore: double, driftVerdict: String, fairnessScore: double,
  fairnessVerdict: String, evaluatedAt: Instant}.

- 1 Workflow ChurnWorkflow per account snapshot with steps: scoreStep -> conditional branch:
  HIGH -> adviseStep -> awaitActionStep -> finaliseStep; MEDIUM -> mediumCloseStep (emits
  AccountClosed reason="medium-risk-auto-closed"); LOW -> lowCloseStep (emits AccountClosed
  reason="low-risk-auto-closed"). scoreStep wraps ChurnScoringAgent with
  WorkflowSettings.builder().stepTimeout(Duration.ofSeconds(15)). adviseStep wraps
  RetentionAdvisorAgent with stepTimeout 30s. awaitActionStep polls AccountChurnEntity
  .getState every 5s; on decision.isPresent() or dismissal advances. No auto-timeout on
  awaitAction — HIGH risk accounts wait indefinitely until actioned or dismissed.
  finaliseStep emits ActionRecorded or AccountDismissed.

- 2 EventSourcedEntities:
  * AccountSnapshotQueue — append-only audit log of inbound snapshots. Command
    receive(AccountSnapshot) emits SnapshotReceived{snapshot}.
  * AccountChurnEntity (one per accountId) — full lifecycle per scoring cycle. State
    AccountChurnState{accountId, snapshot: AccountSnapshot{accountId, accountName,
    contactEmail, contactPhone, industry, accountTier, monthsActive,
    supportTicketsLast90d, usageRatioLast30d, snapshotAt}, Optional<SanitizedSnapshot>
    sanitized, Optional<ChurnScore> score, Optional<RetentionPlan> retentionPlan,
    Optional<ActionDecision> decision (with decidedBy, Optional<String> reason,
    decidedAt), Optional<EvalSummary> latestEval, AccountChurnStatus status,
    Instant createdAt, Optional<Instant> closedAt}. AccountChurnStatus enum: RECEIVED,
    SANITIZED, SCORED, ADVISED, AWAITING_ACTION, ACTIONED, DISMISSED, LOW_RISK_CLOSED,
    MEDIUM_RISK_CLOSED. Events: SnapshotReceived, SnapshotSanitized, AccountScored,
    RetentionAdvised, ActionRecorded, AccountDismissed, AccountClosed, EvalCompleted.
    Commands: registerSnapshot, attachSanitized, attachScore, attachRetentionPlan,
    recordAction, recordDismissal, closeAccount, recordEval, getState.
    emptyState() returns AccountChurnState.initial("", null) without commandContext()
    reference.

- 1 Consumer PiiSanitizer subscribed to AccountSnapshotQueue events; for each
  SnapshotReceived, applies a regex+heuristic redaction pipeline (emails, phone numbers,
  names, account IDs, street addresses) to accountName, contactEmail, contactPhone,
  builds SanitizedSnapshot with piiCategoriesFound, and calls
  AccountChurnEntity.registerSnapshot followed by attachSanitized. Then starts a
  ChurnWorkflow with accountId as the workflow id.

- 1 View ChurnView with row type AccountChurnRow (mirrors AccountChurnState minus raw
  snapshot personally-identifiable fields). Table updater consumes AccountChurnEntity
  events. ONE query getAllAccounts SELECT * AS accounts FROM churn_view. No WHERE status
  filter — caller filters client-side.

- 2 TimedActions:
  * AccountPoller — every 60s, reads next line from src/main/resources/sample-events/
    account-snapshots.jsonl and calls AccountSnapshotQueue.receive.
  * DriftFairnessEvalRunner — every 6 hours (configurable via EVAL_RUNNER_SECONDS for
    testing), queries ChurnView.getAllAccounts, picks accounts scored since the last eval
    run (up to 50, oldest-first), calls DriftFairnessEvalJudge with their score vectors,
    then calls AccountChurnEntity.recordEval(evalSummary) per account in the batch.

- 2 HttpEndpoints:
  * ChurnEndpoint at /api with GET /churn, GET /churn/{accountId}, POST
    /churn/{accountId}/action (body {decidedBy}), POST /churn/{accountId}/dismiss
    (body {decidedBy, reason}), GET /churn/sse, and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/. Action writes
    ActionRecorded; dismiss writes AccountDismissed.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ChurnTasks.java declaring three Task<R> constants: SCORE (ChurnScore), ADVISE
  (RetentionPlan), EVAL (EvalSummary).
- Domain records AccountSnapshot, SanitizedSnapshot, ChurnScore, RetentionAction,
  RetentionPlan, ActionDecision, EvalSummary, ChurnScoreSample.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9608 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/account-snapshots.jsonl with 10 canned account lines
  covering HIGH (low usage + many tickets), MEDIUM (moderate signals), and LOW (healthy
  usage + minimal tickets) risk profiles. Include two accounts with the same industry to
  let the fairness eval find a signal.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls: S1 sanitizer pii, E1 eval-periodic
  drift-fairness-watch. Matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = automated-with-
  review-for-high-risk, oversight.human_in_loop = true (for HIGH risk only),
  oversight.human_on_loop = true (eval runner monitors all predictions); deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/churn-scoring.md, prompts/retention-advisor.md, prompts/drift-fairness-eval.md
  loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Customer Churn Prediction Agent
  (Partner)", prerequisites, generate-the-system, what-you-get, customise-before-
  generating, what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live account risk board sorted by riskLevel; right = selected account detail with
  score, top signals, retention plan actions, Mark Actioned / Dismiss buttons and, after
  eval, drift and fairness chips). Browser title exactly:
  <title>Akka Sample: Customer Churn Prediction Agent</title>. No subtitle on the
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
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    churn-scoring.json — 10–12 ChurnScore entries spanning HIGH (probability
      0.75–0.95, signals like "usage dropped 40% MoM", "6 tickets in 90d",
      "contract renewal within 30d"), MEDIUM (0.40–0.65, mixed signals), and
      LOW (0.05–0.25, healthy engagement). Each entry has topSignals as a
      list of ≤5 strings. The mock should default HIGH under ambiguity to
      match the prompt's conservative behaviour.
    retention-advisor.json — 5–7 RetentionPlan entries with 2–4 ranked
      RetentionAction items each. actionType drawn from: "executive-outreach",
      "discount-offer", "training-session", "feature-demo", "support-upgrade".
      No invented pricing, no SLA promises. summary is 1–2 sentences.
    drift-fairness-eval.json — 4–6 EvalSummary entries. driftScore 0.0–1.0
      (>0.3 = "drift detected"); fairnessScore 0.0–1.0 (<0.7 = "bias flag").
      Verdicts are one short sentence each.
- A `MockModelProvider.seedFor(accountId)` helper makes per-account
  selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- PiiSanitizer runs INSIDE a Consumer before any LLM call — not inside an Agent's prompt
  and not after the LLM has seen the raw snapshot.
- LOW and MEDIUM risk accounts auto-close without requiring human action. Only HIGH risk
  accounts enter AWAITING_ACTION.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
  Without these, state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26). No "hidden"
  zombie panels in the DOM — delete removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block. Per Lesson 25, /akka:specify handles the key during
  generation.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, marketing tone, competitor brand names.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.env` file written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
