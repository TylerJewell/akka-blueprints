# Akka Sample: SecOps Vulnerability Triage Agent

A single vulnerability-triage agent reads incoming security findings, enriches them with asset context and threat-intelligence signals, and returns a structured triage verdict: `CRITICAL_IMMEDIATE` / `HIGH_SCHEDULED` / `MEDIUM_MONITORED` / `LOW_ACCEPTED` with a risk rationale and a recommended remediation action. Each finding enters the agent as a task attachment, not as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a pre-tool-call guardrail that blocks any remediation action the agent might propose before an analyst approves it, a human-in-the-loop approval step that gates execution of the approved remediation, and a periodic drift evaluator that flags when the agent's severity distribution has shifted from its calibration baseline.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the vulnerability feed, asset registry, and threat-intel signals live in-process as seeded data; there is no SIEM, scanner, or CMDB dependency.

## Generate the system

```sh
cp -r ./single-agent.governance-risk.secops-triage  ~/my-projects/secops-triage
cd ~/my-projects/secops-triage
```

(Optional) Edit `SPEC.md` to swap the seeded vulnerability feed for your own (e.g., a real CVE stream or scanner output) or to restrict the analyst-approval workflow to a specific remediation category.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **VulnerabilityTriageAgent** — an AutonomousAgent that accepts a raw finding plus asset context as a task attachment and returns a typed `TriageVerdict`.
- **TriageWorkflow** — orchestrates enrich-wait → triage → approval-gate → remediation per submitted finding.
- **FindingEntity** — an EventSourcedEntity holding the per-finding lifecycle.
- **FindingEnricher** — a Consumer that subscribes to `FindingIngested` events, attaches asset context and threat-intel signals, and emits `FindingEnriched` back to the entity.
- **ApprovalEntity** — an EventSourcedEntity holding the analyst's approve/reject decision.
- **FindingView + FindingEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded vulnerability feed for real scanner output (change `src/main/resources/sample-events/findings.jsonl` after generation).
- `SPEC.md §5` — extend `TriageVerdict` with environment-specific fields (e.g., `complianceImpact`, `exploitabilityScore`, `patchAvailability`).
- `prompts/vulnerability-triage-agent.md` — narrow the agent's role (e.g., restrict to cloud-infrastructure CVEs only, or add a specific CVSS floor below which the agent must return `LOW_ACCEPTED` without human review).
- `eval-matrix.yaml` — wire a real CVSS enrichment service or a live threat-intel feed by naming it under the enricher's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A finding is ingested, enriched, and triaged → the analyst sees the verdict and either approves or rejects the remediation.
2. The agent proposes a remediation action → the pre-tool-call guardrail intercepts it and requires analyst approval before the action can execute.
3. A drift evaluator flags that the agent's severity distribution has shifted — the analyst sees the alert in the UI.
4. A rejected remediation leaves the finding in `REMEDIATION_REJECTED` state; the analyst can re-submit with a different action.

## License

Apache 2.0.
