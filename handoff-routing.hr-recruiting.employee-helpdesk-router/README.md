# Akka Sample: Employee Helpdesk Router

A `TopicRouter` agent classifies an incoming employee question into `HR`, `IT`, or `POLICY` and hands the same `ANSWER` task off to the matching specialist agent that owns the response end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with two governance mechanisms: a PII sanitizer that strips employee identifiers before any LLM call, and a before-agent-response guardrail that checks every draft answer against an internal-policy rubric before it reaches the employee.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Integration-tier requirement: **None** — this blueprint runs out of the box. The inbound question stream and the outbound answer surface are both modeled inside the service. When you are ready to connect a real HR, IT ticketing, or policy knowledge system, supply a valid credential for that service using the same key-sourcing options listed above (env var, env file, secrets URI, or type-once) — Akka never writes the value to disk.

## Generate the system

```sh
cp -r ./handoff-routing.hr-recruiting.employee-helpdesk-router  ~/my-projects/employee-helpdesk-router
cd ~/my-projects/employee-helpdesk-router
```

(Optional) Edit `SPEC.md` to add a `FACILITIES` or `LEGAL` category and a matching specialist agent.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **QuestionSimulator** — TimedAction firing every 30 s that drips canned employee questions from a JSONL file into `QuestionQueue`.
- **QuestionQueue** — EventSourcedEntity append-only audit log of every inbound question (captured before redaction).
- **PiiSanitizer** — Consumer that redacts employee ids, email addresses, and phone numbers before any LLM call.
- **TopicRouter** — typed Agent that classifies the sanitized question into `HR`, `IT`, or `POLICY`.
- **HRSpecialist** — AutonomousAgent that owns the `ANSWER` task for HR questions (leave, benefits, payroll, onboarding).
- **ITSpecialist** — AutonomousAgent that owns the `ANSWER` task for IT questions (access, devices, software, incidents).
- **PolicySpecialist** — AutonomousAgent that owns the `ANSWER` task for policy questions (conduct code, compliance rules, procedures).
- **AnswerGuardrail** — typed Agent used as a before-agent-response guardrail to check every draft answer against the internal-policy rubric.
- **HelpWorkflow** — Workflow per question: sanitize → route → answer → guardrail check → publish.
- **QuestionEntity** — EventSourcedEntity holding each question's lifecycle.
- **QuestionView + HelpdeskEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add a `FACILITIES` category for building-access or facilities requests and a matching `FacilitiesSpecialist`.
- `SPEC.md §5` — extend the `Question` record with deployer-specific fields (`employeeLevel`, `department`, `location`).
- `prompts/topic-router.md` — tighten the routing heuristics for your org's naming conventions and ticket taxonomy.
- `prompts/hr-specialist.md` / `prompts/it-specialist.md` / `prompts/policy-specialist.md` — encode your actual policy references, escalation thresholds, and tone guidelines.
- `eval-matrix.yaml` — connect the PII sanitizer to a real redaction service instead of the in-process regex stage.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips an HR-flavoured question → sanitized, routed to `HR`, answered by `HRSpecialist`, guardrail passes, published.
2. Simulator drips an IT question → routed to `IT`, answered by `ITSpecialist`, guardrail passes, published.
3. Simulator drips a policy question → routed to `POLICY`, answered by `PolicySpecialist`, guardrail passes, published.
4. An ambiguous question routes to `UNCLEAR` and lands in `ESCALATED` without invoking any specialist.
5. A draft that cites a specific policy document number that was not in the sanitized question is blocked; the question lands in `BLOCKED` for HR-team review.
6. Every routed question carries a `RouteScore` (1–5) within ~10 s of the routing decision.

## License

Apache 2.0.
