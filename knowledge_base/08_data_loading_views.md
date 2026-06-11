# Data Loading Views

These views track all data ingestion activities in your Snowflake account — whether via manual `COPY INTO` commands, Snowpipe continuous loading, or Snowflake's external integrations. They are critical for **monitoring data pipeline health**, **auditing load history**, and **debugging ingestion failures**.

---

## COPY_HISTORY

**Purpose:** Records the full history of all file-based data loads, including both manual `COPY INTO <table>` statements and **Snowpipe** (auto-ingest) loads. This view covers the entire account and retains up to 365 days of history.

**Data Latency:** 2 hours  
**Retention:** 1 year

> **Key difference from LOAD_HISTORY:** `COPY_HISTORY` includes Snowpipe loads; `LOAD_HISTORY` does not.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `FILE_NAME` | TEXT | Name and relative path of the source file in the stage |
| `STAGE_LOCATION` | TEXT | Name or path of the stage containing the source file |
| `LAST_LOAD_TIME` | TIMESTAMP_LTZ | Timestamp when the file completed loading |
| `ROW_COUNT` | NUMBER | Number of rows successfully loaded |
| `ROW_PARSED` | NUMBER | Number of rows parsed from the source |
| `FILE_SIZE` | NUMBER | Size of the source file in bytes |
| `FIRST_ERROR_MESSAGE` | TEXT | First error encountered during loading |
| `FIRST_ERROR_LINE_NUMBER` | NUMBER | Line number in the file where the first error occurred |
| `FIRST_ERROR_CHARACTER_POS` | NUMBER | Character position of the first error |
| `FIRST_ERROR_COLUMN_NAME` | TEXT | Column name associated with the first error |
| `ERROR_COUNT` | NUMBER | Total number of rows with errors |
| `ERROR_LIMIT` | NUMBER | Maximum allowed errors before aborting |
| `STATUS` | TEXT | Load result: `Loaded`, `Load failed`, `Partially loaded`, `Load skipped` |
| `TABLE_ID` | NUMBER | Internal identifier of the target table |
| `TABLE_NAME` | TEXT | Name of the target table |
| `TABLE_SCHEMA_ID` | NUMBER | Schema ID of the target table |
| `TABLE_SCHEMA_NAME` | TEXT | Schema name |
| `TABLE_CATALOG_ID` | NUMBER | Database ID |
| `TABLE_CATALOG_NAME` | TEXT | Database name |
| `PIPE_CATALOG_NAME` | TEXT | Database of the associated Snowpipe (NULL for COPY INTO) |
| `PIPE_SCHEMA_NAME` | TEXT | Schema of the Snowpipe (NULL for COPY INTO) |
| `PIPE_NAME` | TEXT | Name of the Snowpipe (NULL for COPY INTO) |
| `PIPE_RECEIVED_TIME` | TIMESTAMP_LTZ | When Snowpipe received the load request (NULL for COPY INTO) |
| `FIRST_COMMIT_TIME` | TIMESTAMP_LTZ | Timestamp of the first commit in chunked loads |

**Sample Queries:**
```sql
-- View recent load history with status
SELECT last_load_time, table_catalog_name, table_schema_name, table_name,
       file_name, row_count, status, first_error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE last_load_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY last_load_time DESC
LIMIT 50;

-- Find failed or partial loads
SELECT last_load_time, table_name, file_name,
       status, error_count, first_error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE last_load_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND status IN ('Load failed', 'Partially loaded')
ORDER BY last_load_time DESC;

-- Daily load volume by target table
SELECT DATE(last_load_time) AS load_date,
       table_catalog_name, table_schema_name, table_name,
       COUNT(*) AS files_loaded,
       SUM(row_count) AS total_rows,
       SUM(file_size) / 1024 / 1024 AS total_mb
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE last_load_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND status = 'Loaded'
GROUP BY 1, 2, 3, 4
ORDER BY 1 DESC, total_rows DESC;

-- Snowpipe vs. manual COPY loads breakdown
SELECT CASE WHEN pipe_name IS NULL THEN 'Manual COPY' ELSE 'Snowpipe' END AS load_method,
       COUNT(*) AS file_count,
       SUM(row_count) AS total_rows,
       COUNT(CASE WHEN status != 'Loaded' THEN 1 END) AS failed_files
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE last_load_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1;
```

