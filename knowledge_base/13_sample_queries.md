# Sample Queries Reference — Snowflake Account Usage Dashboard

This document provides a comprehensive collection of ready-to-use SQL queries organized by **dashboard use case**. All queries use the `SNOWFLAKE.ACCOUNT_USAGE` schema.

> **Tip:** Set your session timezone to UTC for consistent timestamp handling across all queries:
> ```sql
> ALTER SESSION SET TIMEZONE = 'UTC';
> ```

---

## 1. Cost & Billing Queries

### 1.1 Current Month Credit Summary
```sql
SELECT 
  service_type,
  SUM(credits_used) AS credits_used,
  SUM(credits_billed) AS credits_billed,
  ROUND(SUM(credits_billed) * 100.0 / SUM(SUM(credits_billed)) OVER (), 2) AS pct_of_total
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATE_TRUNC('month', CURRENT_DATE())
GROUP BY service_type
ORDER BY credits_billed DESC;
```

### 1.2 Daily Spend Trend (Last 90 Days)
```sql
SELECT usage_date,
  SUM(credits_billed) AS total_credits_billed
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD('day', -90, CURRENT_DATE())
GROUP BY usage_date
ORDER BY usage_date;
```

### 1.3 Warehouse Cost Breakdown by Day
```sql
SELECT usage_date,
  name AS warehouse_name,
  credits_billed
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE service_type = 'WAREHOUSE_METERING'
  AND usage_date >= DATEADD('day', -30, CURRENT_DATE())
ORDER BY usage_date DESC, credits_billed DESC;
```

### 1.4 Month-over-Month Credit Comparison
```sql
SELECT 
  DATE_TRUNC('month', usage_date) AS month,
  SUM(credits_billed) AS total_billed
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD('month', -12, DATE_TRUNC('month', CURRENT_DATE()))
GROUP BY 1
ORDER BY 1 DESC;
```

### 1.5 Monthly Budget Burn Rate Projection
```sql
WITH daily_avg AS (
  SELECT AVG(daily_credits) AS avg_daily_credits
  FROM (
    SELECT usage_date, SUM(credits_billed) AS daily_credits
    FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
    WHERE usage_date >= DATE_TRUNC('month', CURRENT_DATE())
      AND usage_date < CURRENT_DATE()
    GROUP BY usage_date
  )
)
SELECT 
  ROUND(avg_daily_credits, 2) AS avg_daily_credits,
  DAY(LAST_DAY(CURRENT_DATE())) AS days_in_month,
  ROUND(avg_daily_credits * DAY(LAST_DAY(CURRENT_DATE())), 2) AS projected_monthly_credits
FROM daily_avg;
```

---

## 2. Warehouse Performance & Utilization Queries

### 2.1 Hourly Credit Usage by Warehouse
```sql
SELECT DATE_TRUNC('hour', start_time) AS hour,
  warehouse_name,
  SUM(credits_used) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;
```

### 2.2 Warehouse Queue Depth (Concurrency Issues)
```sql
SELECT warehouse_name,
  MAX(avg_queued_load) AS peak_queue_depth,
  AVG(avg_queued_load) AS avg_queue_depth,
  SUM(CASE WHEN avg_queued_load > 1 THEN 1 ELSE 0 END) AS intervals_over_threshold
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY avg_queue_depth DESC;
```

### 2.3 Idle Warehouse Detection (Low Query Activity vs. Credits)
```sql
SELECT wmh.warehouse_name,
  SUM(wmh.credits_used) AS total_credits,
  COUNT(qh.query_id) AS query_count,
  ROUND(SUM(wmh.credits_used) / NULLIF(COUNT(qh.query_id), 0), 4) AS credits_per_query
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY wmh
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
  ON qh.warehouse_name = wmh.warehouse_name
  AND qh.start_time BETWEEN wmh.start_time AND wmh.end_time
WHERE wmh.start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY wmh.warehouse_name
HAVING SUM(wmh.credits_used) > 0
ORDER BY credits_per_query DESC NULLS LAST;
```

### 2.4 Peak Usage Hours by Warehouse
```sql
SELECT EXTRACT(HOUR FROM start_time) AS hour_of_day,
  warehouse_name,
  ROUND(AVG(avg_running), 2) AS avg_concurrent_queries,
  ROUND(AVG(avg_queued_load), 4) AS avg_queue
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 2, 1;
```

---

## 3. Query Performance Queries

