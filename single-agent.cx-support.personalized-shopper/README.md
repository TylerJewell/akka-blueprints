# Akka Sample: Personalized Shopper

A single shopping-recommendation agent reads a shopper's preference profile and a product catalog and returns a structured recommendation set: ranked product suggestions with per-product rationale and a PII-scrubbed evidence trace. The preference profile is forwarded to the agent as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a PII sanitizer that runs before the agent ever sees the shopper's profile, stripping name, contact details, and payment-linked identifiers so only preference signals reach the model.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the product catalog and preference profiles live in-process.

## Generate the system

```sh
cp -r ./single-agent.cx-support.personalized-shopper  ~/my-projects/personalized-shopper
cd ~/my-projects/personalized-shopper
```

(Optional) Edit `SPEC.md` to swap the seeded product catalog (electronics / apparel / home goods) for your own domain's catalog format, or extend `ProductRecommendation` with domain-specific fields such as `subscriptionTier`, `regionalAvailability`, or `bundleId`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ShoppingAdvisorAgent** — an AutonomousAgent that accepts a shopper preference profile as a task attachment and a product catalog snapshot as task instructions and returns a typed `RecommendationSet`.
- **RecommendationWorkflow** — orchestrates sanitize-wait → recommend → eval per submitted session.
- **ShoppingSessionEntity** — an EventSourcedEntity holding the per-session lifecycle.
- **ProfileSanitizer** — a Consumer that subscribes to `SessionStarted` events, strips PII from the preference profile into a `SanitizedProfile`, and emits `ProfileSanitized` back to the entity.
- **RecommendationView + ShoppingEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded product catalog entries for your own (the JSONL file under `src/main/resources/sample-events/catalog.jsonl` after generation).
- `SPEC.md §5` — extend `ProductRecommendation` with domain-specific fields such as `promotionCode`, `subscriptionTier`, or `bundleId`.
- `prompts/shopping-advisor.md` — narrow the agent's role (a luxury-goods deployer would constrain it to high-margin product lines; a B2B deployer to contract-eligible SKUs).
- `eval-matrix.yaml` — wire a real PII redactor (e.g., a presidio-style library) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A shopper submits preferences → the profile is sanitized → the agent returns ranked recommendations → the recommendation set appears in the UI with per-product rationale.
2. Personal identifiers submitted in the profile (email, phone number, loyalty-card number) never appear in the LLM call log; only the redacted form does.
3. A catalog with zero products that match any stated preference yields a `NO_MATCH` recommendation set with a clear explanation card in the UI.
4. Every recommendation set carries a freshness score visible on the session card.

## License

Apache 2.0.
