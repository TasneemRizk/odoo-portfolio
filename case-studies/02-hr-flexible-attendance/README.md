# Case Study 02 — Half-Day Leave × Flexible Hours

> **Stack:** Odoo 18 · Python · QWeb
> **Fictional client:** _Meridian Labs_ — a 120-person software consultancy with a flexible-hours policy.
> **Type:** Computed field chain + timezone-aware temporal math

---

## The Problem

Meridian Labs runs a **flexible hours** policy: engineers must log **8 hours per working day** but are free to arrive anywhere between 08:00 and 11:00 — no fixed start time, no "lateness" below a 30-minute grace threshold.

The bug report: an engineer took an **officially-approved half-day leave**, yet the monthly attendance sheet deducted **6 hours** from the payroll computation. The expected behavior was a straightforward **4-hour deduction** (half of the daily requirement) with **no late penalty** because the employee was on leave during the "missing" half.

On paper this looks like a small accounting mistake. In practice it surfaced a **multi-layer interaction** between:

1. Flexible-hours vs. fixed-schedule expected-hour computation.
2. Leave interval calculation across the employee's timezone.
3. The order of operations when subtracting leave time from expected time vs. from worked time.

---

## Constraints & Considerations

- **Payroll correctness is sacred.** Any fix has to be deterministic, auditable, and reversible — no "mostly works" tolerations.
- **Timezone alignment.** Employees are in `Africa/Cairo` but `hr.leave` stores datetimes in UTC; `hr.attendance` check-in/out are stored in UTC; the "day boundary" for half-day decisions must be computed in the **employee's local timezone**.
- **Flexible vs. fixed employees coexist.** The same module serves both populations — the computation must branch on `employee.is_flexible_hours` without duplicating logic.
- **Existing data.** The fix has to re-run cleanly over historical attendance sheets (a `cron` recompute is part of the delivery, not code golf).

---

## The Root Cause

The original computation did:

```
deducted_hours = max(0, expected_hours - worked_hours - late_grace)
```

…which is correct **for fixed schedules**. For flexible-hours employees on half-day leave, it double-counted: the leave reduced neither `expected_hours` nor `worked_hours`, so the missing 4 leave hours were treated as "missing worked hours" and deducted a second time on top of the real 4-hour absence.

The fix:

```
effective_expected = expected_hours - leave_hours
deducted_hours     = max(0, effective_expected - worked_hours - late_grace)
```

i.e. **leave reduces expected, not worked**. The difference is subtle but changes the arithmetic entirely when half-days are involved.

The second layer was **timezone**: the half-day leave stored as `2026-03-15 06:00 UTC → 10:00 UTC` resolved to `08:00 → 12:00 Cairo time` — the *morning* half. But the sheet computation was running in UTC and mapping it to the *night* half of the previous day for employees configured with late-night flexible windows. Forcing the interval intersection into the employee's timezone fixed it.

---

## Architecture

See [`architecture.md`](./architecture.md) for the full computation diagram.

Summary flow:

```
hr.employee (is_flexible_hours, daily_required_hours, tz)
        │
        ▼
hr.flex.attendance.sheet    (one per employee × date)
  ├── expected_hours        ← from employee policy
  ├── worked_hours          ← sum over hr.attendance for that local day
  ├── leave_hours           ← intersection with hr.leave in employee tz
  ├── late_minutes          ← first check-in vs. policy window (flex=0)
  └── deducted_hours        ← (expected − leave) − worked − grace
```

---

## Key Technical Decisions

- **Persistent model, not a wizard.** The sheet is computed once per day and stored — enables audit, recomputation, and integration with payroll.
- **Per-day granularity.** One record per (employee, date). This matches how payroll slices the month and makes the compute tractable.
- **Timezone conversion at the boundary, not throughout.** All datetimes are kept in UTC internally; conversion to the employee's tz happens only when computing day boundaries and interval intersection. This avoids sprinkling `pytz.timezone(...)` through the business logic.
- **`@api.depends` on the right fields.** The computed chain is dependency-tracked so that editing an attendance or leave record invalidates only the affected sheets, not the whole month.
- **Explicit `late_grace` field on the policy.** The grace threshold was a hard-coded 30 minutes in the legacy code. Exposing it as a policy field made the fix testable and configurable per employee category.

---

## Illustrative Snippets

See [`snippets.md`](./snippets.md) for extracts showing the computed-field shape and the timezone-aware interval math. Full implementation lives in the private companion repo.

---

## Screenshots & Demo

| Screenshot | What it shows |
|---|---|
| ![Sheet](./screenshots/01-sheet-list.png) | Monthly view of flex attendance sheets with the new columns |
| ![Form](./screenshots/02-sheet-form.png) | One day's detail with expected/worked/leave/deducted breakdown |
| ![Edge case](./screenshots/03-halfday-edge-case.png) | Before/after comparison on the specific half-day bug scenario |

🎬 [`demo.gif`](./demo.gif) — 15 sec: open sheet → show wrong legacy value → apply policy with fix → show correct 4h deduction.

---

## What I Learned

> **The bug was not where the bug appeared.** The symptom was "wrong deduction on one day"; the cause was a missing subtraction in `expected_hours` combined with a timezone mismap. I burned two hours staring at the deduction formula before I printed the raw leave interval and realized it was falling outside the day entirely in the employee's local time.

Two takeaways I carry forward:

1. **Always print interval arithmetic in the user's timezone first.** The first thing I now add to any attendance debugger is `.astimezone(ZoneInfo(employee.tz))` on every datetime boundary.
2. **Leave subtracts from expectation, not from work.** This framing is so much cleaner than "subtract leave from the deduction." Naming the intermediate variable `effective_expected_hours` made the fix a one-line change once the framing was right.

---

## Want the Full Code?

Complete module — including the 7-case test matrix for flexible vs. fixed × leave types × timezone edge cases — is in the private companion repo. **Available on request for interview context.**
