# Architecture — Half-Day Leave × Flexible Hours

## Data Model & Computation Chain

```mermaid
flowchart TD
    A[hr.employee<br/>is_flexible_hours<br/>daily_required_hours<br/>tz] --> B[hr.flex.attendance.sheet<br/>employee_id, date]

    C[hr.attendance<br/>check_in, check_out UTC] -.->|overlap with local day| B
    D[hr.leave<br/>date_from, date_to UTC] -.->|intersect with local day| B
    E[attendance.policy<br/>window_start, window_end<br/>late_grace_minutes] --> B

    B --> F[expected_hours<br/>flex: daily_required<br/>fixed: from policy window]
    B --> G[worked_hours<br/>Σ attendance overlaps]
    B --> H[leave_hours<br/>Σ leave overlaps in tz]
    B --> I[late_minutes<br/>flex: 0<br/>fixed: first_checkin − window_start − grace]

    F --> J[effective_expected<br/>= expected − leave_hours]
    H --> J
    J --> K[deducted_hours<br/>= max 0, effective_expected − worked − late_grace]
    G --> K
    I --> K

    classDef config fill:#e1f5ff,stroke:#0277bd
    classDef source fill:#fff4e1,stroke:#f57c00
    classDef computed fill:#f3e5f5,stroke:#6a1b9a
    classDef output fill:#e8f5e9,stroke:#2e7d32
    class A,E config
    class C,D source
    class B,F,G,H,I,J computed
    class K output
```

---

## The Timezone Interval Intersection

The core temporal operation — intersecting leave and attendance intervals with "today" in the employee's local time:

```mermaid
sequenceDiagram
    participant Sheet as FlexAttendanceSheet
    participant TZ as Employee tz (Africa/Cairo)
    participant Leave as hr.leave record

    Note over Leave: date_from: 2026-03-15 06:00 UTC<br/>date_to:   2026-03-15 10:00 UTC
    Sheet->>TZ: day_start = localize(2026-03-15 00:00)
    Sheet->>TZ: day_end   = localize(2026-03-16 00:00)
    TZ-->>Sheet: day_start_utc = 2026-03-14 22:00 UTC<br/>day_end_utc   = 2026-03-15 22:00 UTC
    Sheet->>Leave: overlap(day_start_utc, day_end_utc)
    Leave-->>Sheet: 06:00 → 10:00 UTC = 08:00 → 12:00 Cairo = morning half
    Note over Sheet: leave_hours = 4.0 (correct: morning half)
```

**Legacy bug:** `day_start` / `day_end` were computed in UTC, so `2026-03-15 00:00 UTC → 2026-03-16 00:00 UTC` was taken as "the day." For Cairo, that window actually spans **02:00 Cairo** on the 15th to **02:00 Cairo** on the 16th — a 24-hour slice offset by 2 hours from the real local day. For employees with a late-night flex window this pushed the leave into the *wrong* local day entirely.

---

## Why This Layout

- **Computed, not stored manually.** Every cell in the sheet is derived from source data — no data entry, no drift.
- **One day per record.** Matches the granularity of payroll computation. Aggregations happen at read time via Odoo's `read_group`.
- **Source-of-truth separation.** Policies configure, attendances/leaves record facts, sheets derive. No mutation flows backward.
- **Timezone handled at boundary computation only.** The business logic works in UTC; only the day-boundary lookup converts to local tz.
