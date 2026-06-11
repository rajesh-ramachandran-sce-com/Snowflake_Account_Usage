# Reader Account Usage (`SNOWFLAKE.READER_ACCOUNT_USAGE`)

If your account creates **reader (managed) accounts** to share data with
consumers who don't have their own Snowflake account, you (the provider) pay for
their compute and storage. The `READER_ACCOUNT_USAGE` schema lets you monitor and
charge back that consumption.

> These views mirror many `ACCOUNT_USAGE` views but are scoped to the reader
> accounts you manage, and include a `READER_ACCOUNT_NAME` dimension. Latency and
> 365‑day retention are comparable to `ACCOUNT_USAGE`.

## Key views

| View | Description |
|------|-------------|
| `METERING_DAILY_HISTORY` | Daily credits per reader account by service type. |
| `WAREHOUSE_METERING_HISTORY` | Per‑warehouse credits within reader accounts. |
| `STORAGE_USAGE` | Daily storage consumed by reader accounts. |
| `QUERY_HISTORY` | Queries run inside reader accounts. |
| `LOGIN_HISTORY` | Logins to reader accounts. |
| `RESOURCE_MONITORS` | Resource monitors governing reader account spend. |

Most rows carry identifying columns such as `READER_ACCOUNT_NAME` (and/or a
reader account locator) so you can group usage per consumer.

## Reader‑account dashboard ideas

- **Cost per reader account:** sum credits from `METERING_DAILY_HISTORY` /
  `WAREHOUSE_METERING_HISTORY` grouped by `READER_ACCOUNT_NAME`, for chargeback.
- **Reader workload:** `QUERY_HISTORY` volume and performance per reader account.
- **Reader storage:** `STORAGE_USAGE` per reader account.
- **Spend guardrails:** verify `RESOURCE_MONITORS` are in place and check usage
  against their quotas.

> Confirm the exact set of views and columns with
> `SHOW VIEWS IN SCHEMA SNOWFLAKE.READER_ACCOUNT_USAGE;` and the live docs, as
> availability depends on whether you manage any reader accounts.
