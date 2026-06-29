# Akka Sample: Scored CV Reviewer

A tailor agent produces a CV targeted at a job description; a reviewer agent scores it against a structured hiring rubric; the two iterate until the score meets the acceptance threshold or the loop hits its retry ceiling. Demonstrates the **evaluator-optimizer** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required before `/akka:specify`: **none**. The blueprint runs out of the box; the inbound submission stream and the hiring-decision surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./evaluator-optimizer.hr-recruiting.scored-loop-tailor  ~/my-projects/scored-loop-tailor
cd ~/my-projects/scored-loop-tailor
```

(Optional) Edit `SPEC.md` to change the rubric weights, the acceptance score threshold, the max-retry cap, or the sample job descriptions the simulator drips.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CvTailorAgent** — AutonomousAgent that produces a tailored CV targeting a job description; accepts reviewer feedback when revising.
- **CvReviewerAgent** — AutonomousAgent that scores a CV draft against a structured hiring rubric, returns either `APPROVE` or `REVISE` with a typed `ReviewNotes` payload.
- **TailoringWorkflow** — Workflow that runs the tailor → score → revise loop up to a configurable retry ceiling, transitions the application to `APPROVED` on reviewer acceptance or to `REJECTED_FINAL` when the ceiling is hit.
- **ApplicationEntity** — EventSourcedEntity that holds the application lifecycle, every CV draft, every review, and the final outcome.
- **SubmissionQueue** — EventSourcedEntity that logs each job application submission for replay and audit.
- **ApplicationsView** — read-side projection that the UI lists and streams via SSE.
- **SubmissionConsumer** — Consumer that starts a workflow per inbound submission.
- **SubmissionSimulator** — TimedAction that drips a sample job posting every 60 s so the App UI is never empty.
- **EvalSampler** — TimedAction that records the per-attempt eval event each cycle (control E1).
- **RecruitingEndpoint + AppEndpoint** — REST + SSE + `/api/metadata/*` + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the job postings the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Application` record fields (e.g., raise `maxAttempts`, change the acceptance score threshold).
- `prompts/cv-tailor.md` — narrow the tailoring strategy (e.g., emphasise technical skills, add ATS keyword guidance).
- `prompts/cv-reviewer.md` — change the rubric (e.g., add a culture-fit dimension, adjust dimension weights).
- `eval-matrix.yaml` — tighten enforcement on the output guardrail (currently non-blocking advisory).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a job posting → application progresses `TAILORING` → `REVIEWING` → `APPROVED` within the retry ceiling.
2. Force-fail rubric → application hits `REJECTED_FINAL` after the configured number of attempts; the entity preserves every draft and every review for audit.
3. The output guardrail blocks a CV draft that lacks mandatory sections, so the reviewer never scores an incomplete document.
4. Each completed cycle emits a `ReviewEvalRecorded` event that surfaces in the App UI's per-attempt timeline.

## License

Apache 2.0.
