# Akka Sample: Parallel Multi-Reviewer Workflow

Three independent reviewers — an HR reviewer, a hiring manager, and a team member — evaluate a candidate in parallel; their assessments are aggregated by a panel coordinator into a multi-perspective hiring recommendation. Demonstrates the **debate-multi-perspective** coordination pattern with embedded fairness governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — every external surface is modelled inside the service with Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./debate-multi-perspective.hr-recruiting.parallel-hiring-reviewers  ~/my-projects/parallel-hiring-reviewers
cd ~/my-projects/parallel-hiring-reviewers
```

(Optional) Edit `SPEC.md` to change the reviewer roles, the recommendation schema, or the model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PanelCoordinator** — AutonomousAgent that briefs each reviewer on their evaluation axis, then aggregates the three perspectives into an overall hiring recommendation.
- **HrReviewer** — AutonomousAgent assessing compliance, compensation fit, and protected-attribute neutrality.
- **ManagerReviewer** — AutonomousAgent assessing role fit, skill alignment, and team needs.
- **TeamReviewer** — AutonomousAgent assessing cultural fit and peer working-style compatibility.
- **FairnessJudge** — AutonomousAgent that scores how consistently the three reviewer assessments drive the overall recommendation.
- **ReviewWorkflow** — Workflow that sanitizes the candidate profile, fans out to the three reviewers in parallel, then calls the coordinator for aggregation.
- **CandidateEntity** — EventSourcedEntity holding the full evaluation lifecycle.
- **EvaluationView** — projection used by the UI.
- **ReviewEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample candidate profiles the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `PerspectiveReview` or `HiringRecommendation` record fields (e.g., add a `confidence` field).
- `prompts/hr-reviewer.md` — narrow the HR axis to a specific compliance framework.
- `eval-matrix.yaml` — extend the special-category sanitizer's attribute table, or add a `before-tool-call` guardrail if you wire a real ATS integration.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a candidate profile → evaluation enters `INTAKE`, then `EVALUATING`, then `DECIDED` with three perspective reviews and one overall recommendation; UI reflects each transition via SSE.
2. Protected attributes in the submitted profile are redacted before any reviewer sees them; the redaction count surfaces in the UI.
3. A reviewer timeout drives the evaluation to `DEGRADED` with the recommendation aggregated from whichever perspectives returned.
4. The output guardrail blocks a malformed or policy-violating recommendation; the evaluation enters `BLOCKED`.
5. The fairness sampler scores per-perspective consistency and surfaces the score on the App UI.

## License

Apache 2.0.
