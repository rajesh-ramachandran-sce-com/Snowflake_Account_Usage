# Organization Usage (`SNOWFLAKE.ORGANIZATION_USAGE`)

The `ORGANIZATION_USAGE` schema rolls up usage and cost **across every account in
your organization**, and — critically — it can express cost in **currency**, not
just credits. Query it from the organization's primary account (or an account
with the appropriate organization privileges).

> Use `ACCOUNT_USAGE` for deep, single‑account detail; use `ORGANIZATION_USAGE`
> for the executive, multi‑account, money‑denominated view. Org views generally
> retain **365 days** (some up to ~1 year+; verify) and have similar latency.

## Key views

| View | Description |
|------|-------------|
| `USAGE_IN_CURRENCY_DAILY` | Daily usage converted to **currency** per account & usage type, incl. the effective rate. The headline spend view. |
| `METERING_DAILY_HISTORY` | Daily credits by account and service type across the org. |
| `WAREHOUSE_METERING_HISTORY` | Per‑warehouse credits across all accounts. |
| `STORAGE_DAILY_HISTORY` | Daily storage bytes per account (DB + Fail‑safe + stage). |
| `DATABASE_STORAGE_USAGE_HISTORY` | Per‑database storage across accounts. |
| `STAGE_STORAGE_USAGE_HISTORY` | Stage storage across accounts. |
| `DATA_TRANSFER_HISTORY` | Data transfer (bytes + currency) across accounts. |
| `AUTOMATIC_CLUSTERING_HISTORY` | Clustering credits across accounts. |
| `MATERIALIZED_VIEW_REFRESH_HISTORY` | MV maintenance credits across accounts. |
| `SEARCH_OPTIMIZATION_HISTORY` | SOS credits across accounts. |
| `PIPE_USAGE_HISTORY` | Snowpipe credits across accounts. |
| `REPLICATION_USAGE_HISTORY` | Replication credits across accounts. |
| `CONTRACT_ITEMS` | Contract line items / commitments. |
| `REMAINING_BALANCE_DAILY` | Remaining capacity/credit balance over time. |
| `RATE_SHEET_DAILY` | Effective per‑credit rates by usage type/date. |
| `BALANCE_DAILY` | Daily capacity, on‑demand, and rollover balances. |
| `ACCOUNTS` | Inventory of accounts in the organization. |

## `USAGE_IN_CURRENCY_DAILY` (most important)

| Column | Notes |
|--------|-------|
| `ORGANIZATION_NAME` | Org. |
| `ACCOUNT_NAME` / `ACCOUNT_LOCATOR` | Account. |
| `REGION` | Region. |
| `USAGE_DATE` | Day. |
| `USAGE_TYPE` | e.g. `compute`, `storage`, `data transfer`, serverless types. |
| `CURRENCY` | Currency code. |
| `USAGE` | Quantity (credits or bytes depending on type). |
| `USAGE_IN_CURRENCY` | Cost in currency. |

## Organization dashboard ideas

- **Total org spend (money):** sum `USAGE_IN_CURRENCY` by `USAGE_DATE` /
  `ACCOUNT_NAME` / `USAGE_TYPE`.
- **Budget burn‑down:** `REMAINING_BALANCE_DAILY` / `BALANCE_DAILY` vs
  `CONTRACT_ITEMS` commitments.
- **Account‑level chargeback:** spend per `ACCOUNT_NAME`, then drill into each
  account's `ACCOUNT_USAGE` for detail.
- **Effective rate tracking:** `RATE_SHEET_DAILY` to monitor price changes.

> Exact view/column names in `ORGANIZATION_USAGE` evolve; confirm with
> `SHOW VIEWS IN SCHEMA SNOWFLAKE.ORGANIZATION_USAGE;` and the live docs.
