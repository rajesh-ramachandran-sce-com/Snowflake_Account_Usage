# Storage Views

These views track data storage consumption across your Snowflake account — from account-level summaries down to individual table metrics. They are essential for **storage cost management**, **Time Travel and Fail-safe optimization**, and **capacity planning**.

---

## STORAGE_USAGE

**Purpose:** Provides a daily snapshot of average data storage usage (in bytes) across the entire Snowflake account for the past 365 days. Covers all storage components.

**Data Latency:** 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `USAGE_DATE` | DATE | Date of the storage measurement |
| `STORAGE_BYTES` | NUMBER | Bytes used by database table storage (including Time Travel) |
| `STAGE_BYTES` | NUMBER | Bytes used by all internal stages (named, table, and user stages) |
| `FAILSAFE_BYTES` | NUMBER | Bytes in Fail-safe storage (7-day protection beyond Time Travel) |

**Sample Queries:**
```sql
-- Total account storage trend (last 30 days)
SELECT usage_date,
       ROUND(storage_bytes / 1024 / 1024 / 1024, 2) AS storage_gb,
       ROUND(stage_bytes / 1024 / 1024 / 1024, 2) AS stage_gb,
       ROUND(failsafe_bytes / 1024 / 1024 / 1024, 2) AS failsafe_gb,
       ROUND((storage_bytes + stage_bytes + failsafe_bytes) / 1024 / 1024 / 1024, 2) AS total_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
WHERE usage_date >= DATEADD('day', -30, CURRENT_DATE())
ORDER BY usage_date DESC;

-- Month-over-month storage growth
SELECT DATE_TRUNC('month', usage_date) AS month,
       AVG(storage_bytes + stage_bytes + failsafe_bytes) / 1024 / 1024 / 1024 AS avg_total_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
GROUP BY 1
ORDER BY 1 DESC;
```

---

## DATABASE_STORAGE_USAGE_HISTORY

**Purpose:** Provides daily storage usage broken down by individual database. Allows you to track which databases are consuming the most storage.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `USAGE_DATE` | DATE | Date of the measurement |
| `DATABASE_ID` | NUMBER | Internal database identifier |
| `DATABASE_NAME` | TEXT | Name of the database |
| `DELETED` | TIMESTAMP_LTZ | When the database was dropped (NULL if active) |
| `AVERAGE_DATABASE_BYTES` | NUMBER | Average daily bytes used by the database (including Time Travel) |
| `AVERAGE_FAILSAFE_BYTES` | NUMBER | Average daily bytes in Fail-safe for this database |

**Sample Queries:**
```sql
-- Current storage by database (latest available date)
SELECT database_name,
       ROUND(average_database_bytes / 1024 / 1024 / 1024, 2) AS storage_gb,
       ROUND(average_failsafe_bytes / 1024 / 1024 / 1024, 2) AS failsafe_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE_HISTORY
WHERE usage_date = (
    SELECT MAX(usage_date)
    FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE_HISTORY
)
  AND deleted IS NULL
ORDER BY average_database_bytes DESC;

-- Storage trend per database (last 90 days)
SELECT usage_date, database_name,
       ROUND(average_database_bytes / 1024 / 1024 / 1024, 2) AS storage_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_STORAGE_USAGE_HISTORY
WHERE usage_date >= DATEADD('day', -90, CURRENT_DATE())
  AND deleted IS NULL
ORDER BY usage_date DESC, storage_gb DESC;
```

---

## TABLE_STORAGE_METRICS

**Purpose:** Provides current storage metrics for every table in your account, including active data, Time Travel, Fail-safe, and clone data. This view is unique because it reflects the **current state**, not historical snapshots.

**Data Latency:** Near real-time (few minutes)  
**Retention:** Current state (not historical — for history, use DATABASE_STORAGE_USAGE_HISTORY)

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `ID` | NUMBER | Internal table identifier |
| `TABLE_NAME` | TEXT | Name of the table |
| `TABLE_SCHEMA_ID` | NUMBER | Schema ID |
| `TABLE_SCHEMA` | TEXT | Schema name |
| `TABLE_CATALOG_ID` | NUMBER | Database ID |
| `TABLE_CATALOG` | TEXT | Database name |
| `CLONE_GROUP_ID` | NUMBER | ID of the clone group (if this is a clone) |
| `IS_TRANSIENT` | BOOLEAN | Whether the table is transient |
| `ACTIVE_BYTES` | NUMBER | Bytes in the active (current) data |
| `TIME_TRAVEL_BYTES` | NUMBER | Bytes used by Time Travel data |
| `FAILSAFE_BYTES` | NUMBER | Bytes used by Fail-safe |
| `RETAINED_FOR_CLONE_BYTES` | NUMBER | Bytes retained for clone references |
| `ROW_COUNT` | NUMBER | Current number of rows |
| `CREATED` | TIMESTAMP_LTZ | Table creation timestamp |
| `LAST_ALTERED` | TIMESTAMP_LTZ | Last table alteration timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp (NULL if active) |
| `TABLE_ENTERED_FAILSAFE` | TIMESTAMP_LTZ | When data entered Fail-safe |

