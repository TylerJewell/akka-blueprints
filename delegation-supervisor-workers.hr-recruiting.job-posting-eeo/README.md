# Akka Sample: Job Posting Team

A posting lead delegates company-culture analysis and role-requirements analysis to two specialist agents, then merges their outputs into a polished job posting that is screened for discriminatory language, sanitized of protected-class preferences, and cleared against a hiring-policy documentation gate. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — no external services, no Docker.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.hr-recruiting.job-posting-eeo  ~/my-projects/job-posting-eeo
cd ~/my-projects/job-posting-eeo
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or the posting output fields.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PostingLead** — AutonomousAgent that decomposes a request, synthesises the merged posting, and runs the EEO output guardrail.
- **CultureAnalyst** — AutonomousAgent that analyzes the hiring company's values and tone.
- **RoleAnalyst** — AutonomousAgent that analyzes role requirements, backed by a seeded market-research tool.
- **JobPostingWorkflow** — Workflow that fans work to the two analysts in parallel, then drafts, sanitizes, and clears.
- **JobPostingEntity** — EventSourcedEntity holding the posting lifecycle.
- **JobPostingView** — projection the UI streams via SSE.
- **JobPostingEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the sample requests the simulator drips, or remove the simulator.
- `SPEC.md §5` — adjust the `JobPosting` record fields (e.g., add `seniorityBand`).
- `prompts/role-analyst.md` — narrow the agent to a single job family.
- `eval-matrix.yaml` — extend the protected-class term list the sanitizer screens.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a posting request → posting progresses `PLANNING` → `ANALYZING` → `DRAFTED` → `SANITIZED` → `CLEARED`.
2. A draft with discriminatory language → the EEO guardrail moves it to `BLOCKED`.
3. A draft carrying a protected-class preference → the sanitizer strips it and records the removed terms.

## License

Apache 2.0.
