# User journeys — plumber-data-engineering-assistant

Acceptance criteria. The generated system passes when all four journeys complete as written.

## J1 — Happy path

**Preconditions:** Service running on the declared port; valid model-provider API key set (or `model-provider = mock`).

**Steps:**
1. Open `http://localhost:<port>/`. App UI tab is visible.
2. In the Pipeline prompt field, type "Design a Spark pipeline that reads Parquet from s3://raw-events, filters nulls, and writes Avro to s3://clean-events." Click Submit.
3. A new pipeline card appears with status PLANNING.

**Expected:**
- Within 5 s, status transitions to EXECUTING via SSE.
- Within ~3 minutes, status transitions to FINALISED.
- The expanded view shows:
  - A pipeline ledger with non-empty sources, transforms, sinks, and validationPlan lists, and a `currentStage` that is null at finalisation.
  - A progress ledger with 3–6 entries. At least one entry has `engine = SPARK`, at least one has `engine = SCHEMA`. Every entry has a non-empty `artifact`.
  - A `PipelineDefinition` with a 60–120 word description and a non-empty stages list.

## J2 — Test gate blocks a schema-incompatible stage

**Preconditions:** As J1.

**Steps:**
1. Submit the prompt "Generate a Spark source stage that reads from s3://raw-events and uses column `login_count` typed as `array<string>`."

**Expected:**
- The SparkAgent returns an artifact declaring `login_count` as `array<string>`.
- The test gate's column-type check finds that the schema registry entry for `raw-clickstream-v2` declares `login_count` as `integer` (or the column does not exist in the registry).
- A `StageBlocked` entry appears on the progress ledger with `verdict = BLOCKED_BY_TEST_GATE` and a `blocker` message naming the type mismatch.
- The planner either replans with a corrected column type and the pipeline reaches `FINALISED`, or exhausts the replan budget and the pipeline ends in `FAILED` with a clear `failureReason`. Either outcome is acceptable; what matters is that the incompatible artifact is never recorded as an `OK` stage.

## J3 — Operator halt drains gracefully

**Preconditions:** As J1.

**Steps:**
1. Submit any pipeline request the simulator has not yet run.
2. While the pipeline status is EXECUTING (within the first ~10 seconds), click **Halt new dispatches** and provide a reason such as "schema registry refresh in progress".
3. Observe the in-flight stage completes.

**Expected:**
- The in-flight `ProgressEntry` is recorded normally (the workflow does not abort mid-stage).
- The next loop iteration reads the halt flag, exits the loop, and emits `PipelineHaltedOperator`.
- Pipeline status moves to `HALTED`. `haltReason` is populated with the operator's reason.
- The operator pane's `HALTED` pill reflects the state in real time via the `control-update` SSE event.
- After clicking **Resume**, the operator pane returns to the un-halted state. New pipeline submissions begin dispatching stages normally.

## J4 — Secret scrubber redacts a connection string password

**Preconditions:** As J1, with the canned fixture file `sample-data/dbt-fixtures.jsonl` containing one entry whose SQL field includes the substring `postgresql://dbt_user:S3cr3tPass!@warehouse.internal:5432/analytics`.

**Steps:**
1. Submit "Generate a dBT incremental model that reads from the analytics warehouse and writes a cleaned sessions table."

**Expected:**
- The DbtAgent returns an artifact containing the connection string with the cleartext password.
- The `SecretScrubber` identifies the password segment matching `postgresql://[^:]+:([^@]+)@` and replaces it with `[REDACTED:url-password]`.
- The `ProgressEntry.artifact` contains `postgresql://dbt_user:[REDACTED:url-password]@warehouse.internal:5432/analytics`.
- The PipelinePlannerAgent's next DECIDE prompt never contains the literal password `S3cr3tPass!`.
- The final `PipelineDefinition.description` does not contain the literal password.
