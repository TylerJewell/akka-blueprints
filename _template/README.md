# Akka Sample: <Short Name>

> **Template.** Copy this folder when authoring a new blueprint. Rename it to `<coordination-pattern>.<domain>.<kebab-name>/`. Replace every `TODO:` marker below before opening a PR.

A one-line pitch describing what the generated system does.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`.

## Generate the system

```sh
cp -r ./<this-folder>  ~/my-projects/<system-name>
cd ~/my-projects/<system-name>
```

(Optional) Edit `SPEC.md`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding.

## What you'll get

- TODO: bulleted list of the components the generated system contains.

## Customise before generating

- TODO: pointers to the parts of `SPEC.md` worth editing.

## What gets validated

- TODO: summarise the user journeys in `reference/user-journeys.md`.

## License

Apache 2.0.
