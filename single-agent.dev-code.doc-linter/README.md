# Akka Sample: Doc Linter

A general-assistant agent lints a Markdown file with a custom tool, then summarizes the findings into actionable edits. User feedback tunes which rules the summary emphasizes.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One LLM key option (the generator picks based on what is set, or falls back to a mock model — no key needed):
  - `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`, or
  - choose the mock model when prompted (offline, no key).
- Host software: None. This blueprint runs out of the box — the files it lints are static sample files inside the service.

## Generate the system

```sh
cp -r ./single-agent.dev-code.doc-linter  ~/my-projects/doc-linter
cd ~/my-projects/doc-linter
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or the lint rule set.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `LinterAgent` — a general-assistant agent with a custom in-process Markdown lint tool that returns findings and writes a plain-language summary of suggested edits.
- `LintEntity` — an event-sourced record of each lint run, its findings, gate status, and any feedback.
- `RuleProfileEntity` — learned rule weights updated from user feedback.
- `LintView` — a read model the UI queries and streams over SSE.
- `FeedbackConsumer` — applies recorded feedback to the rule profile.
- `LintSimulator` — drips sample file paths for background lint runs.
- Two HTTP endpoints — the lint API plus the static single-file UI.

## Customise before generating

- **System name / model provider** — SPEC.md Section 1 and Section 11.
- **Lint rule set** — the rule list in `prompts/linter-agent.md` and Section 5 of SPEC.md.
- **Documentation gate threshold** — the error-count cutoff in `eval-matrix.yaml` control `A1` and Section 8.

## What gets validated

- Submitting a file path returns findings and a readable summary within ~30 s.
- A path that escapes the sample-data root is blocked before any file read.
- The documentation gate reports PASS / FAIL based on error-severity findings.
- Recording feedback shifts the rule profile and changes the next summary's emphasis.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
