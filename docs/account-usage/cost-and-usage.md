# Cost, Metering & Consumption Views

These views answer **"where are my credits going?"** They are the backbone of any
cost dashboard. Compute is reported in **credits**; multiply by your price per
credit to get currency (or use `ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY`).

Snowflake credit consumption falls into two buckets:

1. **Virtual warehouse compute** — user‑managed warehouses (`WAREHOUSE_METERING_HISTORY`).
2. **Serverless / cloud‑services** — Snowflake‑managed compute for features like
   clustering, materialized views, search optimization, Snowpipe, serverless
   tasks, query acceleration, replication, and AI functions.

---

## `METERING_HISTORY`

Hourly credit usage for the **whole account**, broken down by service type. Best
single source for "total credits over time" trend lines.

| Column | Type | Notes |
|--------|------|-------|
| `SERVICE_TYPE` | TEXT | e.g. `WAREHOUSE_METERING`, `PIPE`, `AUTO_CLUSTERING`, `MATERIALIZED_VIEW`, `SEARCH_OPTIMIZATION`, `REPLICATION`, `SERVERLESS_TASK`, `QUERY_ACCELERATION`. |
| `NAME` | TEXT | Object name where applicable (e.g. warehouse). |
| `START_TIME` / `END_TIME` | TIMESTAMP_LTZ | Hourly window. |
| `CREDITS_USED_COMPUTE` | NUMBER | Compute credits. |
| `CREDITS_USED_CLOUD_SERVICES` | NUMBER | Cloud‑services credits. |
| `CREDITS_USED` | NUMBER | Total (compute + cloud services). |

---

## `METERING_DAILY_HISTORY`

Daily roll‑up by service type, and the place that reflects the **daily 10%
cloud‑services adjustment/credit**. Use it for daily spend charts and to
reconcile billing.

| Column | Type | Notes |
|--------|------|-------|
| `SERVICE_TYPE` | TEXT | Service category. |
| `USAGE_DATE` | DATE | The day. |
| `CREDITS_USED_COMPUTE` | NUMBER | Compute credits. |
| `CREDITS_USED_CLOUD_SERVICES` | NUMBER | Cloud‑services credits. |
| `CREDITS_USED` | NUMBER | Billable total after adjustment. |
| `CREDITS_ADJUSTMENT_CLOUD_SERVICES` | NUMBER | The free cloud‑services allowance applied. |
| `CREDITS_BILLED` | NUMBER | Net credits billed. |

---

## `WAREHOUSE_METERING_HISTORY`

The most important view for warehouse cost. **One row per warehouse per hour.**

| Column | Type | Notes |
|--------|------|-------|
| `WAREHOUSE_ID` | NUMBER | Stable warehouse identifier (join key). |
| `WAREHOUSE_NAME` | TEXT | Warehouse name. |
| `START_TIME` / `END_TIME` | TIMESTAMP_LTZ | Hourly window. |
| `CREDITS_USED_COMPUTE` | NUMBER | Compute credits in the hour. |
| `CREDITS_USED_CLOUD_SERVICES` | NUMBER | Cloud‑services credits attributed. |
| `CREDITS_USED` | NUMBER | Total credits in the hour. |

**Common uses:** top warehouses by spend, hourly heatmaps, idle‑vs‑busy
analysis (join with `WAREHOUSE_LOAD_HISTORY`), and cloud‑services overhead ratio.

---

## Serverless feature credit views

All follow the same general shape: a time window, the object, and credits.

### `AUTOMATIC_CLUSTERING_HISTORY`
Credits consumed by automatic clustering, **per table**.

| Column | Notes |
|--------|-------|
| `START_TIME` / `END_TIME` | Window. |
| `CREDITS_USED` | Clustering credits. |
| `NUM_BYTES_RECLUSTERED` | Bytes reclustered. |
| `NUM_ROWS_RECLUSTERED` | Rows reclustered. |
| `DATABASE_NAME` / `SCHEMA_NAME` / `TABLE_NAME` (+ IDs) | Target table. |

### `MATERIALIZED_VIEW_REFRESH_HISTORY`
Credits for keeping materialized views up to date.

| Column | Notes |
|--------|-------|
| `START_TIME` / `END_TIME` | Window. |
| `CREDITS_USED` | Maintenance credits. |
| `DATABASE_NAME` / `SCHEMA_NAME` / `TABLE_NAME` | The materialized view. |

### `SEARCH_OPTIMIZATION_HISTORY`
Credits for the Search Optimization Service, per table.

| Column | Notes |
|--------|-------|
| `START_TIME` / `END_TIME` | Window. |
| `CREDITS_USED` | SOS credits. |
| `DATABASE_NAME` / `SCHEMA_NAME` / `TABLE_NAME` | Optimized table. |

### `SERVERLESS_TASK_HISTORY`
Credits for serverless task execution.

| Column | Notes |
|--------|-------|
| `START_TIME` / `END_TIME` | Window. |
| `CREDITS_USED` | Serverless task credits. |
| `DATABASE_NAME` / `SCHEMA_NAME` / `TASK_NAME` | The task. |

### `QUERY_ACCELERATION_HISTORY`
Credits for the Query Acceleration Service, per warehouse.

| Column | Notes |
|--------|-------|
| `START_TIME` / `END_TIME` | Window. |
| `CREDITS_USED` | QAS credits. |
| `WAREHOUSE_ID` / `WAREHOUSE_NAME` | Accelerated warehouse. |

> Related: `QUERY_ACCELERATION_ELIGIBLE` (where exposed) estimates which queries
> *would* benefit from query acceleration — useful for tuning, not billing.

---

## `PIPE_USAGE_HISTORY`

Snowpipe credits and volume, per pipe. Belongs to both Cost and Data Loading.

| Column | Notes |
|--------|-------|
| `PIPE_ID` / `PIPE_NAME` | The pipe. |
| `START_TIME` / `END_TIME` | Window. |
| `CREDITS_USED` | Snowpipe credits. |
| `BYTES_INSERTED` | Bytes loaded. |
| `FILES_INSERTED` | Files loaded. |

---

## `DATA_TRANSFER_HISTORY`

Egress data transfer (cross‑region / cross‑cloud) bytes and credits.

| Column | Notes |
|--------|-------|
| `START_TIME` / `END_TIME` | Window. |
| `SOURCE_CLOUD` / `SOURCE_REGION` | Origin. |
| `TARGET_CLOUD` / `TARGET_REGION` | Destination. |
| `BYTES_TRANSFERRED` | Volume. |
| `TRANSFER_TYPE` | Category (e.g. `COPY`, `REPLICATION`, `EXTERNAL_*`). |

---

## Replication credit views

See [replication.md](replication.md) for `REPLICATION_USAGE_HISTORY`,
`DATABASE_REPLICATION_USAGE_HISTORY`, and `REPLICATION_GROUP_USAGE_HISTORY`.

---

## AI / Cortex

- `CORTEX_FUNCTIONS_USAGE_HISTORY` — token and credit usage for Cortex AI
  functions (LLM functions, etc.). Column set evolves quickly; verify in‑account.

---

## Putting total cost together

To approximate **total account credits** for a period, sum
`METERING_HISTORY.CREDITS_USED` (it already spans all service types). To
attribute cost to teams/workloads, combine `WAREHOUSE_METERING_HISTORY` (by
warehouse) with `QUERY_ATTRIBUTION_HISTORY` (per query) and your own
warehouse→team mapping. See [dashboard-guide.md](dashboard-guide.md).
