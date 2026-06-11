# Dashboard Build Guide

A practical bridge from **monitoring objective → the right view(s) → starter
SQL**. All queries target `SNOWFLAKE.ACCOUNT_USAGE` unless noted. Treat the most
recent hours/day as incomplete due to latency, and remember compute is in
**credits** (multiply by your rate for currency, or use
`ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY`).

## Objective → view map

| Dashboard objective | Primary view(s) |
|---------------------|-----------------|
| Total spend trend | `METERING_DAILY_HISTORY` (org currency: `ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY`) |
| Cost by warehouse | `WAREHOUSE_METERING_HISTORY` |
| Cost by query / user / tag | `QUERY_ATTRIBUTION_HISTORY` + `QUERY_HISTORY` |
| Serverless feature cost | `AUTOMATIC_CLUSTERING_HISTORY`, `PIPE_USAGE_HISTORY`, `MATERIALIZED_VIEW_REFRESH_HISTORY`, `SEARCH_OPTIMIZATION_HISTORY`, `SERVERLESS_TASK_HISTORY`, `QUERY_ACCELERATION_HISTORY` |
| Storage trend & by database | `STORAGE_USAGE`, `DATABASE_STORAGE_USAGE_HISTORY`, `TABLE_STORAGE_METRICS` |
| Warehouse sizing / queuing | `WAREHOUSE_LOAD_HISTORY`, `QUERY_HISTORY` (queue + spill columns) |
| Slow / expensive queries | `QUERY_HISTORY`, `QUERY_ATTRIBUTION_HISTORY` |
| Failed queries | `QUERY_HISTORY` (`EXECUTION_STATUS='FAIL'`) |
| Login / security | `LOGIN_HISTORY`, `SESSIONS`, `USERS` |
| Access reviews | `GRANTS_TO_ROLES`, `GRANTS_TO_USERS` |
| Data ingestion health | `COPY_HISTORY`, `PIPE_USAGE_HISTORY` |
| Governance / lineage | `ACCESS_HISTORY`, `TAG_REFERENCES`, `POLICY_REFERENCES`, `OBJECT_DEPENDENCIES` |
| Data quality | `DATA_QUALITY_MONITORING_RESULTS` |
| Org / multi‑account spend | `ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY` |
| Reader account spend | `READER_ACCOUNT_USAGE.*` |

---

## Starter queries

### 1. Daily total credits by service type (last 30 days)
```sql
SELECT usage_date,
       service_type,
       SUM(credits_used) AS credits
FROM   SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE  usage_date >= DATEADD('day', -30, CURRENT_DATE())
GROUP  BY 1, 2
ORDER  BY 1, 2;
```

### 2. Top 10 warehouses by credits (last 30 days)
```sql
SELECT warehouse_name,
       SUM(credits_used)                 AS total_credits,
       SUM(credits_used_compute)         AS compute_credits,
       SUM(credits_used_cloud_services)  AS cloud_services_credits
FROM   SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE  start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP  BY 1
ORDER  BY total_credits DESC
LIMIT  10;
```

### 3. Most expensive queries by attributed credits (last 7 days)
```sql
SELECT qa.query_id,
       qh.user_name,
       qh.warehouse_name,
       qh.query_tag,
       SUM(qa.credits_attributed_compute) AS credits,
       ANY_VALUE(LEFT(qh.query_text, 200)) AS sample_sql
FROM   SNOWFLAKE.ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY qa
JOIN   SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
       ON qh.query_id = qa.query_id
WHERE  qa.start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP  BY 1, 2, 3, 4
ORDER  BY credits DESC
LIMIT  50;
```

### 4. Warehouses with queuing pressure (right‑sizing signal)
```sql
SELECT warehouse_name,
       AVG(avg_running)             AS avg_running,
       AVG(avg_queued_load)         AS avg_queued_load,
       AVG(avg_queued_provisioning) AS avg_queued_provisioning
FROM   SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE  start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP  BY 1
HAVING AVG(avg_queued_load) > 0
ORDER  BY avg_queued_load DESC;
```

