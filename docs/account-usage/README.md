# Snowflake Account Usage — Knowledge Base

This knowledge base is the **context layer** for building an organization‑wide
dashboard to monitor Snowflake usage, cost, performance, security, and
governance. It documents the views and tables that Snowflake exposes through the
shared **`SNOWFLAKE`** database — primarily the **`ACCOUNT_USAGE`** schema, plus
the related **`ORGANIZATION_USAGE`** and **`READER_ACCOUNT_USAGE`** schemas.

> **Source & accuracy note.** This content is compiled from Snowflake's public
> *Account Usage* reference (`https://docs.snowflake.com/en/sql-reference/account-usage`)
> and verified against Snowflake product knowledge. The documentation site was
> not directly reachable from the build environment, so always confirm exact
> column names and newly added views against the live docs and against your own
> account using `DESC VIEW SNOWFLAKE.ACCOUNT_USAGE.<view_name>;`. Snowflake adds
> and revises views frequently.

## How to read this knowledge base

| Document | What it covers |
|----------|----------------|
| [concepts.md](concepts.md) | Core concepts: the `SNOWFLAKE` shared database, latency, data retention, `ACCOUNT_USAGE` vs `INFORMATION_SCHEMA` vs `ORGANIZATION_USAGE` vs `READER_ACCOUNT_USAGE`, and access control. |
| [views-catalog.md](views-catalog.md) | The complete, categorized catalog of `ACCOUNT_USAGE` views with one‑line descriptions, typical latency, and retention. Start here to find the right view. |
| [cost-and-usage.md](cost-and-usage.md) | Credit consumption / metering views (warehouses, serverless features, Snowpipe, clustering, etc.) with column detail. |
| [storage.md](storage.md) | Storage consumption views (account, database, stage, table‑level) with column detail. |
| [query-and-performance.md](query-and-performance.md) | `QUERY_HISTORY`, `ACCESS_HISTORY`, `QUERY_ATTRIBUTION_HISTORY`, sessions, warehouse load, etc. |
| [security-and-access.md](security-and-access.md) | Logins, sessions, users, roles, grants, and policy views. |
| [data-loading.md](data-loading.md) | `COPY_HISTORY`, `LOAD_HISTORY`, `PIPE_USAGE_HISTORY`. |
| [governance-and-data-quality.md](governance-and-data-quality.md) | Tags, policies, masking/row access, data metric functions, data quality monitoring. |
| [object-metadata.md](object-metadata.md) | Catalog/metadata views (databases, schemata, tables, columns, etc.). |
| [replication.md](replication.md) | Replication & failover usage views. |
| [organization-usage.md](organization-usage.md) | The `ORGANIZATION_USAGE` schema for multi‑account / org‑level reporting. |
| [reader-account-usage.md](reader-account-usage.md) | The `READER_ACCOUNT_USAGE` schema for monitoring reader (managed) accounts. |
| [dashboard-guide.md](dashboard-guide.md) | Practical mapping of dashboard objectives → views → starter SQL. |

## TL;DR for dashboard builders

- **Cost / spend:** `METERING_DAILY_HISTORY`, `WAREHOUSE_METERING_HISTORY`,
  `METERING_HISTORY`, plus serverless views (`AUTOMATIC_CLUSTERING_HISTORY`,
  `PIPE_USAGE_HISTORY`, `MATERIALIZED_VIEW_REFRESH_HISTORY`,
  `SEARCH_OPTIMIZATION_HISTORY`, `SERVERLESS_TASK_HISTORY`,
  `QUERY_ACCELERATION_HISTORY`, `REPLICATION_USAGE_HISTORY`).
- **Storage:** `STORAGE_USAGE`, `DATABASE_STORAGE_USAGE_HISTORY`,
  `STAGE_STORAGE_USAGE_HISTORY`, `TABLE_STORAGE_METRICS`.
- **Performance / workload:** `QUERY_HISTORY`, `QUERY_ATTRIBUTION_HISTORY`,
  `WAREHOUSE_LOAD_HISTORY`, `WAREHOUSE_EVENTS_HISTORY`.
- **Security:** `LOGIN_HISTORY`, `SESSIONS`, `USERS`, `GRANTS_TO_USERS`,
  `GRANTS_TO_ROLES`, policy views.
- **Governance / lineage:** `ACCESS_HISTORY`, `OBJECT_DEPENDENCIES`,
  `TAG_REFERENCES`, `POLICY_REFERENCES`, `DATA_QUALITY_MONITORING_RESULTS`.

See [dashboard-guide.md](dashboard-guide.md) for ready‑to‑adapt queries.
