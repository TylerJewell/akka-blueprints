# Data model — incident-classifier

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `IncidentSubmission` | `incidentId` | `String` | no | UUID minted by `ClassificationEndpoint`. |
| | `shortDescription` | `String` | no | One-line summary supplied by the reporter. |
| | `longDescription` | `String` | no | Full incident description. Passed to the agent as an attachment. |
| | `callerId` | `String` | no | Reporter identifier. |
| | `priorityHint` | `String` | no | P1–P4, recorded but not used for classification. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `TaxonomyScope` | `categoryCount` | `int` | no | Number of categories in the loaded taxonomy. |
| | `subcategoryCount` | `int` | no | Total subcategories across all categories. |
| | `ciCount` | `int` | no | Total CI entries across all categories. |
| | `validatedAt` | `Instant` | no | When `VocabularyValidator` confirmed the taxonomy loaded. |
| `ClassificationResult` | `category` | `String` | no | Taxonomy category code (e.g. `"Database"`). |
| | `subcategory` | `String` | no | Subcategory code under `category` (e.g. `"Connectivity"`). |
| | `affectedCi` | `String` | no | CI name from the registry (e.g. `"PROD-DB-01"`). |
| | `rationale` | `String` | no | 1–2 sentences from the agent explaining the assignment. |
| | `decidedAt` | `Instant` | no | When the agent returned the result. |
| `AccuracyContribution` | `categoryInTaxonomy` | `boolean` | no | `true` if `category` appears in the taxonomy table. |
| | `subcategoryUnderCategory` | `boolean` | no | `true` if `subcategory` is listed under the stated `category`. |
| | `ciInRegistry` | `boolean` | no | `true` if `affectedCi` appears in the CI list. |
| | `score` | `int` | no | 0–5: +2 category, +2 subcategory, +1 CI. |
| | `rationale` | `String` | no | One sentence describing the validation result. |
| | `evaluatedAt` | `Instant` | no | When `AccuracyEvaluator` finished. |
| `Incident` (entity state) | `incidentId` | `String` | no | — |
| | `submission` | `Optional<IncidentSubmission>` | yes | Populated after `IncidentSubmitted`. |
| | `scope` | `Optional<TaxonomyScope>` | yes | Populated after `TaxonomyValidated`. |
| | `classification` | `Optional<ClassificationResult>` | yes | Populated after `ClassificationRecorded`. |
| | `eval` | `Optional<AccuracyContribution>` | yes | Populated after `AccuracyScored`. |
| | `status` | `IncidentStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `IncidentSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Incident` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`IncidentStatus`: `SUBMITTED`, `TAXONOMY_VALIDATED`, `CLASSIFYING`, `CLASSIFICATION_RECORDED`, `EVALUATED`, `FAILED`.

## Events (`IncidentEntity`)

| Event | Payload | Transition |
|---|---|---|
| `IncidentSubmitted` | `submission` | → SUBMITTED |
| `TaxonomyValidated` | `scope` | → TAXONOMY_VALIDATED |
| `ClassificationStarted` | — | → CLASSIFYING |
| `ClassificationRecorded` | `classification` | → CLASSIFICATION_RECORDED |
| `AccuracyScored` | `eval` | → EVALUATED (terminal happy) |
| `ClassificationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Incident.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`IncidentRow` mirrors `Incident`. The view stores the full `ClassificationResult` and `AccuracyContribution` so the UI can render all chips without a secondary fetch.

The view declares TWO queries:

```
getAllIncidents: SELECT * AS incidents FROM classification_view ORDER BY createdAt DESC
getAccuracyWindow: SELECT * AS contributions FROM classification_view
                   WHERE eval IS NOT NULL ORDER BY createdAt DESC LIMIT 50
```

No `WHERE status = :status` filter on `getAllIncidents` — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`IncidentTasks.java`)

```java
public final class IncidentTasks {
  public static final Task<ClassificationResult> CLASSIFY_INCIDENT = Task
      .name("Classify incident")
      .description("Read the attached incident description and produce a ClassificationResult "
          + "with category, subcategory, and affectedCi drawn from the provided taxonomy")
      .resultConformsTo(ClassificationResult.class);

  private IncidentTasks() {}
}
```

The companion Tasks class is mandatory for any `AutonomousAgent` (Lesson 7).

## TaxonomyTable structure (in-process singleton)

Loaded once from `src/main/resources/taxonomy/taxonomy.json` at startup. The JSON schema:

```json
{
  "categories": [
    {
      "code": "Network",
      "subcategories": ["Connectivity", "DNS", "VPN", "Firewall", "Bandwidth"],
      "ciCodes": ["PROD-FW-01", "PROD-SW-01", "CORP-VPN-01", "CORE-RTR-01"]
    },
    {
      "code": "Database",
      "subcategories": ["Connectivity", "Performance", "Replication", "Backup"],
      "ciCodes": ["PROD-DB-01", "PROD-DB-02", "REPORTING-DB-01"]
    }
  ]
}
```

`TaxonomyTable` exposes three methods used by `AccuracyEvaluator`:
- `containsCategory(String category)` — case-sensitive match on `code`.
- `subcategoryBelongsTo(String category, String subcategory)` — checks subcategory list under the given category code.
- `containsCi(String ci)` — checks across all categories' CI lists.
