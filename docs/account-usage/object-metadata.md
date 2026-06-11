# Object / Catalog Metadata Views

These views are the account‑wide catalog. Unlike `INFORMATION_SCHEMA`, they cover
the **entire account** and **include dropped objects** (with a `DELETED` /
`DELETED_ON` timestamp), making them ideal for inventory, growth, and audit
dashboards. Typical latency is ~1–2 hours.

> For up‑to‑the‑second introspection of current objects in one database, use that
> database's `INFORMATION_SCHEMA` instead.

---

## Database & schema level

### `DATABASES`
All databases (incl. dropped).

| Column | Notes |
|--------|-------|
| `DATABASE_ID` | Stable identifier. |
| `DATABASE_NAME` | Name. |
| `DATABASE_OWNER` | Owning role. |
| `IS_TRANSIENT` | Transient flag. |
| `COMMENT` | Description. |
| `RETENTION_TIME` | Time Travel retention (days). |
| `CREATED` / `LAST_ALTERED` / `DELETED` | Lifecycle. |

### `SCHEMATA`
All schemas. Key columns: `SCHEMA_ID`, `SCHEMA_NAME`, `CATALOG_NAME` (database),
`SCHEMA_OWNER`, `IS_TRANSIENT`, `IS_MANAGED_ACCESS`, `RETENTION_TIME`,
`CREATED` / `LAST_ALTERED` / `DELETED`.

---

## Table & column level

### `TABLES`
All tables/views/external/dynamic tables, with size metrics.

| Column | Notes |
|--------|-------|
| `TABLE_ID` | Identifier. |
| `TABLE_CATALOG` / `TABLE_SCHEMA` / `TABLE_NAME` | FQN. |
| `TABLE_OWNER` | Owning role. |
| `TABLE_TYPE` | `BASE TABLE`, `VIEW`, `EXTERNAL TABLE`, `MATERIALIZED VIEW`, etc. |
| `IS_TRANSIENT` | Transient flag. |
| `ROW_COUNT` | Approx. rows. |
| `BYTES` | Approx. active bytes. |
| `CLUSTERING_KEY` | Clustering key if defined. |
| `RETENTION_TIME` | Time Travel days. |
| `AUTO_CLUSTERING_ON` | Whether auto‑clustering is enabled. |
| `CREATED` / `LAST_ALTERED` / `DELETED` | Lifecycle. |

### `COLUMNS`
All columns with data types. Key columns: `TABLE_CATALOG/SCHEMA/NAME`,
`COLUMN_NAME`, `ORDINAL_POSITION`, `DATA_TYPE`, `IS_NULLABLE`,
`COLUMN_DEFAULT`, `CHARACTER_MAXIMUM_LENGTH`, `NUMERIC_PRECISION`,
`NUMERIC_SCALE`, `COMMENT`, `DELETED`.

### `VIEWS`
View definitions. Key columns: `TABLE_CATALOG/SCHEMA/NAME`, `VIEW_DEFINITION`,
`IS_SECURE`, `IS_MATERIALIZED`, `CREATED`, `LAST_ALTERED`, `DELETED`.

---

## Programmable & other objects

| View | Lists |
|------|-------|
| `FUNCTIONS` | UDFs and external functions (signature, language, return type). |
| `PROCEDURES` | Stored procedures. |
| `SEQUENCES` | Sequences. |
| `STAGES` | Internal/external stages (URL, type, owner). |
| `PIPES` | Snowpipe pipes (definition, owner, notification channel). |
| `FILE_FORMATS` | File format objects. |
| `EXTERNAL_TABLES` | External table metadata. |
| `MATERIALIZED_VIEWS` | Materialized view metadata (where exposed). |
| `TABLE_CONSTRAINTS` / `REFERENTIAL_CONSTRAINTS` | Constraint definitions. |

Most of these share the pattern: an `*_ID`, a fully qualified name
(`*_CATALOG`/`*_SCHEMA`/`*_NAME`), an `OWNER`, a `COMMENT`, and lifecycle
columns `CREATED` / `LAST_ALTERED` / `DELETED`.

---

## Tasks

### `TASK_HISTORY`
Run history for tasks (the account‑usage equivalent of the task history table
function).

| Column | Notes |
|--------|-------|
| `NAME` / `DATABASE_NAME` / `SCHEMA_NAME` | The task. |
| `QUERY_ID` | The SQL the task ran (join to `QUERY_HISTORY`). |
| `QUERY_TEXT` | SQL executed. |
| `SCHEDULED_TIME` / `QUERY_START_TIME` / `COMPLETED_TIME` | Timing. |
| `STATE` | `SCHEDULED`, `EXECUTING`, `SUCCEEDED`, `FAILED`, `CANCELLED`, `SKIPPED`. |
| `ERROR_CODE` / `ERROR_MESSAGE` | Failure detail. |
| `ROOT_TASK_ID` / `GRAPH_VERSION` | DAG context. |
| `RUN_ID` | Identifies one DAG run. |

### `TASK_VERSIONS`
Versioned definitions of task DAGs (predecessors, schedule, warehouse).

---

## `TABLE_STORAGE_METRICS`

Per‑table storage detail. Documented under [storage.md](storage.md); also a key
metadata view for finding the largest objects.

---

## Metadata dashboard ideas

- **Object inventory & growth:** counts from `DATABASES` / `SCHEMATA` / `TABLES`
  over `CREATED` vs `DELETED`.
- **Largest tables:** `TABLES.BYTES` or `TABLE_STORAGE_METRICS` ranked.
- **Stale / unused objects:** objects in `TABLES` not appearing in recent
  `ACCESS_HISTORY`.
- **Task reliability:** `TASK_HISTORY` failure rate by task over time.
