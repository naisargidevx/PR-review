# Mermaid Diagram Templates

Ready-to-use Mermaid templates for documenting feature flows in PR descriptions. Pick the best type based on the change. Customize with actual names from the diff.

> **Confidence note:** If code intent is unclear, always prefix the diagram block with:
> `> **Note:** Best-effort generated diagram — please verify accuracy with the author.`

---

## When to Use Which Diagram Type

| Change type | Recommended diagram |
|-------------|---------------------|
| User-facing feature with steps | Flowchart |
| API request/response between services | Sequence diagram |
| Frontend state machine / user journey | Flowchart or State diagram |
| Service-to-service architecture change | Component / C4 diagram |
| Data transformation pipeline | Flowchart |
| Auth/token flow | Sequence diagram |
| Background job / worker flow | Flowchart |
| Database interaction pattern | Sequence diagram |

---

## Template 1 — Flowchart (Process / Decision Flow)

Best for: user flows, business logic, decision trees, data pipelines.

```mermaid
flowchart TD
    A([Start: User Action]) --> B[Step 1: Validate Input]
    B --> C{Valid?}
    C -- Yes --> D[Step 2: Process Data]
    C -- No --> E[Return Validation Error]
    D --> F[Step 3: Save to DB]
    F --> G{Save Success?}
    G -- Yes --> H([Return Success Response])
    G -- No --> I[Log Error]
    I --> J([Return 500 Error])
```

### Flowchart node shapes reference:
- `[Text]` — rectangle (process step)
- `{Text}` — diamond (decision/condition)
- `([Text])` — rounded (start/end)
- `(Text)` — rounded rectangle (subprocess)
- `[[Text]]` — subroutine
- `[(Text)]` — database
- `>Text]` — asymmetric (annotation)

### Flowchart direction options:
- `TD` — top to bottom
- `LR` — left to right
- `BT` — bottom to top
- `RL` — right to left

---

## Template 2 — Sequence Diagram (API / Service Interaction)

Best for: REST API flows, authentication handshakes, service-to-service calls, webhooks.

```mermaid
sequenceDiagram
    participant U as User / Client
    participant API as API Gateway
    participant Auth as Auth Service
    participant DB as Database
    participant Cache as Redis Cache

    U->>API: POST /api/v1/login {email, password}
    API->>Auth: validateCredentials(email, password)
    Auth->>DB: findUser(email)
    DB-->>Auth: User record
    Auth->>Auth: verifyPassword(hash)

    alt Valid credentials
        Auth->>Auth: generateJWT(userId)
        Auth-->>API: {token, expiresIn}
        API->>Cache: setSession(userId, token, ttl)
        API-->>U: 200 OK {token}
    else Invalid credentials
        Auth-->>API: AuthError
        API-->>U: 401 Unauthorized
    end
```

### Sequence diagram arrow types:
- `->>` — solid arrow (synchronous call)
- `-->>` — dashed arrow (response/async)
- `-x` — solid with X (failed call)
- `--x` — dashed with X (failed async)
- `-)` — open arrow (async fire-and-forget)

### Sequence control blocks:
- `alt [condition]` ... `else [condition]` ... `end` — conditional
- `loop [text]` ... `end` — loop
- `par [text]` ... `and [text]` ... `end` — parallel
- `opt [text]` ... `end` — optional
- `rect [color]` ... `end` — grouping

---

## Template 3 — State Diagram (State Machine / Status Flow)

Best for: order status transitions, user account states, workflow state machines.

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Submitted: submit()
    Submitted --> InReview: assign_reviewer()
    Submitted --> Draft: request_changes()
    InReview --> Approved: approve()
    InReview --> Rejected: reject()
    InReview --> Draft: request_changes()
    Approved --> Published: publish()
    Rejected --> [*]
    Published --> [*]

    note right of InReview
        Reviewer assigned
        within 24 hours
    end note
