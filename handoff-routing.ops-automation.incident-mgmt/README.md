# Akka Sample: Incident Management

A `ClassifierAgent` categorizes an incoming incident report, then routes the same `INVESTIGATE` task to an `InfraSpecialist` or `AppSpecialist` that owns the investigation end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with two governance mechanisms (a before-tool-call guardrail that checks every write to external systems before it executes, and an on-incident-reporter eval that scores every classification decision).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound incident stream and the outbound investigation surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.ops-automation.incident-mgmt  ~/my-projects/incident-mgmt
cd ~/my-projects/incident-mgmt
```

(Optional) Edit `SPEC.md` to point `IncidentFeeder` at a real alerting source (PagerDuty, OpsGenie, Datadog) or to add additional incident categories.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **IncidentFeeder** — TimedAction firing every 30 s that drips canned incident reports from a JSONL file into `IncidentQueue`.
- **IncidentQueue** — EventSourcedEntity append-only log of every inbound report (audit before enrichment).
- **ContextEnricher** — Consumer that attaches environment metadata (host group, service tier) before any LLM call.
- **ClassifierAgent** — typed Agent that categorizes the enriched report into `INFRASTRUCTURE`, `APPLICATION`, or `AMBIGUOUS`.
- **InfraSpecialist** — AutonomousAgent that owns the `INVESTIGATE` task for infrastructure incidents (network, compute, storage).
- **AppSpecialist** — AutonomousAgent that owns the `INVESTIGATE` task for application incidents (errors, latency, deploys).
- **RoutingJudge** — typed Agent used by `RoutingEvalScorer` to grade every classification decision against a 1–5 rubric.
- **ToolCallGuardrail** — typed Agent used by `IncidentWorkflow` to pre-screen every write action before the specialist executes it.
- **IncidentWorkflow** — Workflow per incident: enrich → classify → route → investigate → guardrail → publish.
- **IncidentEntity** — EventSourcedEntity holding each incident's lifecycle.
- **IncidentView + IncidentEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **RoutingEvalScorer** — Consumer that listens for `IncidentClassified` events and writes an inline eval score.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the classification taxonomy (add `SECURITY`, `DATA`, etc.) and add the matching specialist agent.
- `SPEC.md §5` — extend the `Incident` record with deployer-specific fields (`serviceOwner`, `slaTier`, `escalationContact`).
- `prompts/classifier-agent.md` — narrow the classification rules (confidence thresholds, category-specific signal patterns).
- `prompts/infra-specialist.md` / `prompts/app-specialist.md` — encode runbook authority limits, escalation triggers, permitted write actions.
- `eval-matrix.yaml` — swap the in-process before-tool-call guardrail for an external policy engine when the deployer has one.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Feeder drips an infrastructure-flavoured incident → classified `INFRASTRUCTURE`, handed to `InfraSpecialist`, investigation action published.
2. Feeder drips an application incident → classified `APPLICATION`, handed to `AppSpecialist`, investigation action published.
3. An ambiguous incident classifies as `AMBIGUOUS` and the workflow terminates in `ESCALATED` without calling any specialist.
4. A specialist that attempts to write a runbook action the guardrail rejects has the action blocked; the incident lands in `BLOCKED` for human review.
5. The routing eval score (1–5) and rationale appear on every classified incident within a few seconds of the decision.

## License

Apache 2.0.
