# Akka Sample: Policy-as-Code

A single policy-enforcement agent evaluates a proposed change — infrastructure, configuration, or code — against a set of encoded compliance policies and returns a structured decision: ALLOW / DENY / WARN with a violation finding per policy rule. The change payload rides into the agent as a task attachment, not as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that intercepts every external lookup the agent initiates and validates it against an allowlist, and a CI gate that enforces policy evaluation is required before any change reaches the deploy pipeline.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the policy corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.governance-risk.policy-as-code  ~/my-projects/policy-as-code
cd ~/my-projects/policy-as-code
```

(Optional) Edit `SPEC.md` to point at a different policy set (e.g., switch from the seeded infrastructure-change rulebook to a software-supply-chain policy set or an API-versioning governance policy).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PolicyEnforcementAgent** — an AutonomousAgent that accepts a change request plus encoded policy rules (passed as a task attachment) and returns a typed `PolicyDecision`.
- **EvaluationWorkflow** — orchestrates validate-wait → enforce → audit per submitted change.
- **ChangeRequestEntity** — an EventSourcedEntity holding the per-change lifecycle.
- **ChangeValidator** — a Consumer that subscribes to `ChangeSubmitted` events, validates the change metadata, and emits `ChangeValidated` back to the entity.
- **PolicyView + PolicyEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded policy rulebook for your own (the JSONL file under `src/main/resources/sample-events/policy-rules.jsonl` after generation).
- `SPEC.md §5` — extend `PolicyDecision` with deployment-specific fields (e.g., `environment`, `changeCategory`, `ruleSetVersion`).
- `prompts/policy-enforcement-agent.md` — narrow the agent's role (an infrastructure team would constrain it to Terraform plan diffs; a platform team to Kubernetes manifest changes).
- `eval-matrix.yaml` — wire the CI gate to a real pipeline trigger by naming the CI integration under the gate mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a change request + policy rules → it is validated → enforced → the decision appears in the UI.
2. The agent attempts a disallowed tool call → the `before-tool-call` guardrail blocks it → the agent continues with the permitted data → a compliant decision lands.
3. A change request submitted with no matching policy rules receives a WARN decision with a clear rationale visible in the UI.
4. A DENY decision triggers the CI gate to block deployment; the gate's response is visible on the same change card.

## License

Apache 2.0.
