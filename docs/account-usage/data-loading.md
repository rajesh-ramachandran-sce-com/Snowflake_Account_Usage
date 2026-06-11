# Data Loading & Unloading Views

Track how data enters Snowflake — bulk loads (`COPY INTO`), legacy load history,
and continuous loads via Snowpipe.

---

## `COPY_HISTORY`

Load activity for `COPY INTO <table>` (bulk and Snowpipe), **per file**, including
errors. Primary view for ingestion monitoring and load‑failure alerting.

| Column | Notes |
|--------|-------|
| `FILE_NAME` | Source file. |
| `STAGE_LOCATION` | Stage path. |
| `LAST_LOAD_TIME` | When the file was loaded. |
| `ROW_COUNT` | Rows loaded from the file. |
| `ROW_PARSED` | Rows parsed. |
| `FILE_SIZE` | Bytes. |
| `FIRST_ERROR_MESSAGE` | First error (if any). |
| `FIRST_ERROR_LINE_NUMBER` / `FIRST_ERROR_CHARACTER_POS` | Error location. |
| `FIRST_ERROR_COLUMN_NAME` | Column where the first error occurred. |
| `ERROR_COUNT` | Number of errors. |
| `ERROR_LIMIT` | Configured error limit. |
| `STATUS` | `Loaded`, `Load failed`, `Partially loaded`. |
| `TABLE_CATALOG_NAME` / `TABLE_SCHEMA_NAME` / `TABLE_NAME` | Target table. |
| `PIPE_CATALOG_NAME` / `PIPE_SCHEMA_NAME` / `PIPE_NAME` | Pipe if loaded via Snowpipe. |
| `PIPE_RECEIVED_TIME` | When Snowpipe received the file. |

> `ACCOUNT_USAGE.COPY_HISTORY` is the account‑wide, historical version. There is
> also an `INFORMATION_SCHEMA.COPY_HISTORY` **table function** for real‑time,
> per‑table queries.

---

## `LOAD_HISTORY`

Historical bulk data load events (the account‑usage counterpart to the classic
load history). Useful where you need a simpler per‑load summary than
`COPY_HISTORY`.

| Column | Notes |
|--------|-------|
| `FILE_NAME` | Source file. |
| `TABLE_CATALOG_NAME` / `TABLE_SCHEMA_NAME` / `TABLE_NAME` | Target. |
| `LAST_LOAD_TIME` | When loaded. |
| `STATUS` | Load status. |
| `ROW_COUNT` | Rows loaded. |
| `ROW_PARSED` | Rows parsed. |
| `FILE_SIZE` | Bytes. |
| `ERROR_COUNT` | Errors. |
| `FIRST_ERROR_MESSAGE` | First error message. |

> Note: `LOAD_HISTORY` excludes Snowpipe loads; use `COPY_HISTORY` /
> `PIPE_USAGE_HISTORY` for Snowpipe.

---

## `PIPE_USAGE_HISTORY`

Snowpipe consumption — credits, bytes, and files per pipe. Documented in detail
under [cost-and-usage.md](cost-and-usage.md). For ingestion monitoring, combine:

- `PIPE_USAGE_HISTORY` → cost & volume trend per pipe.
- `COPY_HISTORY` (filtered to `PIPE_NAME IS NOT NULL`) → per‑file success/errors.

---

## Ingestion dashboard ideas

- **Load failures:** `COPY_HISTORY` where `STATUS <> 'Loaded'`, grouped by table
  and day, surfacing `FIRST_ERROR_MESSAGE`.
- **Ingestion volume:** sum `ROW_COUNT` / `FILE_SIZE` by table over time.
- **Snowpipe efficiency:** `PIPE_USAGE_HISTORY` credits per GB
  (`CREDITS_USED / (BYTES_INSERTED / POWER(1024,3))`) by pipe.
- **Latency:** `COPY_HISTORY.LAST_LOAD_TIME − PIPE_RECEIVED_TIME` for Snowpipe.