---

## LOAD_HISTORY

**Purpose:** Records the history of data loads performed via **manual `COPY INTO <table>` statements only** (excludes Snowpipe). This view provides similar information to `COPY_HISTORY` but is scoped to COPY commands.

**Data Latency:** 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `SCHEMA_NAME` | TEXT | Schema of the target table |
| `FILE_NAME` | TEXT | Name of the source file |
| `TABLE_ID` | NUMBER | Internal table identifier |
| `TABLE_NAME` | TEXT | Target table name |
| `LAST_LOAD_TIME` | TIMESTAMP_LTZ | When the load completed |
| `STATUS` | TEXT | Load status: `Loaded`, `Load failed`, `Partially loaded`, `Load skipped` |
| `ROW_COUNT` | NUMBER | Rows successfully loaded |
| `ROW_PARSED` | NUMBER | Rows parsed |
| `FIRST_ERROR_MESSAGE` | TEXT | First error message if any |
| `FIRST_ERROR_LINE_NUMBER` | NUMBER | Line number of the first error |
| `FIRST_ERROR_CHARACTER_POS` | NUMBER | Character position of the first error |
| `FIRST_ERROR_COLUMN_NAME` | TEXT | Column name of the first error |
| `ERROR_COUNT` | NUMBER | Number of erroneous rows |
| `ERROR_LIMIT` | NUMBER | Error limit configured for the load |

**Note:** The `LOAD_HISTORY` view does **not** include Snowpipe loads. For a unified view of all loads, use `COPY_HISTORY`.

---

## Comparison: COPY_HISTORY vs LOAD_HISTORY vs INFORMATION_SCHEMA.LOAD_HISTORY

| Feature | COPY_HISTORY | LOAD_HISTORY | INFORMATION_SCHEMA.LOAD_HISTORY |
|---------|-------------|-------------|--------------------------------|
| **Scope** | Account-wide | Account-wide | Current database only |
| **Snowpipe included** | ✅ Yes | ❌ No | ❌ No |
| **Retention** | 365 days | 365 days | 14 days |
| **Row limit** | No limit | No limit | 10,000 rows per query |
| **Latency** | ~2 hours | ~2 hours | Near real-time |
| **Best for** | Full audit of all loads | Manual COPY audit | Real-time operational checks |

---

## Pipeline Health Dashboard Queries

### 1. Data Freshness Check
```sql
-- Latest load time per table (detect stale pipelines)
SELECT table_catalog_name, table_schema_name, table_name,
       MAX(last_load_time) AS last_load_time,
       DATEDIFF('hour', MAX(last_load_time), CURRENT_TIMESTAMP()) AS hours_since_last_load
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
GROUP BY 1, 2, 3
HAVING hours_since_last_load > 24
ORDER BY hours_since_last_load DESC;
```

### 2. Error Rate Monitoring
```sql
-- Load error rate by day
SELECT DATE(last_load_time) AS load_date,
       COUNT(*) AS total_loads,
       COUNT(CASE WHEN status = 'Loaded' THEN 1 END) AS successful,
       COUNT(CASE WHEN status != 'Loaded' THEN 1 END) AS failed,
       ROUND(COUNT(CASE WHEN status != 'Loaded' THEN 1 END) * 100.0 / COUNT(*), 2) AS error_rate_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE last_load_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY 1 DESC;
```

### 3. Top Files with Errors
```sql
-- Most problematic files (most error rows)
SELECT file_name, table_name, error_count,
       first_error_message, last_load_time
FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
WHERE last_load_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND error_count > 0
ORDER BY error_count DESC
LIMIT 20;
```
