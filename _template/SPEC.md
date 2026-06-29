# SPEC — <kebab-name>

The natural-language brief `/akka:specify` reads to generate this system. Fill every section below before opening a PR.

---

## 1. System name + pitch

**System name:** TODO.
**One-line pitch:** TODO. State what the user types into the running UI and what the AI does in response.

## 2. What this blueprint demonstrates

TODO — one paragraph naming the coordination pattern and the governance pattern. No marketing, no rhetorical comparisons.

## 3. User-facing flows

TODO — numbered steps the human takes; each step's outcome. These become acceptance tests in `reference/user-journeys.md`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| TODO | AutonomousAgent / Agent / Workflow / EventSourcedEntity / View / Consumer / TimedAction / HttpEndpoint | TODO | TODO | TODO |

Names matter. `/akka:specify` will use them verbatim.

## 5. Data model

TODO — entity state record, event types, view row, status enum (if any). Use `Optional<T>` for lifecycle fields that are null before their transition.

## 6. API contract

TODO — inline the top-level surface; point at `reference/api-contract.md` for payload schemas.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`. Browser title: `<title>Akka Sample: <Short Name></title>`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. TODO — one sentence per mechanism wired by the generated system.

## 9. Agent prompts

TODO — for each agent component named in Section 4, point at its file under `prompts/`. Inline a one-sentence purpose.

## 10. Acceptance

TODO — inline the three or four journeys whose passing means "the blueprint generated correctly." See `reference/user-journeys.md`.

---

## 11. Implementation directives

**Required.** Replace the placeholder block below with the actual implementation directives before the blueprint is considered ready. The whole SPEC.md (Sections 1–11) is consumed by `/akka:specify @SPEC.md`, and the directives in Section 11 carry the Akka-specific details Sections 1–10 don't repeat.

Use the exemplar's directives at `human-in-loop-gate.content-editorial.draft-approve-publish/SPEC.md` Section 11 as the canonical form. Cover, in order:

1. One-paragraph identity (sample name, matrix cell, integration form, Maven group/artifact, Java package, HTTP port).
2. "Components to wire (exactly)" — every Akka primitive instance with its role, behaviour, and any required idioms (definition() shape, capability declarations, step timeouts, view query shape, etc.).
3. "Companion files" — domain records, application.conf, sample-events, metadata copies, eval-matrix.yaml, risk-survey.yaml, README structure, static-resources/index.html.
4. "Constraints" — pointers to AKKA-EXEMPLAR-LESSONS plus the must-not-fail list (Optional<T> on lifecycle fields, emptyState rule, AutonomousAgent never silently downgraded, no forbidden words, etc.).

```
TODO — replace this placeholder with the actual implementation directives following the
exemplar's format. Until this is filled in, /akka:specify cannot generate the system.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

**Required.** Copy this section verbatim from the exemplar (`human-in-loop-gate.content-editorial.draft-approve-publish/SPEC.md` Section 12). Its job is to instruct Claude — after `/akka:specify` finishes scaffolding — to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` without prompting the user, and to print the listening URL when the service is up.

Tune the section only when:

- Your blueprint requires a different chain (e.g., adds `/akka:test` between implement and build).
- Your blueprint has known long-running tasks where the 30-second narration threshold should be raised.

Do not delete the section: without it, the user has to manually type each downstream command.
