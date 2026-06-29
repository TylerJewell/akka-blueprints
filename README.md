# Akka Blueprints

A library of **detailed specifications** that Claude turns into running Akka systems.

Each blueprint is a folder of natural-language and YAML inputs — a SPEC, a PLAN, an eval-matrix, a risk survey, prompt fragments, reference material. You point Claude Code at the folder, run `/akka:specify`, and it generates the project: Akka components, application.conf, tests, embedded UI — the lot.

**You don't get pre-built code. You get a precise, governance-aware recipe Claude can build from.**

---

## Why blueprints, not samples?

A sample is one frozen implementation of one pattern. A blueprint is the *idea* of that pattern made explicit enough that any modern code-generating agent can produce a fresh, current, locally-tuned implementation on demand.

Trade-off you're making by reading a blueprint instead of cloning a sample:
- **More setup** — you need Claude Code + the Akka plugin, and one `/akka:specify` invocation.
- **Less rot** — the blueprint stays current with the latest Akka SDK version because the SDK is read at generation time.
- **Real customisation** — you can edit the SPEC before you generate, and the system reflects your edits.
- **Embedded governance** — every blueprint declares its eval-matrix and risk-survey, so the generated system ships with controls already wired.

---

## How to use a blueprint

### 1. Install Claude Code + the Akka plugin

Follow the Akka docs: [doc.akka.io — Spec-Driven Development with Claude Code](https://doc.akka.io/) → "Install /akka:specify".

(Short version: install Claude Code, then `/akka:setup` once in any working directory to scaffold the Akka plugin globally.)

### 1.5. (Optional) Have a model-provider key — or pick mock LLM

Each blueprint asks during `/akka:specify` how to source its AI key. Five options: (a) **mock LLM** — no real key, random-but-shape-correct outputs per agent, perfect for demos and offline dev; (b) name an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`); (c) point to an env file you maintain; (d) secrets-store URI; (e) type once in the session. Akka **never** writes the key value to disk — only the *reference* is recorded.

### 1.6. Check the integration tier — some blueprints need extra software

Each blueprint declares an **integration tier** (visible on its card on the landing page and in its `README.md`). The tier dictates what additional software you need installed on your machine **before** running `/akka:specify`.

| Tier | What it means | Extra software you need |
|---|---|---|
| **Runs out of the box** | The external system is simulated entirely inside the Akka service. Nothing external is called. | None. Just `/akka:build`. |
| **Real service via env var** | The blueprint includes an adapter that flips between a simulated stub and a real client based on an env var (e.g., `SLACK_BOT_TOKEN`). | A valid credential for the upstream service. The real client is dormant until the env var is set. |
| **Requires Docker sidecar** | The blueprint ships a `docker-compose.yml` next to `pom.xml`. The sidecar runs a real service the blueprint connects to — typically PGVector + Postgres for RAG, Neo4j for graph workloads, MinIO for object storage. | **[Docker Desktop](https://www.docker.com/products/docker-desktop/)** (macOS / Windows) or **Docker Engine** (Linux) — running before you invoke `/akka:specify`. Compose v2 is bundled with Docker Desktop. |
| **Browser or sandbox execution** | The blueprint drives a real browser (Playwright / Stagehand), runs untrusted code in an isolated sandbox (E2B, Modal, Robocorp), or otherwise needs a managed execution environment. | One of: **Playwright** (`npm install -g playwright && npx playwright install`), an **E2B account + `E2B_API_KEY`**, a **Modal account + token**, or **Robocorp**, depending on which environment the specific blueprint targets — its README names the one it expects. |

If you try to generate a blueprint whose tier requires software you don't have, `/akka:specify` will tell you up front rather than failing mid-build.

### 2. Pick a blueprint

Browse `index.html` (open it in your browser) or skim the folder names. Each folder is one blueprint, named `<coordination-pattern>.<domain>.<short-name>/`.

### 3. Clone or copy the blueprint folder into your own project space

```sh
cp -r ./continuous-monitor.cx-support.inbox-watcher  ~/my-projects/inbox-watcher
cd ~/my-projects/inbox-watcher
```

### 4. (Optional) Edit `SPEC.md`

Open `SPEC.md` and tune anything you want to change before generation — system name, model provider, additional fields, business-specific logic. The blueprint ships with sensible defaults but is meant to be edited.

### 5. Run `/akka:specify`

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `@SPEC.md` inlines the file as the skill's argument. SPEC.md's Section 12 ("Post-scaffolding workflow") instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` with no prompts in between. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

### 6. Run it

Set one model-provider API key in your shell (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`), then in Claude Code:

```
/akka:build
```

The skill compiles the project, starts the local Akka runtime, and prints the URL. Open `http://localhost:<port>/` to use the embedded UI.

---

## What's in this repo

| Path | What it is |
|---|---|
| `index.html` | Browsable corpus — filter blueprints by pattern, domain, governance mechanism. |
| `corpus.json` | The 417-entry master corpus. Source of truth for the landing page. |
| `classifications.json` | Filter taxonomies — patterns, domains, mechanisms, integrations. |
| `_template/` | The canonical blueprint skeleton. Copy this folder when authoring a new blueprint. |
| `human-in-loop-gate.content-editorial.draft-approve-publish/` | **The formal exemplar.** A fully-built reference of what `/akka:specify` should produce when given a properly-authored blueprint. Study it before authoring your own. |
| `<pattern>.<domain>.<name>/` | Each named folder = one blueprint. |

---

## Authoring a blueprint

Read [`explainability/specs/eval-matrix/blueprint-program/BLUEPRINT-AUTHORING-GUIDE.md`](../explainability/specs/eval-matrix/blueprint-program/BLUEPRINT-AUTHORING-GUIDE.md) — it's the single source of truth for what every blueprint must contain, what the SPEC must say, and the language/style/format rules. Pair with [`AKKA-EXEMPLAR-LESSONS.md`](../explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md) to understand the failure modes a good SPEC must prevent.

---

## License

Apache 2.0 — both the blueprints and the formal exemplar.
