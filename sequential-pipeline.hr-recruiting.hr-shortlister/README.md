# Akka Sample: HR Candidate Shortlister

An `ShortlistAgent` walks each applicant through three task phases — **PARSE → SCORE → SHORTLIST** — wired together by explicit task dependencies. The PARSE phase extracts structured profile data from a resume; the SCORE phase evaluates the profile against job criteria; the SHORTLIST phase decides accept / hold / reject and writes the result to ERPNext. Protected personal attributes are redacted before the scoring phase runs. A periodic drift monitor compares shortlisting outcome distributions for bias. A recruiter approval gate sits between automated shortlisting and the ERPNext write.

Demonstrates the **sequential-pipeline** coordination pattern with three governance mechanisms: a `before-tool-call` sanitizer that strips special-category fields (age, gender, nationality, disability indicators) before the Score phase sees the profile; a periodic `drift-fairness-watch` evaluator that compares shortlisting-rate distributions across demographic cohorts; and a human-in-the-loop `hitl` gate that holds every shortlist decision until a recruiter explicitly approves or overrides it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software required. Resume parsing, scoring, and shortlisting are all implemented in-process. The ERPNext integration is stubbed with a configurable HTTP adapter; the stub returns deterministic responses out of the box.

## Generate the system

```sh
cp -r ./sequential-pipeline.hr-recruiting.hr-shortlister  ~/my-projects/hr-shortlister
cd ~/my-projects/hr-shortlister
```

(Optional) Edit `SPEC.md` to point at a real ERPNext instance URL, a different model provider, or a narrower job-criteria set.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ShortlistAgent** — one AutonomousAgent declaring three Task constants (`PARSE_RESUME`, `SCORE_PROFILE`, `DECIDE_SHORTLIST`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **ShortlistingWorkflow** — runs `parseStep → scoreStep → shortlistStep → approvalStep`. Each step calls `runSingleTask` and writes the typed result back onto `ApplicationEntity` before the next step starts. `approvalStep` suspends until a recruiter calls `POST /api/applications/{id}/decision`.
- **ApplicationEntity** — an EventSourcedEntity holding the per-application lifecycle (`ResumeReceived`, `ProfileParsed`, `ScoreAssigned`, `ShortlistDecided`, `RecruiterApproved`, `RecruiterOverridden`, `WrittenToErpNext`, `ApplicationFailed`).
- **ParseTools / ScoreTools / ShortlistTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase and that special-category fields are stripped before scoring tools run.
- **SpecialCategoryGuardrail** — the runtime check that backs the sanitizer contract. Any tool call that would expose age, gender, nationality, or disability indicators to the SCORE phase is intercepted; the field values are replaced with `[REDACTED]` before the tool body executes.
- **FairnessScorer** — rule-based periodic evaluator that computes shortlisting-rate ratios across gender and nationality cohorts in a rolling 30-day window and emits a `DriftReport` when a ratio exceeds the configured threshold.
- **ApplicationView + ShortlistEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded applicant profiles under `src/main/resources/sample-data/profiles/*.json` to fit your demo audience or job family.
- `SPEC.md §4` and `prompts/shortlist-agent.md` — narrow the agent's role to a specific job family or scoring rubric by tightening the system prompt and renaming the typed records (`Profile`, `Score`, `ShortlistDecision`).
- `SPEC.md §5` — extend the typed outputs with role-specific fields. The phase-gating guardrail does not need editing — it checks recorded-phase preconditions, not field shapes.
- `eval-matrix.yaml` — wire a real fairness backend (replace the rolling-window stub with a live cohort query) by editing the `H1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A recruiter submits a resume → `PARSE` runs → `SCORE` runs → `SHORTLIST` runs → the application enters the approval queue within ~60 s. Every transition is visible in real time via SSE.
2. A resume containing an explicit age field is submitted → the `before-tool-call` sanitizer intercepts the `evaluateCriteria` call → the age field value is replaced with `[REDACTED]` before the tool body runs → the `SpecialCategoryRedacted` event is recorded → the pipeline completes correctly.
3. After 30+ shortlisting decisions, the periodic fairness scanner computes outcome ratios; a cohort whose shortlisting rate diverges by > 20% triggers a `DriftReport` visible on the Eval Matrix tab.
4. A shortlisted application sits in `AWAITING_APPROVAL` until a recruiter POSTs an approval or override; only after approval does the workflow write the record to ERPNext.

## License

Apache 2.0.
