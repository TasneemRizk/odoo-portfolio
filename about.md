# About

## Tasneem Rezk — Odoo Developer

I customize and extend Odoo for operational businesses — the kind of deployments where the ERP has to be right, because payroll, guest billing, or a government filing depends on it. Over **3+ years** and **500+ delivered tasks across 30+ clients**, I've worked the full lifecycle: requirements analysis, data modeling, view inheritance, security, frontend (OWL), reporting, and version migration — on versions **14 through 19**.

The four case studies in this repo are the kind of problems I actually get handed: a payroll bug that only shows up at a timezone boundary, a POS requirement that looks simple until commission math gets involved, an external system that needs Odoo's data in its shape, not Odoo's. Each one is written up as **problem → constraints → decisions → what I'd change** — not a feature list, because the reasoning is the part worth reading.

---

## What I Actually Do

- **HR & Payroll** — employee lifecycle, contracts, attendance-sheet logic (flexible hours, leave intervals, half-day handling), payroll rules, leave allocation. → [Case 02](./case-studies/02-hr-flexible-attendance/): a timezone-aware leave bug that turned into a lesson about atomic computed-field chains.
- **POS Customization** — order-line extension, OWL component patching, receipt customization, per-line attribution. → [Case 01](./case-studies/01-pos-salesperson-tracking/): why `patch()` beats subclassing, and how OWL reactivity actually propagates to a receipt.
- **Third-Party & Regulatory Integrations** — token-authenticated outbound/inbound gateways (REST, OAuth2 client-credentials), idempotent retry design, multi-entity data reporting to external and government platforms. → [Case 03](./case-studies/03-api-partner-gateway/) and [Case 04](./case-studies/04-gov-ledger-reporting-gateway/): structured error envelopes, post-commit background sends, and turning "must be instant" into an architecture that doesn't block the UI.
- **Multi-Version Migration** — Odoo 14 → 15 → 16 → 17 → 18 → 19, including breaking-API adaptation (`attrs` removal, `<tree>` → `<list>`, OWL2 → OWL3).
- **Operational Reporting** — QWeb PDF, Excel (`xlsxwriter` / `openpyxl`), wizards.

## Technical Stack

- **Languages:** Python, JavaScript (OWL), XML, SQL
- **Framework:** Odoo ORM, OWL 2/3, QWeb, HTTP Controllers
- **Integration:** REST APIs, OAuth2 client-credentials, webhook + polling sync patterns, idempotency design
- **Tools:** Git, PostgreSQL, Linux, Docker, VS Code

## Versions Worked On

`14` · `15` · `16` · `17` · `18` · `19`

---

## How I Work

1. **Inheritance over modification** — extend Odoo, don't fork it.
2. **Computed fields with explicit dependencies** — predictable invalidation, one atomic method over four that can race each other.
3. **Security at the model layer** — `ir.model.access.csv` + `ir.rule`, never bypass with a casual `sudo()`.
4. **OWL patches, not template hacks** — surgical extension via `patch()` and `t-inherit`, additive-only where the framework allows it.
5. **Cut before you add** — an over-modeled abstraction (an unnecessary wizard, an invented error catalogue) is a bug waiting to drift out of sync with reality. Case 04 is half the size it was after removing exactly that.

---

## Contact

- LinkedIn: _add your profile link here_
- GitHub: [TasneemRizk](https://github.com/TasneemRizk)
