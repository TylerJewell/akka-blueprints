# Akka Sample: GitWiki Demo Agent

A single wiki-editor agent accepts page-update requests from a LocalWiki instance, applies them to the in-process wiki state, and commits and pushes each change to a GitHub repository as the canonical source of truth. Every `git push` is gated by a `before-tool-call` guardrail; the repo token and target branch are validated at startup by a configuration gate.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that intercepts every push request to verify the target ref and actor before any write reaches the remote, and a configuration gate enforced at startup to attest that the repo token is present and the target branch is declared.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- A valid GitHub personal access token (PAT) with `repo` scope for the target repository. Supply it using the same key-sourcing options as the LLM key: an existing shell env var (`GITHUB_TOKEN`), an env file, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the token value to disk.

## Generate the system

```sh
cp -r ./single-agent.content-editorial.git-backed-wiki-agent  ~/my-projects/gitwiki
cd ~/my-projects/gitwiki
```

(Optional) Edit `SPEC.md` to point at a different GitHub repository by updating the `repo_owner`, `repo_name`, and `default_branch` fields in `risk-survey.yaml` before scaffolding.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **WikiEditorAgent** — an AutonomousAgent that receives a page-update task (title, body, author) and produces a `CommitOutcome` describing the resulting commit SHA and any merge conflicts.
- **WikiUpdateWorkflow** — orchestrates validate-config → apply-edit → push for each page-update request.
- **PageEntity** — an EventSourcedEntity holding per-page lifecycle from draft through committed.
- **GitPushConsumer** — a Consumer that subscribes to `CommitReady` events, calls the `GitPushGuardrail` before executing the push, and emits `PushCompleted` or `PushRejected` back to the entity.
- **PageView + WikiEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded wiki pages (the JSONL file under `src/main/resources/sample-events/seed-pages.jsonl` after generation) to match your own knowledge base.
- `SPEC.md §5` — extend `CommitOutcome` with additional fields (e.g., `reviewUrl`, `ciRunId`, `mergeStrategy`).
- `prompts/wiki-editor.md` — narrow the agent's role for a specific editorial policy (a technical-documentation team might enforce a strict heading hierarchy; a policy wiki might require citations for every factual claim).
- `eval-matrix.yaml` — wire a real GitHub Actions CI check by naming the workflow file under the configuration-gate's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a page update → the agent applies it → a commit is pushed to the target branch → the commit SHA appears in the UI.
2. The guardrail intercepts a push attempt targeting a protected branch → the push is blocked → `PushRejected` lands on the entity → the UI shows the rejection reason.
3. The system starts without `GITHUB_TOKEN` in the environment → the configuration gate fails at startup with a clear message → the service does not accept update requests.
4. Two concurrent updates to the same page arrive → the second update detects a non-fast-forward condition → the workflow surfaces a `CONFLICT` outcome in the UI.

## License

Apache 2.0.
