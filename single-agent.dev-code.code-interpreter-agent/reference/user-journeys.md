# User journeys — code-interpreter-agent

## J1 — Submit a CSV prompt and get a scalar result

**Preconditions:** Service running on declared port (`http://localhost:9673/`); a valid model-provider API key set, or the mock LLM selected at scaffold time.

**Steps:**
1. Open `http://localhost:9673/` → App UI tab.
2. From the **Dataset format** dropdown, pick `CSV`.
3. Click **Load seeded example** to fill the data payload and prompt (uses the 20-row sales CSV and the prompt "Find the top 5 values in the revenue column and compute their mean.").
4. Click **Run**.

**Expected:**
- The new job card appears in the live list with status `SUBMITTED` within 1 s.
- The card transitions to `CODE_APPROVED` within ~10 s. The right-pane detail shows the generated Python source and the `importsList` chips.
- Within ~15 s the card reaches `EXECUTING` then `RESULT_RECORDED`. The right pane shows: a SCALAR result-kind chip, the `answer` value (e.g., `mean=15940.00`), and the Python source that ran.
- The wall-clock duration chip shows a value under 1,000 ms for the in-process executor.

## J2 — Guardrail blocks a forbidden import

**Preconditions:** Service running with the mock LLM selected (`model-provider = mock`). The mock's `interpret-data.json` includes entries with `import os` or `import requests` that trigger on the first iteration of every 3rd job (seeded by `MockModelProvider.seedFor(jobId)`).

**Steps:**
1. Submit any seeded dataset three times in a row (J1 steps × 3).
2. Watch the third submission's lifecycle in the browser network panel (`/api/jobs/sse`).

**Expected:**
- The third job's first agent iteration produces code containing a forbidden import.
- The `before-tool-call` guardrail rejects the tool call. The forbidden code NEVER runs — `ExecutionSandbox` is not called.
- The agent loop retries on iteration 2 (and 3 if needed) and produces code with only allowed imports. The job transitions to `CODE_APPROVED` then `RESULT_RECORDED`.
- The service log shows one `guardrail.reject` line per rejected iteration, naming the forbidden module.

## J3 — Safety halt terminates a runaway execution

**Preconditions:** Mock LLM mode. The mock contains a `while True: pass` entry that is selected for a specific seed (documented in `src/main/resources/mock-responses/interpret-data.json`).

**Steps:**
1. In the App UI, submit the seeded numeric dataset with the prompt "Compute the sum." using the seeded job count that triggers the runaway mock entry (consult the mock-response file — the entry is labelled "runaway-loop").
2. Wait for 12 seconds.

**Expected:**
- The job transitions to `CODE_APPROVED` (the guardrail approves the code — `while True: pass` contains no forbidden imports or network calls; the halt is the only backstop here).
- Within 10 s of `EXECUTING`, the job transitions to `HALTED`.
- The right-pane detail shows a red banner with breach type `TIMED_OUT` and elapsed wall-clock around 10,000 ms.
- The status pill is red (`HALTED`). No `RESULT_RECORDED` event appears in the SSE stream.
- The service log shows a watchdog interruption line for the job.

## J4 — JSON dataset produces a table result

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Dataset format** dropdown, pick `JSON`.
2. Click **Load seeded example** to fill with the 15-item API-latency metrics JSON array and the prompt "Show me all endpoints where latency_ms exceeds 200, sorted by latency descending."
3. Click **Run**.
4. Wait for `RESULT_RECORDED`.

**Expected:**
- The result-kind chip shows `TABLE`.
- The right pane shows a compact table with `columnNames` matching the JSON object keys (e.g., `endpoint`, `latency_ms`) and a `rows` list containing only entries where `latency_ms > 200`, sorted descending.
- Row count in the table matches the number of qualifying entries in the seeded dataset.

## J5 — Error result on unparseable data

**Preconditions:** Service running. Any model provider.

**Steps:**
1. In the App UI, set format to `CSV` and paste a data payload of `not,valid\ncsv\x00data` (malformed UTF-8 or mismatched column counts).
2. Enter any numeric aggregation prompt.
3. Click **Run**.

**Expected:**
- The agent generates code that attempts to parse the CSV; the code raises an exception at parse time (e.g., `csv.Error`) which the sandbox catches.
- The sandbox returns `SandboxResult{status: RUNTIME_ERROR}`.
- The job transitions to `RESULT_RECORDED` with `kind = ERROR` and a non-empty `errorMessage`.
- The right pane shows a red error message block. No HALTED state — a `RUNTIME_ERROR` is not a safety halt, it is a normal execution failure that produces an `ERROR` result.

## J6 — Numeric array produces a scalar result

**Preconditions:** Service running. Any model provider.

**Steps:**
1. From the **Dataset format** dropdown, pick `Numeric`.
2. Click **Load seeded example** to fill with the 50-measurement response-time numeric array and the prompt "What is the 90th percentile response time?"
3. Click **Run**.
4. Wait for `RESULT_RECORDED`.

**Expected:**
- The result-kind chip shows `SCALAR`.
- The `answer` string contains a numeric value corresponding to the 90th percentile of the seeded 50 measurements (the expected value is documented in `src/main/resources/sample-events/seed-datasets.jsonl` alongside the dataset).
- The generated Python source uses `statistics.quantiles` or equivalent from the allowed module list; no third-party import appears in the `importsList` chips.
