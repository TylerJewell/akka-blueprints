# Akka Sample: AI Security Triage Agent

A security supervisor delegates vulnerability scanning to a VulnerabilityScanner and threat-context gathering to a ThreatContextAgent in parallel, then the SecurityCoordinator merges their findings into one triage report and queues the mitigation for human approval. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance: an eval-event captures each incident for quality scoring, and a human-in-the-loop gate blocks autonomous mitigation actions until a security officer approves.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound incident stream and the vulnerability tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.governance-risk.security-triage-agent  ~/my-projects/security-triage-agent
cd ~/my-projects/security-triage-agent
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or incident schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SecurityCoordinator** — AutonomousAgent that decomposes an incident into scan and context tasks, then merges worker outputs into a triage report.
- **VulnerabilityScanner** — AutonomousAgent that identifies and scores vulnerabilities from the incident signal.
- **ThreatContextAgent** — AutonomousAgent that retrieves threat-actor context and attack-pattern history for the affected asset.
- **TriageWorkflow** — Workflow that fans the work out to VulnerabilityScanner and ThreatContextAgent in parallel, merges results, and holds for human approval before issuing any mitigation command.
- **IncidentEntity** — EventSourcedEntity holding the full incident lifecycle.
- **ApprovalEntity** — EventSourcedEntity tracking the human approval gate.
- **IncidentView** — projection the UI streams via SSE.
- **IncidentEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the incident signals the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `Incident` record fields (e.g., add `cvssScore`).
- `prompts/vulnerability-scanner.md` — narrow the scanner to a single asset class.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real CVE database.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an incident → report enters `TRIAGING`, then `AWAITING_APPROVAL`, then `MITIGATED` after approval.
2. Workers fail-fast → if either VulnerabilityScanner or ThreatContextAgent times out, the incident enters `DEGRADED` with whichever partial output exists.
3. Eval-event sampling captures one triage decision per cycle and surfaces the score on the App UI.
4. Approval rejection routes the incident to `REJECTED` without executing any mitigation.

## License

Apache 2.0.
