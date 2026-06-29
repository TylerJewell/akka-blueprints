# Akka Sample: Basic CV Agent

A single AI-service agent takes a candidate profile and generates a polished CV document — either as free-form prose or as a structured JSON output — with a PII sanitizer running before the profile data ever reaches the model.

Demonstrates the **single-agent** coordination pattern in the HR recruiting domain. One `CvGeneratorAgent` (AutonomousAgent) carries the entire generation step; the surrounding components prepare the input, sanitize it, and record the result. A PII sanitizer strips sensitive identifiers (government IDs, financial data, undeclared medical details) from the raw profile before the LLM call; the raw profile is preserved on the entity for audit.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the candidate profile store lives in-process and the agent's generation call is the only external dependency.

## Generate the system

```sh
cp -r ./single-agent.hr-recruiting.basic-cv-agent  ~/my-projects/basic-cv-agent
cd ~/my-projects/basic-cv-agent
```

(Optional) Edit `SPEC.md` to change the CV format (e.g., switch from chronological to functional, or restrict output to a JSON schema required by your ATS integration).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CvGeneratorAgent** — an AutonomousAgent that accepts a sanitized candidate profile as a task attachment and returns a typed `GeneratedCv`.
- **CvWorkflow** — orchestrates sanitize-wait → generate per submitted profile.
- **CvEntity** — an EventSourcedEntity holding the per-request lifecycle from submission through generation.
- **ProfileSanitizer** — a Consumer that subscribes to `ProfileSubmitted` events, strips sensitive identifiers, and emits `ProfileSanitized` back to the entity.
- **CvView + CvEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded candidate profiles for your own (the JSONL file under `src/main/resources/sample-events/seed-profiles.jsonl` after generation).
- `SPEC.md §5` — extend `GeneratedCv` with fields your ATS needs (e.g., `keywordsMatched`, `targetRoleCode`, `formattedSections`).
- `prompts/cv-generator.md` — narrow the agent's style guide (a technical-recruiting deployer would add instructions for STAR-format achievements; an executive-search deployer would ask for a bio-style narrative header).
- `eval-matrix.yaml` — wire a real PII detector by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a candidate profile → it is sanitized → the CV is generated → the output appears in the UI.
2. A profile containing a government ID and a bank account number is submitted; the LLM call log shows only redacted markers; the entity's `request.rawProfile` retains the originals.
3. A structured-output CV request returns a `GeneratedCv` where every required section is non-empty.
4. A profile submitted with an empty work-history list produces a CV with a clear placeholder section and an eval score that flags it for human review.

## License

Apache 2.0.