### 3.1 Slowest Queries (Last 7 Days)
```sql
SELECT query_id, user_name, warehouse_name, query_type,
  ROUND(total_elapsed_time / 1000.0, 2) AS total_sec,
  ROUND(compilation_time / 1000.0, 2) AS compile_sec,
  ROUND(execution_time / 1000.0, 2) AS exec_sec,
  ROUND(bytes_scanned / 1073741824.0, 2) AS gb_scanned,
  LEFT(query_text, 200) AS query_preview
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
ORDER BY total_elapsed_time DESC
LIMIT 25;
```

### 3.2 Query Volume by Hour of Day
```sql
SELECT EXTRACT(HOUR FROM start_time) AS hour_of_day,
  COUNT(*) AS total_queries,
  COUNT(CASE WHEN execution_status = 'FAIL' THEN 1 END) AS failed_queries,
  ROUND(AVG(total_elapsed_time) / 1000.0, 2) AS avg_elapsed_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY 1;
```

### 3.3 Queries with High Data Spill
```sql
SELECT query_id, user_name, warehouse_name, warehouse_size,
  ROUND(bytes_spilled_to_local_storage / 1073741824.0, 2) AS local_spill_gb,
  ROUND(bytes_spilled_to_remote_storage / 1073741824.0, 2) AS remote_spill_gb,
  ROUND(total_elapsed_time / 1000.0, 2) AS elapsed_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND (bytes_spilled_to_local_storage > 0 OR bytes_spilled_to_remote_storage > 0)
ORDER BY bytes_spilled_to_remote_storage DESC, bytes_spilled_to_local_storage DESC
LIMIT 20;
```

### 3.4 Result Cache Hit Rate
```sql
SELECT DATE(start_time) AS query_date,
  COUNT(*) AS total_queries,
  ROUND(AVG(percentage_scanned_from_cache) * 100, 2) AS avg_cache_hit_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND execution_status = 'SUCCESS'
GROUP BY 1
ORDER BY 1 DESC;
```

### 3.5 Top Users by Query Volume and Cost
```sql
SELECT user_name,
  COUNT(*) AS query_count,
  ROUND(SUM(total_elapsed_time) / 3600000.0, 2) AS total_hours,
  ROUND(SUM(credits_used_cloud_services), 4) AS cloud_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY user_name
ORDER BY query_count DESC
LIMIT 20;
```

---

## 4. Storage Queries

### 4.1 Account Storage Summary
```sql
SELECT usage_date,
  ROUND(storage_bytes / 1073741824.0, 2) AS table_storage_gb,
  ROUND(stage_bytes / 1073741824.0, 2) AS stage_storage_gb,
  ROUND(failsafe_bytes / 1073741824.0, 2) AS failsafe_storage_gb,
  ROUND((storage_bytes + stage_bytes + failsafe_bytes) / 1073741824.0, 2) AS total_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
ORDER BY usage_date DESC
LIMIT 30;
```

### 4.2 Storage by Database
```sql
SELECT database_name,
  ROUND(average_database_bytes / 1073741824.0, 2) AS storage_gb,
  ROUND(average_failsafe_bytes / 1073741824.0, 2) AS failsafe_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE_HISTORY
WHERE usage_date = (SELECT MAX(usage_date) FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE_HISTORY)
  AND deleted IS NULL
ORDER BY average_database_bytes DESC;
```

### 4.3 Tables with Highest Storage (Including Time Travel & Fail-safe)
```sql
SELECT table_catalog, table_schema, table_name,
  ROUND(active_bytes / 1073741824.0, 2) AS active_gb,
  ROUND(time_travel_bytes / 1073741824.0, 2) AS time_travel_gb,
  ROUND(failsafe_bytes / 1073741824.0, 2) AS failsafe_gb,
  ROUND((active_bytes + time_travel_bytes + failsafe_bytes) / 1073741824.0, 2) AS total_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE deleted IS NULL
ORDER BY (active_bytes + time_travel_bytes + failsafe_bytes) DESC
LIMIT 20;
```

---

## 5. Security & Access Queries

### 5.1 Failed Login Attempts (Last 24 Hours)
```sql
SELECT event_timestamp, user_name, client_ip,
  reported_client_type, error_code, error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
  AND is_success = 'NO'
ORDER BY event_timestamp DESC;
```

### 5.2 Users with ACCOUNTADMIN Role
```sql
SELECT grantee_name AS username, granted_by, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE role = 'ACCOUNTADMIN'
  AND deleted_on IS NULL
ORDER BY grantee_name;
```

### 5.3 All Privileges Granted to a Role
```sql
SELECT privilege, granted_on, name AS object_name,
  table_catalog AS database_name, table_schema AS schema_name,
  grant_option, granted_by
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE grantee_name = 'MY_ROLE_NAME'  -- replace with target role
  AND deleted_on IS NULL
ORDER BY granted_on, name;
```

