# Metering & Billing Views

These views provide the most direct insight into **Snowflake credit consumption** across all services — from virtual warehouses to serverless features. They are the foundation of **cost management dashboards**, **budget tracking**, and **chargeback/showback reporting**.

---

## METERING_HISTORY

**Purpose:** Records **hourly** credit usage for every billable Snowflake service in your account. This is the most granular billing view, showing credits consumed by service type and warehouse.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the one-hour billing window |
| `END_TIME` | TIMESTAMP_LTZ | End of the one-hour billing window |
| `SERVICE_TYPE` | TEXT | Type of service consuming credits (see Service Types table below) |
| `NAME` | TEXT | Name of the service instance (e.g., warehouse name, pipe name) |
| `CREDITS_USED` | NUMBER | Total credits used in this period |
| `CREDITS_USED_COMPUTE` | NUMBER | Credits used for compute operations |
| `CREDITS_USED_CLOUD_SERVICES` | NUMBER | Credits used for cloud services operations |

**Service Types:**

| SERVICE_TYPE Value | Description |
|---------------------|-------------|
| `WAREHOUSE_METERING` | Virtual warehouse compute credits |
| `SNOWPIPE` | Snowpipe continuous ingestion credits |
| `AUTOMATIC_CLUSTERING` | Background automatic clustering credits |
| `MATERIALIZED_VIEW` | Materialized view refresh credits |
| `REPLICATION` | Database/group replication credits |
| `SERVERLESS_TASK` | Serverless task execution credits |
| `SEARCH_OPTIMIZATION` | Search Optimization Service credits |
| `SNOWPARK_CONTAINER_SERVICES` | Snowpark Container Services credits |
| `QUERY_ACCELERATION` | Query Acceleration Service credits |
| `HYBRID_TABLE_REQUESTS` | Hybrid table read/write request credits |

**Sample Queries:**
```sql
-- Hourly credit usage by service type (last 7 days)
SELECT start_time, service_type, name, credits_used
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY start_time DESC, credits_used DESC;

-- Total credits by service type this month
SELECT service_type,
       SUM(credits_used) AS total_credits,
       SUM(credits_used_compute) AS compute_credits,
       SUM(credits_used_cloud_services) AS cloud_services_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
GROUP BY service_type
ORDER BY total_credits DESC;
```

---

## METERING_DAILY_HISTORY

**Purpose:** Records **daily** aggregated credit usage by service type. This is the most convenient view for daily billing reports, trend analysis, and cost dashboards.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `USAGE_DATE` | DATE | Date of the usage measurement |
| `SERVICE_TYPE` | TEXT | Type of service (same values as METERING_HISTORY) |
| `NAME` | TEXT | Service instance name |
| `CREDITS_USED` | NUMBER | Total daily credits used |
| `CREDITS_USED_COMPUTE` | NUMBER | Daily compute credits |
| `CREDITS_USED_CLOUD_SERVICES` | NUMBER | Daily cloud services credits |
| `CREDITS_ADJUSTMENT_CLOUD_SERVICES` | NUMBER | Adjustments applied (10% free cloud services rule) |
| `CREDITS_BILLED` | NUMBER | Final billable credits after adjustments |

**About CREDITS_BILLED:**
Snowflake applies a **10% adjustment** on cloud services credits: if cloud services credits are ≤ 10% of total compute credits for the day, those cloud services credits are free (not billed). `CREDITS_BILLED` reflects this adjustment.

**Sample Queries:**
```sql
-- Daily billed credits by service type (last 30 days)
SELECT usage_date, service_type,
       SUM(credits_used) AS credits_used,
       SUM(credits_billed) AS credits_billed
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD('day', -30, CURRENT_DATE())
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;

-- Monthly cost summary (approximate USD based on contract rate)
SELECT DATE_TRUNC('month', usage_date) AS month,
       SUM(credits_billed) AS total_credits_billed,
       ROUND(SUM(credits_billed) * 3.0, 2) AS estimated_cost_usd  -- update rate to match your contract
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
GROUP BY 1
ORDER BY 1 DESC;

-- Daily spend trend for the last 90 days
SELECT usage_date,
       SUM(credits_billed) AS daily_credits_billed
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD('day', -90, CURRENT_DATE())
GROUP BY usage_date
ORDER BY usage_date DESC;

-- Credit breakdown by service type this month
SELECT service_type,
       SUM(credits_billed) AS total_billed,
       ROUND(SUM(credits_billed) * 100 / SUM(SUM(credits_billed)) OVER (), 2) AS pct_of_total
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATE_TRUNC('month', CURRENT_DATE())
GROUP BY service_type
ORDER BY total_billed DESC;

-- Serverless task cost breakdown
SELECT usage_date, name AS task_name, credits_billed
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE service_type = 'SERVERLESS_TASK'
  AND usage_date >= DATEADD('day', -30, CURRENT_DATE())
ORDER BY usage_date DESC, credits_billed DESC;
```

