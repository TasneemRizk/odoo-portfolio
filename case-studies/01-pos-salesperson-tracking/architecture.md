# Architecture — POS Per-Line Salesperson Tracking

## Component & Data Flow

```mermaid
flowchart TD
    subgraph Backend["Odoo Backend (Python)"]
        A[res.config.settings] -->|writes| B[pos.config<br/>allowed_salesperson_ids: M2M]
        B -->|inherited via config_id| C[pos.session]
        D[pos.order.line<br/>sale_person_id: M2O res.users]
    end

    subgraph Loader["POS Data Loader"]
        B -.->|_load_pos_data_fields| E[Browser-side pos.config record]
        F[res.users] -.->|_load_pos_data| G[Browser-side users cache]
    end

    subgraph Frontend["POS Frontend (OWL)"]
        E --> H[SalespersonSelector<br/>OWL Component]
        G --> H
        H -->|writes via patched setter| I[PosOrderline<br/>JS model + sale_person_id]
        I -->|t-inherit reactive| J[OrderlineReceipt template]
    end

    I -.->|persisted on session close| D

    classDef backend fill:#e1f5ff,stroke:#0277bd
    classDef loader fill:#fff4e1,stroke:#f57c00
    classDef frontend fill:#f3e5f5,stroke:#6a1b9a
    class A,B,C,D backend
    class E,F,G loader
    class H,I,J frontend
```

---

## Lifecycle of One Order Line

```mermaid
sequenceDiagram
    actor Cashier
    participant Selector as SalespersonSelector (OWL)
    participant Line as PosOrderline (JS model)
    participant Receipt as OrderlineReceipt template
    participant Server as Odoo Server

    Cashier->>Selector: Click salesperson dropdown
    Selector->>Line: line.setSalesperson(user)
    Line-->>Receipt: reactive update propagates
    Receipt-->>Cashier: "Sold by: Alice" appears live
    Note over Cashier,Server: ... session continues ...
    Cashier->>Server: Validate order
    Server->>Server: Persist sale_person_id on pos.order.line
```

---

## Why this layout

- **One source of truth per concern:** the M2M whitelist lives on `pos.config` (configuration), the assignment lives on `pos.order.line` (transaction). No duplication.
- **Data ships with bootstrap:** `_load_pos_data_fields` augments the existing POS payload — no extra round-trip per order.
- **OWL reactivity does the redraw:** the selector never calls `render()`. Writing to `props.line.sale_person_id` is enough because the receipt template reads from the same reactive proxy.
- **Patch composition:** `patch(PosOrderline.prototype, ...)` keeps the customization composable with other addons that may patch the same class.
