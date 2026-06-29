# Akka Sample: AI-Powered Social Media Amplifier

A single `AmplifierAgent` walks a source article through three task phases — **PARSE → DRAFT → PUBLISH** — wired together by explicit task dependencies. Each phase has its own typed input, typed output, and a small set of phase-specific tools. The user submits an article URL and receives platform-tailored posts published to each configured social channel.

Demonstrates the **sequential-pipeline** coordination pattern wired with two governance mechanisms: a `before-agent-response` guardrail that checks generated post content for brand-policy compliance before the workflow records it, and a `before-tool-call` guardrail that enforces that publishing tools are only callable once the brand-policy check has passed and the posts are approved for each platform.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — every parse / draft / publish tool is implemented in-process inside the same Akka service. Social platform write calls are stubbed with in-process simulators; real credentials are injected by the deployer.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.social-amplifier  ~/my-projects/social-amplifier
cd ~/my-projects/social-amplifier
```

(Optional) Edit `SPEC.md` to point at a different article seed list, a different set of target platforms, or a richer brand-policy ruleset.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AmplifierAgent** — one AutonomousAgent declaring three Task constants (`PARSE_ARTICLE`, `DRAFT_POSTS`, `PUBLISH_POSTS`); the workflow runs them in order, feeding each output forward as the next task's instruction context.
- **AmplificationWorkflow** — runs `parseStep → draftStep → publishStep → evalStep`. Each step calls `runSingleTask` and writes the typed result back onto `AmplificationEntity` before the next step starts.
- **AmplificationEntity** — an EventSourcedEntity holding the per-run lifecycle (`ArticleParsed`, `DraftsProduced`, `PostsPublished`, `BrandCheckScored`).
- **ParseTools / DraftTools / PublishTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that publish tools are only callable once brand-policy approval is recorded.
- **BrandPolicyGuardrail** — two hooks. The `before-agent-response` hook intercepts generated post content and scores it against the brand-policy ruleset before the workflow records drafts. The `before-tool-call` hook blocks any publish tool whose target platform has not yet passed brand review.
- **BrandPolicyScorer** — deterministic, rule-based on-decision evaluator that runs after `PostsPublished` and emits a per-platform publication receipt with a pass/fail per rule.
- **AmplificationView + AmplificationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded article set under `src/main/resources/sample-events/articles.jsonl` to match your demo content calendar.
- `SPEC.md §4` and `prompts/amplifier-agent.md` — constrain the agent's drafting instructions (e.g., limit platforms to LinkedIn + X, adjust character-count rules, add a CTA pattern).
- `SPEC.md §5` — extend the typed outputs (`ParsedArticle`, `DraftSet`, `PublishedSet`) with brand-specific fields. The brand-policy guardrail checks registered rules, not field shapes, so extending records does not require modifying the guardrail class.
- `eval-matrix.yaml` — replace the deterministic brand-policy stub with a vector-similarity check against your own tone corpus by editing the `G1` and `G2` implementation paragraphs.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits an article URL → `PARSE` runs → `DRAFT` runs → `PUBLISH` runs → publication receipts for all platforms land in the UI within ~60 s. Every transition is visible in real time.
2. A draft for one platform fails the `before-agent-response` brand-policy check (forced via the mock LLM) → the guardrail rejects the response → the agent retries within its iteration budget → a compliant draft is eventually recorded → the run completes correctly.
3. A publish tool call fires before the brand-policy check has passed (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the agent retries → the run completes correctly.
4. Each task receives only its own typed inputs; the PARSE task does not see the publishing instructions, and the PUBLISH task does not see the raw article text — the workflow's task-chaining is the only path information travels between phases.

## License

Apache 2.0.
