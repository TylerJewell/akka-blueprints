# Akka Sample: AI Ads Generation Agent with Crypto Payments

An Ad Campaign Planner drafts ad creatives for a given campaign brief, submits each creative to a Copywriter agent for refinement, then routes the approved placement buy through a crypto-payment tool. Human approval gates every spend decision before the on-chain transaction fires. An operator halt switch stops new payment dispatches at any time.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration form (Runs out of the box): **None.** The crypto-payment tool, ad-network placement surface, and brand-asset library are all simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.sales-marketing.ad-spend-payment-agent  ~/my-projects/ad-spend-payment-agent
cd ~/my-projects/ad-spend-payment-agent
```

(Optional) Edit `SPEC.md` to change the campaign brief schema, payment token, or any agent's behaviour.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CampaignPlannerAgent** — AutonomousAgent that reads a campaign brief, builds a creative plan on a campaign ledger, and dispatches each creative step to the Copywriter or the PaymentExecutor.
- **CopywriterAgent** — AutonomousAgent that drafts or refines ad copy and returns a typed `CreativeResult`.
- **PaymentExecutorAgent** — AutonomousAgent that constructs a crypto payment instruction from an approved placement record and calls the simulated on-chain tool.
- **CampaignWorkflow** — Workflow with plan → dispatch-guarded → execute → sanitize → record → approval-gate → decide loop, plus replan and halt branches.
- **CampaignEntity** — EventSourcedEntity holding the campaign lifecycle, creative ledger, payment ledger, and final campaign report.
- **ApprovalEntity** — EventSourcedEntity holding one pending spend approval per placement. Human approvers act through the UI.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **CampaignQueue** — EventSourcedEntity acting as the audit log for submitted campaign briefs.
- **CampaignView** — projection consumed by the UI.
- **CampaignRequestConsumer** — Consumer that starts a `CampaignWorkflow` per brief submission.
- **BriefSimulator** — TimedAction that drips a sample campaign brief every 90 s.
- **StaleCampaignMonitor** — TimedAction that marks campaigns stuck in `AWAITING_APPROVAL` or `EXECUTING` past 10 minutes as `STUCK`.
- **CampaignEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the simulator's canned briefs or disable it entirely.
- `SPEC.md §5` — add fields to `CampaignBrief` (e.g., `targetAudienceSegment`).
- `prompts/campaign-planner.md` — narrow the planner to a specific ad format (e.g., video-only).
- `eval-matrix.yaml` — add an `eval-event` control for creative quality scoring.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a campaign brief → planner builds a creative plan, copywriter drafts copy, spend approval is requested, human approves, payment fires, campaign reaches `COMPLETED`.
2. Submit a brief whose plan proposes an above-limit spend → the before-payment guardrail blocks the payment dispatch; the planner revises the placement or the campaign ends in `FAILED`.
3. Submit a brief, approve one placement, then click **Halt new dispatches** — no further payment dispatches fire; in-flight copywriter work finishes; campaign moves to `HALTED`.
4. A copywriter result containing a wallet-private-key-shaped string is scrubbed before it reaches the planner's next prompt.

## License

Apache 2.0.
