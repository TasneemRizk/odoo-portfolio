# Case Study 06 — Card Payment Settlement with a Payment Gateway (Paymob)

> **Stack:** Odoo 19 · Python · REST (read-only, token auth) · Shopify connector
> **Fictional client:** _Saffron & Sage_ — the same direct-to-consumer brand as [Case 05](../05-cod-courier-settlement/), now for customers who pay online by card through Paymob.
> **Type:** Inbound, read-only settlement integration, built from scratch.

> ℹ️ **A note on this case study.** It is based on a real integration. The client is fictionalized, every number shown is a placeholder, and no source code is reproduced. Paymob is named because it is a public payment gateway; nothing here describes its API beyond what any merchant can see.

---

## For a business reader

After [Case 05](../05-cod-courier-settlement/) fixed cash-on-delivery orders, the natural next question was: *"And the customer who paid online by card? That one is surely paid from the first second, right?"*

It turned out to be the same problem in a different shape. The customer pays on Shopify through Paymob, Shopify says *Paid*, and the order was closed in Odoo immediately. But the money hadn't reached the bank. It sat in the Paymob balance, fees still to be deducted, and was transferred days later in one batch covering many orders.

So we kept **one rule for every payment method**:

> ✅ An order is paid when its money reaches the company's bank account. Not when the courier collects it, and not when the customer pays by card.

The result: each card order shows whether its money is still at Paymob or already transferred, and its invoice is closed automatically on the transfer date.

## For a technical reader

**Problem.** For every Shopify order paid through Paymob, find the Paymob payment, find the **transfer** (payout) that carried it to the bank, and once that transfer is settled, pay the invoice, exactly once, without ever writing to Paymob.

1. **Capture the key at import time.** The *Payment ID* Shopify shows on the order's Paymob transaction is the order's reference at Paymob. The Shopify connector's order-values hook is extended to store it on the sale order as the order is created, from the successful Paymob transaction only.
2. **One order, several attempts.** A customer can retry a declined card several times; each attempt is a separate transaction at the gateway. The successful one wins; otherwise the latest is kept, so a failed order is visible as *Not paid*.
3. **Payment → transfer.** The link between a payment and the transfer that settled it is not a direct field lookup, so it takes one extra resolution step. Transfers are fetched once per company per run and shared by every order in the batch.
4. **Settled means settled.** Only a transfer the gateway reports as settled to the bank turns the order into *Transferred to bank*; until then it is *Paid, awaiting transfer*.
5. **Pay the residual, on the transfer date.** Same idempotent mechanism as Case 05, in a configurable journal, with the transfer reference in the payment memo.

---

## Constraints & Considerations

- **Read-only, strictly.** The client was explicit: nothing is ever sent from Odoo to the gateway. Only GET-style reads after a token exchange.
- **The API key is a secret, not a setting.** It is stored on the company, visible only to a dedicated *Gateway User* group, never logged, and never kept in the repository or test data.
- **Statuses are monotonic.** Once *Transferred to bank*, a later sync can only confirm it.
- **Don't fight the connector.** The Shopify connector has its own *auto-workflow* that can register a payment the moment an order is imported. Leaving it on would close the invoice before the transfer, so the deployment relies on it being off for gateway orders (see *What I Learned*).
- **Independence from Case 05.** The courier module and the gateway module don't depend on each other. Both pay only the open residual, so whichever runs first wins and the other has nothing left to pay.

---

## Architecture

See [`architecture.md`](./architecture.md) for the full diagrams.

```
Shopify order (paid via Paymob) ──► Odoo sale order + Payment ID
                                              │
                       scheduled sync / manual button (read-only)
                                              │
                                              ▼
                         gateway: payment (successful attempt wins)
                                              │
                                              ▼
                              transfer that carried the payment
                                              │
                    ┌─────────────────────────┼──────────────────────┐
                    ▼                         ▼                      ▼
          Paid, awaiting transfer     Transferred to bank        Not paid / Refunded
                                              │
                        register payment for the open residual,
                        dated on the transfer date
```

---

## Key Technical Decisions

- **Hook the import, don't re-query Shopify.** The Payment ID is already in the order payload the connector receives, so the module extends the connector's order-values method instead of calling Shopify again.
- **Share expensive reads across a batch.** One token and one list of recent transfers per company per run, passed into each order's sync, instead of one transfer scan per order.
- **Thin API layer.** A small abstract model owns auth, timeouts, and error mapping (404 → "unknown", 401/403 → "key rejected", other → readable error with the gateway's own message). Business code never builds URLs.
- **Error isolation.** A rejected key or an outage skips that company for the run; a bad order rolls back to its own savepoint; progress is committed as the batch goes.
- **Bounded polling.** Orders already settled or refunded are never polled again, and only the last 60 days are in scope.
- **A second job catches late invoices.** If an order is settled before its invoice is posted, a follow-up step pays invoices of already-settled orders.
- **Access as a feature.** Fields, filters, and the sync button are bound to the *Gateway User* group, and the action itself re-checks the group server-side.

---

## Illustrative Snippets

See [`snippets.md`](./snippets.md): rewritten, generic patterns, not the production source.

---

## Screenshots

| Screenshot | What it shows |
|---|---|
| ![Concept](./screenshots/01-concept.png) | The money journey (card → gateway balance → transfer → Odoo) and where "paid" is now recorded. All values are placeholders. |

---

## What I Learned

> The question "is this order paid?" has one answer, whatever the payment method: has the money reached the bank? The courier and the gateway are just two different roads to the same answer.

1. **Reuse the rule, not just the code.** Case 05's rule carried over unchanged, which is why this module was fast to design: only the "how do we know?" part was new.
2. **Test against the real configuration, not only the code.** On the test instance, one order's invoice was already *Paid* before any transfer existed. The module was fine: the order had been imported a few minutes before the connector's auto-workflow was reconfigured, and that workflow registers a payment on import. The finding became a deployment checklist item rather than a code change.
3. **Find the key that already exists.** In both cases, the solution wasn't building something large. It was finding the one identifier both systems share, and it was in the data from the start.

---

## Want to Talk Through This One?

The module runs for a real client, so the source stays private. I'm glad to walk through the import hook, the batch strategy, and the idempotency design live in an interview.
