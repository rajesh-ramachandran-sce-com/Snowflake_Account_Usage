# Governance, Tags & Data Quality Views

These views support data governance dashboards: classification & tagging, policy
coverage, lineage, dependencies, and data quality monitoring.

---

## `TAGS`

Definitions of object tags (used for classification, cost attribution, and
governance).

| Column | Notes |
|--------|-------|
| `TAG_ID` | Stable identifier. |
| `TAG_NAME` | Tag name. |
| `TAG_SCHEMA` / `TAG_DATABASE` | Where the tag is defined. |
| `TAG_ALLOWED_VALUES` | Allowed values (if constrained). |
| `OWNER` | Owning role. |
| `CREATED` / `LAST_ALTERED` / `DELETED` | Lifecycle. |

---

## `TAG_REFERENCES`

Where each tag is **applied** (object/column level) and its value. Central to
governance reporting and tag‑based cost attribution.

| Column | Notes |
|--------|-------|
| `TAG_NAME` / `TAG_DATABASE` / `TAG_SCHEMA` | The tag. |
| `TAG_VALUE` | Value assigned. |
| `OBJECT_DATABASE` / `OBJECT_SCHEMA` / `OBJECT_NAME` | Tagged object. |
| `DOMAIN` | Object domain (e.g. `TABLE`, `COLUMN`, `WAREHOUSE`). |
| `COLUMN_NAME` | Column, when tag is on a column. |
| `OBJECT_ID` | Object identifier. |

---

## `POLICY_REFERENCES`

Where masking, row‑access, and other policies are **attached**. Pair with the
policy definition views in [security-and-access.md](security-and-access.md) to
find coverage gaps.

| Column | Notes |
|--------|-------|
| `POLICY_NAME` / `POLICY_DB` / `POLICY_SCHEMA` | The policy. |
| `POLICY_KIND` | e.g. `MASKING_POLICY`, `ROW_ACCESS_POLICY`, `SESSION_POLICY`, `PASSWORD_POLICY`, `AUTHENTICATION_POLICY`. |
| `REF_DATABASE_NAME` / `REF_SCHEMA_NAME` / `REF_ENTITY_NAME` | Object the policy is applied to. |
| `REF_ENTITY_DOMAIN` | Domain of that object. |
| `REF_COLUMN_NAME` | Protected column (for masking). |
| `REF_ARG_COLUMN_NAMES` | Columns passed to a row access policy. |

---

## `OBJECT_DEPENDENCIES`

Object‑to‑object dependency graph (e.g. a view depends on its base tables).
Foundational for impact analysis ("what breaks if I drop this?").

| Column | Notes |
|--------|-------|
| `REFERENCED_DATABASE` / `REFERENCED_SCHEMA` / `REFERENCED_OBJECT_NAME` | The object depended on. |
| `REFERENCED_OBJECT_ID` | Its ID. |
| `REFERENCED_OBJECT_DOMAIN` | Its type. |
| `REFERENCING_DATABASE` / `REFERENCING_SCHEMA` / `REFERENCING_OBJECT_NAME` | The dependent object. |
| `REFERENCING_OBJECT_ID` | Its ID. |
| `REFERENCING_OBJECT_DOMAIN` | Its type. |
| `DEPENDENCY_TYPE` | Nature of the dependency (e.g. `BY_NAME`, `BY_ID`). |

---

## `ACCESS_HISTORY`

Per‑query read/write **lineage** at column granularity. Documented under
[query-and-performance.md](query-and-performance.md). For governance it answers:
who read sensitive columns, and which queries wrote to a table.

---

## Data Metric Functions & Data Quality

### `DATA_METRIC_FUNCTION_REFERENCES`
Where Data Metric Functions (DMFs) are attached to tables/columns and their
schedules.

| Column | Notes |
|--------|-------|
| `METRIC_NAME` / `METRIC_DATABASE` / `METRIC_SCHEMA` | The DMF. |
| `REF_ENTITY_NAME` / `REF_ENTITY_DOMAIN` | Object measured. |
| `REF_ARGUMENTS` | Columns/arguments measured. |
| `SCHEDULE` / `SCHEDULE_STATUS` | When/whether it runs. |

### `DATA_QUALITY_MONITORING_RESULTS`
Measured results from DMFs (e.g. null counts, duplicate counts, freshness),
powering data‑quality dashboards and alerts.

| Column | Notes |
|--------|-------|
| `MEASUREMENT_TIME` | When measured. |
| `METRIC_NAME` / `METRIC_DATABASE` / `METRIC_SCHEMA` | The metric. |
| `TABLE_NAME` / `TABLE_SCHEMA` / `TABLE_DATABASE` | Measured table. |
| `ARGUMENT_IDS` / `ARGUMENT_NAMES` | Columns measured. |
| `VALUE` | Measured value. |

> Classification metadata (sensitive data categories) may be exposed via
> additional views/functions depending on edition. Verify availability in‑account.

---

## Governance dashboard ideas

- **PII coverage:** join `TAG_REFERENCES` (e.g. a `PII` tag) with
  `POLICY_REFERENCES` to confirm tagged columns have masking policies.
- **Sensitive data access:** `ACCESS_HISTORY` flattened to columns, filtered to
  tagged sensitive columns, by `USER_NAME`.
- **Lineage / impact:** recursive walk of `OBJECT_DEPENDENCIES`.
- **Data quality trend:** `DATA_QUALITY_MONITORING_RESULTS.VALUE` over
  `MEASUREMENT_TIME`, alert when thresholds breach.
