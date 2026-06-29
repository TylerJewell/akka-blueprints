# Akka Sample: SRE Incident Response Agent

An SRE agent that responds to on-call alerts — gathering telemetry from seeded fixtures, forming hypotheses about root causes, and either proposing or executing remediation actions. Demonstrates the **planner-executor** coordination pattern with destructive-action guardrails, human-in-the-loop approval, automatic safety halt, and post-incident evaluation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host software required by this blueprint's integration tier (Runs out of the box): **None.** Telemetry sources — metrics, logs, traces, runbook access — are simulated inside the same Akka service using seeded fixtures.

## Generate the system

```sh
cp -r ./planner-executor.ops-automation.sre-incident-responder  ~/my-projects/sre-incident-responder
cd ~/my-projects/sre-incident-responder
```

(Optional) Edit `SPEC.md` to change the alert prompts the simulator drips, narrow the remediation allow-list, or adjust approval workflow timeouts.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **IncidentCommanderAgent** — AutonomousAgent that maintains an investigation ledger (known facts, open hypotheses, investigation plan, current probe) and a remediation ledger (proposed actions, approvals, execution outcomes). Drives TRIAGE, INVESTIGATE, PROPOSE, and COMPOSE_REPORT modes.
- **TelemetryAgent** — AutonomousAgent that retrieves metric time-series, log snippets, and trace summaries from seeded fixtures.
- **RunbookAgent** — AutonomousAgent that looks up service runbooks and configuration baselines from fixture files.
- **RemediationAgent** — AutonomousAgent that executes approved remediation actions (restart, rollback, traffic-shift) against a simulated control plane.
- **IncidentWorkflow** — Workflow with a triage → investigate-loop → propose → approve → execute → evaluate chain, plus automatic halt and timeout branches.
- **IncidentEntity** — EventSourcedEntity holding the incident lifecycle, both ledgers, and the post-incident report.
- **ApprovalEntity** — EventSourcedEntity holding the pending approval request and the SRE's decision.
- **SystemControlEntity** — EventSourcedEntity holding the operator halt flag.
- **IncidentView** — projection used by the UI.
- **IncidentEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the alert prompts the simulator drips, or disable the simulator.
- `SPEC.md §5` — adjust the `Incident` record (e.g., add `severityScore`).
- `prompts/incident-commander.md` — narrow the commander to a specific infrastructure domain (e.g., Kubernetes-only).
- `eval-matrix.yaml` — tune the remediation allow-list entries under the `G1` control.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an alert → commander triages, investigates with telemetry and runbook probes, proposes a remediation, SRE approves, action executes, post-incident report is generated.
2. Inject an out-of-policy remediation proposal (e.g., `DROP TABLE` on a production DB) → guardrail blocks it; commander revises the proposal.
3. Reject an approval request → commander replans the investigation; task reaches `MITIGATED` via an alternate remediation or `UNRESOLVED` after exhausting options.
4. Click **Halt new probes** while the workflow is investigating → in-flight telemetry probe finishes; no further probes dispatch; incident ends in `HALTED`.

## License

Apache 2.0.
