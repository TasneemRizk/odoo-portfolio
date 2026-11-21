# Illustrative Snippets — POS Per-Line Salesperson Tracking

> ⚠️ **These are intentionally incomplete extracts.** Imports, decorators, security files, manifest, and surrounding logic are omitted. They illustrate the _patterns_ used, not a runnable implementation. Full source available privately on request.

---

## 1. Model extension — Many2one with a domain referencing a related M2M

Demonstrates: model inheritance, related field, domain expressed against a related field so the dropdown is filtered to the POS config's whitelist.

```python
# Illustrative extract — pos.order.line extension.
class PosOrderLine(models.Model):
    _inherit = "pos.order.line"

    sale_person_id = fields.Many2one(
        "res.users",
        string="Salesperson",
        domain="[('id', 'in', allowed_salesperson_ids)]",
    )
    allowed_salesperson_ids = fields.Many2many(
        related="order_id.config_id.allowed_salesperson_ids",
    )
    # _load_pos_data_fields override is handled in a separate file.
```

---

## 2. OWL patch — extending the JS PosOrderline class

Demonstrates: `patch()` over the prototype, and letting the POS data layer do the work. `sale_person_id` is shipped by `_load_pos_data_fields`, so the framework already loads, links, and serializes it; the patch only adds a setter and a display helper.

```javascript
// Illustrative extract — OWL patch on the JS-side PosOrderline.
import { patch } from "@web/core/utils/patch";
import { PosOrderline } from "@point_of_sale/app/models/pos_order_line";

patch(PosOrderline.prototype, {
    setSalesperson(user) { this.update({ sale_person_id: user || false }); },
    getSalespersonName() { return this.sale_person_id?.name || ""; },
    // Component logic and template binding live in separate files.
});
```

---

## 3. Receipt template — `t-inherit` extension

Demonstrates: surgical xpath extension of an existing template without copying the whole template, conditional rendering.

```xml
<!-- Illustrative extract — in Odoo 19 one Orderline template renders both the cart and the receipt. -->
<t t-name="pos_orderline_salesperson.Orderline"
   t-inherit="point_of_sale.Orderline" t-inherit-mode="extension">
    <xpath expr="//ul[hasclass('info-list')]" position="inside">
        <li t-if="props.mode === 'receipt' and line.sale_person_id" class="text-muted">
            Sold by: <t t-esc="line.sale_person_id.name"/>
        </li>
    </xpath>
</t>
```

---

## What's deliberately not shown here

- The full `__manifest__.py` (asset bundle declarations, dependency list).
- The `_load_pos_data_fields` override on `res.users` and `pos.config`.
- The complete `SalespersonSelector` OWL component (props validation, dropdown lifecycle, keyboard handling).
- Security access rules (`ir.model.access.csv`).
- Unit tests covering whitelist enforcement and receipt rendering.

These are part of the full module in the private companion repository.
