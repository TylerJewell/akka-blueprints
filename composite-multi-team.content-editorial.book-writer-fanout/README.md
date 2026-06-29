# Akka Sample: Book Writer

Type a book topic. One agent plans the chapters; a fresh writing agent drafts each chapter; a final step joins them into one manuscript.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- An AI model key — one of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`. If none is set, `/akka:specify` offers a mock-model path that needs no key (see SPEC.md Section 11).
- Host software: none. This blueprint runs out of the box — the web-search service it would call in production is modeled in-process.

## Generate the system

```sh
cp -r ./composite-multi-team.content-editorial.book-writer-fanout  ~/my-projects/book-writer
cd ~/my-projects/book-writer
```

(Optional) Edit `SPEC.md` — system name, model provider, chapter count, sample topics.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, then print the listening URL.

## What you'll get

- `OutlineAgent` (AutonomousAgent) — turns a topic into a titled chapter plan.
- `ChapterAgent` (AutonomousAgent) — drafts one chapter from its brief; calls a guarded in-process web-search tool.
- `BookWritingWorkflow` (Workflow) — outline → per-chapter writing fan-out → consolidate.
- `BookEntity` and `ChapterEntity` (EventSourcedEntity) — book lifecycle and per-chapter state.
- `BooksView` (View) — the read model the UI lists and streams over SSE.
- `ChapterEvalConsumer` (Consumer) — scores each chapter as it is drafted.
- `RequestSimulator` (TimedAction) — drips sample book topics.
- `WebSearchEndpoint`, `BookEndpoint`, `AppEndpoint` (HttpEndpoint) — search facade, `/api` surface, UI.

## Customise before generating

- **System name + topic list** — SPEC.md Section 1 and the sample-events file named in Section 11.
- **Model provider** — SPEC.md Section 11 "Generation workflow"; defaults to whichever key is set.
- **Chapter count and quality threshold** — SPEC.md Section 5 and the eval control in `eval-matrix.yaml`.

## What gets validated

- Submit a topic; an outline with chapters appears; each chapter drafts; the consolidated manuscript appears — `reference/user-journeys.md` J1–J3.
- A web-search query that drifts off-topic is blocked by the before-tool-call guardrail — J4.
- A book whose manuscript is missing a chapter does not reach `COMPLETED` — J5.

## License

Apache 2.0.
