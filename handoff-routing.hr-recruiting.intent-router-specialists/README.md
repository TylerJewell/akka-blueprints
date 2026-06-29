# Akka Sample: Core Semantic Router

An `IntentRouterAgent` classifies an inbound HR or Finance query, then hands the same `ANSWER` task off to an `HrSpecialist` or `FinanceSpecialist` that owns the response end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with two governance mechanisms: a PII sanitizer that strips employee identifiers and financial account data before any LLM call, and a before-agent-invocation guardrail that validates the routing classification and confirms the caller is authorized for the target specialist domain before that specialist runs.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound query stream and the outbound response surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.hr-recruiting.intent-router-specialists  ~/my-projects/core-semantic-router
cd ~/my-projects/core-semantic-router
```

(Optional) Edit `SPEC.md` to point `QuerySimulator` at a real HR or Finance system (Workday, SAP SuccessFactors, Concur) or to extend the intent taxonomy with additional domains such as `IT-HELPDESK` or `LEGAL`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **QuerySimulator** — TimedAction firing every 30 s that drips canned HR and Finance queries from a JSONL file into `QueryQueue`.
- **QueryQueue** — EventSourcedEntity append-only log of every inbound query (audit record before redaction).
- **PiiSanitizer** — Consumer that redacts employee IDs, SSNs, salary figures, and account references before any LLM call.
- **IntentRouterAgent** — typed Agent that classifies the sanitized query into `HR`, `FINANCE`, or `AMBIGUOUS`.
- **HrSpecialist** — AutonomousAgent that owns the `ANSWER` task for HR queries (policies, benefits, onboarding, leave, performance).
- **FinanceSpecialist** — AutonomousAgent that owns the `ANSWER` task for Finance queries (payroll, reimbursements, budgets, expense reports).
- **RoutingGuardrail** — typed Agent used as the before-agent-invocation check; validates routing classification and caller authorization before the specialist is invoked.
- **RoutingJudge** — typed Agent invoked by `RoutingEvalScorer` to score each routing decision on a 1–5 rubric.
- **RoutingEvalScorer** — Consumer that listens for `IntentRouted` events and calls `RoutingJudge` to grade every routing decision.
- **RouterWorkflow** — Workflow per query: sanitize → guardrail-check → route → answer → publish.
- **QueryEntity** — EventSourcedEntity holding each query's lifecycle.
- **QueryView + RouterEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — extend the intent taxonomy (add `IT-HELPDESK`, `LEGAL`, etc.) and add matching specialist agents.
- `SPEC.md §5` — add deployer-specific fields to the `Query` record (`employeeTier`, `departmentCode`, `urgencyFlag`).
- `prompts/intent-router-agent.md` — tighten classification rules (keyword lists, confidence thresholds, tie-break rules).
- `prompts/hr-specialist.md` / `prompts/finance-specialist.md` — encode policy-document references, authorization limits, escalation triggers.
- `eval-matrix.yaml` — swap the in-process PII regex for a real redactor (a compliance-approved masking service via env var).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips an HR-flavoured query → sanitized, classified `HR`, guardrail passes, `HrSpecialist` answers, response published.
2. Simulator drips a Finance query → classified `FINANCE`, `FinanceSpecialist` answers, response published.
3. An ambiguous query classifies as `AMBIGUOUS` and the workflow terminates in `ESCALATED` without invoking any specialist.
4. A query whose routing classification fails the before-agent-invocation guardrail (low confidence or unauthorized caller) is blocked before any specialist runs.
5. The routing eval score (1–5) and rationale appear on every routed query within ~10 s of the routing decision.

## License

Apache 2.0.
