# Architecture — Cash-on-Delivery Settlement with a Courier

## Payment Status Lifecycle

```mermaid
flowchart TD
    A[Delivery with tracking number] --> B[Read shipment from courier<br/>read-only]
    B --> C{Returned / cancelled /<br/>terminated?}
    C -->|yes| NC[Not collected]
    C -->|no| D{Amount to collect<br/>is zero?}
    D -->|yes| PP[Prepaid<br/>nothing for the courier to collect]
    D -->|no| E{Delivered?}
    E -->|no| PE[Pending collection]
    E -->|yes| CO[Collected]
    CO --> F{Part of a payout<br/>that was transferred?}
    F -->|not yet| CO
    F -->|yes| G[Register payment<br/>open residual only<br/>dated on payout date]
    G --> PO[Paid by courier]

    classDef ok fill:#e8f5e9,stroke:#2e7d32
    classDef wait fill:#fff8e1,stroke:#f9a825
    classDef bad fill:#ffebee,stroke:#c62828
    classDef decision fill:#e1f5ff,stroke:#0277bd
    class PO,PP ok
    class PE,CO wait
    class NC bad
    class C,D,E,F decision
```

Order of the checks matters: a return shipment carries no amount to collect, so checking "zero amount" first would label returns as prepaid.

---

## Systems and the Shared Key

```mermaid
flowchart LR
    S[Shopify order] -->|imported by the Shopify connector| O[Odoo sale order]
    S -->|courier app creates the shipment| C[Courier shipment]
    C -->|tracking number<br/>via fulfillment sync| D[Odoo delivery]
    O --> D
    O --> I[Invoice]
    D -->|tracking number| C
    C -->|state + payout| D
    D -->|payment for the residual| I
```

The tracking number is the only identifier all three systems agree on, so every lookup goes through it.

---

## Sync Strategy

```mermaid
flowchart TD
    CR[Scheduled job, every 30 min] --> Q[Select deliveries:<br/>open or collected-not-paid-out<br/>last 60 days<br/>least recently synced first<br/>bounded batch]
    BT[Manual 'Sync' button<br/>Courier User group only] --> R
    Q --> R[For each delivery]
    R --> SP[Savepoint]
    SP --> API[Read shipment]
    API -->|ok| AP[Apply status, payout, pay invoice]
    API -->|key rejected / outage| SK[Skip this company<br/>for the rest of the run]
    AP --> CM[Commit progress]
```

---

## Module Layout (illustrative — generic names)

```
courier_cod_settlement/
├── models/
│   ├── courier_api.py        thin read-only HTTP client: auth header, timeouts, 404/401 handling
│   ├── stock_picking.py      status mapping, apply shipment data, register payment, cron
│   ├── sale_order.py         stored computed status of the latest delivery
│   └── res_company.py        API key, API URL, payout journal
├── security/
│   └── res_groups.xml        "Courier User" group
├── data/
│   └── ir_cron.xml           30-minute safety-net sync
├── views/                    delivery tab, order list badge, filters, group-by
└── tests/                    status mapping, idempotent payment, returns, group access
```

## Why This Layout

- **The HTTP client knows nothing about Odoo business rules**, and the business rules never build URLs. Each can be tested on its own.
- **The status mapping is a pure function** of (state, amount to collect), so every edge case is a one-line test.
- **Payment goes through Odoo's standard register-payment wizard**, not hand-built journal entries, so reconciliation, payment state, and reports behave exactly like a manual payment.
