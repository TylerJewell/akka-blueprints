# Fair CV Matcher

The entire file is the input to `/akka:specify @SPEC.md`. Sections 1–12 together are the brief.

## 1. System name + one-line pitch

**Fair CV Matcher** (short name: `Fair CV Matcher`). A user submits a CV and a set of job postings; the system extracts a structured profile, strips protected-class signals, scores fit for each role, and presents the ranked matches to a human reviewer who monitors the outcome.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern: a workflow whose steps run in fixed order — extract, sanitize, match, review — each step's output feeding the next. The governance pattern combines a **sanitizer** that redacts special-category signals before scoring, a **periodic fairness eval** that watches selection-rate parity across demographic slices, and a **human-on-the-loop** monitoring gate appropriate to a high-risk hiring use case. Employment screening is a high-risk use under the EU AI Act, so the controls carry regulation anchors.

## 3. User-facing flows

1. A user submits raw CV text plus a list of job postings through the App UI or `POST /api/candidates`. The response carries the `candidateId`. The candidate appears in `SUBMITTED` state.
2. The workflow runs `ExtractionAgent`, producing a structured `CvProfile`. The candidate moves to `EXTRACTED`.
3. The sanitize step redacts protected-class signals (name, photo reference, address, age, gender markers) from the profile before any scoring. The candidate moves to `SANITIZED`.
4. The workflow runs `MatchingAgent` over the sanitized profile against each posting, producing a scored, ranked list of `MatchResult`. The candidate moves to `MATCHED`.
5. A human reviewer opens the candidate in the App UI, reads the matches and the sanitization summary, and records oversight (`POST /api/candidates/{id}/review`). The candidate moves to `REVIEWED`.
6. Independently, `FairnessMonitor` periodically computes selection-rate parity across declared slices and flags drift.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ExtractionAgent | AutonomousAgent | Extracts `CvProfile` from raw CV text | MatchingWorkflow | MatchingWorkflow |
| MatchingAgent | AutonomousAgent | Scores sanitized profile against each posting | MatchingWorkflow | MatchingWorkflow |
| MatchingWorkflow | Workflow | Sequential extract → sanitize → match → review | CvIntakeConsumer | CandidateEntity |
| CandidateEntity | EventSourcedEntity | Candidate lifecycle + matches | MatchingWorkflow, MatchingEndpoint | MatchesView |
| CvIntakeQueue | EventSourcedEntity | Records each inbound submission | MatchingEndpoint, CvSimulator | CvIntakeConsumer |
| MatchesView | View | Read model of candidates + matches | CandidateEntity | MatchingEndpoint |
| CvIntakeConsumer | Consumer | Starts a workflow per submission | CvIntakeQueue | MatchingWorkflow |
| CvSimulator | TimedAction | Drips sample CVs every 30s | sample-data | CvIntakeQueue |
| FairnessMonitor | TimedAction | Periodic selection-rate parity check | MatchesView | CandidateEntity |
| MatchingEndpoint | HttpEndpoint | `/api` submit, review, list, SSE, metadata | UI / clients | CandidateEntity, MatchesView |
| AppEndpoint | HttpEndpoint | Serves the embedded UI | browser | static-resources |

## 5. Data model

Authoritative records (see `reference/data-model.md` for full field tables). Every nullable lifecycle field is `Optional<T>` (Lesson 6).

- `CvProfile(int yearsExperience, List<String> skills, String education, String currentTitle)` — extraction output.
- `RedactedSignal(String field, String category)` — one stripped special-category signal.
- `MatchResult(String jobId, String jobTitle, int score, String rationale)` — one scored posting.
- `Candidate` — entity state and view row: `id`, `status` (enum), `submittedAt`, plus `Optional` fields `extractedAt`, `sanitizedAt`, `matchedAt`, `reviewedAt`, `reviewedBy`, `reviewNote`, `profile`, `slice`, and lists `redactions`, `matches`.
- Status enum `CandidateStatus { SUBMITTED, EXTRACTED, SANITIZED, MATCHED, REVIEWED }`.
- Events: `CvSubmitted`, `ProfileExtracted`, `ProfileSanitized`, `MatchesScored`, `ReviewRecorded`.

## 6. API contract

Inline surface (full schemas in `reference/api-contract.md`):

```
POST /api/candidates                 -> { candidateId }
POST /api/candidates/{id}/review     -> 200 | 404
GET  /api/candidates ?status=...     -> { candidates: [Candidate, ...] }
GET  /api/candidates/{id}            -> Candidate
GET  /api/candidates/sse             -> Server-Sent Events of Candidate
GET  /api/fairness                   -> { slices: [SliceStat, ...] }
GET  /api/metadata/eval-matrix       -> text/yaml
GET  /api/metadata/risk-survey       -> text/yaml
GET  /api/metadata/readme            -> text/markdown
GET  /                               -> 302 /app/index.html
GET  /app/{*path}                    -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/`, no npm). Browser title `<title>Akka Sample: Fair CV Matcher</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` → `data-panel` attribute, never NodeList index (Lesson 26); removed tabs are deleted from the DOM, not hidden. Mermaid diagrams on the Architecture tab carry the state-label colour and edge-label `overflow:visible` CSS overrides (Lesson 24). The App UI tab submits a CV + postings, streams candidates over SSE, shows the redaction summary and ranked matches, and exposes a Review action on `MATCHED` candidates. See `reference/ui-mockup.md`.

