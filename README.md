# Snowflake Account Usage — Knowledge Base & Dashboard Reference

This repository contains a comprehensive knowledge base covering every **Snowflake Account Usage** view (`SNOWFLAKE.ACCOUNT_USAGE` schema) — including column definitions, data latency, retention periods, and ready-to-use SQL queries for monitoring, cost analysis, security auditing, and governance.

## Purpose

This knowledge base is the foundation for building a comprehensive **Snowflake Account Usage Monitoring Dashboard**. It documents all views available in the `SNOWFLAKE.ACCOUNT_USAGE` schema so that analysts and administrators have a single reference for:

- Understanding what data each view provides
- Knowing the latency and retention characteristics of each view
- Finding production-ready SQL queries for common monitoring scenarios

## Knowledge Base Structure

All documentation is in the [`knowledge_base/`](./knowledge_base/) folder:

| File | Contents |
|------|----------|
| [01_overview.md](./knowledge_base/01_overview.md) | Schema overview, access setup, latency, best practices |
| [02_object_metadata_views.md](./knowledge_base/02_object_metadata_views.md) | DATABASES, SCHEMAS, TABLES, VIEWS, COLUMNS, STAGES, WAREHOUSES, TASKS, PIPES, FUNCTIONS |
| [03_user_role_views.md](./knowledge_base/03_user_role_views.md) | USERS, ROLES, GRANTS_TO_ROLES, GRANTS_TO_USERS, ROLE_GRANTS |
| [04_query_history_views.md](./knowledge_base/04_query_history_views.md) | QUERY_HISTORY, LOGIN_HISTORY, SESSIONS |
| [05_security_governance_views.md](./knowledge_base/05_security_governance_views.md) | ACCESS_HISTORY, MASKING_POLICIES, ROW_ACCESS_POLICIES, POLICY_REFERENCES, TAG_REFERENCES, OBJECT_DEPENDENCIES |
| [06_warehouse_compute_views.md](./knowledge_base/06_warehouse_compute_views.md) | WAREHOUSE_METERING_HISTORY, WAREHOUSE_LOAD_HISTORY, WAREHOUSE_EVENTS_HISTORY, RESOURCE_MONITORS |
| [07_storage_views.md](./knowledge_base/07_storage_views.md) | STORAGE_USAGE, DATABASE_STORAGE_USAGE_HISTORY, TABLE_STORAGE_METRICS, STAGE_STORAGE_USAGE_HISTORY |
| [08_data_loading_views.md](./knowledge_base/08_data_loading_views.md) | COPY_HISTORY, LOAD_HISTORY |
| [09_pipelines_tasks_views.md](./knowledge_base/09_pipelines_tasks_views.md) | PIPE_USAGE_HISTORY, TASK_HISTORY |
| [10_clustering_mv_views.md](./knowledge_base/10_clustering_mv_views.md) | AUTOMATIC_CLUSTERING_HISTORY, MATERIALIZED_VIEW_REFRESH_HISTORY |
| [11_replication_sharing_views.md](./knowledge_base/11_replication_sharing_views.md) | REPLICATION_GROUP_USAGE_HISTORY, DATABASE_REPLICATION_USAGE_HISTORY, SHARES, DATA_TRANSFER_HISTORY |
| [12_metering_billing_views.md](./knowledge_base/12_metering_billing_views.md) | METERING_HISTORY, METERING_DAILY_HISTORY, QUERY_ACCELERATION_HISTORY, SEARCH_OPTIMIZATION_HISTORY |
| [13_sample_queries.md](./knowledge_base/13_sample_queries.md) | 30+ ready-to-use queries for dashboards across all categories |

## Quick Start

### Access the ACCOUNT_USAGE Schema
```sql
-- Option 1: Fully qualified
SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -1, CURRENT_TIMESTAMP())
LIMIT 10;

-- Option 2: Grant access to a custom role (run as ACCOUNTADMIN)
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <my_monitor_role>;
```

### Top Dashboard Queries

**Current month credit breakdown:**
```sql
SELECT service_type, SUM(credits_billed) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE usage_date >= DATE_TRUNC('month', CURRENT_DATE())
GROUP BY service_type ORDER BY credits DESC;
```

**Slowest queries today:**
```sql
SELECT user_name, warehouse_name,
       ROUND(total_elapsed_time/1000, 2) AS elapsed_sec,
       LEFT(query_text, 100) AS query_preview
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= CURRENT_DATE()
ORDER BY total_elapsed_time DESC LIMIT 10;
```

**Failed logins in the last hour:**
```sql
SELECT event_timestamp, user_name, client_ip, error_message
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE event_timestamp >= DATEADD('hour', -1, CURRENT_TIMESTAMP())
  AND is_success = 'NO';
```

## Key Facts

| Property | Value |
|----------|-------|
| Schema location | `SNOWFLAKE.ACCOUNT_USAGE` |
| Data latency | 45 minutes – 3 hours (varies by view) |
| Historical retention | Up to 1 year (365 days) |
| Required role | `ACCOUNTADMIN` or role with `IMPORTED PRIVILEGES` on the SNOWFLAKE database |
| Dropped objects included | Yes |

## Reference

- Official Documentation: https://docs.snowflake.com/en/sql-reference/account-usage