**Sample Queries:**
```sql
-- Find tables with the most Time Travel storage overhead
SELECT table_catalog, table_schema, table_name,
       ROUND(active_bytes / 1024 / 1024 / 1024, 2) AS active_gb,
       ROUND(time_travel_bytes / 1024 / 1024 / 1024, 2) AS time_travel_gb,
       ROUND(failsafe_bytes / 1024 / 1024 / 1024, 2) AS failsafe_gb,
       ROUND(time_travel_bytes / NULLIF(active_bytes, 0) * 100, 1) AS tt_overhead_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE deleted IS NULL
  AND active_bytes > 0
ORDER BY time_travel_bytes DESC
LIMIT 20;

-- Account-wide storage breakdown
SELECT
  SUM(active_bytes) / 1024 / 1024 / 1024 AS total_active_gb,
  SUM(time_travel_bytes) / 1024 / 1024 / 1024 AS total_time_travel_gb,
  SUM(failsafe_bytes) / 1024 / 1024 / 1024 AS total_failsafe_gb,
  SUM(retained_for_clone_bytes) / 1024 / 1024 / 1024 AS total_clone_retained_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE deleted IS NULL;

-- Tables with large Fail-safe footprint (candidates for transient tables)
SELECT table_catalog, table_schema, table_name,
       ROUND(failsafe_bytes / 1024 / 1024 / 1024, 2) AS failsafe_gb,
       is_transient, created
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE failsafe_bytes > 1073741824  -- > 1 GB
  AND deleted IS NULL
ORDER BY failsafe_bytes DESC;
```

---

## STAGE_STORAGE_USAGE_HISTORY

**Purpose:** Tracks daily average storage used by all internal stages (named, table, and user stages) over time. Useful for monitoring file accumulation and optimizing staging storage costs.

**Data Latency:** 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `USAGE_DATE` | DATE | Date of the measurement |
| `AVERAGE_STAGE_BYTES` | NUMBER | Average daily bytes used across all internal stages |

**Sample Query:**
```sql
-- Stage storage trend over the last 30 days
SELECT usage_date,
       ROUND(average_stage_bytes / 1024 / 1024 / 1024, 2) AS stage_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.STAGE_STORAGE_USAGE_HISTORY
WHERE usage_date >= DATEADD('day', -30, CURRENT_DATE())
ORDER BY usage_date DESC;
```

---

## Storage Cost Optimization Strategies

### 1. Identify Tables to Convert to Transient
Transient tables have no Fail-safe period (0 days) vs. permanent tables (7 days default). Converting frequently-truncated staging tables can significantly reduce storage costs.

```sql
-- Staging/temp tables with large Fail-safe footprint — consider making transient
SELECT table_catalog, table_schema, table_name,
       ROUND(failsafe_bytes / 1024 / 1024 / 1024, 2) AS failsafe_gb,
       ROUND(active_bytes / 1024 / 1024 / 1024, 2) AS active_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE deleted IS NULL
  AND is_transient = FALSE
  AND failsafe_bytes > 536870912  -- > 512 MB in Fail-safe
  AND LOWER(table_name) RLIKE '(stg|staging|tmp|temp|load).*'
ORDER BY failsafe_bytes DESC;
```

### 2. Reduce Time Travel Retention for Old Tables
```sql
-- Tables with high Time Travel that haven't been modified recently
SELECT table_catalog, table_schema, table_name,
       ROUND(time_travel_bytes / 1024 / 1024 / 1024, 2) AS time_travel_gb,
       last_altered
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE deleted IS NULL
  AND time_travel_bytes > 1073741824  -- > 1 GB
  AND last_altered < DATEADD('day', -90, CURRENT_TIMESTAMP())
ORDER BY time_travel_bytes DESC;
```

### 3. Storage Growth Forecast
```sql
-- Monthly storage growth rate
WITH monthly AS (
  SELECT DATE_TRUNC('month', usage_date) AS month,
         AVG(storage_bytes + stage_bytes + failsafe_bytes) AS avg_bytes
  FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
  GROUP BY 1
)
SELECT month,
       ROUND(avg_bytes / 1024 / 1024 / 1024, 2) AS avg_total_gb,
       ROUND((avg_bytes - LAG(avg_bytes) OVER (ORDER BY month)) /
             NULLIF(LAG(avg_bytes) OVER (ORDER BY month), 0) * 100, 2) AS mom_growth_pct
FROM monthly
ORDER BY month DESC;
```
