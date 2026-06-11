# Warehouse & Compute Views

These views provide comprehensive data on virtual warehouse usage, credit consumption, load patterns, and operational events. They are the primary source for **cost optimization**, **capacity planning**, and **performance tuning** of your compute resources.

---

## WAREHOUSE_METERING_HISTORY

**Purpose:** Records hourly credit consumption for every virtual warehouse. This is the most direct view for understanding compute costs.

**Data Latency:** Up to 3 hours (CREDITS_USED_CLOUD_SERVICES can lag up to 6 hours)  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the one-hour measurement window |
| `END_TIME` | TIMESTAMP_LTZ | End of the one-hour measurement window |
| `WAREHOUSE_ID` | NUMBER | Internal unique identifier for the warehouse |
| `WAREHOUSE_NAME` | TEXT | Name of the virtual warehouse |
| `CREDITS_USED` | NUMBER | Total credits used (compute + cloud services) |
| `CREDITS_USED_COMPUTE` | NUMBER | Credits used by the compute cluster |
| `CREDITS_USED_CLOUD_SERVICES` | NUMBER | Credits used by cloud services (metadata, compilation, etc.) |
| `CREDITS_ATTRIBUTED_COMPUTE_QUERIES` | NUMBER | Credits attributed to active query execution (excludes idle time; NULL for adaptive warehouses) |

**Sample Queries:**
```sql
-- Daily credit consumption by warehouse (last 30 days)
SELECT DATE(start_time) AS usage_date,
       warehouse_name,
       SUM(credits_used) AS total_credits,
       SUM(credits_used_compute) AS compute_credits,
       SUM(credits_used_cloud_services) AS cloud_services_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;

-- Top 10 most expensive warehouses this month
SELECT warehouse_name,
       SUM(credits_used) AS total_credits,
       ROUND(SUM(credits_used) * 3.0, 2) AS estimated_cost_usd  -- adjust rate as needed
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY total_credits DESC
LIMIT 10;

-- Hourly usage trend for a specific warehouse
SELECT start_time, credits_used, credits_used_compute
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE warehouse_name = 'MY_WAREHOUSE'
  AND start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY start_time;

-- Identify warehouses with high cloud services vs compute ratio
SELECT warehouse_name,
       SUM(credits_used_compute) AS compute,
       SUM(credits_used_cloud_services) AS cloud_services,
       ROUND(SUM(credits_used_cloud_services) /
             NULLIF(SUM(credits_used_compute), 0) * 100, 2) AS cloud_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY cloud_pct DESC;
```

---

## WAREHOUSE_LOAD_HISTORY

**Purpose:** Records the average query load (running, queued, blocked) for each warehouse over time. Essential for **right-sizing warehouses** and detecting **query queuing issues**.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the measurement interval |
| `END_TIME` | TIMESTAMP_LTZ | End of the measurement interval |
| `WAREHOUSE_ID` | NUMBER | Internal warehouse identifier |
| `WAREHOUSE_NAME` | TEXT | Name of the warehouse |
| `AVG_RUNNING` | FLOAT | Average number of queries executing concurrently |
| `AVG_QUEUED_LOAD` | FLOAT | Average number of queries queued due to warehouse concurrency limits |
| `AVG_QUEUED_PROVISIONING` | FLOAT | Average queries queued while warehouse was provisioning |
| `AVG_BLOCKED` | FLOAT | Average queries blocked by transaction locks |

**Sample Queries:**
```sql
-- Identify peak load periods for a warehouse
SELECT start_time, warehouse_name,
       avg_running, avg_queued_load, avg_queued_provisioning, avg_blocked
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE warehouse_name = 'MY_WAREHOUSE'
  AND start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY avg_queued_load DESC
LIMIT 20;

-- Find warehouses consistently hitting concurrency limits (avg_queued > 1)
SELECT warehouse_name,
       COUNT(*) AS intervals_with_queuing,
       AVG(avg_queued_load) AS avg_queue_depth,
       MAX(avg_queued_load) AS max_queue_depth
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND avg_queued_load > 0
GROUP BY warehouse_name
ORDER BY avg_queue_depth DESC;
```

---

## WAREHOUSE_EVENTS_HISTORY

