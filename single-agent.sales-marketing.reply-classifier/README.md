# Akka Sample: Reply Classifier

A single reply-classifier agent reads an inbound sales-email reply and returns a structured classification: `INTERESTED`, `NOT_INTERESTED`, `OBJECTION`, `OUT_OF_OFFICE`, or `UNSUBSCRIBE`, together with a confidence score and a short rationale. On classification, the system calls the Pipedrive CRM API to update the deal's stage — gated by a `before-tool-call` guardrail that rejects unsafe CRM mutations before they execute.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that validates every proposed CRM mutation against a safe-action allowlist before the agent's tool call reaches Pipedrive.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The Pipedrive integration is simulated in-process; no live Pipedrive account is required for local development. A deployer wiring a real Pipedrive account supplies the API token at run time via the same key-sourcing mechanism.

## Generate the system

```sh
cp -r ./single-agent.sales-marketing.reply-classifier  ~/my-projects/reply-classifier
cd ~/my-projects/reply-classifier
```

(Optional) Edit `SPEC.md` to point at a different set of seeded email replies or to swap in a real Pipedrive API token reference.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReplyClassifierAgent** — an AutonomousAgent that accepts a raw email reply as a task attachment, classifies it, proposes a Pipedrive stage update, and returns a typed `ReplyClassification`.
- **ClassificationWorkflow** — orchestrates ingest-wait → classify → update-crm per submitted reply.
- **ReplyEntity** — an EventSourcedEntity holding the per-reply lifecycle.
- **CrmMutationGuardrail** — a `before-tool-call` guardrail wired on `ReplyClassifierAgent` that validates every proposed stage-update call against the deal's current stage and the allowed-transitions map before the call executes.
- **ReplyView + ReplyEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded reply corpus for your own (the JSONL file under `src/main/resources/sample-events/seed-replies.jsonl` after generation).
- `SPEC.md §5` — extend `ReplyClassification` with deal-specific fields (e.g., `followUpDate`, `assignedRep`, `campaignId`).
- `prompts/reply-classifier.md` — narrow the agent's classification taxonomy to match your sales motion (e.g., add `DEMO_REQUESTED` or `PRICING_QUERY` categories).
- `eval-matrix.yaml` — wire the guardrail to a real Pipedrive stage-transition policy by updating the `implementation` paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an inbound reply → it is classified → the deal stage updates → the classification card appears in the UI.
2. The agent proposes an invalid stage transition (e.g., moving a `CLOSED_WON` deal to `PROSPECTING`) → the `before-tool-call` guardrail rejects the mutation → the agent receives a structured rejection error → it either corrects the call or marks the update as skipped.
3. An `UNSUBSCRIBE` reply causes a `SKIP_CRM_UPDATE` action — the guardrail never fires because no mutation is proposed, but the entity records the signal for the sales rep.
4. Every classified reply's full raw text is preserved on the entity for audit; the UI shows only the classification outcome and rationale.

## License

Apache 2.0.