### 5. Queries spilling to remote storage (memory pressure / scale‑up)
```sql
SELECT query_id, user_name, warehouse_name, warehouse_size,
       bytes_spilled_to_remote_storage,
       total_elapsed_time
FROM   SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE  start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND  bytes_spilled_to_remote_storage > 0
ORDER  BY bytes_spilled_to_remote_storage DESC
LIMIT  50;
```

### 6. Storage trend (TB) by database (last 30 days)
```sql
SELECT usage_date,
       database_name,
       (average_database_bytes + average_failsafe_bytes)
         / POWER(1024, 4) AS total_tb
FROM   SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE_HISTORY
WHERE  usage_date >= DATEADD('day', -30, CURRENT_DATE())
ORDER  BY usage_date, database_name;
```

### 7. Largest tables incl. Time Travel & Fail‑safe overhead
```sql
SELECT table_catalog, table_schema, table_name,
       active_bytes              / POWER(1024,3) AS active_gb,
       time_travel_bytes         / POWER(1024,3) AS time_travel_gb,
       failsafe_bytes            / POWER(1024,3) AS failsafe_gb,
       retained_for_clone_bytes  / POWER(1024,3) AS clone_gb
FROM   SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE  deleted = FALSE
ORDER  BY active_bytes DESC
LIMIT  50;
```

### 8. Failed logins by user / IP (last 7 days)
```sql
SELECT user_name, client_ip,
       COUNT(*) AS failed_attempts
FROM   SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE  event_timestamp >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND  is_success = 'NO'
GROUP  BY 1, 2
ORDER  BY failed_attempts DESC;
```

### 9. Dormant (unused) enabled users
```sql
SELECT name, login_name, last_success_login, created_on
FROM   SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE  deleted_on IS NULL
  AND  disabled = 'false'
  AND  (last_success_login IS NULL
        OR last_success_login < DATEADD('day', -90, CURRENT_TIMESTAMP()))
ORDER  BY last_success_login NULLS FIRST;
```

### 10. Load failures by table (last 7 days)
```sql
SELECT table_catalog_name, table_schema_name, table_name,
       COUNT(*)                       AS failed_files,
       ANY_VALUE(first_error_message) AS sample_error
FROM   SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE  last_load_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND  status <> 'Loaded'
GROUP  BY 1, 2, 3
ORDER  BY failed_files DESC;
```

### 11. Estimated cost in currency (apply your rate)
```sql
-- Replace 3.00 with your effective price per credit
WITH params AS (SELECT 3.00::NUMBER AS credit_price)
SELECT m.usage_date,
       SUM(m.credits_used)                  AS credits,
       SUM(m.credits_used) * p.credit_price  AS est_cost
FROM   SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY m
CROSS  JOIN params p
WHERE  m.usage_date >= DATEADD('day', -30, CURRENT_DATE())
GROUP  BY m.usage_date, p.credit_price
ORDER  BY m.usage_date;
```

### 12. Org‑wide spend in currency (multi‑account)
```sql
SELECT usage_date, account_name, usage_type,
       SUM(usage_in_currency) AS cost
FROM   SNOWFLAKE.ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY
WHERE  usage_date >= DATEADD('day', -30, CURRENT_DATE())
GROUP  BY 1, 2, 3
ORDER  BY 1, 2, 3;
```

---

## Implementation tips

- **Materialize for speed & history.** Querying `ACCOUNT_USAGE` directly consumes
  cloud‑services credits and is subject to latency and the 365‑day window.
  Schedule a task to copy daily snapshots into your own reporting tables (a
  "usage mart") for fast BI and history beyond one year.
- **Normalize time zones** (e.g. to UTC) in the modeling layer; source timestamps
  are `TIMESTAMP_LTZ`.
- **Join on IDs, not names.** Names can be reused after a drop; `*_ID` columns are
  stable.
- **Use `QUERY_TAG`** in your applications/sessions to make per‑team/per‑pipeline
  attribution trivial in `QUERY_HISTORY` / `QUERY_ATTRIBUTION_HISTORY`.
- **Least privilege.** Grant the dashboard role only the `SNOWFLAKE` database
  roles it needs (see [concepts.md](concepts.md) §7) rather than `ACCOUNTADMIN`.
- **Validate columns in‑account** with `DESC VIEW SNOWFLAKE.ACCOUNT_USAGE.<view>;`
  before wiring a chart, since Snowflake evolves these views.
