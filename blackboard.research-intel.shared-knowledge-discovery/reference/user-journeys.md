# User journeys

## J1 — Happy path: inquiry to converged report

**Preconditions:** Service running on declared port (9876); model-provider key set or mock LLM selected.

**Steps:**
1. Open http://localhost:9876/. App UI tab visible.
2. In the form, enter a research question (e.g., "What is the current consensus on room-temperature superconductors?"). Click Submit.
3. An `inquiryId` is returned and a blackboard panel appears with status `OPEN`.

**Expected:**
- Within seconds, the panel shows the first source proposal from `SourceScout`.
- `ClaimExtractor` adds claims tied to that source; the iteration counter advances.
- `HypothesisSynthesizer` proposes a hypothesis backed by claim ids that resolve on the board.
- `Critic` runs and either leaves the hypothesis alone or moves an item to `disputed`.
- When every open sub-question has at least one supporting claim and the latest hypothesis is not disputed, status moves to `CONVERGED` and a report appears.
- SSE keeps the panel live; no page reload.

## J2 — Schema-violating write rejected

**Preconditions:** As J1. Under the mock LLM, `hypothesis-synthesizer.json` includes one entry whose payload references a dangling `claimId`.

**Steps:**
1. Submit the canned inquiry that triggers the malformed entry.

**Expected:**
- The schema guardrail rejects the write; the entity emits `WriteRejected` with the failing field listed.
- The blackboard's structured payload is unchanged — no `Hypothesis` is appended.
- The rejected-writes strip in the UI shows the rejection with reason.
- The control shell continues; on its next pick it routes to a different specialist or retries the synthesizer.

## J3 — Low-utility contributions bench a specialist

**Preconditions:** As J1, run long enough that one specialist has produced 3 contributions in a row that the scorer rates below threshold (mock LLM is seeded so that `SourceScout` re-proposes near-duplicates).

**Steps:**
1. Submit the inquiry that exercises the seed.
2. Watch the per-specialist score chips on the panel.

**Expected:**
- The scorer records progressively lower scores for the offending specialist.
- After the third low score, the control shell stops invoking that specialist; the panel's "next specialist" hint shows a different agent.
- When a new sub-question is opened or the critic flags an item, the benched specialist becomes eligible again.

## J4 — Critic challenge reroutes the shell

**Preconditions:** As J1. The blackboard has at least one hypothesis present.

**Steps:**
1. Submit a canned inquiry whose seeded critic entry challenges the live hypothesis.

**Expected:**
- The critic posts a `Challenge`; the target hypothesis moves into the blackboard's `disputed` list.
- On the next tick, the control shell does not route to `HypothesisSynthesizer` again immediately; it routes to `ClaimExtractor` (to back the hypothesis with stronger claims) or `SourceScout` (to find a counter-source).
- Once new claims arrive, the synthesizer is invoked again and a fresh hypothesis is proposed.

## J5 — Operator halt and resume

**Preconditions:** As J1, with at least one inquiry in progress.

**Steps:**
1. Click Halt in the control strip (or POST `/api/control/halt` with reason and actor).
2. Observe the shell for ~10 s.
3. Click Resume (or POST `/api/control/resume`).

**Expected:**
- After Halt: no new specialist invocations; the schema guardrail rejects any in-flight `ProposedWrite` with reason "system halted". `GET /api/control` reports `halted: true` with reason and actor; the App UI shows a red halted banner.
- After Resume: `halted` returns to false; the control shell continues and the inquiry progresses to convergence.
