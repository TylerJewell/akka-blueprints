# Akka Sample: Financial Model Builder

A sequential-pipeline agent that walks SEC filings through three task phases — EXTRACT, BUILD, VALIDATE — with a mandatory analyst review gate before scoring.

---

## Prerequisites

- **Claude Code** installed and the Akka plugin enabled.
- An AI model key. Options (pick one — never write a key value to disk):
  - **Mock LLM** — no key needed; the Akka local runtime ships a mock provider.
  - **Environment variable** — export `ANTHROPIC_API_KEY` (or the provider variable for your chosen model) before starting.
  - **`.env` file** — create `.env` in the service root; the runtime loads it automatically.
  - **Secrets URI** — configure `akka.secrets.uri` in `application.conf` to point at a vault entry.
  - **Type-once prompt** — the Akka local console prompts for the key on first run and stores it in the session keyring only.
- No additional host software (no Python, no database, no message broker).

---

## Generate the service

1. Copy this folder to your working directory.
2. Optionally edit `SPEC.md` to adjust the seed ticker list (§3), agent scope (§4), data model extensions (§5), or eval-matrix controls (`eval-matrix.yaml`).
3. Open Claude Code in the folder and run:

```
/akka:specify @SPEC.md
```

The specify → plan → tasks → implement chain generates a complete, runnable Akka service.

---

## What you'll get

| Component | Kind |
|---|---|
| `FinancialModelAgent` | AutonomousAgent |
| `FinancialModelPipelineWorkflow` | Workflow |
| `FinancialModelEntity` | EventSourcedEntity |
| `ExtractTools` | Tool class |
| `BuildTools` | Tool class |
| `ValidateTools` | Tool class |
| `ModelPhaseGuardrail` | Guardrail (before-tool-call) |
| `FilingFidelityScorer` | Scorer (on-decision-eval) |
| `FinancialModelView` | View |
| `ModelEndpoint` | HTTP endpoint (`/api/models/*`) |
| `AppEndpoint` | Static-asset endpoint (`/app/*`) |

The generated UI has five tabs: Overview, Architecture, Risk Survey, Eval Matrix, and App UI. The App UI tab shows a two-column layout — a live model list on the left and a detail panel on the right — with phase panels for extracted line items, built model rows and assumptions, validation flags, analyst review controls, and eval score.

---

## Customise before generating

| Section | What to adjust |
|---|---|
| SPEC.md §3 | Seed ticker list for mock filings |
| SPEC.md §4 | Agent task scope (add or remove tool calls) |
| SPEC.md §5 | Data model extensions (extra fields on records) |
| `eval-matrix.yaml` | Control thresholds (e.g., assumption deviation %, confidence floor) |

---

## What gets validated

`reference/user-journeys.md` defines four acceptance journeys:

- **J1** — Submit ticker and get an approved model (analyst approves, score emitted).
- **J2** — Phase-gate guardrail blocks a misordered tool call during EXTRACT.
- **J3** — Analyst rejects a model; entity reaches REJECTED terminal state.
- **J4** — `FilingFidelityScorer` flags aggressive assumptions; score ≤ 2.

Run `/akka:build` after generation to execute them locally.

---

## License

Apache 2.0 — see [LICENSE](https://www.apache.org/licenses/LICENSE-2.0).
