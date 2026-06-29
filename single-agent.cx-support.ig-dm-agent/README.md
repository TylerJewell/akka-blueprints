# Akka Sample: IG DM Agent

A single customer-support agent reads each incoming Instagram direct message and replies with a brand-voice response that has already passed a brand-and-safety guardrail and had any user-shared PII removed before the model ever sees it.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that strips identifiers from the incoming message before the agent call, and a `before-agent-response` guardrail that validates every candidate reply against brand tone and safety rules before the response is dispatched.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the inbound message corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.cx-support.ig-dm-agent  ~/my-projects/ig-dm-agent
cd ~/my-projects/ig-dm-agent
```

(Optional) Edit `SPEC.md` to adjust the brand-voice rules or safety policies (e.g., switch from the seeded retail brand guidelines to a hospitality or SaaS brand).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DmReplyAgent** — an AutonomousAgent that receives a sanitized DM text and brand context and returns a typed `DmReply`.
- **DmWorkflow** — orchestrates sanitize-wait → reply → log per inbound message.
- **DmEntity** — an EventSourcedEntity holding the per-message lifecycle.
- **MessageSanitizer** — a Consumer that subscribes to `MessageReceived` events, strips PII into a `SanitizedMessage`, and emits `MessageSanitized` back to the entity.
- **DmView + DmEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded brand-voice profile for your own (the JSONL file under `src/main/resources/sample-events/brand-profiles.jsonl` after generation).
- `SPEC.md §5` — extend `DmReply` with brand-specific metadata (e.g., `campaignTag`, `escalationTier`, `responseLanguage`).
- `prompts/dm-reply-agent.md` — narrow the agent's role (a fashion brand would constrain it to product-related inquiries; a SaaS brand to support triage).
- `eval-matrix.yaml` — wire a real PII redactor by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an inbound DM → it is sanitized → the agent replies → the reply appears in the UI with status REPLIED.
2. The agent returns a policy-violating draft on its first iteration → the `before-agent-response` guardrail rejects it → the agent retries → a compliant reply lands.
3. PII strings in the inbound message never appear in the LLM call log; only the redacted form does.
4. A reply containing a prohibited phrase (e.g., a competitor brand mention) is caught by the guardrail and never dispatched.

## License

Apache 2.0.
