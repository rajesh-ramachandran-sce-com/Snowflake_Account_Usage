# Core Concepts

Understanding these concepts is essential before querying Account Usage data or
designing dashboards on top of it.

## 1. The shared `SNOWFLAKE` database

Every Snowflake account includes a system‑defined, read‑only database named
**`SNOWFLAKE`**, imported from a share provided by Snowflake. It contains several
schemas; the ones relevant to monitoring are:

| Schema | Purpose |
|--------|---------|
| `ACCOUNT_USAGE` | Account‑level usage, metering, metadata, and activity for the **current account**, including dropped objects. |
| `ORGANIZATION_USAGE` | Usage and cost rolled up across **all accounts in the organization**. |
| `READER_ACCOUNT_USAGE` | Usage for **reader (managed) accounts** created by the account. |
| `DATA_SHARING_USAGE` | Activity related to listings/data shares (Marketplace & Data Exchange). |
| `CORE` | Utility functions/procedures (e.g., trust center, tags helpers). |

This knowledge base focuses on the three `*_USAGE` schemas.

## 2. `ACCOUNT_USAGE` vs `INFORMATION_SCHEMA`

Snowflake exposes overlapping object metadata in two places. Choosing correctly
matters for both correctness and dashboard latency.

| Dimension | `SNOWFLAKE.ACCOUNT_USAGE` | `<db>.INFORMATION_SCHEMA` |
|-----------|---------------------------|----------------------------|
| Scope | Entire account | A single database (+ account‑level table functions) |
| Dropped objects | **Included** (with a `DELETED` timestamp) | Excluded — only current objects |
| Latency | Minutes to ~3 hours depending on view | Real‑time / no latency |
| History retention | Up to **365 days** (1 year) | 7 days to 6 months via table functions |
| Best for | Trend analysis, cost, audit, long history | Live introspection of current objects |

**Rule of thumb:** use `ACCOUNT_USAGE` for dashboards and historical trends; use
`INFORMATION_SCHEMA` table functions when you need up‑to‑the‑second data or
per‑database detail.

## 3. Latency

`ACCOUNT_USAGE` views are **not real time**. Each view has its own typical
latency, ranging from a couple of minutes to roughly 3 hours. Common examples:

- ~45 minutes: `QUERY_HISTORY`, `WAREHOUSE_METERING_HISTORY`, `SESSIONS`.
- ~2 hours: `LOGIN_HISTORY`, `COPY_HISTORY`, `DATABASE_STORAGE_USAGE_HISTORY`.
- ~3 hours: `STORAGE_USAGE`, `STAGE_STORAGE_USAGE_HISTORY`, `METERING_DAILY_HISTORY`.
- ~2 hours: `ACCESS_HISTORY`.

Always treat the most recent window (e.g., the last few hours / current day) as
**incomplete**. Per‑view latency is listed in [views-catalog.md](views-catalog.md);
confirm current numbers against the live Snowflake docs.

## 4. Data retention

Most `ACCOUNT_USAGE` views retain **365 days (1 year)** of history. A few differ
(for example, `QUERY_HISTORY` retains 365 days; `LOGIN_HISTORY` retains 365 days;
storage views retain 365 days as daily snapshots). If your dashboard needs longer
history, periodically **materialize** the data into your own tables.

## 5. Time semantics

- Most history views are keyed by event time columns such as `START_TIME` /
  `END_TIME` (queries, metering) or `USAGE_DATE` (daily roll‑ups).
- Timestamps are typically `TIMESTAMP_LTZ` (local time zone of the session).
  Normalize to a single zone (often UTC) in your dashboard layer for consistency.
- Daily roll‑up views (`*_DAILY_HISTORY`, storage views) aggregate to a date and
  are the cheapest source for long‑range trend charts.

## 6. Credits, not currency

Compute usage is reported in **credits**, not money. To show currency you must
multiply credits by your contract's price‑per‑credit (which varies by edition,
region, and cloud). For account‑level currency reporting across the org, prefer
`ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY`, which includes the effective rate.

## 7. Access control (who can query these views)

By default, only the **`ACCOUNTADMIN`** role can access `ACCOUNT_USAGE`. To let a
dashboard service role read it, grant the relevant database roles of the
`SNOWFLAKE` database, for example:

```sql
-- Broad read access to all usage data
GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE my_dashboard_role;

-- Or grant only the specific SNOWFLAKE database roles you need, e.g.:
GRANT DATABASE ROLE SNOWFLAKE.OBJECT_VIEWER       TO ROLE my_dashboard_role;
GRANT DATABASE ROLE SNOWFLAKE.USAGE_VIEWER        TO ROLE my_dashboard_role;
GRANT DATABASE ROLE SNOWFLAKE.GOVERNANCE_VIEWER   TO ROLE my_dashboard_role;
GRANT DATABASE ROLE SNOWFLAKE.SECURITY_VIEWER     TO ROLE my_dashboard_role;
```

Snowflake exposes fine‑grained **database roles** (such as `USAGE_VIEWER`,
`OBJECT_VIEWER`, `SECURITY_VIEWER`, `GOVERNANCE_VIEWER`) on the `SNOWFLAKE`
database so you can grant least‑privilege access instead of all of
`ACCOUNTADMIN`. Confirm the exact set of database roles available in your edition.

## 8. Naming & casing

Object identifiers in these views are usually stored in **uppercase** unless they
were created with quoted, case‑sensitive identifiers. Account for this when
joining on names (e.g., `UPPER(...)`) and prefer joining on **IDs**
(`*_ID` columns) where available, since names can be reused after a drop.
