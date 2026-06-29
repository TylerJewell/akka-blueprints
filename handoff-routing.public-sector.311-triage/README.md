# Akka Sample: 311 Constituent Triage Agent

A `TriageAgent` classifies an inbound 311 service request, then hands the same `HANDLE` task off to the correct city department specialist that owns the response end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with two governance mechanisms (PII sanitizer before any LLM call, before-agent-invocation guardrail that audits the route decision and flags misroutes for triage review).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound 311 request stream and the department response surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.public-sector.311-triage  ~/my-projects/311-triage
cd ~/my-projects/311-triage
```

(Optional) Edit `SPEC.md` to add more department categories (`PARKS_REC`, `NOISE_COMPLAINT`, etc.) and a matching specialist agent, or to point `RequestSimulator` at a real 311 feed.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RequestSimulator** — TimedAction firing every 30 s that drips canned 311 requests from a JSONL file into `RequestQueue`.
- **RequestQueue** — EventSourcedEntity append-only audit log of every inbound 311 request (captured before redaction).
- **PiiSanitizer** — Consumer that redacts constituent names, addresses, phone numbers, and email addresses before any LLM call.
- **TriageAgent** — typed Agent that classifies the sanitized request into `PUBLIC_WORKS`, `PERMITS_ZONING`, or `UNCLEAR`.
- **RouteGuardrail** — typed Agent used as a before-agent-invocation guardrail; audits the routing decision and returns a `RouteVerdict { approved, flags }` before the department specialist is invoked.
- **PublicWorksSpecialist** — AutonomousAgent that owns the `HANDLE` task for public-works and sanitation requests.
- **PermitsSpecialist** — AutonomousAgent that owns the `HANDLE` task for permits and zoning requests.
- **TriageJudge** — typed Agent used by `TriageEvalScorer` to grade every triage decision against a 1–5 rubric.
- **RequestWorkflow** — Workflow per request: sanitize → triage → route-guardrail → route → handle → publish.
- **ServiceRequestEntity** — EventSourcedEntity holding each request's lifecycle.
- **RequestView + TriageEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **TriageEvalScorer** — Consumer that listens for `TriageDecided` events and writes an inline eval score.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add department categories (`PARKS_REC`, `SANITATION`, `NOISE`) and a matching specialist agent.
- `SPEC.md §5` — extend the `ServiceRequest` record with deployer-specific fields (`priorityLevel`, `gisCoordinates`, `serviceAreaCode`).
- `prompts/triage-agent.md` — adjust category keywords for the specific city's 311 taxonomy.
- `prompts/public-works-specialist.md` / `prompts/permits-specialist.md` — encode the department's response templates and escalation rules.
- `eval-matrix.yaml` — replace the in-process PII regex with a city-approved redactor.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a public-works request → sanitized → triaged `PUBLIC_WORKS` → route guardrail approves → handed to `PublicWorksSpecialist` → response published.
2. Simulator drips a permits request → triaged `PERMITS_ZONING` → route guardrail approves → handed to `PermitsSpecialist` → response published.
3. An ambiguous request triages as `UNCLEAR` and the workflow terminates in `ESCALATED` without ever calling a specialist.
4. A request the route guardrail cannot confidently approve (e.g. a public-works description that is actually a zoning question) is flagged; the request lands in `FLAGGED_FOR_REVIEW` for a human triage reviewer.
5. Every triaged request carries a `TriageScore` (1–5) and rationale within ~10 s of the routing decision.

## License

Apache 2.0.
