# Case Study 01 — POS Per-Line Salesperson Tracking

> **Stack:** Odoo 19 · Python · OWL JS · QWeb
> **Fictional client:** _NorthLane Retail_ — a 40-store fashion retailer used here to illustrate the scenario.
> **Type:** Frontend + backend extension on a transactional model

---

## The Problem

NorthLane Retail pays its in-store sales staff a **commission per item sold**, not a flat salary. Their existing POS workflow attributed the entire order to a single cashier, which made commission accounting inaccurate when **multiple salespeople contributed to one transaction** (a common pattern in their fitting-room → checkout flow).

The standard Odoo POS exposes a salesperson at the order level only. NorthLane needed:

1. A **per-line** salesperson assignment in the POS UI (not per order).
2. A **whitelist** of allowed salespeople per POS config (not all internal users).
3. The salesperson **printed on the receipt line** so the customer's slip doubles as a commission proof.
4. The data persisted on `pos.order.line` so it survives session close and shows up in reports.

---

## Constraints & Considerations

- **No fork of `point_of_sale`.** The customization had to install cleanly alongside vanilla POS and survive Odoo upgrades.
- **OWL3 reactive state.** The selector had to participate in OWL's reactivity so the receipt updated live as the cashier picked a salesperson.
- **POS data loading discipline.** The salespeople list had to ship to the browser via Odoo 19's `_load_pos_data` mechanism — _not_ via an `rpc.query` at runtime — to avoid latency on every order line.
- **Receipt printing without t-inherit conflicts.** Other addons in the stack also patch the receipt template; the xpath had to be specific enough not to clash.

---

## Architecture

See [`architecture.md`](./architecture.md) for the data flow diagram.

At a glance:

```
res.config.settings  ──►  pos.config.allowed_salesperson_ids  (Many2many)
                              │
                              ▼
                    _load_pos_data_fields  ──►  browser-side pos.config record
                              │
                              ▼
                    SalespersonSelector (OWL)  ──►  pos.order.line.sale_person_id
                              │
                              ▼
                    OrderlineReceipt (t-inherit)  ──►  printed receipt
```

---

## Key Technical Decisions

- **Many2many on `pos.config`, not on `pos.session`.** The whitelist is a configuration concern, not a runtime one. Sessions inherit it through `config_id`.
- **Many2one on `pos.order.line`, not Many2many.** A single line is sold by one salesperson; M2M would imply commission splitting which the business explicitly didn't want.
- **`patch(PosOrderline.prototype)` instead of subclassing.** Subclassing breaks when other addons patch the same class; `patch()` composes cleanly.
- **`_load_pos_data_fields` override** instead of a custom RPC: keeps the data shipped with the standard POS bootstrap and benefits from Odoo's caching.
- **No JS state outside the orderline.** The selector reads/writes directly on `props.line` so the receipt auto-updates via OWL reactivity — no manual re-render.

---

## Illustrative Snippets

See [`snippets.md`](./snippets.md) for 3 small extracts demonstrating the patterns used (model inheritance, OWL `patch()`, `t-inherit`).

> Snippets are intentionally incomplete — imports, manifest, and security are omitted. They illustrate _approach_, not a runnable implementation.

---

## Screenshots & Demo

| Screenshot | What it shows |
|---|---|
| ![Settings](./screenshots/01-settings.png) | POS config — "Allowed Salespersons" Many2many in the settings tab |
| ![POS UI](./screenshots/02-pos-ui.png) | Cashier picking a salesperson on an order line |
| ![Receipt](./screenshots/03-receipt.png) | Printed receipt showing "Sold by: …" under each line |

🎬 [`demo.gif`](./demo.gif) — 12 sec walkthrough: open POS → add product → assign salesperson → print receipt.

---

## What I Learned

The hardest part wasn't the model extension — it was the **OWL reactivity contract**. My first version stored the selected salesperson in the `SalespersonSelector` component's local state and called a setter on the orderline; the receipt didn't update until the next render cycle.

Once I removed the local state entirely and made the component a **thin controller over `props.line`**, the receipt updated instantly because OWL's reactive proxy on the orderline propagated the change to every subscriber, including the receipt template. This is the kind of design where _doing less_ is the right answer.

I also internalized the difference between `_load_pos_data_fields` (cheap, declarative) and runtime RPCs (expensive, imperative) for data the cashier needs at every line — a pattern I now reach for first in any POS customization.

---

## Want the Full Code?

The complete, runnable module — including manifest, security, tests, and demo data — is in a private companion repo. **Available on request for interview / technical-review context.**

Reach out via GitHub issue or LinkedIn.
