# Clustering & Materialized View Refresh History

These views track two important Snowflake automated maintenance operations: **Automatic Clustering** (background re-clustering of tables for query performance) and **Materialized View Refreshes** (keeping materialized view data in sync with underlying tables). Both consume credits and are tracked via these views.

---

## AUTOMATIC_CLUSTERING_HISTORY

**Purpose:** Records all automatic clustering operations performed by Snowflake's background service. Automatic clustering continuously re-organizes table micro-partitions based on the defined clustering key to maintain query performance. Each row represents one clustering operation.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the clustering operation |
| `END_TIME` | TIMESTAMP_LTZ | End of the clustering operation |
| `TABLE_ID` | NUMBER | Internal table identifier |
| `TABLE_NAME` | TEXT | Name of the table being clustered |
| `SCHEMA_NAME` | TEXT | Schema of the table |
| `DATABASE_NAME` | TEXT | Database of the table |
| `CREDITS_USED` | NUMBER | Credits consumed by this clustering operation |
| `NUM_BYTES_RECLUSTERED` | NUMBER | Bytes re-organized during this operation |
| `NUM_ROWS_RECLUSTERED` | NUMBER | Rows re-organized during this operation |
| `PARTITION_BEFORE` | NUMBER | Number of micro-partitions before clustering |
| `PARTITION_AFTER` | NUMBER | Number of micro-partitions after clustering |

**Sample Queries:**
```sql
-- Daily automatic clustering credit cost by table (last 30 days)
SELECT DATE(start_time) AS cluster_date,
       database_name, schema_name, table_name,
       SUM(credits_used) AS credits_used,
       SUM(num_bytes_reclustered) / 1024 / 1024 / 1024 AS gb_reclustered
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3, 4
ORDER BY 1 DESC, 5 DESC;

-- Top tables by clustering cost this month
SELECT database_name, schema_name, table_name,
       SUM(credits_used) AS total_credits,
       SUM(num_rows_reclustered) AS total_rows_reclustered,
       COUNT(*) AS clustering_operations
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3
ORDER BY total_credits DESC
LIMIT 10;

-- Evaluate clustering effectiveness (partition reduction rate)
SELECT database_name, schema_name, table_name,
       AVG(partition_before) AS avg_partitions_before,
       AVG(partition_after) AS avg_partitions_after,
       ROUND(1 - AVG(partition_after) / NULLIF(AVG(partition_before), 0), 4) AS avg_reduction_rate
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3
ORDER BY avg_reduction_rate DESC;

-- Clustering cost trend over time
SELECT DATE_TRUNC('week', start_time) AS week,
       SUM(credits_used) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE start_time >= DATEADD('day', -90, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY 1 DESC;
```

---

## MATERIALIZED_VIEW_REFRESH_HISTORY

**Purpose:** Records all automatic refresh operations for materialized views in your account. Snowflake automatically keeps materialized views in sync with their base tables; this view tracks when those refreshes happened, how long they took, and how much they cost.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | When the refresh operation started |
| `END_TIME` | TIMESTAMP_LTZ | When the refresh operation completed |
| `MATERIALIZED_VIEW_ID` | NUMBER | Internal identifier of the materialized view |
| `MATERIALIZED_VIEW_NAME` | TEXT | Name of the materialized view |
| `MATERIALIZED_VIEW_SCHEMA_ID` | NUMBER | Schema ID |
| `MATERIALIZED_VIEW_SCHEMA` | TEXT | Schema name |
| `MATERIALIZED_VIEW_DATABASE_ID` | NUMBER | Database ID |
| `MATERIALIZED_VIEW_DATABASE` | TEXT | Database name |
| `TABLE_ID` | NUMBER | Internal ID of the base table |
| `TABLE_NAME` | TEXT | Name of the base table |
| `TABLE_SCHEMA` | TEXT | Schema of the base table |
| `TABLE_DATABASE` | TEXT | Database of the base table |
| `CREDITS_USED` | NUMBER | Credits consumed by this refresh |
| `BYTES_INSERTED` | NUMBER | Bytes inserted into the materialized view during refresh |
| `ROWS_INSERTED` | NUMBER | Rows inserted |
| `BYTES_DELETED` | NUMBER | Bytes deleted from the materialized view |
| `ROWS_DELETED` | NUMBER | Rows deleted |

**Sample Queries:**
```sql
-- Materialized view refresh cost summary (last 30 days)
SELECT materialized_view_database, materialized_view_schema,
       materialized_view_name, table_name,
       COUNT(*) AS refresh_count,
       SUM(credits_used) AS total_credits,
       AVG(credits_used) AS avg_credits_per_refresh,
       AVG(DATEDIFF('second', start_time, end_time)) AS avg_refresh_duration_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3, 4
ORDER BY total_credits DESC;

-- Most frequently refreshed materialized views
SELECT materialized_view_name,
       COUNT(*) AS refresh_count,
       SUM(credits_used) AS total_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY refresh_count DESC
LIMIT 20;

-- Detect high-cost refreshes
SELECT start_time, materialized_view_name,
       credits_used, rows_inserted, rows_deleted,
       DATEDIFF('second', start_time, end_time) AS duration_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND credits_used > 0.1  -- adjust threshold
ORDER BY credits_used DESC;
```

---

## Combined Automated Maintenance Cost View

Understand the total cost of Snowflake-managed background operations:

```sql
-- Total automated maintenance cost breakdown this month
SELECT 'Automatic Clustering' AS service,
       SUM(credits_used) AS credits_used_this_month
FROM SNOWFLAKE.ACCOUNT_USAGE.AUTOMATIC_CLUSTERING_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP())

UNION ALL

SELECT 'Materialized View Refresh' AS service,
       SUM(credits_used) AS credits_used_this_month
FROM SNOWFLAKE.ACCOUNT_USAGE.MATERIALIZED_VIEW_REFRESH_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP())

ORDER BY credits_used_this_month DESC;
```

---

## Optimization Guidance

### Automatic Clustering
- **High clustering cost** on a table may indicate a poor clustering key choice or high data ingestion volatility.
- Consider using **CLUSTER BY** on the most frequently filtered column(s) in queries.
- If clustering cost exceeds query performance gains, consider **suspending automatic clustering** for that table:
  ```sql
  ALTER TABLE my_table SUSPEND RECLUSTER;
  ```

### Materialized Views
- **High refresh frequency** with small data changes wastes credits; evaluate if the base table changes are frequent enough to justify materialized views.
- For large base tables with infrequent full refreshes, consider **dynamic tables** (a newer alternative) which only process changed data incrementally.
- Monitor refresh duration — if a single refresh consistently takes a long time, the materialized view definition may need optimization.
