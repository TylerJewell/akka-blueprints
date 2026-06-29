# Akka Sample: Hiring Supervisor Orchestration

A hiring supervisor delegates candidate screening, job-description evaluation, and interview scheduling to specialist sub-agents, then consolidates decisions into a structured hiring recommendation. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance, including a pre-delegation policy check and sanitization of protected-attribute data.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the candidate pipeline and hiring tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.hr-recruiting.hiring-supervisor  ~/my-projects/hiring-supervisor
cd ~/my-projects/hiring-supervisor
```

(Optional) Edit `SPEC.md` to change the candidate pipeline topics, model provider, or recommendation schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **HiringSupervisor** — AutonomousAgent that receives a candidate application, plans the evaluation stages, and consolidates the sub-agent results into a final hiring recommendation.
- **ScreeningAgent** — AutonomousAgent that evaluates a candidate's qualifications against job requirements, returning a scored screening report.
- **SchedulingAgent** — AutonomousAgent that proposes interview slots given a candidate's availability and role tier.
- **HiringWorkflow** — Workflow that fans screening and scheduling out in parallel, then calls HiringSupervisor for consolidation.
- **ApplicationEntity** — EventSourcedEntity holding the application lifecycle.
- **ApplicationView** — projection the UI streams via SSE.
- **HiringEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the candidate pipeline drip rate or remove the simulator entirely.
- `SPEC.md §5` — adjust the `CandidateApplication` record fields (e.g., add `diversityConsent`).
- `prompts/hiring-supervisor.md` — narrow the supervisor to a specific job family.
- `eval-matrix.yaml` — add an additional sanitizer stage if your deployment includes résumé attachments with photo data.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a candidate application → application enters `RECEIVED`, then `UNDER_REVIEW`, then `RECOMMENDED` or `REJECTED`.
2. Guardrail fires before a delegate call when a policy check fails → application enters `POLICY_HOLD` with a reason.
3. Sanitizer strips protected-attribute fields before any agent sees the application payload.
4. Wait after a successful recommendation; the application row in the UI shows an eval score.

## License

Apache 2.0.
