# Replication & Failover Views

These views report the credits and bytes consumed by database replication and
replication/failover groups — important for cost dashboards in multi‑region or
business‑continuity setups.

---

## `REPLICATION_USAGE_HISTORY`

Per‑database replication usage (credits + bytes transferred).

| Column | Notes |
|--------|-------|
| `DATABASE_NAME` | Replicated database. |
| `START_TIME` / `END_TIME` | Hourly window. |
| `CREDITS_USED` | Credits consumed by replication. |
| `BYTES_TRANSFERRED` | Bytes transferred for replication. |

---

## `DATABASE_REPLICATION_USAGE_HISTORY`

Per‑database replication detail (similar shape; use whichever your account
exposes for database‑level granularity).

| Column | Notes |
|--------|-------|
| `DATABASE_NAME` / `DATABASE_ID` | Replicated database. |
| `START_TIME` / `END_TIME` | Window. |
| `CREDITS_USED` | Replication credits. |
| `BYTES_TRANSFERRED` | Bytes transferred. |

---

## `REPLICATION_GROUP_USAGE_HISTORY`

Usage for **replication / failover groups** (the modern grouping construct that
replicates multiple objects together).

| Column | Notes |
|--------|-------|
| `REPLICATION_GROUP_NAME` | The group. |
| `START_TIME` / `END_TIME` | Window. |
| `CREDITS_USED` | Group replication credits. |
| `BYTES_TRANSFERRED` | Bytes transferred. |

---

## Replication dashboard ideas

- **Replication spend:** sum `CREDITS_USED` by database/group over time.
- **Transfer volume vs cost:** plot `BYTES_TRANSFERRED` against `CREDITS_USED`.
- **Cross‑reference egress:** combine with `DATA_TRANSFER_HISTORY`
  ([cost-and-usage.md](cost-and-usage.md)) for total cross‑region movement.
