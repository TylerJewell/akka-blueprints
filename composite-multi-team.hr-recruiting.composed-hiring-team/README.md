# Akka Sample: Composed Hiring Workflow

A hiring-team agent runs a candidate from application to offer through two nested pipelines — a `CandidateWorkflow` that screens each applicant and a `CvImprovementLoop` that critiques and rewrites the candidate's CV before advancing it — demonstrating the **composite-multi-team** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. The screening pipeline, the improvement loop, the panel interview, and the offer approval are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./composite-multi-team.hr-recruiting.composed-hiring-team  ~/my-projects/composed-hiring-workflow
cd ~/my-projects/composed-hiring-workflow
```

(Optional) Edit `SPEC.md` to change the screener roster, the interview panel, the model provider, or the sample applications the simulator drips.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **HiringManager** — AutonomousAgent that opens a requisition (produces a `HiringBrief`) and, at the end, approves or rejects the final offer under an output guardrail.
- **ScreeningLead + Screener** — the screening desk: the lead assigns screening criteria; several screener instances each evaluate one dimension and write a `ScreeningNote`. The lead then synthesises a `ScreeningReport`. Delegation.
- **CvCoach + CvCritic** — the improvement loop: the coach rewrites the candidate's CV; the critic scores it; the coach revises until the critic passes or the iteration cap is reached. A team running an embedded feedback loop.
- **Interviewer** — the panel interview: several interviewer instances each score one competency dimension; a deterministic `PanelRule` turns their scores into a `PanelVerdict`. Moderation.
- **HiringTeamWorkflow** — the top-level pipeline: open → screen → improve CV → interview → offer, with a revision round when the panel requests re-assessment.
- **CandidateWorkflow** — nested workflow driving the screening desk for one applicant.
- **CvImprovementLoop** — nested workflow running the coach-critic feedback loop.
- **ApplicationEntity + CvEntity** — the shared application workspace and the per-candidate CV whose improvement iterations are tracked atomically.
- **ApplicationBoardView, CvBoardView** — the read models the UI streams.
- **StageEvalConsumer** — fires a non-blocking quality eval on every stage result.
- **HiringEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample application topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — change the screener roster (`screener-1`, `screener-2`) or the interview panel axes (`technical`, `behavioural`, `cultural`).
- `prompts/screener.md`, `prompts/cv-coach.md`, `prompts/cv-critic.md`, `prompts/interviewer.md` — narrow each desk to a specific role or seniority level.
- `eval-matrix.yaml` — extend the before-agent-invocation guardrail's refusal conditions, or change the stage-eval thresholds.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an application → the hiring manager opens a brief → the screening desk delegates and synthesises → the CV improvement loop runs the coach-critic cycle → the interview panel scores and delivers a verdict → a passing verdict triggers an offer that is approved and published.
2. Two screeners claim the same dimension concurrently → only one wins; no double-scoring.
3. The panel returns a `REASSESS` verdict → the application loops back for one CV improvement round, then proceeds.
4. A screener or coach attempts to write into a workspace they were not assigned → the before-agent-invocation guardrail blocks execution and the stage is recorded with the refusal reason.
5. After an offer is `EXTENDED`, a compliance officer posts a post-hire review → it is recorded without changing the offer state.

## License

Apache 2.0.
