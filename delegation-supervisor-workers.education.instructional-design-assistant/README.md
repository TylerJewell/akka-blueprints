# Akka Sample: Instructional Design Assistant

A lead agent decomposes a unit-design request into three parallel sub-tasks — outline, learning materials, and assessment — delegated to specialist agents, then merges the results into a publishable unit package. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance: a human-in-the-loop faculty review gate and an academic-integrity guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the unit-design request stream and content tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.education.instructional-design-assistant  ~/my-projects/instructional-design-assistant
cd ~/my-projects/instructional-design-assistant
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **UnitDesignSupervisor** — AutonomousAgent that decomposes a unit-design request and later merges specialist outputs into a unit package.
- **OutlineSpecialist** — AutonomousAgent that drafts the learning-objective outline.
- **MaterialsSpecialist** — AutonomousAgent that authors lesson content and activities.
- **AssessmentSpecialist** — AutonomousAgent that writes quiz questions and rubrics.
- **UnitDesignWorkflow** — Workflow that fans work out to the three specialists in parallel, joins results, calls the Supervisor for synthesis, runs the academic-integrity guardrail, then waits for faculty approval.
- **UnitPackageEntity** — EventSourcedEntity holding the unit's full lifecycle.
- **FacultyReviewQueue** — EventSourcedEntity that manages pending review decisions.
- **UnitPackageView** — projection the UI streams via SSE.
- **UnitDesignEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the subjects the simulator drips, or remove the simulator.
- `SPEC.md §5` — adjust the `UnitPackage` record fields (e.g., add `bloomsLevel`).
- `prompts/outline-specialist.md` — narrow the outline to a specific curriculum framework.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real content library.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a unit-design request → unit progresses `DRAFT → IN_PROGRESS → PENDING_REVIEW`, faculty approves, unit enters `PUBLISHED`.
2. Academic-integrity guardrail trips on planted plagiarism → unit enters `BLOCKED`.
3. Faculty rejects a unit → unit enters `REVISION_REQUESTED`; a new design cycle begins.

## License

Apache 2.0.
