# Illustrative Snippets — Cash-on-Delivery Settlement

> ⚠️ **Rewritten, generic patterns — not the production source.** The module runs for a real client, so the code below only shows the *shape* of the solution.

---

## 1. Status mapping: returns first

Demonstrates: why the order of checks is part of the business rule.

```python
# Illustrative pattern only — not the production source.
def _payment_status_for(self, state_code, amount_to_collect):
    if state_code in NOT_COLLECTED_STATES:   # a return carries no amount to collect,
        return 'not_collected'              # so it must be caught before "prepaid"
    if not amount_to_collect:
        return 'prepaid'
    if state_code == DELIVERED:
        return 'collected'
    return 'pending'
```

---

## 2. Idempotent payment: pay the residual only

Demonstrates: why running the sync twice can't pay twice.

```python
# Illustrative pattern only — not the production source.
def _register_payout_payment(self):
    if not self._is_in_settled_payout():
        return
    unpaid = self.sale_id.invoice_ids.filtered(
        lambda m: m.state == 'posted' and m.payment_state in ('not_paid', 'partial'))
    if unpaid:
        self.env['account.payment.register'].with_context(
            active_model='account.move', active_ids=unpaid.ids,
        ).create({'journal_id': self._payout_journal().id,
                  'payment_date': self.payout_date}).action_create_payments()
```

---

## 3. Batch-safe cron

Demonstrates: one broken record or one rejected key doesn't stop the run.

```python
# Illustrative pattern only — not the production source.
def _cron_sync(self, limit=100):
    failed_companies = self.env['res.company']
    for picking in self._deliveries_to_sync(limit):   # least recently synced first
        if picking.company_id in failed_companies:
            continue
        try:
            with self.env.cr.savepoint():
                picking._sync_from_courier()
        except UserError:
            failed_companies |= picking.company_id
        self.env['ir.cron']._commit_progress(1)
```

---

## What's deliberately not shown

- How a shipment is resolved to the payout that settled it.
- The real API routes, payloads, and field names.
- The full state-code table and the views.
