# Data model — landing-page-builder

Authoritative record shapes. `/akka:implement` writes them as specified. Nullable
lifecycle fields are `Optional<T>` (Lesson 6).

## `Page` — PageEntity state and PagesView row

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Page id (workflow id). |
| `concept` | `Optional<String>` | yes | The submitted one-line concept. |
| `status` | `PageStatus` | no | Lifecycle status (enum below). |
| `analyzedAt` | `Optional<Instant>` | yes | When the brief was recorded. |
| `brief` | `Optional<String>` | yes | Serialized `ConceptBrief`. |
| `selectedAt` | `Optional<Instant>` | yes | When the template was recorded. |
| `templateName` | `Optional<String>` | yes | Chosen template name. |
| `templateUrl` | `Optional<String>` | yes | Source URL of the chosen template. |
| `customizedAt` | `Optional<Instant>` | yes | When markup was recorded. |
| `html` | `Optional<String>` | yes | Generated, sanitized markup. |
| `validatedAt` | `Optional<Instant>` | yes | When the lint gate ran. |
| `lintPassed` | `Optional<Boolean>` | yes | Lint outcome. |
| `publishedAt` | `Optional<Instant>` | yes | When the page was published. |
| `publishedUrl` | `Optional<String>` | yes | In-process published URL. |
| `rejectedAt` | `Optional<Instant>` | yes | When the page was rejected. |
| `rejectReason` | `Optional<String>` | yes | Why the lint gate rejected it. |

`emptyState()` returns `Page.initial("")` with no `commandContext()` reference
(Lesson 3).

## `PageStatus` enum

`QUEUED · ANALYZED · SELECTED · CUSTOMIZED · PUBLISHED · REJECTED`

## Events (PageEntity)

| Event | Trigger |
|---|---|
| `ConceptAnalyzed` | `recordAnalysis` after IdeaAnalyst returns a brief. |
| `TemplateSelected` | `recordTemplate` after TemplateSelector returns a choice. |
| `PageCustomized` | `recordCustomization` after CustomizationAgent returns markup. |
| `PagePublished` | `publish` after the lint gate passes. |
| `PageRejected` | `reject` after the lint gate fails. |

## InboundRequestQueue

Single instance `"default"`. Command `enqueueConcept(concept)` emits
`ConceptQueued`. `RequestConsumer` starts a `GenerationWorkflow` per event.

## Agent result records

- `ConceptBrief(String audience, List<String> valueProps, String tone, String colorTheme, List<String> sections)`
- `TemplateChoice(String templateName, String templateUrl, String rationale)`
- `LandingPage(String title, String html)`

## GenerationTasks constants (Lesson 7)

- `ANALYZE` — `resultConformsTo(ConceptBrief.class)`
- `SELECT` — `resultConformsTo(TemplateChoice.class)`
- `CUSTOMIZE` — `resultConformsTo(LandingPage.class)`

## HtmlValidator

Pure helper returning pass/fail + reason: balanced tags, no `<script>`, has a
`<title>`, non-empty body.
