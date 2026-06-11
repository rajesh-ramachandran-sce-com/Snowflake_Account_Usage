# Query Activity & Performance Views

These views power workload, performance, and per‑query cost analysis.

---

## `QUERY_HISTORY`

The central view for workload analysis — **one row per executed statement**
(SQL, DDL, DML, calls). Heavily used for performance tuning, error monitoring,
and warehouse right‑sizing.

### Identity & classification

| Column | Notes |
|--------|-------|
| `QUERY_ID` | Unique statement ID (join key to `ACCESS_HISTORY`, `QUERY_ATTRIBUTION_HISTORY`). |
| `QUERY_TEXT` | The SQL text (may be truncated; can contain sensitive literals). |
| `QUERY_TYPE` | e.g. `SELECT`, `INSERT`, `CREATE`, `MERGE`. |
| `QUERY_TAG` | Session/statement tag — use it to attribute queries to apps/teams. |
| `QUERY_HASH` / `QUERY_PARAMETERIZED_HASH` | Hashes for grouping identical/parameterized queries. |
| `SESSION_ID` | Owning session (join to `SESSIONS`). |

### Who / where

| Column | Notes |
|--------|-------|
| `USER_NAME` | User who ran it. |
| `ROLE_NAME` | Active role. |
| `WAREHOUSE_NAME` / `WAREHOUSE_ID` | Warehouse used. |
| `WAREHOUSE_SIZE` / `WAREHOUSE_TYPE` | Size/type at execution time. |
| `DATABASE_NAME` / `SCHEMA_NAME` | Context. |

### Timing

| Column | Notes |
|--------|-------|
| `START_TIME` / `END_TIME` | Execution window. |
| `TOTAL_ELAPSED_TIME` | Total ms end‑to‑end. |
| `COMPILATION_TIME` | ms compiling. |
| `EXECUTION_TIME` | ms executing. |
| `QUEUED_OVERLOAD_TIME` | ms queued due to warehouse overload (right‑sizing signal). |
| `QUEUED_PROVISIONING_TIME` | ms waiting for warehouse to start. |
| `TRANSACTION_BLOCKED_TIME` | ms blocked by locks. |

### Data volume & performance

| Column | Notes |
|--------|-------|
| `BYTES_SCANNED` | Bytes scanned. |
| `PERCENTAGE_SCANNED_FROM_CACHE` | Cache effectiveness. |
| `ROWS_PRODUCED` / `ROWS_INSERTED` / `ROWS_UPDATED` / `ROWS_DELETED` | Row counts. |
| `PARTITIONS_SCANNED` / `PARTITIONS_TOTAL` | Pruning effectiveness. |
| `BYTES_SPILLED_TO_LOCAL_STORAGE` | Local spill (memory pressure). |
| `BYTES_SPILLED_TO_REMOTE_STORAGE` | Remote spill (severe memory pressure — strong "scale up" signal). |
| `CREDITS_USED_CLOUD_SERVICES` | Cloud‑services credits attributed to the query. |
| `QUERY_ACCELERATION_BYTES_SCANNED` | Bytes handled by Query Acceleration Service. |

### Outcome

| Column | Notes |
|--------|-------|
| `EXECUTION_STATUS` | `SUCCESS`, `FAIL`, `INCIDENT`. |
| `ERROR_CODE` / `ERROR_MESSAGE` | Failure detail. |

> For per‑query **credit cost**, join to `QUERY_ATTRIBUTION_HISTORY` on
> `QUERY_ID` — `QUERY_HISTORY` itself only reports cloud‑services credits, not
> the warehouse compute attributed to the query.

---

## `QUERY_ATTRIBUTION_HISTORY`

Estimates the **warehouse compute credits attributable to each query**, enabling
true per‑query / per‑user / per‑tag cost analysis.

