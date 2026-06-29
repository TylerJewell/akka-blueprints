# Akka Sample: Cross-Agency Case Handoff Mesh

A `JurisdictionRouter` classifies an incoming constituent case, then routes it through a mesh of agency-owned segments — each agency's `AutonomousAgent` processes its portion, a case-owner approves the handoff via a HITL step, and a `JurisdictionGuardrail` verifies the receiving agency's scope before any agent is invoked. Demonstrates the **handoff-routing** coordination pattern with three governance mechanisms: per-agency PII scoping, application-level HITL approval, and a before-agent-invocation jurisdiction guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound case stream and the agency-side processing surfaces are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.public-sector.cross-agency-handoff  ~/my-projects/cross-agency-handoff
cd ~/my-projects/cross-agency-handoff
```

(Optional) Edit `SPEC.md` to point `CaseSimulator` at a real intake channel (a forms API, a case-management webhook) or to add a third agency segment.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CaseSimulator** — TimedAction firing every 30 s that drips canned constituent cases from a JSONL file into `CaseInbox`.
- **CaseInbox** — EventSourcedEntity append-only audit log of every inbound case submission (captured before any redaction).
- **PiiScopingFilter** — Consumer that redacts out-of-scope PII fields before each agency's agent sees the case payload.
- **JurisdictionRouter** — typed Agent that classifies the scoped case into `INTAKE`, `BENEFITS`, or `APPEALS`.
- **IntakeAssessor** — AutonomousAgent that owns the `ASSESS` task for intake-segment cases (eligibility screening, document checklist).
- **BenefitsReviewer** — AutonomousAgent that owns the `REVIEW` task for benefits-segment cases (entitlement calculation, award determination).
- **JurisdictionGuardrail** — typed Agent implementing the before-agent-invocation guardrail: verifies the receiving agency's jurisdiction before any agent call.
- **HandoffWorkflow** — Workflow per case: scope → route → guardrail check → agent segment → HITL approval → re-emit or close.
- **CaseEntity** — EventSourcedEntity holding each case's cross-agency lifecycle.
- **CaseView + CaseEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add a third agency segment (e.g. `APPEALS`) and the matching `AppealsOfficer` AutonomousAgent.
- `SPEC.md §5` — extend the `Case` record with deployer-specific fields (`programCode`, `geographicRegion`, `priorityFlag`).
- `prompts/jurisdiction-router.md` — tighten the routing taxonomy (additional program codes, sub-agency rules).
- `prompts/intake-assessor.md` / `prompts/benefits-reviewer.md` — encode agency-specific business rules, document requirements, entitlement thresholds.
- `eval-matrix.yaml` — swap the in-process PII regex for an agency-approved redaction service; update `integration_tier` accordingly.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips an eligibility-relevant case → PII-scoped, routed to `IntakeAssessor`, case-owner approves the handoff, jurisdiction guardrail passes, assessment published.
2. Simulator drips a benefits case → routed to `BenefitsReviewer`, approved, guardrail passes, award determination published.
3. A case routed outside both agencies' jurisdiction is blocked by the guardrail before any agent is invoked.
4. A case-owner rejects the handoff; the case lands in `HANDOFF_REJECTED` and no downstream agent is invoked.
5. A case that cannot be jurisdiction-mapped routes to `UNROUTABLE` and the workflow terminates without invoking any agency agent.

## License

Apache 2.0.
