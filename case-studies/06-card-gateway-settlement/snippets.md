# Illustrative Snippets — Card Payment Settlement

> ⚠️ **Rewritten, generic patterns — not the production source.** The module runs for a real client, so the code below only shows the *shape* of the solution.

---

## 1. Capture the key while the order is imported

Demonstrates: extending the connector instead of calling Shopify again.

```python
# Illustrative pattern only — not the production source.
def prepare_shopify_order_vals(self, instance, partner, shipping, invoice, order_response, *args, **kwargs):
    vals = super().prepare_shopify_order_vals(
        instance, partner, shipping, invoice, order_response, *args, **kwargs)
    payment_id = self._gateway_payment_id(order_response.get('transaction'))
    if payment_id:                      # successful gateway transaction only
        vals['gateway_payment_ref'] = payment_id
    return vals
```

---

## 2. Several attempts, one answer

Demonstrates: a retried card payment creates several transactions for one order.

```python
# Illustrative pattern only — not the production source.
def _pick_transaction(self, transactions):
    paid = [t for t in transactions if t.get('success')]
    return (paid or transactions)[-1] if transactions else None
```

---

## 3. Share the expensive read across the batch

Demonstrates: one token and one transfer list per company, not per order.

```python
# Illustrative pattern only — not the production source.
for company, orders in orders.grouped('company_id').items():
    try:
        token = api._get_token(company)
        transfers = api._get_transfers(company, token, since=oldest(orders))
    except UserError:
        continue                        # rejected key / outage: skip this company
    for order in orders:
        with self.env.cr.savepoint():
            order._sync_gateway_payment(token=token, transfers=transfers)
        self.env['ir.cron']._commit_progress(1)
```

---

## What's deliberately not shown

- How a payment is resolved to the transfer that settled it.
- The real API routes, payloads, and field names.
- Token handling details and the views.