```

---

## Template 4 — Component Diagram (Architecture / Module Structure)

Best for: new service integration, module architecture changes, dependency structure.

```mermaid
flowchart LR
    subgraph Client["Client Layer"]
        UI[React Frontend]
        Mobile[Mobile App]
    end

    subgraph API["API Layer"]
        GW[API Gateway]
        Auth[Auth Service]
        Orders[Order Service]
        Notify[Notification Service]
    end

    subgraph Data["Data Layer"]
        PG[(PostgreSQL)]
        Redis[(Redis Cache)]
        S3[(S3 Storage)]
        MQ[Message Queue]
    end

    UI --> GW
    Mobile --> GW
    GW --> Auth
    GW --> Orders
    Orders --> PG
    Orders --> Redis
    Orders --> MQ
    MQ --> Notify
    Notify --> S3
    Auth --> PG
```

---

## Template 5 — Background Job / Worker Flow

Best for: async processing, queue consumers, scheduled jobs, event handlers.

```mermaid
flowchart TD
    subgraph Trigger["Trigger"]
        E1[HTTP Request] --> Q
        E2[Scheduled Cron] --> Q
        E3[Webhook Event] --> Q
    end

    Q[(Message Queue)] --> W

    subgraph Worker["Worker Process"]
        W[Dequeue Job] --> V{Valid Job?}
        V -- No --> DLQ[(Dead Letter Queue)]
        V -- Yes --> P[Process Job]
        P --> R{Success?}
        R -- Yes --> ACK[Acknowledge & Remove]
        R -- No, Retry < Max --> NACK[Nack → Retry with backoff]
        R -- No, Retry >= Max --> DLQ
    end

    subgraph Outcome["Side Effects"]
        ACK --> DB[(Update DB)]
        ACK --> N[Send Notification]
    end
```

---

## Template 6 — Data Flow / Transformation Pipeline

Best for: ETL pipelines, data processing flows, import/export features.

```mermaid
flowchart LR
    Raw[("Raw Input\n(CSV/JSON/API)")] --> Parse[Parse & Validate]
    Parse --> |Valid| Transform[Transform & Enrich]
    Parse --> |Invalid| ErrLog[(Error Log)]
    Transform --> Dedupe[Deduplicate]
    Dedupe --> Store[(Database)]
    Store --> Index[Update Search Index]
    Store --> Cache[Invalidate Cache]
    Index --> Done([Processing Complete])
    Cache --> Done
```

---

## Template 7 — Auth / Token Flow

Best for: OAuth, JWT refresh, SSO flows, API key validation.

```mermaid
sequenceDiagram
    participant App as Client App
    participant API as Resource API
    participant AuthServer as Auth Server

    App->>AuthServer: POST /oauth/token (client_credentials)
    AuthServer-->>App: access_token (exp: 1hr) + refresh_token (exp: 7d)

    App->>API: GET /resource (Bearer: access_token)

    alt Token valid
        API-->>App: 200 Resource Data
    else Token expired
        API-->>App: 401 Unauthorized
        App->>AuthServer: POST /oauth/token (refresh_token)

        alt Refresh valid
            AuthServer-->>App: new access_token
            App->>API: GET /resource (Bearer: new_access_token)
            API-->>App: 200 Resource Data
        else Refresh expired
            AuthServer-->>App: 401 Re-authenticate
            App->>App: Redirect to Login
        end
    end
```

---

## Mermaid Embedding in PR Description

Wrap diagrams in markdown fenced blocks for GitHub/GitLab rendering:

```markdown
## Feature Flow

```mermaid
flowchart TD
    ...
```
```

GitHub, GitLab, and most modern PR tools render Mermaid natively.

For platforms that don't (e.g., Bitbucket), provide a text description as fallback:

```markdown
## Feature Flow

> **Note:** Mermaid diagram — renders on GitHub/GitLab. Plain-text flow below for other platforms.

**Flow summary:**
1. User submits form → API validates input
2. On success → save to DB → return token
3. On failure → return validation error with field details
```

---

## Diagram Accuracy Rules

1. **Only generate diagrams when code intent is clear** — if the flow is ambiguous from the diff alone, do not fabricate
2. **Prefix best-effort diagrams** with: `> **Note:** Best-effort generated — please verify with the author.`
3. **Use real names from the code** — actual function names, service names, endpoint paths from the diff
4. **Show error paths** — always include at least one failure/error branch in flowcharts and sequence diagrams
5. **Keep it simple** — if a feature has 3 steps, don't make a 15-node diagram. Aim for clarity over completeness
6. **No mind maps** — avoid `mindmap` diagram type; it produces unclear, non-actionable diagrams for PRs
