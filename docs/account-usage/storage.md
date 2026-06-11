# Storage Views

Storage is billed on **average bytes stored per month** across active data, Time
Travel, and Fail‑safe, plus staged files. These views provide daily snapshots and
per‑table detail for storage dashboards.

> Storage views report **bytes**. Convert to TB (`/ POWER(1024,4)`) for charts,
> and multiply by your storage rate for currency.

---

## `STORAGE_USAGE`

Account‑level daily storage snapshot — the headline "how much are we storing"
number.

| Column | Type | Notes |
|--------|------|-------|
| `USAGE_DATE` | DATE | The day. |
| `STORAGE_BYTES` | NUMBER | Average bytes of active database storage. |
| `STAGE_BYTES` | NUMBER | Average bytes in internal stages. |
| `FAILSAFE_BYTES` | NUMBER | Average bytes in Fail‑safe. |

Total storage for a day ≈ `STORAGE_BYTES + STAGE_BYTES + FAILSAFE_BYTES`.

---

## `DATABASE_STORAGE_USAGE_HISTORY`

Daily storage **per database**, so you can attribute storage cost to a database
owner / team.

| Column | Type | Notes |
|--------|------|-------|
| `USAGE_DATE` | DATE | The day. |
| `DATABASE_ID` | NUMBER | Stable database identifier. |
| `DATABASE_NAME` | TEXT | Database name (`NULL`/retained for dropped DBs still in Fail‑safe). |
| `AVERAGE_DATABASE_BYTES` | NUMBER | Avg active bytes for the DB that day. |
| `AVERAGE_FAILSAFE_BYTES` | NUMBER | Avg Fail‑safe bytes for the DB that day. |
| `AVERAGE_HYBRID_TABLE_STORAGE_BYTES` | NUMBER | Hybrid table storage (where applicable). |
| `DELETED` | BOOLEAN | Whether the database is dropped. |

---

## `STAGE_STORAGE_USAGE_HISTORY`

Daily bytes held in internal named/table/user stages.

| Column | Type | Notes |
|--------|------|-------|
| `USAGE_DATE` | DATE | The day. |
| `AVERAGE_STAGE_BYTES` | NUMBER | Avg bytes stored in stages that day. |

---

## `TABLE_STORAGE_METRICS`

Per‑table, current‑state storage breakdown — essential for finding the biggest
tables and understanding Time Travel / Fail‑safe / clone retention overhead. This
is a current snapshot (not a daily history) and includes dropped tables still
incurring storage.

| Column | Type | Notes |
|--------|------|-------|
| `ID` | NUMBER | Table ID. |
| `TABLE_CATALOG` / `TABLE_SCHEMA` / `TABLE_NAME` (+ `*_ID`) | TEXT/NUMBER | Fully qualified table. |
| `ACTIVE_BYTES` | NUMBER | Bytes in the active/current table. |
| `TIME_TRAVEL_BYTES` | NUMBER | Bytes retained for Time Travel. |
| `FAILSAFE_BYTES` | NUMBER | Bytes in Fail‑safe. |
| `RETAINED_FOR_CLONE_BYTES` | NUMBER | Bytes retained because a clone references them. |
| `IS_TRANSIENT` | TEXT | Transient tables have no Fail‑safe. |
| `DELETED` | BOOLEAN | Whether the table is dropped. |
| `TABLE_CREATED` / `TABLE_DROPPED` | TIMESTAMP | Lifecycle timestamps. |
| `CATALOG_CREATED` / `CATALOG_DROPPED` | TIMESTAMP | DB lifecycle. |
| `SCHEMA_CREATED` / `SCHEMA_DROPPED` | TIMESTAMP | Schema lifecycle. |

**Tip:** total table storage = `ACTIVE_BYTES + TIME_TRAVEL_BYTES + FAILSAFE_BYTES
+ RETAINED_FOR_CLONE_BYTES`. A large gap between `ACTIVE_BYTES` and the rest often
signals an opportunity to reduce `DATA_RETENTION_TIME_IN_DAYS` or clean up clones.

---

## Choosing the right storage view

| Question | View |
|----------|------|
| Total account storage trend | `STORAGE_USAGE` |
| Storage by database (charge‑back) | `DATABASE_STORAGE_USAGE_HISTORY` |
| Staged file storage trend | `STAGE_STORAGE_USAGE_HISTORY` |
| Biggest tables / Time Travel & Fail‑safe overhead | `TABLE_STORAGE_METRICS` |
