# Snowflake_Account_Usage

Context and knowledge base for building a comprehensive dashboard to monitor an
organization's Snowflake usage, cost, performance, security, and governance.

It documents the views Snowflake exposes through the shared **`SNOWFLAKE`**
database — primarily the **`ACCOUNT_USAGE`** schema, plus **`ORGANIZATION_USAGE`**
and **`READER_ACCOUNT_USAGE`** — based on Snowflake's
[Account Usage reference](https://docs.snowflake.com/en/sql-reference/account-usage).

## 📚 Knowledge base

Start here: **[docs/account-usage/README.md](docs/account-usage/README.md)**

| Topic | Document |
|-------|----------|
| Core concepts (latency, retention, schema comparison, access) | [concepts.md](docs/account-usage/concepts.md) |
| Full categorized catalog of views | [views-catalog.md](docs/account-usage/views-catalog.md) |
| Cost, metering & consumption | [cost-and-usage.md](docs/account-usage/cost-and-usage.md) |
| Storage | [storage.md](docs/account-usage/storage.md) |
| Query activity & performance | [query-and-performance.md](docs/account-usage/query-and-performance.md) |
| Security & access | [security-and-access.md](docs/account-usage/security-and-access.md) |
| Data loading & unloading | [data-loading.md](docs/account-usage/data-loading.md) |
| Governance, tags & data quality | [governance-and-data-quality.md](docs/account-usage/governance-and-data-quality.md) |
| Object / catalog metadata | [object-metadata.md](docs/account-usage/object-metadata.md) |
| Replication & failover | [replication.md](docs/account-usage/replication.md) |
| Organization usage (multi‑account, currency) | [organization-usage.md](docs/account-usage/organization-usage.md) |
| Reader account usage | [reader-account-usage.md](docs/account-usage/reader-account-usage.md) |
| Dashboard build guide (objective → views → SQL) | [dashboard-guide.md](docs/account-usage/dashboard-guide.md) |

> **Accuracy note:** Snowflake adds and revises Account Usage views frequently.
> Always confirm exact view/column names against the live documentation and your
> own account (`SHOW VIEWS IN SCHEMA SNOWFLAKE.ACCOUNT_USAGE;` and
> `DESC VIEW SNOWFLAKE.ACCOUNT_USAGE.<view_name>;`).