| Column | Notes |
|--------|-------|
| `QUERY_ID` | Join to `QUERY_HISTORY`. |
| `QUERY_HASH` / `QUERY_PARAMETERIZED_HASH` | Group similar queries. |
| `START_TIME` / `END_TIME` | Window. |
| `WAREHOUSE_ID` / `WAREHOUSE_NAME` | Warehouse. |
| `USER_NAME` | User. |
| `QUERY_TAG` | Attribution tag. |
| `CREDITS_ATTRIBUTED_COMPUTE` | Estimated compute credits for the query. |
| `CREDITS_USED_QUERY_ACCELERATION` | QAS credits for the query. |

> Note: very short/cheap queries may be excluded; attribution is an estimate that
> distributes warehouse credits across concurrent queries.

---

## `ACCESS_HISTORY`

Per‑query **data lineage**: which columns/objects were read and which were
written. Foundational for governance, impact analysis, and "who touched this
table" auditing.

| Column | Notes |
|--------|-------|
| `QUERY_ID` | Join to `QUERY_HISTORY`. |
| `QUERY_START_TIME` | When it ran. |
| `USER_NAME` | Who ran it. |
| `DIRECT_OBJECTS_ACCESSED` | ARRAY of objects/columns referenced directly. |
| `BASE_OBJECTS_ACCESSED` | ARRAY of underlying base table columns (through views). |
| `OBJECTS_MODIFIED` | ARRAY of objects/columns written. |
| `OBJECT_MODIFIED_BY_DDL` | DDL change descriptor (where applicable). |

These are semi‑structured (`VARIANT`/`ARRAY`) columns — use `LATERAL FLATTEN` to
expand them for column‑level lineage.

---

## `SESSIONS`

Session metadata, including how the session authenticated and the client used.

| Column | Notes |
|--------|-------|
| `SESSION_ID` | Join to `QUERY_HISTORY` / `LOGIN_HISTORY`. |
| `CREATED_ON` | Session start. |
| `USER_NAME` | User. |
| `AUTHENTICATION_METHOD` | Password, SSO/SAML, key‑pair, OAuth, etc. |
| `LOGIN_EVENT_ID` | Join to `LOGIN_HISTORY.EVENT_ID`. |
| `CLIENT_APPLICATION_ID` / `CLIENT_APPLICATION_VERSION` | Driver/client. |
| `CLIENT_ENVIRONMENT` | Client details (VARIANT). |

---

## `WAREHOUSE_LOAD_HISTORY`

Warehouse load over time — concurrency and queuing pressure. Pair with
`WAREHOUSE_METERING_HISTORY` to see whether spend matches demand.

| Column | Notes |
|--------|-------|
| `START_TIME` / `END_TIME` | Window. |
| `WAREHOUSE_ID` / `WAREHOUSE_NAME` | Warehouse. |
| `AVG_RUNNING` | Avg concurrently running queries. |
| `AVG_QUEUED_LOAD` | Avg queries queued due to load (scale‑out signal for multi‑cluster). |
| `AVG_QUEUED_PROVISIONING` | Avg queued waiting for warehouse start. |
| `AVG_BLOCKED` | Avg blocked by locks. |

---

## `WAREHOUSE_EVENTS_HISTORY`

Lifecycle events for warehouses — useful to explain spend spikes and verify
auto‑suspend behavior.

| Column | Notes |
|--------|-------|
| `TIMESTAMP` | When the event occurred. |
| `WAREHOUSE_ID` / `WAREHOUSE_NAME` | Warehouse. |
| `EVENT_NAME` | e.g. `CREATE`, `RESUME_WAREHOUSE`, `SUSPEND_WAREHOUSE`, `RESIZE_WAREHOUSE`. |
| `EVENT_REASON` | Why (e.g. user action, auto‑suspend). |
| `EVENT_STATE` | `STARTED` / `COMPLETED`. |
| `SIZE` / `CLUSTER_COUNT` | Size and cluster count where relevant. |

---

## `AGGREGATE_QUERY_HISTORY`

Pre‑aggregated query statistics (where available), reducing the cost of scanning
`QUERY_HISTORY` for high‑volume trend dashboards. Verify availability and exact
columns in your account.
