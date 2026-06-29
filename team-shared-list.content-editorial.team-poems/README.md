# Akka Sample: Team Poems Multi-Agent

A poetry-director agent breaks a writing prompt into verse assignments on a shared board; poet agents claim stanzas on their own, draft lines, pass a quality gate, and message each other when they need to coordinate tone or meter. Demonstrates the **team-shared-list** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the verse workspace, the quality gate, and the prompt intake are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./team-shared-list.content-editorial.team-poems  ~/my-projects/team-poems-multi-agent
cd ~/my-projects/team-poems-multi-agent
```

(Optional) Edit `SPEC.md` to change the system name, the number of poet agents, the model provider, or the writing prompts the simulator drips.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **PoetryDirector** — AutonomousAgent that decomposes a writing prompt into a list of verse assignments with ordering constraints.
- **PoetAgent** — AutonomousAgent (run as several poet instances) that claims a stanza assignment, drafts the lines, and raises a coordination request when it needs another poet's output for rhythm or continuity.
- **DirectionWorkflow** — Workflow that runs the poetry director and writes one verse assignment per stanza onto the board.
- **PoetWorkflow** — one durable loop per poet: poll the board, claim an open assignment atomically, draft verse under a content guardrail, pass the quality gate, and mark the stanza done or blocked.
- **StanzaEntity** — one EventSourcedEntity per stanza; the atomic claim that stops two poets grabbing the same assignment.
- **PoemEntity**, **CoordinationMailbox**, **PromptQueue** — poem lifecycle, per-poet coordination messages, and the submission log.
- **EditorialControl** — KeyValueEntity holding the operator pause flag.
- **VerseBoardView** — the shared assignment list the UI and the poets read.
- **PoetryEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the writing prompts the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — change the poet roster (`poet-1`, `poet-2`, `poet-3`) to a different count.
- `prompts/poet.md` — narrow the poet to a specific form (haiku, sonnet, free verse).
- `eval-matrix.yaml` — this baseline carries no governance controls; add controls if you extend the system with consequential outputs or external publishing.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a writing prompt → the poetry director assigns stanzas → poets claim and complete them → the poem reaches `PUBLISHED`.
2. Two poets claim concurrently → each stanza is claimed by exactly one poet; no double-claim.
3. A poet raises a coordination request → the stanza goes `BLOCKED`, a message lands in the peer's mailbox, the reply unblocks it.
4. The editor pauses the team → in-flight claiming pauses; resume continues the drafting.

## License

Apache 2.0.
