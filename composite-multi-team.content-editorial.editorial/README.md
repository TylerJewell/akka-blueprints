# Akka Sample: Editorial Desk

An editor-in-chief agent runs a story from topic to published article through three section desks — a research desk that delegates to several researchers, a writing desk whose writers claim sections off a shared board, and a review desk whose reviewers each score the draft. Demonstrates the **composite-multi-team** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the research desk, the writing board, the review panel, and the story intake are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./composite-multi-team.content-editorial.editorial  ~/my-projects/editorial-desk
cd ~/my-projects/editorial-desk
```

(Optional) Edit `SPEC.md` to change the system name, the writer roster, the review panel, the model provider, or the story topics the simulator drips.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **EditorInChief** — AutonomousAgent that assigns a story (produces an editorial brief) and, at the end, assembles the final article under an output guardrail.
- **ResearchLead + Researcher** — the research desk: the lead plans research subtopics and synthesises the findings; several researcher instances each produce one note. The lead delegating to a roster of researchers is the delegation capability.
- **WritingLead + Writer** — the writing desk: the lead plans the article sections onto a shared board; a roster of writer loops each claim an open section, write it under a tool guardrail, and mark it written. Writers self-organising around a shared list is the team capability.
- **Reviewer** — the review desk: several reviewer instances each score the draft on one axis (fact-check, style, legal); a deterministic moderation rule turns their notes into a pass-or-revise verdict. A scored panel feeding a moderation rule is the moderation capability.
- **EditorialWorkflow** — the editor-in-chief pipeline: brief → research → writing → review → publish, with a revision loop when review asks for changes.
- **DocumentEntity + SectionEntity** — the shared document workspace and the per-section board entry whose atomic claim stops two writers grabbing the same section.
- **StoryQueue, DocumentBoardView, SectionBoardView** — the submission log and the two read models the UI streams.
- **StageEvalConsumer** — fires a non-blocking quality eval on every stage result.
- **EditorialEndpoint + AppEndpoint** — REST + SSE + static UI serving, including the post-publication compliance-review surface.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the story topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — change the writer roster (`writer-1`, `writer-2`) or the review panel (`factcheck`, `style`, `legal`) to a different count.
- `prompts/researcher.md`, `prompts/writer.md`, `prompts/reviewer.md` — narrow each desk to a house style or a subject area.
- `eval-matrix.yaml` — extend the before-tool-call guardrail's refused-operation list, or change the stage-eval thresholds.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a topic → the editor-in-chief assigns a brief → the research desk delegates and synthesises → the writing desk fills sections on a shared board → the review panel scores the draft → a passing verdict assembles and publishes the article.
2. Two writers claim concurrently → each section is claimed by exactly one writer; no double-claim.
3. The review panel returns a revise verdict → the story loops back to writing for one revision round, then publishes.
4. A researcher, writer, or reviewer attempts a refused document-workspace write → the before-tool-call guardrail blocks it before execution and the stage is recorded with the guardrail reason.
5. After an article publishes, a compliance officer posts a post-publication review → it is recorded against the published article without changing its published state.

## License

Apache 2.0.
