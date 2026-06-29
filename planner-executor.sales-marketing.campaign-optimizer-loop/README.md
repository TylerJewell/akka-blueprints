# Akka Sample: Campaign Optimizer

A Planner analyzes campaign goals, generates a multi-step execution plan, and dispatches work to specialist agents — CopyWriter, AudienceSegmenter, AssetPublisher, PerformanceAnalyst — tracking progress on a campaign ledger plus a run ledger. Requires marketer approval before launch. Demonstrates the **planner-executor** coordination pattern with brand guardrails, human-in-the-loop approval, and periodic performance monitoring.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Marketing platform credential): **A valid API credential for your marketing platform (e.g., campaign management API key or OAuth token). Supply it via the same key-sourcing options as the LLM key above — env var, env file, secrets-store URI, or type-once in session. Akka never writes the credential value to disk.**

## Generate the system

```sh
cp -r ./planner-executor.sales-marketing.campaign-optimizer-loop  ~/my-projects/campaign-optimizer-loop
cd ~/my-projects/campaign-optimizer-loop
```

(Optional) Edit `SPEC.md` to change the campaign types the simulator creates, the approval threshold, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PlannerAgent** — AutonomousAgent that owns a campaign ledger (goals, target audience, channel plan, current dispatch) and a run ledger (per-step attempts, verdicts, metrics). Decides who runs the next step. Replans when steps fail.
- **CopyWriterAgent** — AutonomousAgent that drafts email subject lines, ad headlines, and body copy from seeded brand-voice fixtures.
- **AudienceSegmenterAgent** — AutonomousAgent that selects audience segments from seeded CRM cohort fixtures.
- **AssetPublisherAgent** — AutonomousAgent that simulates publishing assets to a channel endpoint from seeded channel fixtures.
- **PerformanceAnalystAgent** — AutonomousAgent that reads simulated KPI fixtures and returns a performance assessment.
- **CampaignWorkflow** — Workflow with a plan → approval-gate → dispatch → execute → record → evaluate loop, replan branch, and terminal exit states.
- **CampaignEntity** — EventSourcedEntity holding the campaign lifecycle, both ledgers, and the final report.
- **ApprovalEntity** — EventSourcedEntity holding a marketer's approval decision for a campaign prior to launch.
- **CampaignView** — projection used by the UI.
- **CampaignEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the campaign briefs the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Campaign` record fields (e.g., add `budgetCap`).
- `prompts/planner.md` — narrow the planner to a single channel type (e.g., email-only).
- `eval-matrix.yaml` — tune the KPI thresholds in the performance-monitor control.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a campaign brief → planner creates a plan, marketer approves, specialists execute, campaign reaches `LIVE` within ~4 minutes.
2. Inject copy that violates the brand-voice guardrail → the offending copy step is blocked before publishing; planner replans.
3. Marketer rejects the approval request → campaign moves to `REJECTED`; no assets are published.
4. Periodic performance check reads KPIs below threshold → planner replans with a revised audience or copy variant.

## License

Apache 2.0.
