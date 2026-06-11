# Pipelines & Tasks Views

These views track the execution and credit consumption of Snowflake's data pipeline components: **Snowpipe** (continuous data ingestion) and **Tasks** (scheduled SQL execution). They are critical for monitoring **pipeline throughput**, **cost attribution**, and **failure diagnosis**.

---

## PIPE_USAGE_HISTORY

**Purpose:** Records hourly credit and file usage statistics for every Snowpipe in your account. Snowpipes use a serverless compute model, and this view lets you track their cost and throughput.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the one-hour measurement window |
| `END_TIME` | TIMESTAMP_LTZ | End of the one-hour measurement window |
| `PIPE_ID` | NUMBER | Internal Snowpipe identifier |
| `PIPE_NAME` | TEXT | Name of the Snowpipe |
| `PIPE_SCHEMA` | TEXT | Schema where the pipe is defined |
| `PIPE_CATALOG` | TEXT | Database where the pipe is defined |
| `CREDITS_USED` | NUMBER | Serverless compute credits consumed |
| `BYTES_INSERTED` | NUMBER | Total bytes inserted into target tables |
| `FILES_INSERTED` | NUMBER | Number of files loaded |

**Sample Queries:**
```sql
-- Daily credit usage by Snowpipe (last 30 days)
SELECT DATE(start_time) AS usage_date,
       pipe_name,
       SUM(credits_used) AS daily_credits,
       SUM(files_inserted) AS files_loaded,
       SUM(bytes_inserted) / 1024 / 1024 AS mb_inserted
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;

-- Top 10 most expensive pipes this month
SELECT pipe_name, pipe_catalog, pipe_schema,
       SUM(credits_used) AS total_credits,
       SUM(files_inserted) AS total_files,
       SUM(bytes_inserted) / 1024 / 1024 / 1024 AS total_gb_inserted
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3
ORDER BY total_credits DESC
LIMIT 10;

-- Snowpipe throughput trend
SELECT DATE_TRUNC('day', start_time) AS day,
       SUM(files_inserted) AS files,
       SUM(bytes_inserted) / 1024 / 1024 / 1024 AS gb_inserted
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY 1 DESC;
```

---

## TASK_HISTORY

**Purpose:** Records the execution history of every **Snowflake Task** run — including the SQL statement executed, execution state, duration, errors, and scheduling metadata. Tasks are used to schedule SQL procedures and queries on a recurring basis.

**Data Latency:** 45 minutes  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `QUERY_ID` | TEXT | ID of the query executed by this task run |
| `NAME` | TEXT | Name of the task |
| `DATABASE_NAME` | TEXT | Database where the task is defined |
| `SCHEMA_NAME` | TEXT | Schema where the task is defined |
| `QUERY_TEXT` | TEXT | SQL statement executed by the task |
| `CONDITION_TEXT` | TEXT | WHEN condition evaluated before execution |
| `STATE` | TEXT | Execution state: SUCCEEDED, FAILED, SKIPPED, CANCELLED |
| `ERROR_CODE` | TEXT | Error code if the task failed |
| `ERROR_MESSAGE` | TEXT | Error message if the task failed |
| `SCHEDULED_TIME` | TIMESTAMP_LTZ | When the task was scheduled to run |
| `QUERY_START_TIME` | TIMESTAMP_LTZ | When the task actually started executing |
| `COMPLETED_TIME` | TIMESTAMP_LTZ | When the task finished |
| `RETURN_VALUE` | TEXT | Return value from the task (for stored procedures) |
| `GRAPH_VERSION` | NUMBER | Version of the task graph (DAG) |
| `RUN_ID` | NUMBER | Unique identifier for the task run |
| `ROOT_TASK_ID` | TEXT | ID of the root task in a DAG |
| `SCHEDULED_FROM` | TEXT | What triggered the schedule: SCHEDULE or TRIGGER |
| `ATTEMPT_NUMBER` | NUMBER | Retry attempt number (1 for first attempt) |
| `INSTANCE_ID` | NUMBER | Specific instance of the task run |
| `CONFIG` | VARIANT | Task configuration parameters |

