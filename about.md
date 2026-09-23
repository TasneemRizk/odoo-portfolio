# About

## Tasneem Rezk — Odoo Developer

I build and maintain Odoo customizations for medium-to-large operational deployments. My work spans the full development lifecycle: requirements analysis, model design, view inheritance, security, frontend (OWL), reporting, and version migration.

---

## Specializations

- **HR & Payroll** — employee lifecycle, contracts, attendance sheet logic (flexible hours, leave intervals, half-day handling), payroll rules, leave allocation.
- **POS Customization** — order line extension, OWL component patching, receipt customization, salesperson attribution.
- **Multi-Version Migration** — Odoo 14 → 15 → 16 → 17 → 18 → 19, including breaking-API adaptation (`attrs` removal, `<tree>` → `<list>`, OWL3 migration).
- **Operational Reporting** — QWeb PDF, Excel (xlsxwriter / openpyxl), wizards.
- **Third-Party & Regulatory Integrations** — OAuth2/token-authenticated outbound gateways, idempotent retry design, multi-entity data reporting to external and government platforms.

## Technical Stack

- **Languages:** Python, JavaScript (OWL), XML, SQL
- **Framework:** Odoo ORM, OWL 2/3, QWeb, HTTP Controllers
- **Tools:** Git, PostgreSQL, Linux, Docker, VS Code

## Versions Worked On

`14` · `15` · `16` · `17` · `18` · `19`

---

## Engineering Principles

1. **Inheritance over modification** — extend Odoo, don't fork it.
2. **Computed fields with explicit dependencies** — predictable invalidation.
3. **Security at the model layer** — `ir.model.access.csv` + `ir.rule`, never bypass with sudo() casually.
4. **OWL patches, not template hacks** — surgical extension via `patch()` and `t-inherit`.

---

## Contact

- LinkedIn: _your-link-here_
- GitHub: [TasneemRizk](https://github.com/TasneemRizk)
