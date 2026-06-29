# User journeys — basic-cv-agent

## J1 — Submit a profile in Prose mode and get a formatted CV

**Preconditions:** Service running on declared port (`http://localhost:9871/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9871/` → App UI tab.
2. From the **Load seeded profile** dropdown, pick `Software Engineer`.
3. Confirm output mode is set to `Prose CV`.
4. Click **Generate CV**.

**Expected:**
- The new card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `SANITIZED` within 1 s. The right-pane detail shows the sanitized notes with any PII category chips visible (at minimum `email` from the seeded profile's contact email).
- Within 30 s the card reaches `CV_GENERATED`. The right pane shows: a headline (max 12 words), a rendered Markdown block with at minimum four sections (Professional Summary, Work History, Education, Skills), and a keywords chip row with 8–15 entries.
- Every Work History entry in the Markdown has a title, company, date range, and at least one bulleted achievement beginning with a past-tense verb.

## J2 — Submit the same profile in Structured mode

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Load seeded profile** dropdown, pick `Software Engineer`.
2. Toggle output mode to `Structured CV`.
3. Click **Generate CV**.

**Expected:**
- The card reaches `CV_GENERATED` within 30 s.
- The right pane shows a formatted JSON tree, not Markdown prose.
- Each `section.content` is a parseable JSON string. Work History section content parses as an array of objects with keys `title`, `company`, `start`, `end`, `bullets`. Professional Summary section content parses as `{"text":"..."}`.
- A copy-to-clipboard button is visible above the JSON output.

## J3 — PII never reaches the LLM (audit check)

**Preconditions:** Service running with `LOG_LEVEL=DEBUG` so the LLM call body is logged. Any model provider (real or mock — the sanitizer runs either way).

**Steps:**
1. In the Additional notes textarea, type: `National Insurance: AB123456C and my email is test@candidate.example`.
2. Fill in the rest of the form with any values and click **Generate CV**.
3. Wait for `CV_GENERATED`.
4. Inspect the service log for the LLM call body (`debug:agent.task.attachment`).
5. Fetch `GET /api/cv-requests/{id}` and read `request.additionalNotes` and `request.contactEmail`.

**Expected:**
- The logged LLM call body (profile.json attachment) contains `[REDACTED-NINO]` and `[REDACTED-EMAIL]` in place of the raw identifiers. The raw strings `AB123456C` and `test@candidate.example` do not appear in the logged attachment.
- `request.additionalNotes` in the JSON response still contains the raw text — the audit log preserves it.
- `sanitized.piiCategoriesFound` lists `nino` and `email` (at minimum).

## J4 — Empty work history produces a placeholder section

**Preconditions:** Service running. Any model provider.

**Steps:**
1. Fill the submission form with only a name, contact email, target role, and one education entry. Leave the work history empty.
2. Click **Generate CV** in Prose mode.

**Expected:**
- The card reaches `CV_GENERATED` without error (no `FAILED` transition).
- The generated CV includes a Work History section whose content contains a clear placeholder message (e.g., `_No work history provided. Please add your employment history._`).
- The Professional Summary, Education, and Skills sections are still present and non-empty.

## J5 — Recent Graduate profile with structured output

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Load seeded profile** dropdown, pick `Recent Graduate`.
2. Toggle output mode to `Structured CV`.
3. Click **Generate CV**.

**Expected:**
- The card reaches `CV_GENERATED` within 30 s.
- Because the seeded graduate profile includes an internship entry, the Work History section content is a JSON array with one object.
- The keywords list includes terms relevant to the Data Science domain (e.g., `Python`, `data analysis`, `machine learning`, `internship`).
- No PII redaction markers (`[REDACTED-...]`) appear anywhere in the generated CV output — the CV section content is clean of sanitizer artefacts.
