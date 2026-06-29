# User journeys — team-poems-multi-agent

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path: prompt to published poem

**Preconditions:** Service running on declared port (9299); a model-provider key set, or the mock LLM selected during scaffolding. The poet roster (`poet-1`, `poet-2`, `poet-3`) is running and idle-polling.

**Steps:**
1. Open `http://localhost:9299/`. The App UI tab is visible.
2. In the form, enter title "After the Rain" and a one-line inspiration note. Click Submit.
3. A `poemId` is returned; the poem appears with status `RECEIVED`.

**Expected:**
- Within a few seconds the poetry director's plan lands: three to five stanza cards appear in the `Open` column, and the poem moves to `ASSIGNED` then `DRAFTING`.
- Poets claim eligible stanzas; each claimed card shows a poet id and moves through `Claimed` → `Drafting` → `In review` → `Done`.
- A stanza with `dependsOn` stays in `Open` until its dependencies are `Done`.
- When every stanza is `Done`, the poem moves to `PUBLISHED`.
- The board updates live via SSE throughout, with no page reload.

## J2 — Atomic claim: no double-claim under contention

**Preconditions:** As J1, with at least two poets idle and at least one `OPEN` stanza with no unmet dependency.

**Steps:**
1. Submit a prompt whose plan has a single dependency-free stanza so multiple poets race for it.

**Expected:**
- Exactly one poet's id appears on the contested stanza; the stanza is claimed once.
- The other poets return to polling and pick up different stanzas (or idle if none remain).
- No stanza is ever shown claimed by two poets; the claim count per stanza is one.

## J3 — Coordination request

**Preconditions:** As J1. Under the mock LLM, the `poet.json` set includes an entry whose `coordinationRequest` targets `poet-2`; with a real model, a prompt whose stanzas force a tonal dependency.

**Steps:**
1. Submit a prompt that produces a stanza a poet cannot finish without another poet's line for rhythm continuity.
2. Watch that stanza.

**Expected:**
- The stanza moves to `BLOCKED` with a reason naming the peer (e.g., "waiting on poet: poet-2").
- A `CoordinationMessage` appears in `poet-2`'s mailbox panel with the concrete question.
- `POST /api/mailbox/poet-2/messages/{id}/reply` (via the reply box) posts a reply; the blocked stanza returns to `OPEN` and is then picked up and completed.

## J4 — Editorial pause and resume

**Preconditions:** As J1, with at least one poem in progress.

**Steps:**
1. Click Pause in the control strip (or `POST /api/control/pause` with a reason and actor).
2. Observe the team for ~10 s.
3. Click Resume (or `POST /api/control/resume`).

**Expected:**
- After Pause: no new stanzas are claimed; poets idle. `GET /api/control` reports `paused: true` with the reason and actor; the App UI shows the red paused banner.
- After Resume: `paused` returns to `false`; poets resume claiming and the in-progress poem continues to completion.
