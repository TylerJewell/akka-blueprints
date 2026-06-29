# Akka Sample: ITSM Incident Categorization Agent

An IT incident arrives; one AI agent reads its short description and long description, then returns a structured `ClassificationResult` carrying a category, subcategory, and the affected configuration item (CI). Downstream workflows pick up the result without waiting on a human to route it.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a continuous-accuracy evaluator that tracks how category assignments drift over time, and an on-decision evaluator that spot-checks each classification against a closed-label vocabulary the moment it lands.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the category taxonomy lives in-process and all external ITSM integrations are simulated.

## Generate the system

```sh
cp -r ./single-agent.ops-automation.incident-classifier  ~/my-projects/incident-classifier
cd ~/my-projects/incident-classifier
```

(Optional) Edit `SPEC.md` to swap the seeded category taxonomy for your own (e.g., change the default IT service catalog to match your org's CMDB structure or switch from a generic taxonomy to an industry-specific one).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **IncidentClassifierAgent** — an AutonomousAgent that accepts an incident's description fields as a task attachment and returns a typed `ClassificationResult`.
- **ClassificationWorkflow** — orchestrates ingest-wait → classify → eval per submitted incident.
- **IncidentEntity** — an EventSourcedEntity holding the per-incident lifecycle from submission through classification.
- **VocabularyValidator** — a Consumer that subscribes to `IncidentSubmitted` events, confirms all taxonomy codes in the target scope, and emits `TaxonomyValidated` back to the entity.
- **ClassificationView + ClassificationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded incident catalog for your own (the JSONL file under `src/main/resources/sample-events/incidents.jsonl` after generation).
- `SPEC.md §5` — extend `ClassificationResult` with org-specific fields (e.g., `priority`, `assignmentGroup`, `businessService`).
- `prompts/incident-classifier.md` — narrow the agent's role to your taxonomy depth (a financial-services deployer might restrict to a 3-level ITIL taxonomy; a healthcare deployer to a clinical systems hierarchy).
- `eval-matrix.yaml` — configure the continuous-accuracy evaluator to compare against your own historical ground-truth labels.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an incident description → it is taxonomy-validated → classified → the result appears in the UI.
2. The agent returns a classification with an unrecognised category code → the on-decision evaluator flags it with score 1 → the UI highlights the card for human review.
3. Every classified incident's accuracy contribution is reflected in the continuous-accuracy trend visible on the Eval Matrix tab.
4. A classification result that lands outside the closed label set is never silently accepted; the eval pipeline surfaces the mismatch.

## License

Apache 2.0.
