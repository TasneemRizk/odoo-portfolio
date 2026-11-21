# Architecture — Card Payment Settlement with a Payment Gateway

## From Shopify Order to Paid Invoice

```mermaid
flowchart TD
    A[Shopify order imported] --> B{Successful gateway<br/>transaction in payload?}
    B -->|yes| C[Store Payment ID on sale order]
    B -->|no| Z[Not a gateway order — ignored]
    C --> D[Sync: token exchange<br/>read-only]
    D --> E[Read payment by Payment ID<br/>successful attempt wins, else latest]
    E --> F{Refunded / voided?}
    F -->|yes| RF[Refunded]
    F -->|no| G{Successful?}
    G -->|no| NP[Not paid]
    G -->|yes| H[Resolve the transfer<br/>that carried this payment]
    H --> I{Transfer settled<br/>to the bank?}
    I -->|no / not yet| PA[Paid, awaiting transfer]
    I -->|yes| ST[Transferred to bank]
    ST --> J[Register payment<br/>open residual only<br/>dated on transfer date]

    classDef ok fill:#e8f5e9,stroke:#2e7d32
    classDef wait fill:#fff8e1,stroke:#f9a825
    classDef bad fill:#ffebee,stroke:#c62828
    classDef decision fill:#e1f5ff,stroke:#0277bd
    class ST,J ok
    class PA wait
    class NP,RF bad
    class B,F,G,I decision
```

---

## Batch Strategy

```mermaid
flowchart TD
    CR[Scheduled job, hourly] --> Q[Orders with a Payment ID<br/>not settled / refunded<br/>last 60 days<br/>least recently synced first]
    Q --> G[Group by company]
    G --> T[One token + one list of recent transfers<br/>per company]
    T -->|key rejected / outage| SK[Skip company this run]
    T --> L[For each order: savepoint → sync<br/>reuse token + transfers]
    L --> CM[Commit progress]
    CM --> LATE[Pay invoices of orders<br/>settled before they were invoiced]
```

---

## Two Settlement Modules, One Rule

```mermaid
flowchart LR
    subgraph COD["Case 05 — courier"]
        C1[Delivery] --> C2[Payout settled?]
    end
    subgraph CARD["Case 06 — gateway"]
        G1[Sale order] --> G2[Transfer settled?]
    end
    C2 -->|pay open residual| INV[Invoice]
    G2 -->|pay open residual| INV
```

No dependency between the two modules: residual-only payments make them safe side by side.

---

## Module Layout (illustrative — generic names)

```
gateway_card_settlement/
├── models/
│   ├── gateway_api.py      token exchange, request wrapper, payment lookup, paginated transfers
│   ├── sale_order.py       import hook, status mapping, transfer resolution, payment, crons
│   └── res_company.py      API URL, API key (group-restricted), settlement journal
├── security/
│   └── res_groups.xml      "Gateway User" group
├── data/
│   └── ir_cron.xml         hourly sync
├── views/                  order tab, list badge, filters, company settings
└── tests/                  import hook, retries, status mapping, idempotent payment, access
```
