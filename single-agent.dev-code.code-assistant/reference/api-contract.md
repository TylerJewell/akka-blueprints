# API contract — code-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/edits` | `SubmitEditRequest` | `201 { editId }` | `EditEndpoint` → `EditEntity` |
| `GET` | `/api/edits` | — | `200 [ Edit... ]` (newest-first) | `EditEndpoint` ← `EditView` |
| `GET` | `/api/edits/{id}` | — | `200 Edit` / `404` | `EditEndpoint` ← `EditView` |
| `GET` | `/api/edits/sse` | — | `text/event-stream` | `EditEndpoint` ← `EditView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitEditRequest (request body)

```json
{
  "taskDescription": "Add null-check validation to the submit endpoint and cover it with unit tests.",
  "repoSnapshot": {
    "snapshotId": "java-rest-v1",
    "projectName": "My REST Service",
    "files": [
      {
        "filePath": "src/main/java/io/example/SubmitEndpoint.java",
        "content": "public class SubmitEndpoint { public String submit(String input) { return process(input); } }",
        "language": "java"
      },
      {
        "filePath": "src/test/java/io/example/SubmitEndpointTest.java",
        "content": "public class SubmitEndpointTest { // TODO: add tests }",
        "language": "java"
      }
    ]
  },
  "author": "dev-42"
}
```

### Edit (response body)

```json
{
  "editId": "e-7c3...",
  "request": {
    "editId": "e-7c3...",
    "taskDescription": "Add null-check validation to the submit endpoint and cover it with unit tests.",
    "repoSnapshot": {
      "snapshotId": "java-rest-v1",
      "projectName": "My REST Service",
      "files": [
        {
          "filePath": "src/main/java/io/example/SubmitEndpoint.java",
          "content": "public class SubmitEndpoint { public String submit(String input) { return process(input); } }",
          "language": "java"
        }
      ]
    },
    "author": "dev-42",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "plan": {
    "confidence": "HIGH",
    "rationale": "The null check is isolated to a single method; the test class already has a scaffold in place.",
    "changes": [
      {
        "filePath": "src/main/java/io/example/SubmitEndpoint.java",
        "changeType": "MODIFY",
        "beforeContent": "public class SubmitEndpoint { public String submit(String input) { return process(input); } }",
        "afterContent": "public class SubmitEndpoint { public String submit(String input) { if (input == null || input.isBlank()) throw new IllegalArgumentException(\"input required\"); return process(input); } }",
        "diffSummary": "Added null/blank guard that throws IllegalArgumentException for missing input."
      },
      {
        "filePath": "src/test/java/io/example/SubmitEndpointTest.java",
        "changeType": "MODIFY",
        "beforeContent": "public class SubmitEndpointTest { // TODO: add tests }",
        "afterContent": "public class SubmitEndpointTest { @Test void rejectsNull() { assertThrows(IllegalArgumentException.class, () -> endpoint.submit(null)); } }",
        "diffSummary": "Added unit test for null input rejection."
      }
    ],
    "testCommand": "mvn test -Dtest=SubmitEndpointTest",
    "proposedAt": "2026-06-28T10:00:15Z"
  },
  "gateResult": {
    "status": "PASSED",
    "testOutput": "Tests run: 3, Failures: 0, Errors: 0, Skipped: 0",
    "testsPassed": 3,
    "testsFailed": 0,
    "evaluatedAt": "2026-06-28T10:00:18Z"
  },
  "status": "GATE_PASSED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:18Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: edit-update
data: { "editId": "e-7c3...", "status": "GATE_PASSED", "plan": { ... }, "gateResult": { ... } }
```

One event per state transition (`SUBMITTED`, `ANALYZING`, `EDIT_PROPOSED`, `GATE_PASSED`, `GATE_FAILED`, `FAILED`). Clients reconcile by `editId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `author` from the authenticated principal rather than the request body.