## 8. Governance

Controls are in `eval-matrix.yaml`; deployer survey in `risk-survey.yaml`. Three mechanisms wired:

- **Sanitizer (S1)** — the workflow's sanitize step strips name, photo reference, address, age, and gender markers from `CvProfile`, recording each as a `RedactedSignal`, before `MatchingAgent` runs. The matching agent never receives raw special-category data.
- **Periodic fairness eval (E1)** — `FairnessMonitor` runs every 60s, computes selection rate per declared slice from `MatchesView`, and flags any slice whose rate drifts below an 0.8 parity ratio.
- **Human-on-the-loop (H1)** — matched candidates are released for a reviewer to monitor; `POST /api/candidates/{id}/review` records oversight without blocking the pipeline.

## 9. Agent prompts

- `prompts/extraction-agent.md` — extracts a structured profile from raw CV text.
- `prompts/matching-agent.md` — scores a sanitized profile against each job posting.

## 10. Acceptance

Full journeys in `reference/user-journeys.md`. The four that define correct generation:

1. **Submit and match.** Submitting a CV + postings yields a candidate that reaches `MATCHED` with a ranked `matches` list within ~30s.
2. **Sanitization holds.** The matched candidate's `redactions` list is non-empty and the matching agent's input carried no name/address/age tokens.
3. **Reviewer oversight.** Posting a review on a `MATCHED` candidate moves it to `REVIEWED` with `reviewedBy` set.
4. **Fairness flag.** When sample data skews one slice, `GET /api/fairness` reports that slice below the parity ratio.

## 11. Implementation directives

