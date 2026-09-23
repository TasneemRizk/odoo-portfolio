# Illustrative Snippets — Government Ledger Reporting Gateway

> ⚠️ **Rewritten, generic pattern — not the production source.** This integration reports to a live government system for a real client, so unlike cases 01–03, no code from the actual module is reproduced here, not even a short extract. The snippets below describe the *shape* of the solution in generic terms only. See also the code screenshot in [`screenshots/04-code-pattern-snap.png`](./screenshots/04-code-pattern-snap.png), which is the same rewritten pattern rendered as an image.

---

## 1. Instant send, fired after commit

Demonstrates: why the submission can't block the posting transaction, and can't run inside it either.

```python
# Illustrative pattern only — not the production source.
def _submit_ledger_entry(self):
    if not self._in_reporting_scope():
        return

    for facility, lines in self._split_by_facility().items():
        batch = self._build_balanced_batch(facility, lines)
        idem_key = self._idempotency_key(facility, batch)

        # Fired after the DB commit — a slow or down gateway
        # can never block posting or roll back a real entry.
        threading.Thread(
            target=self._send_after_commit,
            args=(facility, batch, idem_key),
        ).start()
```

---

## 2. Retryable vs. terminal failure

Demonstrates: why not every failure gets retried the same way.

```python
# Illustrative pattern only — not the production source.
def _handle_failure(self, error):
    if error.is_retryable and self.attempts < self.max_attempts:
        self._schedule_retry(backoff=self._next_backoff())
    else:
        self._notify_responsible_user(error)
```

---

## 3. Facility split (concept)

Demonstrates: the idea, not the implementation, of turning one mixed entry into balanced per-facility batches.

```python
# Illustrative pattern only — not the production source.
def _split_by_facility(self):
    lines_by_facility = defaultdict(list)
    for line in self._ntmp_accounting_lines():          # (renamed for this write-up)
        facility = self._resolve_facility(line)
        lines_by_facility[facility].append(line)
    return {
        facility: self._rebalance(lines)                # shared lines allocated pro-rata
        for facility, lines in lines_by_facility.items()
    }
```

---

## What's deliberately not shown

- The real OAuth2 token exchange and caching implementation.
- The real facility-resolution and pro-rata allocation logic.
- The real retry/backoff schedule and attempt-count bookkeeping.
- The submission-log data model and its views.
- Any real endpoint, header name, or domain used by the actual gateway.

This is by design — the client's integration reports to a live production government system, so the real module stays private even in interview contexts. The architecture and decisions above are real; the code is not.
