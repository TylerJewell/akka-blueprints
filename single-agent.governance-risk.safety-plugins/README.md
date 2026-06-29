# Akka Sample: Safety Plugins

A single content-safety agent receives a text payload — a user prompt, a model response, or any string destined for or produced by an LLM — and returns a structured safety decision: ALLOW / BLOCK / REDACT. Before the agent sees the payload, a PII sanitizer strips personal identifiers. After the agent returns its decision, a second guardrail validates the decision structure and category set. Every safety decision is scored by a deterministic evaluator that checks whether the blocking rationale is evidence-backed.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a PII sanitizer that runs before the agent call, a `before-llm-call` input guardrail that inspects the payload for known harmful patterns, an `after-llm-response` output guardrail that validates the agent's structured decision, and an on-decision evaluator that scores each safety decision for rationale quality.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — payloads are submitted in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.governance-risk.safety-plugins  ~/my-projects/safety-plugins
cd ~/my-projects/safety-plugins
```

(Optional) Edit `SPEC.md` to add or remove safety categories from the seeded category list (e.g., add `CSAM` or narrow to only `HATE_SPEECH` and `SELF_HARM` for a specific deployment).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SafetyAgent** — an AutonomousAgent that accepts a payload and a safety profile and returns a typed `SafetyDecision`.
- **SafetyWorkflow** — orchestrates sanitize-wait → screen → eval per submitted payload.
- **SafetyEntity** — an EventSourcedEntity holding the per-screening lifecycle.
- **PayloadSanitizer** — a Consumer that subscribes to `PayloadSubmitted` events, redacts PII from the payload, and emits `PayloadSanitized` back to the entity.
- **SafetyView + SafetyEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded safety profiles for your own (the JSONL file under `src/main/resources/sample-events/safety-profiles.jsonl` after generation).
- `SPEC.md §5` — extend `SafetyDecision` with deployment-specific fields (e.g., `confidenceScore`, `appealable`, `regulatoryCategory`).
- `prompts/safety-agent.md` — narrow the agent's scope (a children's-platform deployer would restrict it to `CSAM`, `SELF_HARM`, and `HATE_SPEECH`; an enterprise internal-tool deployer might focus on `DATA_EXFILTRATION` and `PROMPT_INJECTION`).
- `eval-matrix.yaml` — wire a real PII redactor (e.g., a presidio-style library) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a payload + safety profile → it is sanitized → screened → the decision appears in the UI.
2. The agent returns a malformed decision on first try → the `after-llm-response` guardrail rejects it → the agent retries → a well-formed decision lands.
3. Every recorded decision has an on-decision eval score visible on the same UI card.
4. PII strings in the submitted payload never appear in the LLM call log; only the redacted form does.

## License

Apache 2.0.
