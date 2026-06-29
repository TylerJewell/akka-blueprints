# Akka Sample: Antom Payment Action Agent

A single payment-action agent receives an instruction to initiate, query, or refund a payment through Ant International's Antom APIs. The agent validates the instruction against authorization rules before invoking any payment tool, holds high-value transactions for operator confirmation, and halts the entire session on a fraud signal — giving operators and regulators a hard stop that no retry path circumvents.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a before-tool-call guardrail that checks authorization before the agent touches any payment API, an application-level human-in-the-loop confirmation gate for payments above a configurable threshold, and an operator/regulator halt that freezes the agent immediately on a detected fraud signal.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The Antom payment API calls are simulated in-process; no live Antom credentials are required to run the blueprint.

## Generate the system

```sh
cp -r ./single-agent.finance-analysis.payment-action-agent  ~/my-projects/antom-payment
cd ~/my-projects/antom-payment
```

(Optional) Edit `SPEC.md` to adjust the high-value threshold (`§3`), the list of permitted payment methods (`§5`), or the fraud-signal patterns the halt mechanism checks (`§8`).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PaymentActionAgent** — an AutonomousAgent that accepts a payment task (currency, amount, recipient, method) and either executes, queries, or refunds via simulated Antom API calls.
- **PaymentWorkflow** — orchestrates auth-check → optional HITL gate → agent execution → result recording per payment instruction.
- **PaymentEntity** — an EventSourcedEntity holding the per-payment lifecycle from `REQUESTED` through `SETTLED` or `HALTED`.
- **FraudSignalConsumer** — a Consumer that subscribes to `FraudSignalDetected` events and triggers an immediate halt on the payment entity.
- **PaymentView + PaymentEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the high-value threshold (default: 10 000 USD-equivalent) or the list of currencies the guardrail permits.
- `SPEC.md §5` — extend `PaymentInstruction` with merchant category codes, cost-centre tags, or FX rate fields relevant to your deployer.
- `prompts/payment-action-agent.md` — narrow the agent's permitted tool calls (a treasury deployer might restrict to `initiatePayment` + `queryPaymentStatus`; a refunds-only deployment would remove `initiatePayment` entirely).
- `eval-matrix.yaml` — point the before-tool-call guardrail at a real authorization service by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A payment instruction below the high-value threshold flows end-to-end: requested → authorized → agent executes → settled.
2. A payment above the threshold pauses at the HITL confirmation gate; the agent does not call the payment tool until the operator approves.
3. A fraud signal halts the session immediately; no further tool calls are made regardless of iteration budget.
4. An instruction naming a disallowed payment method is rejected by the before-tool-call guardrail; the tool call never reaches the simulated Antom API.

## License

Apache 2.0.
