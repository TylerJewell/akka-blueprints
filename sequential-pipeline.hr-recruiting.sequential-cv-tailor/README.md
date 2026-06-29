# Akka Sample: Sequential CV Tailor

A two-step pipeline: `CvGeneratorAgent` produces a structured base CV from a candidate profile, then `CvTailorAgent` rewrites it for a specific job posting. Each stage has its own typed input, typed output, and a PII sanitizer that strips candidate personal data before any payload crosses the task boundary and before anything is stored in the read model.

Demonstrates the **sequential-pipeline** coordination pattern wired with one governance mechanism: a `before-call` sanitizer that removes PII fields from every inter-stage payload so that candidate names, contact details, and demographic markers never appear in logs, evaluator output, or the UI beyond the fields explicitly surfaced on the review pane.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Both pipeline stages run in-process inside the same Akka service.

## Generate the system

```sh
cp -r ./sequential-pipeline.hr-recruiting.sequential-cv-tailor  ~/my-projects/sequential-cv-tailor
cd ~/my-projects/sequential-cv-tailor
```

(Optional) Edit `SPEC.md` to point at a different candidate fixture set, a different model provider, or a narrower set of job-posting fixtures.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CvGeneratorAgent** — an AutonomousAgent with a single `Task<BaseCv>` constant (`GENERATE_BASE_CV`). Given a `CandidateProfile`, it calls `ProfileTools` to expand and structure the profile into a `BaseCv`.
- **CvTailorAgent** — a second AutonomousAgent with a single `Task<TailoredCv>` constant (`TAILOR_CV`). Given a `BaseCv` and a `JobPosting`, it calls `TailorTools` to adapt each section for the posting.
- **CvPipelineWorkflow** — one workflow per tailoring request. Steps: `generateStep → tailorStep → evalStep`. Each step calls `runSingleTask` on the appropriate agent, reads the typed result, writes the event back onto `CvRequestEntity`, and advances.
- **CvRequestEntity** — an EventSourcedEntity holding the per-request lifecycle (`ProfileReceived`, `GenerationStarted`, `BaseCvReady`, `TailoringStarted`, `TailoredCvReady`, `QualityScored`, `RequestFailed`).
- **PiiSanitizer** — the `before-call` sanitizer registered on both agents. Strips `candidateName`, `email`, `phone`, `address`, and `dateOfBirth` from any payload presented to the model.
- **AlignmentScorer** — deterministic, rule-based on-decision evaluator that runs after `TailoredCvReady` and emits a 1–5 alignment score.
- **CvRequestView + CvEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded candidate fixture set under `src/main/resources/sample-data/candidates/*.json` to match your demo audience (tech, finance, healthcare).
- `SPEC.md §4` and `prompts/cv-generator-agent.md` / `prompts/cv-tailor-agent.md` — tighten the system prompts to a specific industry vertical.
- `SPEC.md §5` — add fields to `CandidateProfile` (e.g., `certifications`, `publications`). The PII sanitizer does not need editing — it strips by field name from a registry, not by record shape.
- `eval-matrix.yaml` — replace the deterministic `AlignmentScorer` stub with a keyword-overlap or embedding-based scorer by editing the `E1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A recruiter submits a candidate profile + job posting → `generateStep` runs → `tailorStep` runs → a typed `TailoredCv` appears in the UI within ~60 s. Every transition is visible in real time.
2. The `before-call` sanitizer strips PII before each agent receives its payload. The service log contains no `candidateName`, `email`, `phone`, or `address` values in model-call lines.
3. Every `TailoredCv` emitted has an alignment score chip on the UI card; CVs whose required-keyword coverage falls below 50 % receive a score ≤ 2 and are flagged for recruiter review.
4. The `BaseCv` produced by `CvGeneratorAgent` is the only context handed to `CvTailorAgent`; the raw `CandidateProfile` is never forwarded. The workflow's typed handoff is the only path data travels between stages.

## License

Apache 2.0.