**Sample Queries:**
```sql
-- Recent task execution summary
SELECT name, database_name, schema_name, state,
       scheduled_time, query_start_time, completed_time,
       DATEDIFF('second', query_start_time, completed_time) AS duration_seconds
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE scheduled_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY scheduled_time DESC
LIMIT 50;

-- Find all failed tasks in the last 24 hours
SELECT name, database_name, schema_name,
       scheduled_time, error_code, error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE scheduled_time >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
  AND state = 'FAILED'
ORDER BY scheduled_time DESC;

-- Task success rate by task name (last 30 days)
SELECT name, database_name, schema_name,
       COUNT(*) AS total_runs,
       COUNT(CASE WHEN state = 'SUCCEEDED' THEN 1 END) AS succeeded,
       COUNT(CASE WHEN state = 'FAILED' THEN 1 END) AS failed,
       COUNT(CASE WHEN state = 'SKIPPED' THEN 1 END) AS skipped,
       ROUND(COUNT(CASE WHEN state = 'SUCCEEDED' THEN 1 END) * 100.0 / COUNT(*), 2) AS success_rate_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE scheduled_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3
ORDER BY failed DESC;

-- Average task duration trend over time
SELECT DATE(scheduled_time) AS run_date,
       name,
       AVG(DATEDIFF('second', query_start_time, completed_time)) AS avg_duration_sec,
       MAX(DATEDIFF('second', query_start_time, completed_time)) AS max_duration_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE scheduled_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND state = 'SUCCEEDED'
  AND query_start_time IS NOT NULL
  AND completed_time IS NOT NULL
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;

-- Tasks that ran longer than expected (SLA breach detection)
SELECT name, database_name, schema_name,
       scheduled_time, completed_time,
       DATEDIFF('minute', query_start_time, completed_time) AS duration_minutes
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE scheduled_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND state = 'SUCCEEDED'
  AND DATEDIFF('minute', query_start_time, completed_time) > 60  -- adjust threshold
ORDER BY duration_minutes DESC;
```

---

## Monitoring Pipelines End-to-End

Combine `PIPE_USAGE_HISTORY` and `COPY_HISTORY` for a complete pipeline view:

```sql
-- Pipeline health summary: pipe name, recent activity, and status
SELECT 
    p.pipe_name,
    MAX(c.last_load_time) AS last_successful_load,
    SUM(c.row_count) AS rows_loaded_30d,
    SUM(pu.credits_used) AS credits_used_30d,
    COUNT(CASE WHEN c.status != 'Loaded' THEN 1 END) AS failed_loads_30d
FROM SNOWFLAKE.ACCOUNT_USAGE.PIPES p
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY c
    ON c.pipe_name = p.pipe_name
    AND c.last_load_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.PIPE_USAGE_HISTORY pu
    ON pu.pipe_name = p.pipe_name
    AND pu.start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
WHERE p.deleted IS NULL
GROUP BY p.pipe_name
ORDER BY last_successful_load DESC NULLS LAST;
```

---

## Task DAG (Directed Acyclic Graph) Monitoring

Tasks in Snowflake can be organized into DAGs where one task triggers others. Monitor the entire graph:

```sql
-- All runs for a specific DAG root task in the last 7 days
SELECT name, state, scheduled_time, completed_time, attempt_number,
       DATEDIFF('second', query_start_time, completed_time) AS duration_sec
FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
WHERE root_task_id = (
    SELECT QUERY_ID  -- Use the root task's run_id or name
    FROM SNOWFLAKE.ACCOUNT_USAGE.TASK_HISTORY
    WHERE name = 'MY_ROOT_TASK'
    ORDER BY scheduled_time DESC
    LIMIT 1
)
  AND scheduled_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY scheduled_time DESC, name;
```
