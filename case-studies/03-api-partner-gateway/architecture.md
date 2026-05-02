# Architecture — Partner Sync REST Gateway

## Request Lifecycle

```mermaid
flowchart TD
    A[External Client<br/>curl / Postman / Node.js] -->|HTTPS + Bearer token| B[Odoo werkzeug]
    B --> C[/api/v1/* route dispatcher/]
    C --> D{validate_token<br/>decorator}
    D -->|missing/invalid| E[401 envelope]
    D -->|ok| F{rate_limit<br/>decorator}
    F -->|over limit| G[429 envelope]
    F -->|ok| H[Controller method]
    H --> I[env with token.user_id]
    I --> J[res.partner ORM]
    J --> K[_envelope helper]
    K -->|JSON success| A
    E --> A
    G --> A

    classDef client fill:#fff4e1,stroke:#f57c00
    classDef middleware fill:#e1f5ff,stroke:#0277bd
    classDef core fill:#f3e5f5,stroke:#6a1b9a
    classDef error fill:#ffebee,stroke:#c62828
    class A client
    class B,C,D,F middleware
    class H,I,J,K core
    class E,G error
```

---

## Token Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Issued: POST /api/v1/auth/token
    Issued --> Active: first successful request
    Active --> Active: request (updates last_used_at)
    Active --> Expired: expires_at < now()
    Active --> Revoked: admin sets is_active=False
    Expired --> [*]
    Revoked --> [*]
```

---

## Module Layout

```
api_partner_gateway/
├── models/
│   └── api_token.py            api.token model (issue, validate, rate_limit)
├── controllers/
│   └── main.py                 /api/v1 routes + decorators + envelope helper
├── security/
│   └── ir.model.access.csv     admin-only on api.token
└── views/
    └── api_token_views.xml     backend CRUD for tokens
```

---

## Why This Layout

- **Tokens are a model, not config.** Every issued token is auditable — who, when, from where, last used.
- **Decorators compose.** `@validate_token` and `@rate_limit` each do one thing. Endpoints stay thin.
- **Envelope is a single function.** One place to enforce the response contract; adding fields later (e.g., `meta.request_id`) is a one-line change.
- **No `sudo()` sprinkling.** The token validation rebinds `self.env` to the token owner; endpoints use plain ORM from there on.
- **Rate limiter is swappable.** In-memory for the demo; swap to Redis by implementing the same `check_rate_limit(key)` interface.
