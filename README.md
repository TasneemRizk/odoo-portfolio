# Odoo Engineering Portfolio — Tasneem Rezk

Hands-on case studies from real Odoo engineering work across **HR, POS, payments, and integrations**, distilled into the **problem → constraints → decisions → what I learned** format.

> This repo is intentionally **case studies, not source code**. See [Want to See the Full Code?](#want-to-see-the-full-code) below for what's shared and what isn't.

Odoo Developer, 3+ years, 500+ delivered tasks across 30+ clients, Odoo 14 → 19. Full background, stack, and engineering principles in [`about.md`](./about.md).

---

## Case Studies

| # | Case Study | Stack | Topic |
|---|---|---|---|
| 01 | [POS Per-Line Salesperson Tracking](./case-studies/01-pos-salesperson-tracking/) | Odoo 19 · Python · OWL JS | Per-line attribution in POS for commission tracking |
| 02 | [Half-Day Leave × Flexible Hours](./case-studies/02-hr-flexible-attendance/) | Odoo 18 · Python | Timezone-aware leave interval bug in attendance sheet |
| 03 | [Partner Sync REST Gateway](./case-studies/03-api-partner-gateway/) | Odoo 18 · Python · HTTP | Token-authenticated REST endpoints for external sync |
| 04 | [Real-Time Government Ledger Reporting Gateway](./case-studies/04-gov-ledger-reporting-gateway/) | Odoo 18 · Python · OAuth2 | Instant, idempotent multi-facility ledger reporting to a government compliance platform |
| 05 | [Cash-on-Delivery Settlement with a Courier](./case-studies/05-cod-courier-settlement/) | Odoo 19 · Python · REST · Shopify | Close COD invoices only when the courier's payout reaches the bank (Bosta) |
| 06 | [Card Payment Settlement with a Payment Gateway](./case-studies/06-card-gateway-settlement/) | Odoo 19 · Python · REST · Shopify | Same rule for card payments: paid when the gateway transfer settles (Paymob) |

Each case study contains:
- **README** — narrative of the problem and the engineering decisions.
- **architecture.md** — Mermaid diagram of the data flow / components.
- **snippets.md** — small illustrative code extracts (5–10 lines, intentionally incomplete).
- **screenshots/** — UI captures or concept images, where available (cases 04–06; cases 01–03 have the folder scaffolded but not yet filled in).

---

## Want to See the Full Code?

The complete, runnable modules for cases 01–03 live in a private companion repo. I share access on request for interview / technical-review context.

Cases 04–06 are based on real production integrations for real clients — their full source isn't shared even privately, but I'm glad to walk through their architecture and decisions live.

📩 Contact: open a GitHub issue, or reach out via [LinkedIn](https://www.linkedin.com/in/tasneem-rezk/).

---

## License

Documentation and diagrams in this repository are licensed under [Creative Commons BY-NC-ND 4.0](./LICENSE). Code snippets are illustrative extracts only and are not provided as a redistributable software work.
