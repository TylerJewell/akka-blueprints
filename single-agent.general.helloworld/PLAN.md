# Implementation Plan — `helloworld`

The architecture this blueprint resolves to once [`SPEC.md`](./SPEC.md) runs through `/akka:specify` → `/akka:plan`. Diagrams are rendered on the Architecture tab of the embedded UI; they carry the Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels.

---

## Component graph

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryTextColor':'#ffffff','primaryBorderColor':'#e6c200','lineColor':'#e6c200','nodeTextColor':'#ffffff'}}}%%
flowchart TB
  client([Browser / API client]) -->|POST /api/ask| ASK[AskEndpoint]
  ASK -->|ask command| QE[(QuestionEntity)]
  QE -. QuestionAsked .-> CONS[AnswerConsumer]
  CONS -->|runSingleTask ANSWER| AGENT[QuestionAnswerer]
  AGENT -->|Answer text confidence| CONS
  CONS -->|recordAnswer / blockAnswer| QE
  QE -. events .-> VIEW[QuestionsView]
  VIEW -->|getAllQuestions / SSE| ASK
  ASK -->|Question list| client
  APP[AppEndpoint] -->|/app static UI| client
```

Solid arrows are synchronous commands; dashed arrows are event subscriptions.

## Interaction sequence

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryTextColor':'#ffffff','primaryBorderColor':'#e6c200','lineColor':'#e6c200','actorTextColor':'#ffffff','noteTextColor':'#ffffff'}}}%%
sequenceDiagram
  participant U as User
  participant E as AskEndpoint
  participant Q as QuestionEntity
  participant C as AnswerConsumer
  participant A as QuestionAnswerer
  U->>E: POST /api/ask { question }
  E->>Q: ask(text)
  Q-->>E: questionId
  E-->>U: { questionId }
  Q-->>C: QuestionAsked
  C->>A: runSingleTask(ANSWER)
  Note over A: before-agent-response check on confidence/content
  A-->>C: Answer { text, confidence }
  alt passes check
    C->>Q: recordAnswer(text, confidence)
  else blocked
    C->>Q: blockAnswer(reason)
  end
  Q-->>E: SSE update (ANSWERED or BLOCKED)
```

## State machine

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryTextColor':'#ffffff','primaryBorderColor':'#e6c200','lineColor':'#e6c200','nodeTextColor':'#ffffff','stateLabelColor':'#ffffff','transitionLabelColor':'#cccccc'}}}%%
stateDiagram-v2
  [*] --> ASKED: ask(text)
  ASKED --> ANSWERED: recordAnswer (check passed)
  ASKED --> BLOCKED: blockAnswer (check failed)
  ANSWERED --> [*]
  BLOCKED --> [*]
```

State boxes and transition labels need the CSS overrides in Lesson 24 (white `.stateLabel`, `overflow:visible` on edge-label `foreignObject`). The generated `index.html` must inherit them.

## Entity model

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#1a1a1a','primaryTextColor':'#ffffff','primaryBorderColor':'#e6c200','lineColor':'#e6c200'}}}%%
erDiagram
  QUESTION ||--o{ QUESTION_EVENT : "event-sourced from"
  QUESTION_EVENT ||--|| QUESTIONS_VIEW : "projected into"
  QUESTION {
    string id
    string text
    enum status
    instant askedAt
    string answerText
    double confidence
    instant answeredAt
    string blockedReason
    instant blockedAt
  }
  QUESTION_EVENT {
    string variant "QuestionAsked | AnswerRecorded | AnswerBlocked"
    instant timestamp
  }
  QUESTIONS_VIEW {
    string id
    enum status
  }
```

## Component table

| Component | Kind | File | Purpose |
|---|---|---|---|
| `QuestionAnswerer` | AutonomousAgent | `application/QuestionAnswerer.java` | Answers one question; returns `Answer`. |
| `AnswererTasks` | task definitions | `application/AnswererTasks.java` | The `ANSWER` `Task<Answer>` constant. |
| `Answer` | record | `application/Answer.java` | `{ text, confidence }`. |
| `QuestionEntity` | EventSourcedEntity | `application/QuestionEntity.java` | Per-question lifecycle. Commands: `ask`, `recordAnswer`, `blockAnswer`, `getQuestion`. |
| `QuestionsView` | View | `application/QuestionsView.java` | Row type `Question`; one query `getAllQuestions`. |
| `AnswerConsumer` | Consumer | `application/AnswerConsumer.java` | On `QuestionAsked`, runs the ANSWER task and records the result. |
| `AskEndpoint` | HttpEndpoint | `api/AskEndpoint.java` | `/api/*` HTTP API + SSE + metadata. |
| `AppEndpoint` | HttpEndpoint | `api/AppEndpoint.java` | Serves `/` redirect and `/app/*`. |
| `Question` / `QuestionStatus` / `QuestionEvent` | domain | `domain/*.java` | State, enum, sealed events. |
| `Bootstrap` | service-setup | `Bootstrap.java` | Startup wiring. |

Akka component count: **2 http-endpoint · 1 view · 1 consumer · 1 autonomous-agent · 1 event-sourced-entity · 1 service-setup**.

## Concurrency notes

- **Step / call timeouts (Lesson 4).** `AnswerConsumer`'s call to `runSingleTask` + `forTask(taskId).result(ANSWER)` must allow for a 10–30s LLM round-trip; configure the consumer's effect timeout to at least 60s. Never rely on the 5s default.
- **Idempotency.** `AnswerConsumer` keys the agent session on `"answer-"+questionId`, so a redelivered `QuestionAsked` event reuses the same task session rather than spawning a duplicate answer.
- **No saga / compensation.** Single agent, single write-back; there is no multi-step transaction to compensate. A failed answer call surfaces as the question remaining in `ASKED`; the consumer retries on redelivery.
- **View indexing (Lesson 2).** `QuestionsView` exposes one query over all rows; status filtering happens client-side in `AskEndpoint` because the enum column cannot be auto-indexed.
