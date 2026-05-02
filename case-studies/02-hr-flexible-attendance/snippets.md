# Illustrative Snippets — Half-Day Leave × Flexible Hours

> ⚠️ **Intentionally incomplete extracts.** Imports, full `@api.depends` graphs, and surrounding logic are omitted. Full source available privately on request.

---

## 1. The computed-field shape that fixes the bug

Demonstrates: the one-line rearrangement that moves `leave_hours` from the deduction formula to the expectation formula.

```python
# Illustrative extract — flex attendance computation.
@api.depends("expected_hours", "leave_hours", "worked_hours", "policy_id.late_grace_minutes")
def _compute_deducted_hours(self):
    for sheet in self:
        effective_expected = sheet.expected_hours - sheet.leave_hours
        grace = sheet.policy_id.late_grace_minutes / 60.0
        sheet.deducted_hours = max(
            0.0,
            effective_expected - sheet.worked_hours - grace,
        )
```

---

## 2. Timezone-aware day boundary

Demonstrates: how "today for this employee" is computed in the employee's local timezone, not the server's UTC.

```python
# Illustrative extract — computing UTC bounds of a local day.
from datetime import datetime, time
from zoneinfo import ZoneInfo

def _local_day_bounds_utc(self, local_date, tz_name):
    tz = ZoneInfo(tz_name or "UTC")
    day_start_local = datetime.combine(local_date, time.min, tz)
    day_end_local   = datetime.combine(local_date, time.max, tz)
    return day_start_local.astimezone(ZoneInfo("UTC")), \
           day_end_local.astimezone(ZoneInfo("UTC"))
    # Usage: intersect hr.leave and hr.attendance records against this window.
```

---

## 3. Interval intersection for leave hours

Demonstrates: overlap math between a leave interval and the local-day window, in hours.

```python
# Illustrative extract — leave hours falling inside the sheet's day.
def _leave_hours_in_window(self, leaves, day_start_utc, day_end_utc):
    total = 0.0
    for leave in leaves:
        start = max(leave.date_from, day_start_utc)
        end   = min(leave.date_to,   day_end_utc)
        if end > start:
            total += (end - start).total_seconds() / 3600.0
    return total
    # Policy-specific rules (e.g. half-day rounding) handled elsewhere.
```

---

## What's deliberately not shown

- The full `hr.flex.attendance.sheet` model with all computed fields and `_sql_constraints`.
- The `hr.employee` extension (flexible flag, default policy).
- The cron job that recomputes sheets after a historical leave is approved.
- The 7-scenario test matrix (fixed vs. flex × morning/afternoon half-day × timezone DST boundary).
- QWeb report for the monthly summary.

These are in the full module in the private repo.
