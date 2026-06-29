# Akka Sample: Mortgage Assistant (Multi-Agent)

A mortgage supervisor delegates eligibility screening, documentation review, underwriting analysis, and applicant communications to four specialist agents. After the workers return their assessments, the supervisor requests a human underwriter sign-off before the decision is finalised. Demonstrates the **delegation-supervisor-workers** coordination pattern with PII sanitisation, sectoral compliance filtering, human-in-the-loop (HITL) gating, and lending-decision quality evaluation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. The inbound application stream and the mortgage tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.finance-analysis.mortgage-team  ~/my-projects/mortgage-team
cd ~/my-projects/mortgage-team
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MortgageSupervisor** — AutonomousAgent that decomposes an application into a work plan and merges worker assessments into a final recommendation packet.
- **EligibilityAgent** — AutonomousAgent that evaluates income, credit, and LTV criteria.
- **DocumentationAgent** — AutonomousAgent that verifies completeness and authenticity flags on submitted documents.
- **UnderwritingAgent** — AutonomousAgent that models risk exposure and proposes a lending term structure.
- **CustomerCommsAgent** — AutonomousAgent that drafts applicant-facing notifications at each stage.
- **MortgageWorkflow** — Workflow that fans eligibility, documentation, and underwriting out in parallel, then gates on a human sign-off before finalising.
- **ApplicationEntity** — EventSourcedEntity holding the full application lifecycle.
- **ApplicationView** — projection the UI streams via SSE.
- **MortgageEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the loan products the simulator cycles through, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `MortgageApplication` record fields (e.g., add `loanPurpose`).
- `prompts/supervisor.md` — change the decomposition strategy for different lending products.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real credit bureau API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an application → application enters `RECEIVED`, then `UNDER_REVIEW`, then `PENDING_SIGNOFF`, then `APPROVED` or `DECLINED`.
2. PII sanitiser redacts applicant identity fields before they enter any agent prompt; raw values are never logged.
3. A worker's sectoral compliance filter blocks a term structure that violates applicable lending rules; the application enters `COMPLIANCE_HOLD`.
4. HITL gate halts the workflow at `PENDING_SIGNOFF`; it only continues after a human underwriter approves via `POST /api/applications/{id}/decision`.
5. Eval-event sampling captures one underwriting recommendation and surfaces the quality score on the App UI.

## License

Apache 2.0.
