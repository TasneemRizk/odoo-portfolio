# Architecture — Real-Time Government Ledger Reporting Gateway

## Submission Lifecycle

```mermaid
flowchart TD
    A[Journal entry posted] --> B{In reporting scope?<br/>posted · dated after go-live · accounts mapped}
    B -->|no| Z[Ignored]
    B -->|yes| C[Resolve facility per accounting line]
    C --> D{Single facility?}
    D -->|yes| E[One balanced batch]
    D -->|no, spans facilities| F[Split into balanced<br/>per-facility batches]
    E --> G[Attach idempotency key]
    F --> G
    G --> H[Background thread,<br/>fired AFTER commit]
    H --> I[Gateway]
    I -->|2xx| J[Mark sent + log response]
    I -->|error| K{Retryable?}
    K -->|yes, under attempt cap| L[Backoff + requeue]
    K -->|no, or cap reached| M[Notify responsible user]
    L -.retry.-> H
    N[Scheduled job] -. safety net only,<br/>retries what H could not land .-> H

    classDef entry fill:#fff4e1,stroke:#f57c00
    classDef decision fill:#e1f5ff,stroke:#0277bd
    classDef core fill:#f3e5f5,stroke:#6a1b9a
    classDef error fill:#ffebee,stroke:#c62828
    class A entry
    class B,D,K decision
    class C,E,F,G,H,I,J core
    class L,M error
```

---

## Facility Split — Why a Single Entry Can Become Several Batches

```mermaid
flowchart LR
    subgraph Entry["One posted journal entry"]
        L1[Line: Property A revenue]
        L2[Line: Property B revenue]
        L3[Line: shared tax line]
    end
    L1 --> R{Resolve facility<br/>per line}
    L2 --> R
    L3 --> R
    R --> PA[Batch: Property A<br/>balanced, own idempotency key]
    R --> PB[Batch: Property B<br/>balanced, own idempotency key]
    PA --> G[Gateway]
    PB --> G
```

A shared line (e.g. a tax or discount line not tied to one property) is allocated across the facilities it affects before the batches are built, so each outgoing batch is independently balanced — the gateway rejects anything that doesn't net to zero.

---

## Credential & Environment Model

```mermaid
flowchart TD
    P1[Property A] -->|own facility ID + secret| T1[OAuth2 token, cached per facility]
    P2[Property B] -->|own facility ID + secret| T2[OAuth2 token, cached per facility]
    T1 --> M{Test Mode?}
    T2 --> M
    M -->|on| TT[Submission-Mode: test<br/>→ non-reportable tables]
    M -->|off| PP[Submission-Mode: production<br/>→ reportable tables]
```

Credentials, tokens, and the test/production mode are all scoped per property — a misconfiguration on one facility cannot leak into another, and flipping test mode off is the explicit, deliberate step that makes a facility's submissions count for real.

---

## Module Layout (illustrative — generic names, not the real module)

```
gov_ledger_reporting_gateway/
├── models/
│   ├── gateway_client.py        OAuth2 token fetch/cache, HTTP client, envelope handling
│   ├── account_move.py          scope check, facility split, batch build, send, retry, failure handling
│   ├── account_account.py       one field: mapped government account number
│   └── submission_log.py        durable, queryable record of every request/response
├── data/
│   └── ir_cron.xml              safety-net retry job only
├── security/
│   └── ir.model.access.csv
└── views/
    ├── res_config_settings_views.xml   gateway URL, test mode, instant-send, retry config
    └── submission_log_views.xml
```

---

## Why This Layout

- **The send/retry state machine lives on the document being sent** (`account_move`), not in a separate service object — the entry's own state (sent / pending / failed) is always consistent with what's actually happened to it.
- **The HTTP client is a thin, swappable layer.** Token caching and request/response shaping live in one place, so the retry logic never has to know about OAuth mechanics directly.
- **The submission log is append-only and queryable** — every attempt, not just the latest one, so a support conversation about "did entry X get filed" is a lookup, not a guess.
- **Facility resolution is computed once per entry, before any network call** — the split decision is pure data logic, independently testable without a live (or mocked) gateway.
