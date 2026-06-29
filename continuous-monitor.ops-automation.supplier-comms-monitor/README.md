# Akka Sample: Supplier Communications Agent

A continuous background worker polls open purchase orders in a simulated Dynamics 365 Supply Chain feed, sends confirmation requests to suppliers, and surfaces potential delivery delays. Every outbound supplier email passes through a before-call guardrail. Material PO changes require buyer sign-off before the agent proceeds. A periodic eval measures delivery-prediction accuracy over time.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None** — the blueprint ships with an in-process simulator for both the Dynamics 365 feed and the outbound email stub.

## Generate the system

```sh
cp -r ./continuous-monitor.ops-automation.supplier-comms-monitor  ~/my-projects/supplier-comms-monitor
cd ~/my-projects/supplier-comms-monitor
```

(Optional) Edit `SPEC.md §3` to point `Poller` at a real Dynamics 365 REST endpoint or to keep the in-memory feed simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PoOrderPoller** — TimedAction firing every 20 s that reads from a simulated open-PO feed.
- **DeliveryRiskAgent** — Agent (typed) that classifies each PO as `ON_TRACK`, `AT_RISK`, or `CRITICAL`.
- **SupplierOutreachAgent** — AutonomousAgent that drafts outbound supplier confirmation requests.
- **PurchaseOrderEntity** — EventSourcedEntity holding each PO's collaboration lifecycle (opened → at-risk detected → outreach drafted → awaiting buyer approval → outreach sent / escalated).
- **PoOrderWorkflow** — per-PO orchestration: assess → (maybe draft outreach) → guardrail check → await buyer if material → send/escalate.
- **PoOrderView + PoOrderEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **AccuracyEvalRunner** — TimedAction running every 60 minutes; samples closed POs, compares predicted risk to actual delivery outcome.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated PO feed for a real Dynamics 365 REST connection.
- `SPEC.md §5` — extend `PurchaseOrder` with org-specific fields (`supplierTier`, `contractCeiling`, etc.).
- `prompts/delivery-risk.md` — tune classification thresholds for your commodity categories.
- `eval-matrix.yaml` — add regulation anchors when deploying in a regulated supply-chain context.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A simulated PO arrives → it is assessed → outreach drafted → held for buyer approval if AT_RISK.
2. A buyer clicks Approve → the outreach transitions to SENT (simulated; no real email leaves).
3. A buyer clicks Escalate → the PO is flagged for procurement review.
4. Accuracy eval runs and surfaces a prediction score on at least one closed PO.

## License

Apache 2.0.