**Purpose:** Records all operational events for warehouses — such as starts, suspensions, resumes, resizing, and failures. Useful for understanding warehouse lifecycle and unexpected interruptions.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `TIMESTAMP` | TIMESTAMP_LTZ | When the event occurred |
| `WAREHOUSE_ID` | NUMBER | Internal warehouse identifier |
| `WAREHOUSE_NAME` | TEXT | Name of the warehouse |
| `CLUSTER_NUMBER` | NUMBER | Cluster number in a multi-cluster warehouse |
| `EVENT_NAME` | TEXT | Event type (e.g., STARTED, SUSPENDED, RESUMED, RESIZED, FAILURE) |
| `EVENT_REASON` | TEXT | Reason for the event (e.g., AUTO_SUSPEND, USER_REQUEST) |
| `EVENT_STATE` | TEXT | State after the event |
| `USER_NAME` | TEXT | User who triggered the event (NULL for automatic events) |
| `ROLE_NAME` | TEXT | Role used when triggering the event |

**Sample Queries:**
```sql
-- View recent warehouse events
SELECT timestamp, warehouse_name, event_name, event_reason, user_name
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE timestamp >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY timestamp DESC;

-- Count auto-suspend events per warehouse (useful for sizing analysis)
SELECT warehouse_name,
       COUNT(*) AS auto_suspend_count
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
WHERE event_reason = 'AUTO_SUSPEND'
  AND timestamp >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY auto_suspend_count DESC;
```

---

## RESOURCE_MONITORS

**Purpose:** Lists all resource monitors in your account. Resource monitors control credit usage limits and trigger alerts or automatic suspension when thresholds are hit.

**Data Latency:** 1 hour  
**Retention:** Current state only (no history)

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `NAME` | TEXT | Name of the resource monitor |
| `CREDIT_QUOTA` | NUMBER | Total credit quota for the monitoring period |
| `USED_CREDITS` | NUMBER | Credits used so far in the current period |
| `REMAINING_CREDITS` | NUMBER | Credits remaining before quota is reached |
| `LEVEL` | TEXT | Level of monitoring: ACCOUNT or WAREHOUSE |
| `FREQUENCY` | TEXT | Reset frequency: DAILY, WEEKLY, MONTHLY, YEARLY, NEVER |
| `START_TIME` | TIMESTAMP_LTZ | When the current monitoring period started |
| `END_TIME` | TIMESTAMP_LTZ | When the monitoring period ends (NULL for never-ending) |
| `NOTIFY_AT` | TEXT | Percentage thresholds that trigger notifications |
| `SUSPEND_AT` | NUMBER | Percentage at which warehouses are suspended |
| `SUSPEND_IMMEDIATELY_AT` | NUMBER | Percentage at which warehouses are immediately suspended |
| `CREATED_ON` | TIMESTAMP_LTZ | When the monitor was created |
| `OWNER` | TEXT | Role that owns the monitor |
| `COMMENT` | TEXT | Optional comment |
| `NOTIFY_USERS` | TEXT | Users to notify on threshold events |

**Sample Query:**
```sql
-- Check credit quota utilization across all resource monitors
SELECT name,
       credit_quota,
       used_credits,
       remaining_credits,
       ROUND(used_credits / NULLIF(credit_quota, 0) * 100, 2) AS pct_used,
       level, frequency
FROM SNOWFLAKE.ACCOUNT_USAGE.RESOURCE_MONITORS
ORDER BY pct_used DESC;
```

---

## Warehouse Cost Optimization Queries

### 1. Idle Warehouse Detection
```sql
-- Warehouses with high idle time (lots of credits used but few queries)
SELECT wmh.warehouse_name,
       SUM(wmh.credits_used) AS total_credits,
       COUNT(DISTINCT qh.query_id) AS total_queries,
       ROUND(SUM(wmh.credits_used) / NULLIF(COUNT(DISTINCT qh.query_id), 0), 4) AS credits_per_query
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY wmh
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
  ON qh.warehouse_name = wmh.warehouse_name
  AND qh.start_time BETWEEN wmh.start_time AND wmh.end_time
WHERE wmh.start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY wmh.warehouse_name
ORDER BY credits_per_query DESC NULLS LAST
LIMIT 20;
```

### 2. Warehouse Sizing Recommendations
```sql
-- Warehouses that are frequently queuing (may need to scale up)
SELECT warehouse_name,
       AVG(avg_running) AS avg_concurrent_queries,
       AVG(avg_queued_load) AS avg_queued,
       SUM(CASE WHEN avg_queued_load > 1 THEN 1 ELSE 0 END) AS intervals_with_high_queue
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
HAVING avg_queued > 0.5
ORDER BY avg_queued DESC;
```

### 3. Peak Hours Analysis
```sql
-- Average credits used by hour of day (to identify peak cost periods)
SELECT EXTRACT(HOUR FROM start_time) AS hour_of_day,
       ROUND(AVG(credits_used), 4) AS avg_hourly_credits,
       ROUND(SUM(credits_used), 2) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY 1;
```
