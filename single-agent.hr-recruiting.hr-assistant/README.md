# Akka Sample: HR Assistant

An HR assistant agent answers employee questions about HR policies, benefits, and job postings by looking up authoritative records and generating structured, cited responses. Two sanitizers run before the agent sees any data — one strips PII, a second redacts special-category fields (health status, disability, religion, union membership) — and a `before-agent-response` guardrail validates every answer before it leaves the agent loop.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a PII sanitizer, a special-category sanitizer, and an employee-facing output guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the policy corpus and job-posting catalogue live in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.hr-recruiting.hr-assistant  ~/my-projects/hr-assistant
cd ~/my-projects/hr-assistant
```

(Optional) Edit `SPEC.md` to point at a different policy corpus (e.g., switch from the seeded benefits and leave policies to a custom onboarding guide or compensation handbook).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **HrPolicyAgent** — an AutonomousAgent that accepts an HR query plus a query context attachment and returns a typed `PolicyAnswer`.
- **QueryWorkflow** — orchestrates sanitize-wait → answer → verify per submitted query.
- **QueryEntity** — an EventSourcedEntity holding the per-query lifecycle.
- **QuerySanitizer** — a Consumer that subscribes to `QuerySubmitted` events, redacts PII and special-category fields from both the query text and the resolved employee profile, and emits `QuerySanitized` back to the entity.
- **QueryView + QueryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded policy documents for your own (the JSONL file under `src/main/resources/sample-events/policies.jsonl` after generation).
- `SPEC.md §5` — extend `PolicyAnswer` with org-specific fields (e.g., `benefitsPlanCode`, `jurisdiction`, `effectiveDate`).
- `prompts/hr-policy-agent.md` — narrow the agent's role (a healthcare employer might constrain it to FMLA and ADA policy lookups only; a global employer might add jurisdiction routing).
- `eval-matrix.yaml` — wire a real redaction library for special-category fields by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. An employee submits an HR query → it is sanitized → answered → the policy answer appears in the UI with citations.
2. The agent returns a malformed answer on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed, cited answer lands.
3. Every recorded answer has a citation list visible on the same UI card, linking each claim to a policy section.
4. Special-category strings submitted in the query context (health condition, disability status) never appear in the LLM call log; only the redacted form does.

## License

Apache 2.0.
