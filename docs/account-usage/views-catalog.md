# Views Catalog (`SNOWFLAKE.ACCOUNT_USAGE`)

The complete, categorized catalog of `ACCOUNT_USAGE` views. Use this as the index
to find the right view; detailed column documentation for the high‑value views
lives in the per‑category files linked from the [README](README.md).

> **Latency / retention values are typical defaults** and can change. Verify
> against the live Snowflake docs and your account. Unless noted, retention is
> **365 days**.

## Legend

- **Latency** — approximate delay before data appears.
- **Retention** — how far back history is kept.

---

## Cost, Metering & Consumption

| View | Description | Latency | Retention |
|------|-------------|---------|-----------|
| `METERING_HISTORY` | Hourly credit usage for the account across all services. | ~2 h | 365 d |
| `METERING_DAILY_HISTORY` | Daily credit usage by service type, incl. daily caps/adjustments. | ~3 h | 365 d |
| `WAREHOUSE_METERING_HISTORY` | Hourly credits consumed **per virtual warehouse** (compute + cloud services). | ~3 h (often noted ~45 m) | 365 d |
| `AUTOMATIC_CLUSTERING_HISTORY` | Credits for serverless automatic clustering, per table. | ~3 h | 365 d |
| `MATERIALIZED_VIEW_REFRESH_HISTORY` | Credits for serverless materialized view maintenance. | ~3 h | 365 d |
| `SEARCH_OPTIMIZATION_HISTORY` | Credits for the Search Optimization Service, per table. | ~3 h | 365 d |
| `PIPE_USAGE_HISTORY` | Credits + bytes/files loaded by Snowpipe, per pipe. | ~3 h | 365 d |
| `SERVERLESS_TASK_HISTORY` | Credits for serverless task execution. | ~3 h | 365 d |
| `QUERY_ACCELERATION_HISTORY` | Credits for the Query Acceleration Service, per warehouse. | ~3 h | 365 d |
| `REPLICATION_USAGE_HISTORY` | Credits + bytes for database replication. | ~3 h | 365 d |
| `REPLICATION_GROUP_USAGE_HISTORY` | Credits + bytes for replication/failover **groups**. | ~3 h | 365 d |
| `DATABASE_REPLICATION_USAGE_HISTORY` | Per‑database replication credits/bytes. | ~3 h | 365 d |
| `DATA_TRANSFER_HISTORY` | Bytes + credits for data transfer egress (cross‑region/cloud). | ~2 h | 365 d |
| `SNOWPIPE_STREAMING_CLIENT_HISTORY` | Credits for Snowpipe Streaming clients. | ~3 h | 365 d |
| `CORTEX_FUNCTIONS_USAGE_HISTORY` | Token/credit usage for Cortex AI functions. | varies | 365 d |
| `LISTING_AUTO_FULFILLMENT_USAGE_HISTORY` | Credits for auto‑fulfillment of cross‑region listings. | ~3 h | 365 d |

➡️ Details: [cost-and-usage.md](cost-and-usage.md)

---

## Storage

| View | Description | Latency | Retention |
|------|-------------|---------|-----------|
| `STORAGE_USAGE` | Daily account‑level storage (DB + Fail‑safe + stage), in bytes. | ~2 h | 365 d |
| `DATABASE_STORAGE_USAGE_HISTORY` | Daily storage **per database** (active + Fail‑safe). | ~2 h | 365 d |
| `STAGE_STORAGE_USAGE_HISTORY` | Daily bytes stored in internal stages. | ~3 h | 365 d |
| `TABLE_STORAGE_METRICS` | Per‑table storage breakdown (active, Time Travel, Fail‑safe, retained for clones). | ~90 m | current |

➡️ Details: [storage.md](storage.md)

---

## Query Activity & Performance

| View | Description | Latency | Retention |
|------|-------------|---------|-----------|
| `QUERY_HISTORY` | One row per executed statement with timing, bytes, credits‑driving metrics, errors. | ~45 m | 365 d |
| `QUERY_ATTRIBUTION_HISTORY` | Estimated **credit attribution per query** (compute cost per query). | ~6 h | 365 d |
| `ACCESS_HISTORY` | Per‑query column/object read & write lineage (objects accessed/modified). | ~3 h | 365 d |
| `AGGREGATE_QUERY_HISTORY` | Aggregated query stats (reduces row volume vs `QUERY_HISTORY`). | ~varies | 365 d |
| `SESSIONS` | Session metadata incl. authentication method and client. | ~3 h | 365 d |
| `WAREHOUSE_LOAD_HISTORY` | Warehouse load: running/queued/blocked query counts over time. | ~3 h | 365 d |
| `WAREHOUSE_EVENTS_HISTORY` | Warehouse lifecycle events (start, suspend, resize, auto‑resume). | ~3 h | 365 d |

➡️ Details: [query-and-performance.md](query-and-performance.md)

---

## Security & Access

