# Akka Sample: Routing Workflow

A `RoutingClassifier` reads an incoming customer message and routes it to one of three specialist agents — `GeneralAgent`, `RefundAgent`, or `TechnicalAgent` — that owns the reply end-to-end. Two governance mechanisms layer on top: a before-invocation guardrail validates the route decision against the allowed-route registry before any specialist is called, and a before-response guardrail screens the specialist's draft reply before it reaches the customer.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound message stream and the outbound reply surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.routing-classifier-pattern  ~/my-projects/routing-workflow
cd ~/my-projects/routing-workflow
```

(Optional) Edit `SPEC.md` to add extra route categories (`ACCOUNT`, `SHIPPING`) and matching specialist agents, or to tighten the classifier's confidence threshold for defaulting to `UNROUTABLE`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MessageSimulator** — TimedAction firing every 30 s that drips canned customer messages from a JSONL file into `MessageQueue`.
- **MessageQueue** — EventSourcedEntity append-only log of every inbound message (audit before any processing).
- **RoutingClassifier** — typed Agent that classifies the raw message into `GENERAL`, `REFUND`, `TECHNICAL`, or `UNROUTABLE`, then produces a `RouteDecision { route, confidence, reason }`.
- **RouteGuardrail** — typed Agent that validates the `RouteDecision` against the allowed-route registry before any specialist is invoked (before-agent-invocation guardrail).
- **GeneralAgent** — AutonomousAgent that owns the `REPLY` task for general enquiries.
- **RefundAgent** — AutonomousAgent that owns the `REPLY` task for refund and billing requests.
- **TechnicalAgent** — AutonomousAgent that owns the `REPLY` task for technical and integration questions.
- **ReplyGuardrail** — typed Agent that screens the specialist's draft reply before it is published (before-agent-response guardrail).
- **RoutingWorkflow** — Workflow per message: classify → validate-route → route → reply → screen → publish.
- **MessageEntity** — EventSourcedEntity holding each message's lifecycle.
- **MessageView + RoutingEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add route categories (`ACCOUNT`, `SHIPPING`) and matching specialist agents.
- `SPEC.md §5` — extend the `Message` record with fields like `customerId`, `locale`, `channelPriority`.
- `prompts/routing-classifier.md` — adjust confidence thresholds and taxonomy definitions.
- `prompts/route-guardrail.md` — extend the allowed-route registry or add route-transition rules.
- `prompts/general-agent.md` / `prompts/refund-agent.md` / `prompts/technical-agent.md` — encode brand voice, authority limits, and escalation triggers per specialist.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a general-enquiry message → classifier routes to `GENERAL` → route guardrail allows → `GeneralAgent` produces a draft → reply guardrail passes → reply published.
2. Simulator drips a refund-flavoured message → classifier routes to `REFUND` → `RefundAgent` resolves it → published.
3. Simulator drips a technical message → classifier routes to `TECHNICAL` → `TechnicalAgent` resolves it → published.
4. An ambiguous message is classified `UNROUTABLE` and the workflow terminates in `ABANDONED` without calling any specialist.
5. The route guardrail rejects an invalid route decision before the specialist is ever invoked; message lands in `ROUTE_BLOCKED`.
6. A specialist draft that fails the reply guardrail (e.g. promises a specific refund date outside policy) is blocked; message lands in `REPLY_BLOCKED` for operator review.

## License

Apache 2.0.
