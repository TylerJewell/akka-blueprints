# Akka Sample: Guardrails Side-by-Side

A single governance agent evaluates a user prompt before it reaches the model and then validates the model's reply before it returns to the caller. The two guardrail agents — an input screener and an output validator — run as separate steps inside a workflow, directly mirroring the `input_guardrails` / `output_guardrails` pattern from the OpenAI Agents SDK.

Demonstrates the **single-agent** coordination pattern wired with two blocking guardrails: a `before-agent-invocation` input screener that rejects unsafe or out-of-scope prompts before the main agent is called, and a `before-agent-response` output validator that checks the response for policy violations before it is returned to the caller.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the agent conversation lives in-process and the guardrail agents are separate AutonomousAgent instances.

## Generate the system

```sh
cp -r ./single-agent.governance-risk.akka-guardrails-side-by-side  ~/my-projects/guardrails-side-by-side
cd ~/my-projects/guardrails-side-by-side
```

(Optional) Edit `SPEC.md` to substitute your own input-screening policy (Section 8, G1) or output-validation rules (G2). The guardrail agents load their rules from `prompts/input-screener.md` and `prompts/output-validator.md`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PolicyAgent** — an AutonomousAgent that handles user prompts that cleared the input guardrail and have a validated reply from the output guardrail.
- **InputScreenerAgent** — an AutonomousAgent acting as the input guardrail, deciding whether a prompt is safe and in-scope before `PolicyAgent` is invoked.
- **OutputValidatorAgent** — an AutonomousAgent acting as the output guardrail, deciding whether `PolicyAgent`'s reply meets policy rules before the caller receives it.
- **GuardrailWorkflow** — orchestrates inputScreen → agentCall → outputValidate per submitted prompt, with explicit timeouts and retry/fail paths.
- **PromptEntity** — an EventSourcedEntity holding the per-prompt lifecycle from RECEIVED through RESPONDED or BLOCKED.
- **PromptView + PromptEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the three seeded prompt categories (governance-question, off-topic, policy-violating response) for your own scenarios.
- `SPEC.md §5` — extend `ScreeningVerdict` or `ValidationVerdict` with organisation-specific fields (e.g., `riskCategory`, `triggeredRule`).
- `prompts/input-screener.md` — tighten the scope: a financial deployer would constrain the screener to allow only regulatory-inquiry prompts.
- `prompts/output-validator.md` — add domain-specific rules: a healthcare deployer would add PHI-disclosure and clinical-advice checks.
- `eval-matrix.yaml` — add a `regulation_anchor` once your compliance team maps the guardrail checks to a specific framework.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a well-formed governance question → input screener passes → `PolicyAgent` answers → output validator passes → the response appears in the UI.
2. A user submits a prompt flagged as off-topic by the input screener → the prompt is blocked at `inputScreenStep`; `PolicyAgent` is never invoked; the UI shows BLOCKED with the screener's reason.
3. `PolicyAgent` produces a response that the output validator flags as a policy violation → the response is blocked at `outputValidateStep`; the caller receives the block reason rather than the bad reply.
4. Both guardrail agents log their PASS/BLOCK verdict alongside every prompt, giving a complete audit trail even for blocked requests.

## License

Apache 2.0.