### 5.4 Users Who Haven't Logged in Recently
```sql
SELECT name, email, last_success_login, created_on,
  DATEDIFF('day', last_success_login, CURRENT_TIMESTAMP()) AS days_since_login
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND disabled = FALSE
  AND (last_success_login IS NULL
       OR last_success_login < DATEADD('day', -60, CURRENT_TIMESTAMP()))
ORDER BY days_since_login DESC NULLS FIRST;
```

### 5.5 Column-Level Access Audit (Access History)
```sql
SELECT ah.query_start_time,
  ah.user_name,
  f.value:objectName::TEXT AS table_accessed,
  col.value:columnName::TEXT AS column_accessed
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah,
     LATERAL FLATTEN(input => ah.base_objects_accessed) f,
     LATERAL FLATTEN(input => f.value:columns) col
WHERE ah.query_start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND col.value:columnName::TEXT IN ('SSN', 'EMAIL', 'SALARY')  -- adjust to your PII columns
ORDER BY ah.query_start_time DESC;
```

---

## 6. Data Pipeline Monitoring Queries

### 6.1 Recent Load Activity Summary
```sql
SELECT DATE(last_load_time) AS load_date,
  table_catalog_name, table_schema_name, table_name,
  COUNT(*) AS files_loaded,
  SUM(row_count) AS rows_loaded,
  COUNT(CASE WHEN status != 'Loaded' THEN 1 END) AS failed_files
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE last_load_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3, 4
ORDER BY 1 DESC, rows_loaded DESC;
```

### 6.2 Failed Task Executions (Last 7 Days)
```sql
SELECT name, database_name, schema_name, state,
  scheduled_time, error_code, error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE scheduled_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND state = 'FAILED'
ORDER BY scheduled_time DESC;
```

### 6.3 Snowpipe Throughput by Day
```sql
SELECT DATE(start_time) AS day,
  pipe_name,
  SUM(files_inserted) AS files,
  SUM(bytes_inserted) / 1048576.0 AS mb_inserted,
  SUM(credits_used) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 1 DESC, credits DESC;
```

---

## 7. Governance & Compliance Queries

### 7.1 Tables with No Masking Policies on Sensitive Columns
```sql
SELECT c.table_catalog, c.table_schema, c.table_name, c.column_name
FROM SNOWFLAKE.ACCOUNT_USAGE.COLUMNS c
WHERE c.deleted IS NULL
  AND LOWER(c.column_name) REGEXP '.*(ssn|email|phone|salary|credit|dob|birth|gender|race).*'
  AND NOT EXISTS (
    SELECT 1
    FROM SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES pr
    WHERE pr.ref_database_name = c.table_catalog
      AND pr.ref_schema_name = c.table_schema
      AND pr.ref_entity_name = c.table_name
      AND pr.ref_column_name = c.column_name
      AND pr.policy_kind = 'MASKING_POLICY'
  )
ORDER BY c.table_catalog, c.table_schema, c.table_name, c.column_name;
```

### 7.2 Orphaned Objects (No Dependents)
```sql
SELECT t.table_catalog, t.table_schema, t.table_name, t.table_type
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
WHERE t.deleted IS NULL
  AND t.table_type = 'BASE TABLE'
  AND NOT EXISTS (
    SELECT 1
    FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES d
    WHERE d.referenced_database = t.table_catalog
      AND d.referenced_schema = t.table_schema
      AND d.referenced_object_name = t.table_name
  )
ORDER BY t.table_catalog, t.table_schema, t.table_name;
```

### 7.3 Privilege Grant Audit (Last 30 Days)
```sql
SELECT created_on, grantee_name, privilege, granted_on, name, granted_by
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE created_on >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND deleted_on IS NULL
ORDER BY created_on DESC;
```

---

## 8. Automated Maintenance Cost Queries

### 8.1 Automatic Clustering Cost by Table
```sql
SELECT database_name, schema_name, table_name,
  SUM(credits_used) AS total_credits,
  SUM(num_bytes_reclustered) / 1073741824.0 AS gb_reclustered,
  COUNT(*) AS clustering_operations
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3
ORDER BY total_credits DESC
LIMIT 20;
```

### 8.2 Materialized View Refresh Cost
```sql
SELECT materialized_view_name,
  SUM(credits_used) AS total_credits,
  COUNT(*) AS refresh_count,
  AVG(DATEDIFF('second', start_time, end_time)) AS avg_duration_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY total_credits DESC;
```