| View | Description | Latency | Retention |
|------|-------------|---------|-----------|
| `LOGIN_HISTORY` | Authentication attempts (success/failure, method, client, IP). | ~2 h | 365 d |
| `SESSIONS` | Sessions and their authentication context. | ~3 h | 365 d |
| `USERS` | All users (incl. dropped), with status and key properties. | ~2 h | 365 d |
| `ROLES` | All roles (incl. dropped). | ~2 h | 365 d |
| `GRANTS_TO_USERS` | Role → user grants. | ~2 h | 365 d |
| `GRANTS_TO_ROLES` | Privilege → role and role → role grants. | ~2 h | 365 d |
| `GRANTS_TO_SHARE` | Privileges granted to shares. | ~2 h | 365 d |
| `MASKING_POLICIES` | Defined column masking policies. | ~2 h | 365 d |
| `ROW_ACCESS_POLICIES` | Defined row access policies. | ~2 h | 365 d |
| `PASSWORD_POLICIES` | Defined password policies. | ~2 h | 365 d |
| `SESSION_POLICIES` | Defined session policies. | ~2 h | 365 d |
| `AUTHENTICATION_POLICIES` | Defined authentication policies. | ~2 h | 365 d |
| `NETWORK_POLICIES` | Defined network policies. | ~2 h | 365 d |
| `NETWORK_RULES` | Defined network rules. | ~2 h | 365 d |
| `POLICY_REFERENCES` | Where masking/row‑access/other policies are applied. | ~2 h | 365 d |
| `DELEGATED_AUTHORIZATIONS` | OAuth / delegated authorization grants. | ~2 h | 365 d |

➡️ Details: [security-and-access.md](security-and-access.md)

---

## Data Loading & Unloading

| View | Description | Latency | Retention |
|------|-------------|---------|-----------|
| `COPY_HISTORY` | `COPY INTO` load activity per table/file, incl. errors. | ~2 h | 365 d |
| `LOAD_HISTORY` | Bulk data load history (legacy `COPY` history). | ~90 m | 365 d |
| `PIPE_USAGE_HISTORY` | Snowpipe credits + bytes/files (also in Cost section). | ~3 h | 365 d |

➡️ Details: [data-loading.md](data-loading.md)

---

## Governance, Tags & Data Quality

| View | Description | Latency | Retention |
|------|-------------|---------|-----------|
| `TAGS` | Defined tags. | ~2 h | 365 d |
| `TAG_REFERENCES` | Tag assignments to objects/columns. | ~2 h | 365 d |
| `POLICY_REFERENCES` | Policy → object associations. | ~2 h | 365 d |
| `MASKING_POLICIES` / `ROW_ACCESS_POLICIES` | Policy definitions. | ~2 h | 365 d |
| `OBJECT_DEPENDENCIES` | Object‑to‑object dependency graph. | ~3 h | 365 d |
| `ACCESS_HISTORY` | Read/write lineage (also under Query). | ~3 h | 365 d |
| `DATA_METRIC_FUNCTION_REFERENCES` | Where Data Metric Functions (DMFs) are attached. | ~2 h | 365 d |
| `DATA_QUALITY_MONITORING_RESULTS` | Results of DMF / data quality measurements. | ~varies | 365 d |
| `CLASSES` / `CLASSIFICATION` (where available) | Sensitive‑data classification metadata. | ~2 h | 365 d |

➡️ Details: [governance-and-data-quality.md](governance-and-data-quality.md)

---

## Object / Catalog Metadata

| View | Description |
|------|-------------|
| `DATABASES` | All databases (incl. dropped). |
| `SCHEMATA` | All schemas. |
| `TABLES` | All tables (incl. external/dynamic), row & byte counts. |
| `COLUMNS` | All columns with data types. |
| `VIEWS` | All views with definitions. |
| `MATERIALIZED_VIEWS` | Materialized view metadata (where exposed). |
| `FUNCTIONS` | UDFs / external functions. |
| `PROCEDURES` | Stored procedures. |
| `SEQUENCES` | Sequences. |
| `STAGES` | Internal/external stages. |
| `PIPES` | Snowpipe pipes. |
| `FILE_FORMATS` | File format objects. |
| `TASK_HISTORY` | Task run history (success/failure, timing). |
| `TASK_VERSIONS` | Task DAG version definitions. |
| `EXTERNAL_TABLES` | External table metadata. |
| `TABLE_CONSTRAINTS` / `REFERENTIAL_CONSTRAINTS` | Constraint definitions. |
| `TABLE_STORAGE_METRICS` | Per‑table storage (also under Storage). |
| `OBJECT_DEPENDENCIES` | Dependency graph (also under Governance). |

➡️ Details: [object-metadata.md](object-metadata.md)

---

## Replication & Failover

| View | Description |
|------|-------------|
| `REPLICATION_USAGE_HISTORY` | Per‑database replication credits + bytes. |
| `DATABASE_REPLICATION_USAGE_HISTORY` | Per‑database replication detail. |
| `REPLICATION_GROUP_USAGE_HISTORY` | Replication/failover group usage. |

➡️ Details: [replication.md](replication.md)

---

## Related schemas (separate files)

- **`ORGANIZATION_USAGE`** — org‑wide rollups incl. currency. See
  [organization-usage.md](organization-usage.md).
- **`READER_ACCOUNT_USAGE`** — reader (managed) account monitoring. See
  [reader-account-usage.md](reader-account-usage.md).

---

## Discovering the live list in your account

The authoritative list for *your* edition/region is always your account itself:

```sql
SHOW VIEWS IN SCHEMA SNOWFLAKE.ACCOUNT_USAGE;

SELECT table_name, comment
FROM   SNOWFLAKE.INFORMATION_SCHEMA.TABLES
WHERE  table_schema = 'ACCOUNT_USAGE'
ORDER  BY table_name;

-- Inspect any view's columns:
DESC VIEW SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY;
```
