# Akka Sample: Support Ticket Next Steps

A single resolution-advisor agent reads a support ticket against a library of past resolutions and returns a ranked list of recommended next steps: the top resolution path, an explanation grounded in historical precedent, and a confidence indicator per recommendation. The ticket rides into the agent as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that runs before the agent ever sees the ticket body, and an on-decision evaluator that scores every recommendation set for grounding quality — are the suggestions actually backed by past resolutions or are they free-form guesses?

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the resolution library lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.cx-support.support-next-steps  ~/my-projects/support-next-steps
cd ~/my-projects/support-next-steps
```

(Optional) Edit `SPEC.md` to point at a different resolution library (e.g., swap the seeded customer-tier-specific resolution sets for your own product-area taxonomy).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResolutionAdvisorAgent** — an AutonomousAgent that accepts ticket context plus a resolution-library attachment and returns a typed `RecommendationSet`.
- **TicketWorkflow** — orchestrates sanitize-wait → advise → eval per submitted ticket.
- **TicketEntity** — an EventSourcedEntity holding the per-ticket recommendation lifecycle.
- **TicketSanitizer** — a Consumer that subscribes to `TicketSubmitted` events, redacts PII into a `SanitizedTicket`, and emits `TicketSanitized` back to the entity.
- **TicketView + TicketEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded resolution library for your product's knowledge base (the JSONL file under `src/main/resources/sample-events/resolution-library.jsonl` after generation).
- `SPEC.md §5` — extend `RecommendationSet` with domain-specific fields (e.g., `escalationPath`, `productArea`, `estimatedResolutionMinutes`).
- `prompts/resolution-advisor.md` — narrow the agent's scope to a specific product surface or customer tier.
- `eval-matrix.yaml` — wire a production PII redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a ticket → it is sanitized → the agent recommends steps → the recommendation appears in the UI with a confidence indicator.
2. A ticket whose recommendation set has no grounding citations receives an eval score of 1 and the card is flagged.
3. PII strings submitted in the ticket body never appear in the LLM call log; only the redacted form does.
4. The live list updates in real time over SSE as each ticket moves through its lifecycle.

## License

Apache 2.0.
