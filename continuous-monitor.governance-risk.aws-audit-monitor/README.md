# Akka Sample: AWS Audit Monitor

A continuous background worker scans an AWS environment for misconfigurations and policy violations, enriches each finding with risk context via an AI agent, and holds every compliance report for risk-officer review before it is published. Demonstrates the **continuous-monitor** coordination pattern wired with two governance mechanisms (deployer-runtime HITL and periodic eval of audit accuracy).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier host software: **None** — the AWS scanner is simulated in-process. Wire a real AWS SDK call in `AuditScanPoller` when deploying to a live account.

## Generate the system

```sh
cp -r ./continuous-monitor.governance-risk.aws-audit-monitor  ~/my-projects/aws-audit-monitor
cd ~/my-projects/aws-audit-monitor
```

(Optional) Edit `SPEC.md` to point `AuditScanPoller` at a real AWS account (replace the in-process simulator with an AWS SDK call) or to keep the built-in canned-findings simulator.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md`'s Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AuditScanPoller** — TimedAction firing every 60 s that pulls simulated AWS findings from a canned dataset.
- **FindingNormalizer** — Consumer that normalizes raw AWS findings into a canonical `NormalizedFinding` form before any LLM sees them.
- **FindingAnalystAgent** — Agent (typed) that assigns severity, maps the finding to a control framework, and drafts a remediation recommendation.
- **ReportCompilerAgent** — AutonomousAgent that groups per-finding analyses into a structured audit report.
- **AuditFindingEntity** — EventSourcedEntity holding each finding's lifecycle (detected → normalized → analyzed → compiled → pending-review → published/dismissed).
- **AuditReportEntity** — EventSourcedEntity accumulating findings into a draft report awaiting risk-officer sign-off.
- **AuditView + AuditEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **AccuracyEvalRunner** — TimedAction running every 4 hours; samples published findings, scores AI analysis accuracy.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the simulated scanner for a real AWS SDK call.
- `SPEC.md §5` — extend the `AwsFinding` record with account-specific fields (`awsAccountId`, `region`, `remediationOwner`).
- `prompts/finding-analyst.md` — narrow the analyst to a specific control framework (CIS, NIST, SOC 2).
- `eval-matrix.yaml` — add a regulation anchor when deploying under a specific compliance regime.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. The scanner produces a finding → it is normalized → analyzed → compiled into a pending report within 60 s.
2. A risk officer clicks Publish → the report transitions to PUBLISHED (simulated; no real outbound).
3. A risk officer clicks Dismiss with a reason → the finding transitions to DISMISSED.
4. The accuracy eval runner scores at least one published finding within 4 hours of system start; the score appears in the UI.

## License

Apache 2.0.
