# Akka Sample: Gitty

A single-agent CLI tool that reads a GitHub issue or pull-request thread and produces a draft reply for the maintainer to review and edit before posting. The draft never reaches GitHub automatically — the maintainer is the final gate.

Demonstrates the **single-agent** coordination pattern in the developer-tooling domain. One `ReplyDrafterAgent` (AutonomousAgent) authors the draft; a `before-agent-response` guardrail screens it for tone and off-topic content before it leaves the agent loop; the maintainer's explicit approval is required before the draft is posted (advisory HITL).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid GitHub personal access token with `repo` scope (for reading issues/PRs and posting comments). Supply it the same way as the LLM key: an existing shell env var (`GITHUB_TOKEN`), an env file, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the token value to disk.

## Generate the system

```sh
cp -r ./single-agent.dev-code.gh-draft-replier  ~/my-projects/gitty
cd ~/my-projects/gitty
```

(Optional) Edit `SPEC.md §3` to change the seeded GitHub repositories used for demonstration examples, or adjust `SPEC.md §7` to tune the App UI layout.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ReplyDrafterAgent** — an AutonomousAgent that receives a GitHub thread (issue or PR) as a task attachment and returns a typed `DraftReply`.
- **DraftWorkflow** — orchestrates fetch → draft → HITL-queue per submitted thread.
- **DraftEntity** — an EventSourcedEntity holding the per-draft lifecycle.
- **ThreadFetcher** — a Consumer that subscribes to `ThreadQueued` events, calls the GitHub API to pull the full thread, and emits `ThreadFetched` back to the entity.
- **DraftView + DraftEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded GitHub repository examples for your own org/repo slugs.
- `SPEC.md §5` — extend `DraftReply` with project-specific fields (e.g., `labelSuggestions`, `closingRecommendation`).
- `prompts/reply-drafter.md` — narrow the agent's tone policy (a project with strict code-of-conduct language would add those rules here).
- `eval-matrix.yaml` — replace the built-in tone-check guardrail with a call to your content-policy endpoint by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A maintainer submits a GitHub issue URL → the thread is fetched → a draft reply appears in the HITL queue in the UI.
2. The agent returns a draft that fails the tone check → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed, on-topic draft is queued.
3. A maintainer approves a draft → the UI marks it APPROVED (posting to GitHub is a deployer-wired step, not performed by the sample).
4. A maintainer discards a draft → the entity records DISCARDED; the draft is removed from the queue.

## License

Apache 2.0.