---

## QUERY_ACCELERATION_HISTORY

**Purpose:** Records the credits used by the **Query Acceleration Service (QAS)** — a Snowflake serverless feature that offloads parts of eligible queries to automatically provisioned compute resources.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the measurement window |
| `END_TIME` | TIMESTAMP_LTZ | End of the measurement window |
| `WAREHOUSE_NAME` | TEXT | Warehouse with QAS enabled |
| `WAREHOUSE_ID` | NUMBER | Internal warehouse identifier |
| `CREDITS_USED` | NUMBER | QAS credits consumed |
| `NUM_FILES_SCANNED` | NUMBER | Files scanned by QAS |
| `NUM_BYTES_SCANNED` | NUMBER | Bytes scanned using QAS |

---

## SEARCH_OPTIMIZATION_HISTORY

**Purpose:** Records credits consumed by the **Search Optimization Service** — a Snowflake feature that speeds up selective point lookups on large tables.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the measurement window |
| `END_TIME` | TIMESTAMP_LTZ | End of the measurement window |
| `TABLE_ID` | NUMBER | Internal table identifier |
| `TABLE_NAME` | TEXT | Name of the table with search optimization |
| `TABLE_SCHEMA` | TEXT | Schema name |
| `TABLE_CATALOG` | TEXT | Database name |
| `CREDITS_USED` | NUMBER | Credits consumed for search optimization maintenance |
| `NUM_BYTES_UPDATED` | NUMBER | Bytes updated in the search access path |

---

## Comprehensive Cost Dashboard Queries

### 1. All-in Cost by Service Category
```sql
-- Complete cost breakdown by category (current month)
WITH costs AS (
  SELECT 
    CASE service_type
      WHEN 'WAREHOUSE_METERING' THEN '1. Compute (Warehouses)'
      WHEN 'SNOWPIPE' THEN '2. Data Ingestion (Snowpipe)'
      WHEN 'AUTOMATIC_CLUSTERING' THEN '3. Automatic Clustering'
      WHEN 'MATERIALIZED_VIEW' THEN '4. Materialized Views'
      WHEN 'REPLICATION' THEN '5. Replication'
      WHEN 'SERVERLESS_TASK' THEN '6. Serverless Tasks'
      WHEN 'SEARCH_OPTIMIZATION' THEN '7. Search Optimization'
      WHEN 'QUERY_ACCELERATION' THEN '8. Query Acceleration'
      WHEN 'SNOWPARK_CONTAINER_SERVICES' THEN '9. Snowpark Container Services'
      ELSE '10. Other (' || service_type || ')'
    END AS category,
    SUM(credits_billed) AS credits_billed
  FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
  WHERE usage_date >= DATE_TRUNC('month', CURRENT_DATE())
  GROUP BY 1
)
SELECT category,
       credits_billed,
       ROUND(credits_billed * 100 / SUM(credits_billed) OVER (), 2) AS pct_of_total
FROM costs
ORDER BY category;
```

### 2. Month-over-Month Comparison
```sql
-- Compare current vs. prior month spend
SELECT 
  SUM(CASE WHEN usage_date >= DATE_TRUNC('month', CURRENT_DATE())
           THEN credits_billed ELSE 0 END) AS current_month_credits,
  SUM(CASE WHEN usage_date >= DATEADD('month', -1, DATE_TRUNC('month', CURRENT_DATE()))
            AND usage_date < DATE_TRUNC('month', CURRENT_DATE())
           THEN credits_billed ELSE 0 END) AS prior_month_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATEADD('month', -1, DATE_TRUNC('month', CURRENT_DATE()));
```

### 3. Budget Burn Rate Projection
```sql
-- At current run rate, how many credits will be used this month?
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
