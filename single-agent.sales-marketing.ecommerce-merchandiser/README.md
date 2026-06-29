# Akka Sample: Merchandiser

A single merchandising agent reads product data, catalog context, and a merchant's objective, then proposes storefront changes — product descriptions, promotion rules, category rankings, and pricing nudges — as a structured `MerchandisingProposal`. Every tool call that would write to the live storefront is screened by a `before-tool-call` guardrail, and every proposal that passes that screen waits for explicit merchant approval before it is published.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that blocks unvalidated writes to the live catalog, and an application-level human-in-the-loop approval gate that holds each proposal in `PENDING_APPROVAL` until a merchant acts on it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid credential for the integrated storefront service (e.g., a read/write API token for the ecommerce platform). Supply it using the same key-sourcing options described above — an env var, env file, secrets-store URI, or type-once in session. Akka never writes the token value to disk.

## Generate the system

```sh
cp -r ./single-agent.sales-marketing.ecommerce-merchandiser  ~/my-projects/merchandiser
cd ~/my-projects/merchandiser
```

(Optional) Edit `SPEC.md` to point at a different product catalog seed (e.g., switch from the general-retail seed to an apparel or electronics catalog) or to change the approval gate timeout.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **MerchandiserAgent** — an AutonomousAgent that accepts a merchandising objective and catalog context, calls storefront tools (read: fetch product, search catalog; write: update description, set promotion, reorder category), and returns a typed `MerchandisingProposal`.
- **ProposalWorkflow** — orchestrates generate → await-approval → publish per submitted objective.
- **ProposalEntity** — an EventSourcedEntity holding the per-proposal lifecycle.
- **CatalogReader** — a Consumer that subscribes to `ObjectiveSubmitted` events, fetches current catalog context from the seed data, and emits `CatalogContextLoaded` back to the entity.
- **ProposalView + ProposalEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded product catalog for your own (the JSONL file under `src/main/resources/sample-events/catalog-seed.jsonl` after generation).
- `SPEC.md §5` — extend `MerchandisingProposal` with domain-specific fields (e.g., `targetSegment`, `marginImpact`, `channelScope`).
- `prompts/merchandiser-agent.md` — narrow the agent's role (a fashion deployer would constrain it to seasonal-collection promotions; a grocery deployer to expiry-driven markdown rules).
- `eval-matrix.yaml` — wire a real storefront API by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A merchant submits an objective → catalog context is loaded → the agent generates a proposal → the proposal appears in `PENDING_APPROVAL` state in the UI.
2. The agent attempts a direct write-tool call that fails the `before-tool-call` guardrail → the guardrail blocks it and returns a structured error → the agent retries with a read-first pattern → the proposal is generated without the blocked call.
3. A merchant approves the proposal → the publish step fires and the entity transitions to `PUBLISHED`.
4. A merchant rejects the proposal → the entity transitions to `REJECTED` and the raw agent output is preserved for audit.

## License

Apache 2.0.
