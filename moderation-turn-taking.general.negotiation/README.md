# Akka Sample: Negotiation Facilitator

A Facilitator runs up to ten rounds of offers and counteroffers between a Buyer and a Seller until terms converge, then returns the outcome and the final offer.

This folder is a **blueprint** — a set of natural-language and YAML inputs. You run `/akka:specify` against it and Claude generates a working Akka project. The blueprint itself contains no Java, no `pom.xml`, and no built UI.

## Prerequisites

- **Claude Code** with the **Akka plugin** installed. Install docs: <https://doc.akka.io/>.
- **One model-provider API key**, sourced however you prefer. When you run `/akka:specify`, Claude detects which of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` is set and wires it. If none is set, Claude asks how you want to source the key and offers five options:
  - a mock provider that returns schema-valid offers with no key,
  - name an existing environment variable,
  - point at an env file you already maintain,
  - a secrets-store reference (`1password://…`, `aws-secretsmanager://…`, `vault://…`),
  - type the key once into the session.
  The key value is never written to any file Claude creates — only the reference is recorded.
- **Host software:** none. This blueprint runs out of the box; the Buyer, the Seller, and the market are all modeled inside the one service.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` — the system name, the model provider, or the negotiation defaults (item, buyer budget, seller floor, convergence tolerance, round cap).
3. In Claude Code, from inside the folder, run:

   ```
   /akka:specify @SPEC.md
   ```

That is the only command you type. `SPEC.md` Section 12 instructs Claude to continue through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` on its own and to print the listening URL when the service is up.

## What you'll get

- **BuyerAgent** and **SellerAgent** — two agents that each produce a typed offer per turn, held to their own constraints.
- **FacilitatorAgent** — the moderator that reads both latest offers each round and decides continue, converged, or no deal, proposing final terms on convergence.
- **NegotiationWorkflow** — the durable turn-taking loop that alternates Buyer → Seller → Facilitator for up to ten rounds.
- **NegotiationEntity** — the event-sourced record of every offer and the concluded outcome.
- **OutcomeEvaluator** — scores each concluded negotiation (surplus split, rounds used) when it ends.
- **NegotiationsView**, a request queue, a request simulator, a stalled-negotiation watch, an operator halt switch, and the two HTTP endpoints that serve the API and the embedded five-tab UI.

## Customise before generating

The parts of `SPEC.md` most worth editing before you run `/akka:specify`:

- **System name** — Section 1, plus the UI title in Section 7.
- **Model provider** — Section 11's identity block; or just set the matching env var and let Claude detect it.
- **Negotiation defaults** — Section 5 and `reference/data-model.md`: the convergence tolerance, the ten-round cap, and the canned scenarios in the request simulator.
- **Agent behavior** — the three files under `prompts/` carry the Buyer, Seller, and Facilitator system prompts.

## What gets validated

The generated system is correct when the journeys in `reference/user-journeys.md` pass:

1. Submitting a negotiation request starts a workflow and the Buyer's first offer appears within seconds.
2. A negotiation whose budget exceeds the floor converges to a deal with a final price between the seller floor and the buyer budget.
3. A negotiation whose budget sits below the floor ends in `NO_DEAL` at or before round ten.
4. Every concluded negotiation receives an outcome score from the evaluator.

## License

Apache 2.0.
