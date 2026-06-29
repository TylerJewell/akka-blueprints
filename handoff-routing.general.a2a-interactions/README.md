# Akka Sample: Agent-to-Agent Interactions Sample

A `DelegatorAgent` receives a task over a peer interactions API, verifies the receiver agent's identity, then hands the same task off to a `ReceiverAgent` that owns the resolution end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with two governance mechanisms (a before-agent-invocation guardrail that checks receiver identity before any handoff, and an on-handoff eval that scores every delegation decision against a correctness rubric).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound interaction stream and the outbound resolution surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.general.a2a-interactions  ~/my-projects/a2a-interactions
cd ~/my-projects/a2a-interactions
```

(Optional) Edit `SPEC.md` to point `InteractionSimulator` at a real A2A message bus or to change the task taxonomy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **InteractionSimulator** — TimedAction firing every 30 s that drips canned A2A interaction payloads from a JSONL file into `InteractionQueue`.
- **InteractionQueue** — EventSourcedEntity append-only log of every inbound interaction (audit before routing).
- **IdentityVerifier** — Consumer that checks the sender's claimed agent identity and capability declaration before any handoff is initiated.
- **DelegatorAgent** — typed Agent that matches the incoming task against a registered `ReceiverAgent` and emits a `DelegationDecision`.
- **ReceiverAgent** — AutonomousAgent that owns the `FULFILL` task after the handoff and produces a typed `Fulfillment`.
- **HandoffJudge** — typed Agent used by `HandoffEvalScorer` to grade every delegation decision against a 1–5 rubric.
- **InteractionWorkflow** — Workflow per interaction: verify → delegate → fulfill → publish.
- **InteractionEntity** — EventSourcedEntity holding each interaction's lifecycle.
- **InteractionView + InteractionEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **HandoffEvalScorer** — Consumer that listens for `DelegationDecided` events and writes an inline eval score.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the task taxonomy (add `QUERY`, `REPORT`, etc.) and add matching receiver agents.
- `SPEC.md §5` — extend the `Interaction` record with deployer-specific fields (`senderOrg`, `priorityLevel`, `expiresAt`).
- `prompts/delegator-agent.md` — narrow the routing rules (default-to-UNROUTABLE thresholds, capability-matching keywords).
- `prompts/receiver-agent.md` — encode resolution depth limits, tool-call permissions, escalation triggers.
- `eval-matrix.yaml` — swap the in-process identity verifier for a real registry (e.g., an external agent directory — make it a real-service-via-env-var if you do).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a routable interaction → identity is verified, delegated to `ReceiverAgent`, and a fulfillment is published.
2. An interaction whose sender fails identity verification is rejected before any handoff; the interaction lands in `REJECTED`.
3. An interaction whose task type does not match any registered receiver routes to `UNROUTABLE` without invoking `ReceiverAgent`.
4. A fulfillment that violates the before-agent-invocation guardrail (e.g. the receiver agent was not the registered handler for the task type) is blocked; the interaction lands in `BLOCKED` for human review.
5. The eval score (1–5) and rationale appear on every delegated interaction within a few seconds of the delegation decision.

## License

Apache 2.0.