```
Create a sample named cv-matcher-fair demonstrating the sequential-pipeline ×
hr-recruiting cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact cv-matcher-fair. Java package
io.akka.samples.cvmatcherfair. Akka 3.6.0. HTTP port 9736.

Matrix cell: coordination_pattern = sequential-pipeline, domain = hr-recruiting.
Integration form: runs out of the box (pure-process simulation).

Components to wire (exactly):
- 2 AutonomousAgents: ExtractionAgent (returns typed CvProfile{yearsExperience,
  skills,education,currentTitle}) and MatchingAgent (returns typed
  List<MatchResult>{jobId,jobTitle,score,rationale}). Each declares definition()
  with capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Never
  silently downgrade AutonomousAgent to Agent (Lesson 1).
- 1 companion CvMatcherTasks.java declaring Task<R> constants: EXTRACT
  (resultConformsTo CvProfile), MATCH (resultConformsTo MatchList wrapper).
  Required — omitting it is a compile error (Lesson 7).
- 1 Workflow MatchingWorkflow with steps extractStep -> sanitizeStep ->
  matchStep -> reviewStep. extractStep calls
  forAutonomousAgent(ExtractionAgent.class,...).runSingleTask(...) then
  forTask(taskId).result(...) and writes ProfileExtracted. sanitizeStep runs a
  deterministic Redactor over CvProfile, records RedactedSignal entries, writes
  ProfileSanitized. matchStep calls MatchingAgent over the sanitized profile and
  postings, writes MatchesScored. reviewStep marks the candidate ready for
  human-on-the-loop review and ends (non-blocking; the reviewer acts via REST).
  Override settings() with stepTimeout(60s) on extractStep and matchStep and a
  defaultStepRecovery failoverTo an error step (Lesson 4). WorkflowSettings is
  nested in Workflow — no import (Lesson 5).
- 1 EventSourcedEntity CandidateEntity holding a Candidate record: id,
  CandidateStatus enum, submittedAt, and Optional lifecycle fields extractedAt,
  sanitizedAt, matchedAt, reviewedAt, reviewedBy, reviewNote, profile, slice;
  plus List<RedactedSignal> redactions and List<MatchResult> matches. Events:
  CvSubmitted, ProfileExtracted, ProfileSanitized, MatchesScored,
  ReviewRecorded. Commands: recordExtraction, recordSanitization, recordMatches,
  review, getCandidate. emptyState() returns Candidate.initial("") with NO
  commandContext() reference (Lesson 3).
- 1 EventSourcedEntity CvIntakeQueue with command enqueue(cvText, postings,
  slice) emitting CvSubmittedToQueue.
- 1 View MatchesView with row type Candidate, table updater consuming
  CandidateEntity events. ONE query: getAllCandidates SELECT * AS candidates
  FROM matches_view. No WHERE status filter — Akka cannot auto-index enum
  columns (Lesson 2); filter client-side in callers.
- 1 Consumer CvIntakeConsumer subscribed to CvIntakeQueue events; starts a
  MatchingWorkflow per submission with a fresh UUID.
- 2 TimedActions: CvSimulator (every 30s, reads next record from
  src/main/resources/sample-data/cvs.jsonl and calls CvIntakeQueue.enqueue);
  FairnessMonitor (every 60s, queries MatchesView.getAllCandidates, groups
  MATCHED candidates by slice, computes selection-rate parity, records drift on
  a SystemControl KVE or logs a flag).
- 2 HttpEndpoints: MatchingEndpoint at /api with candidates submit, review,
  list (filter client-side from getAllCandidates), single candidate, SSE stream,
  fairness summary, and /api/metadata/* serving the YAML/MD from
  src/main/resources/metadata/. AppEndpoint serving / -> 302 /app/index.html and
  /app/* -> static-resources/*.

Companion files:
- Records: CvProfile, RedactedSignal, MatchResult, plus a MatchList wrapper for
  the agent result. Candidate state record with the Optional fields above.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9736 and model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), keys read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Verify model names are current before locking (Lesson 8).
- src/main/resources/sample-data/cvs.jsonl with 8 CV records, each carrying
  raw cv text, a postings list, and a declared slice label; skew the slices so
  the fairness monitor has something to flag.
- src/main/resources/metadata/{eval-matrix.yaml,risk-survey.yaml,README.md}
  copied from the root files for the endpoint to serve from classpath.
- eval-matrix.yaml at the project root with controls S1, E1, H1, A1 and a
  matching simplified_view. Employment is industry-anchored: include EU AI Act
  Annex III employment anchors.
- risk-survey.yaml at the project root pre-filling sector, decisions (employment
  decision support = true), data.types, capability.*, model.*; marking deployer
  fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root (already authored): title, elevator, component
  inventory, integration descriptor, how to run, the tabs, license. No
  governance-mechanisms section, no configuration section.
- src/main/resources/static-resources/index.html — single self-contained file
  (no ui/, no npm). Five tabs. Mermaid CSS overrides + theme variables from
  Lesson 24. Tab switching by data-tab/data-panel attribute, no zombie panels
  (Lesson 26). Risk Survey and Eval Matrix render in matrix-card/matrix-row
  style with mechanism pills.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding, /akka:specify inspects the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (ExtractionAgent -> CvProfile, MatchingAgent -> List<MatchResult>);
  write src/main/resources/mock-responses/{extraction-agent,matching-agent}.json
  with 4-6 shape-correct entries each. Sets model-provider = mock.
  (b) Name an existing env var — record only the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives only in Claude session memory;
  passed to the JVM via the MCP tool's environment parameter; gone at session
  end.
- NEVER write the key value to any file Akka creates. Record only the reference.
- Bootstrap.java fails fast with a clear message naming the configured reference
  if it does not resolve at runtime; never echo captured key material.

Mock LLM provider per-agent shapes:
- ExtractionAgent -> CvProfile{ yearsExperience: random 1-20, skills: 3-6 from a
  fixed pool, education: one of a fixed list, currentTitle: one of a fixed list }.
- MatchingAgent -> List<MatchResult>, one per posting, score 0-100 descending,
  rationale a short fixed phrase referencing skill overlap.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23,
24, 25, 26:
- (1) AutonomousAgent never downgraded to Agent.
- (4) Workflow step timeouts set explicitly on agent-calling steps.
- (6) Optional<T> for every nullable field on the Candidate view row record.
- (7) CvMatcherTasks.java companion present.
- (8) Model names verified current before writing application.conf.
- (9) Run command is "/akka:build", never "mvn akka:run".
- (10) Port 9736 declared in application.conf dev-mode.
- (11) source.platform / brand origin is corpus-internal — never user-facing.
- (12) UI fits the 1080px content column, no horizontal scroll.
- (13) Integration label is "Runs out of the box" — never T1/T2/T3/T4.
- (23) No competitor / alternative-product brand names anywhere user-facing.
- (24) Mermaid state-label colour + edge-label overflow:visible overrides.
- (25) Five-option key sourcing; never write the key value to disk.
- (26) Tab switching by data-tab/data-panel attribute; no zombie panels.
```

## 12. Post-scaffolding workflow

When `/akka:specify` finishes scaffolding the SDD specification, **do not stop and wait for the user**. Continue automatically:

1. Run `/akka:plan` — produce the architectural plan; pick the most conservative option for any ambiguity, consistent with Sections 4 and PLAN.md.
2. Run `/akka:tasks` — break the plan into tasks with the same default-everything policy.
3. Run `/akka:implement` — execute every task in order. On a task failure, capture it, continue with the next task, and summarise all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

When `/akka:build` reports the service is running, output just the listening URL (`http://localhost:9736`) and a one-line summary of any failures from step 3. Stop earlier only on a hard error you cannot work around without the user: an unresolved API key, an unrecoverable compile error after exhausting auto-fix, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
