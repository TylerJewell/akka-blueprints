# Akka Sample: Social Post Team

A pipeline of agents turns a creative concept into a finished Instagram post — caption copy plus visual direction — using simulated web search and page browsing, then pauses for a marketer to approve before the post is marked published.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- An AI provider key: one of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`. If none is set, `/akka:specify` offers a mock-model option that needs no key.
- Host software: none. This sample runs out of the box — the web-search and browsing tools are modeled in-process.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.social-post-team ~/my-projects/social-post-team
cd ~/my-projects/social-post-team
```

(Optional) Edit `SPEC.md` — system name, model provider, brand-voice rules.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` and print the listening URL when the service is up.

## What you'll get

- Three agents: a research agent (web search + page browse tools), a copy agent (caption + hashtags), a visual-director agent (image brief).
- A workflow chaining research → compose → brand check → await approval → publish.
- An event-sourced post entity, a read-model view, and an SSE stream to the UI.
- A concept simulator and a stale-post monitor as scheduled actions.
- Two guardrails (brand/platform check before content emits; URL allowlist before any external fetch) and a human approval gate.
- A single-file UI with five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI.

## Customise before generating

- `SPEC.md` Section 1 — system name and pitch.
- `SPEC.md` Section 11 — model provider and port.
- `prompts/copy-agent.md` and `prompts/visual-director-agent.md` — brand voice, tone, hashtag policy.
- `eval-matrix.yaml` — the brand/platform-policy rules the before-agent-response guardrail enforces.

## What gets validated

- Submitting a concept produces a post that moves through researching and composing to awaiting-approval with a non-empty caption and visual brief.
- Approving a post drives it to published with a permalink.
- Rejecting a post records the reason and ends the pipeline.
- A post left unapproved past the threshold escalates automatically.

See `reference/user-journeys.md`.

## License

Apache 2.0.
