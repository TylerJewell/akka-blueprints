# Akka Sample: IT Help Desk

A `ClassifierAgent` reads an incoming IT request, then hands the same `RESOLVE` task to an `AccessSpecialist`, `InfraSpecialist`, or `SoftwareSpecialist` that owns the resolution end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with two governance mechanisms: a credential sanitizer that strips secrets before any LLM call, and a before-tool-call guardrail that gates ticket creation (a write operation) against an IT policy rubric.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound request stream and the outbound ticket surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.ops-automation.it-helpdesk  ~/my-projects/it-helpdesk
cd ~/my-projects/it-helpdesk
```

(Optional) Edit `SPEC.md` to add routing categories (`NETWORKING`, `SECURITY`) or point `RequestSimulator` at a real ticketing source.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RequestSimulator** — TimedAction firing every 30 s that drips canned IT requests from a JSONL file into `RequestQueue`.
- **RequestQueue** — EventSourcedEntity append-only log of every inbound request (audit before redaction).
- **SecretSanitizer** — Consumer that strips API keys, passwords, tokens, and certificate material from request bodies before any LLM call.
- **ClassifierAgent** — typed Agent that classifies the sanitized request into `ACCESS`, `INFRASTRUCTURE`, `SOFTWARE`, or `UNCLEAR`.
- **AccessSpecialist** — AutonomousAgent that owns the `RESOLVE` task for access and identity issues (password resets, permission grants, account lockouts).
- **InfraSpecialist** — AutonomousAgent that owns the `RESOLVE` task for infrastructure issues (network, VPN, hardware, server).
- **SoftwareSpecialist** — AutonomousAgent that owns the `RESOLVE` task for software issues (installs, configuration, OS, application errors).
- **RoutingJudge** — typed Agent used by `RoutingEvalScorer` to grade every classification decision against a 1–5 rubric.
- **TicketGuardrail** — typed Agent used as the before-tool-call guardrail; checks whether a proposed ticket write is permissible under IT policy.
- **HelpdeskWorkflow** — Workflow per request: sanitize → classify → route → resolve → guardrail check → file ticket → publish.
- **RequestEntity** — EventSourcedEntity holding each request's lifecycle.
- **RequestView + HelpdeskEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **RoutingEvalScorer** — Consumer that listens for `RequestClassified` events and writes an inline eval score.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add routing categories (`NETWORKING`, `SECURITY`) and the matching specialist agent.
- `SPEC.md §5` — extend the `Request` record with deployer-specific fields (`urgency`, `assetTag`, `departmentCode`).
- `prompts/classifier-agent.md` — adjust category keywords and the confidence thresholds.
- `prompts/access-specialist.md` / `prompts/infra-specialist.md` / `prompts/software-specialist.md` — encode internal runbook links, escalation paths, SLA commitments.
- `eval-matrix.yaml` — swap the in-process secret regex for a vault-aware sidecar service.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips an access-flavoured request → sanitized, classified `ACCESS`, resolved by `AccessSpecialist`, ticket filed, response published.
2. Simulator drips an infrastructure request → classified `INFRASTRUCTURE`, resolved by `InfraSpecialist`, published.
3. Simulator drips a software request → classified `SOFTWARE`, resolved by `SoftwareSpecialist`, published.
4. An ambiguous request classifies as `UNCLEAR` and terminates in `ESCALATED` without invoking any specialist.
5. A request body containing a password string has the credential stripped before the LLM sees it; the audit record lists `secret` as a detected category.
6. A specialist response that proposes creating a ticket outside IT policy is blocked; the request lands in `BLOCKED` for a technician review.

## License

Apache 2.0.
