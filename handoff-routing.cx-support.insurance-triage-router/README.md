# Akka Sample: Auto Insurance Agent

A `ClaimTriageAgent` classifies an incoming insurance member request — claim status, policy inquiry, rewards redemption, or roadside assistance dispatch — then hands the same `HANDLE` task off to the matching specialist agent that owns the resolution end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with two governance mechanisms: a PII sanitizer that strips member identifiers before any LLM call, and a before-agent-response guardrail that checks every specialist draft against an insurance-communications policy rubric.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound member request stream and the outbound response surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.insurance-triage-router  ~/my-projects/insurance-triage-router
cd ~/my-projects/insurance-triage-router
```

(Optional) Edit `SPEC.md` to point `RequestSimulator` at a real member-portal intake (IVR feed, mobile app webhook, email gateway) or to change the triage categories.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RequestSimulator** — TimedAction firing every 30 s that drips canned member requests from a JSONL file into `RequestQueue`.
- **RequestQueue** — EventSourcedEntity append-only audit log of every inbound request (audit before redaction).
- **PiiSanitizer** — Consumer that redacts member IDs, policy numbers, VIN numbers, driver's license numbers, phone numbers, and account numbers before any LLM call.
- **ClaimTriageAgent** — typed Agent that classifies the sanitized request into `CLAIM`, `POLICY`, `REWARDS`, `ROADSIDE`, or `UNCLEAR`.
- **ClaimsSpecialist** — AutonomousAgent that owns the `HANDLE` task for claim status and first-notice-of-loss requests.
- **PolicySpecialist** — AutonomousAgent that owns the `HANDLE` task for policy changes, coverage questions, and billing inquiries.
- **RewardsSpecialist** — AutonomousAgent that owns the `HANDLE` task for rewards point redemptions and program inquiries.
- **RoadsideSpecialist** — AutonomousAgent that dispatches roadside assistance and returns a typed confirmation.
- **TriageJudge** — typed Agent used by `TriageEvalScorer` to grade every triage decision against a 1–5 rubric.
- **InsuranceWorkflow** — Workflow per request: sanitize → triage → route → resolve → guardrail check → publish.
- **RequestEntity** — EventSourcedEntity holding each member request's lifecycle.
- **RequestView + InsuranceEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **TriageEvalScorer** — Consumer that listens for `TriageDecided` events and writes an inline eval score.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the triage taxonomy (add `BILLING_DISPUTE`, `SUBROGATION`, etc.) and add the matching specialist agent.
- `SPEC.md §5` — extend the `MemberRequest` record with deployer-specific fields (`tierLevel`, `vehicleYear`, `coverageCode`).
- `prompts/claim-triage-agent.md` — narrow classifier rules (confidence thresholds, roadside-vs-claims ambiguity handling).
- `prompts/claims-specialist.md` / `prompts/roadside-specialist.md` — encode brand voice, authority limits (claim amount caps, dispatch radius), escalation triggers.
- `eval-matrix.yaml` — swap the in-process PII regex for a real redactor or external member-data vault lookup.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a claim-status request → it is sanitized, triaged as `CLAIM`, handed off to `ClaimsSpecialist`, guardrail passes, and a response is published.
2. Simulator drips a roadside request → triaged as `ROADSIDE`, dispatched by `RoadsideSpecialist`, published.
3. An ambiguous or off-topic request triages as `UNCLEAR` and the workflow terminates in `ESCALATED` without any specialist being invoked.
4. A draft that cites a specific claim settlement amount not stated by the member is blocked; the ticket lands in `BLOCKED` for supervisor review.
5. The triage eval score (1–5) and rationale appear on every triaged request within ~10 s of the decision.

## License

Apache 2.0.
