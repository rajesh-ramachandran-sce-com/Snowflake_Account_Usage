# Snowflake Account Usage Schema — Overview

## What is ACCOUNT_USAGE?

The `SNOWFLAKE.ACCOUNT_USAGE` schema is a system-provided, read-only schema within the built-in `SNOWFLAKE` database. It contains a comprehensive set of views that expose historical metadata and usage statistics for every aspect of a Snowflake account — including object definitions, query activity, credit consumption, user access, security policies, data loading, and more.

These views are essential for:
- **Cost monitoring and optimization** — track credit usage by warehouse, service, or user
- **Security auditing** — review logins, query access patterns, and privilege grants
- **Governance** — monitor masking policies, row access policies, tags, and data lineage
- **Operational intelligence** — analyze query performance, task execution, and load history
- **Capacity planning** — understand storage growth and warehouse utilization trends

---

## How to Access ACCOUNT_USAGE Views

```sql
-- Full qualified reference
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.<view_name>;

-- Or set your database context first
USE DATABASE SNOWFLAKE;
USE SCHEMA ACCOUNT_USAGE;
SELECT * FROM <view_name>;
```

### Required Privileges

| Role | Access Level |
|------|-------------|
| `ACCOUNTADMIN` | Full access to all ACCOUNT_USAGE views |
| Any role with `IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE` | Access granted by ACCOUNTADMIN |
| Database roles (e.g., `SNOWFLAKE.OBJECT_VIEWER`) | Scoped access to specific view sets |

```sql
-- Grant access to a custom role
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <my_monitor_role>;
```

---

## Key Characteristics

| Property | Value |
|----------|-------|
| **Database** | `SNOWFLAKE` (system database) |
| **Schema** | `ACCOUNT_USAGE` |
| **Data Latency** | Up to 45 minutes to 3 hours (varies by view) |
| **Historical Retention** | Up to 365 days (1 year) for most views |
| **Includes Dropped Objects** | Yes — records for deleted objects are retained |
| **Real-time** | No — use `INFORMATION_SCHEMA` for near-real-time queries |

---

## ACCOUNT_USAGE vs INFORMATION_SCHEMA

| Feature | ACCOUNT_USAGE | INFORMATION_SCHEMA |
|---------|--------------|-------------------|
| **Scope** | Entire account | Current database only |
| **Historical Retention** | Up to 1 year | 7–90 days (varies by view) |
| **Data Latency** | 45 min – 3 hours | Near real-time |
| **Dropped Objects** | Included | Not included |
| **Usage** | Historical analysis, auditing, cost monitoring | Real-time operational queries |

---

## View Categories

The ACCOUNT_USAGE schema organizes views into the following functional categories:

| # | Category | Key Views |
|---|----------|-----------|
| 1 | [Object Metadata](./02_object_metadata_views.md) | DATABASES, SCHEMAS, TABLES, VIEWS, COLUMNS, STAGES, FILE_FORMATS, PIPES, FUNCTIONS, PROCEDURES |
| 2 | [User & Role Management](./03_user_role_views.md) | USERS, ROLES, GRANTS_TO_ROLES, GRANTS_TO_USERS, ROLE_GRANTS |
| 3 | [Query & Session History](./04_query_history_views.md) | QUERY_HISTORY, LOGIN_HISTORY, SESSIONS |
| 4 | [Security & Governance](./05_security_governance_views.md) | ACCESS_HISTORY, MASKING_POLICIES, ROW_ACCESS_POLICIES, POLICY_REFERENCES, TAG_REFERENCES, OBJECT_DEPENDENCIES |
| 5 | [Warehouse & Compute](./06_warehouse_compute_views.md) | WAREHOUSE_METERING_HISTORY, WAREHOUSE_LOAD_HISTORY, WAREHOUSE_EVENTS_HISTORY, RESOURCE_MONITORS |
| 6 | [Storage](./07_storage_views.md) | STORAGE_USAGE, DATABASE_STORAGE_USAGE_HISTORY, TABLE_STORAGE_METRICS, STAGE_STORAGE_USAGE_HISTORY |
| 7 | [Data Loading](./08_data_loading_views.md) | COPY_HISTORY, LOAD_HISTORY |
| 8 | [Pipelines & Tasks](./09_pipelines_tasks_views.md) | PIPE_USAGE_HISTORY, TASK_HISTORY |
| 9 | [Clustering & Materialized Views](./10_clustering_mv_views.md) | AUTOMATIC_CLUSTERING_HISTORY, MATERIALIZED_VIEW_REFRESH_HISTORY |
| 10 | [Replication & Sharing](./11_replication_sharing_views.md) | REPLICATION_GROUP_USAGE_HISTORY, DATABASE_REPLICATION_USAGE_HISTORY, SHARES, DATA_TRANSFER_HISTORY |
| 11 | [Metering & Billing](./12_metering_billing_views.md) | METERING_DAILY_HISTORY, METERING_HISTORY |

---

## Quick Reference: All Views at a Glance

```sql
-- List all views in ACCOUNT_USAGE
SHOW VIEWS IN SCHEMA SNOWFLAKE.ACCOUNT_USAGE;

-- Or query via INFORMATION_SCHEMA
SELECT TABLE_NAME, COMMENT
FROM SNOWFLAKE.INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA = 'ACCOUNT_USAGE'
ORDER BY TABLE_NAME;
```

---

## General Best Practices

1. **Always filter by time** — Most history views contain up to 1 year of data; always add a date predicate to avoid full scans.
   ```sql
   WHERE query_start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
   ```

2. **Use UTC consistently** — Set your session timezone to UTC to avoid confusion across schemas.
   ```sql
   ALTER SESSION SET TIMEZONE = 'UTC';
   ```

3. **Understand latency** — ACCOUNT_USAGE views are not real-time. Do not use them for operational alerting; use `INFORMATION_SCHEMA` or Snowflake notifications for that.

4. **Cache results** — Because of latency, consider creating a daily snapshot table from ACCOUNT_USAGE views for dashboard consumption.

5. **Prefer role-based access** — Grant a dedicated monitoring role `IMPORTED PRIVILEGES` on the SNOWFLAKE database rather than giving everyone ACCOUNTADMIN.

---

## Documentation Reference

- Official Documentation: https://docs.snowflake.com/en/sql-reference/account-usage
- Query Performance Guide: https://docs.snowflake.com/en/user-guide/performance-query-warehouse
- Cost Management: https://docs.snowflake.com/en/user-guide/cost-understanding-overall
