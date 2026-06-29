# Akka Sample: WP Autotagger

A single tag-proposer agent reads a new WordPress post, generates an ordered list of SEO-relevant tags, and applies them to the post via the WordPress REST API. The post body rides into the agent as a task attachment; tags are written back through a scoped tool call after a `before-tool-call` guardrail confirms the operation stays within the allowed scope.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-tool-call` guardrail that validates every CMS write before it executes — confirming the tag list is well-formed, within the allowed count, and targets the correct post.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the WordPress API is stubbed in-process and the agent's tag-write tool call is simulated.

## Generate the system

```sh
cp -r ./single-agent.content-editorial.wp-autotagger  ~/my-projects/wp-autotagger
cd ~/my-projects/wp-autotagger
```

(Optional) Edit `SPEC.md` to point at a different tag vocabulary (e.g., switch from a general SEO tag list to a domain-specific taxonomy like `tech`, `finance`, or `health`).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TagProposerAgent** — an AutonomousAgent that accepts a post body as a task attachment and returns a typed `TagProposal`.
- **TaggingWorkflow** — orchestrates fetch-post → tag → apply per submitted post.
- **PostEntity** — an EventSourcedEntity holding the per-post tagging lifecycle.
- **WpApiConsumer** — a Consumer that subscribes to `PostIngested` events, fetches the full post body from the stub WP API, and emits `PostBodyFetched` back to the entity.
- **TagProposalGuardrail** — a `before-tool-call` guardrail that validates tag-write operations before they execute.
- **PostView + PostEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded post examples for your own (the JSONL file under `src/main/resources/sample-events/seed-posts.jsonl` after generation).
- `SPEC.md §5` — extend `TagProposal` with domain-specific fields (e.g., `primaryCategory`, `targetAudience`, `contentPillar`).
- `prompts/tag-proposer.md` — narrow the agent's role to a specific publishing vertical (a tech blog would constrain tags to software keywords; a news site would constrain to current event taxonomies).
- `eval-matrix.yaml` — add a real WP credential guard by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a post URL → it is fetched → tagged → the tag proposal appears in the UI with the applied tags visible.
2. The agent attempts to write more tags than the configured maximum → the `before-tool-call` guardrail rejects the tool call → the agent revises the proposal within its iteration budget.
3. The agent proposes tags for a post ID that does not match the active job → the guardrail rejects the tool call as a scope violation → the correct post is tagged.
4. A post body containing a CMS API credential token is submitted → the token never appears in the LLM call log; only `[REDACTED-TOKEN]` does.

## License

Apache 2.0.
