# Security & Governance Views

These views support Snowflake's **data governance**, **data security**, and **compliance** capabilities. They track data access patterns, policy definitions, tag assignments, and object-level dependencies.

> **Note:** `ACCESS_HISTORY` and `OBJECT_DEPENDENCIES` require **Snowflake Enterprise Edition** or higher.

---

## ACCESS_HISTORY

**Purpose:** Records column-level access history for every query — capturing exactly which base objects and columns were read from or written to. This is the cornerstone of **data lineage** and **column-level security auditing**.

**Edition Required:** Enterprise or higher  
**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `QUERY_ID` | TEXT | Links to QUERY_HISTORY |
| `QUERY_START_TIME` | TIMESTAMP_LTZ | When the query started |
| `USER_NAME` | TEXT | User who executed the query |
| `DIRECT_OBJECTS_ACCESSED` | ARRAY | JSON array of objects directly referenced in the query |
| `BASE_OBJECTS_ACCESSED` | ARRAY | JSON array of underlying base tables/columns accessed (resolving views) |
| `OBJECTS_MODIFIED` | ARRAY | JSON array of objects written to |
| `OBJECT_MODIFIED_BY_DDL` | VARIANT | Object changed by DDL (CREATE, ALTER, DROP) |
| `POLICIES_REFERENCED` | ARRAY | Masking and row access policies triggered |
| `PARENT_QUERY_ID` | TEXT | Parent query ID (for child queries in tasks/procedures) |
| `ROOT_QUERY_ID` | TEXT | Root query ID in a chain |

**Structure of DIRECT_OBJECTS_ACCESSED / BASE_OBJECTS_ACCESSED:**
Each element is a JSON object with:
- `objectId` — internal object ID
- `objectName` — fully-qualified object name (db.schema.table)
- `objectDomain` — domain type (Table, View, Stage, etc.)
- `columns` — array of column objects accessed, each with:
  - `columnId`
  - `columnName`

**Sample Queries:**
```sql
-- See who accessed a specific sensitive table recently
SELECT query_id, query_start_time, user_name,
       f.value:objectName::TEXT AS object_accessed,
       f.value:columns::VARIANT AS columns_accessed
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
     LATERAL FLATTEN(input => base_objects_accessed) f
WHERE query_start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND f.value:objectName::TEXT ILIKE '%CUSTOMER%'
ORDER BY query_start_time DESC;

-- Column-level access audit for PII columns
SELECT ah.query_start_time, ah.user_name,
       f.value:objectName::TEXT AS table_accessed,
       col.value:columnName::TEXT AS column_accessed
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah,
     LATERAL FLATTEN(input => base_objects_accessed) f,
     LATERAL FLATTEN(input => f.value:columns) col
WHERE ah.query_start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
  AND col.value:columnName::TEXT IN ('SSN', 'EMAIL', 'CREDIT_CARD_NUMBER')
ORDER BY ah.query_start_time DESC;

-- Objects written to (INSERT, UPDATE, COPY INTO)
SELECT query_start_time, user_name,
       f.value:objectName::TEXT AS object_written,
       f.value:objectDomain::TEXT AS object_type
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
     LATERAL FLATTEN(input => objects_modified) f
WHERE query_start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY query_start_time DESC;
```

---

## MASKING_POLICIES

**Purpose:** Lists all dynamic data masking policies defined in your account. Masking policies control how sensitive data is displayed to different roles.

**Data Latency:** 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `POLICY_ID` | NUMBER | Internal unique identifier |
| `POLICY_NAME` | TEXT | Name of the masking policy |
| `POLICY_SCHEMA_ID` | NUMBER | Schema ID |
| `POLICY_SCHEMA` | TEXT | Schema name |
| `POLICY_CATALOG_ID` | NUMBER | Database ID |
| `POLICY_CATALOG` | TEXT | Database name |
| `POLICY_OWNER` | TEXT | Role that owns the policy |
| `POLICY_SIGNATURE` | TEXT | Input argument types |
| `POLICY_RETURN_TYPE` | TEXT | Return data type |
| `POLICY_BODY` | TEXT | SQL expression defining the masking rule |
| `POLICY_EXEMPTIONS` | VARIANT | Roles exempt from masking |
| `COMMENT` | TEXT | Optional comment |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `LAST_ALTERED` | TIMESTAMP_LTZ | Last modification timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp (NULL if active) |

**Sample Query:**
```sql
-- List all active masking policies
SELECT policy_catalog, policy_schema, policy_name,
       policy_owner, policy_return_type, created
FROM SNOWFLAKE.ACCOUNT_USAGE.MASKING_POLICIES
WHERE deleted IS NULL
ORDER BY policy_catalog, policy_schema, policy_name;
```

---

## ROW_ACCESS_POLICIES

**Purpose:** Lists all row access policies in your account. Row access policies filter which rows a user can see based on context (role, user attributes, etc.).

**Data Latency:** 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `POLICY_ID` | NUMBER | Internal unique identifier |
| `POLICY_NAME` | TEXT | Name of the policy |
| `POLICY_SCHEMA_ID` | NUMBER | Schema ID |
| `POLICY_SCHEMA` | TEXT | Schema name |
| `POLICY_CATALOG_ID` | NUMBER | Database ID |
| `POLICY_CATALOG` | TEXT | Database name |
| `POLICY_OWNER` | TEXT | Owning role |
| `POLICY_SIGNATURE` | TEXT | Input argument types |
| `POLICY_RETURN_TYPE` | TEXT | Always BOOLEAN |
| `POLICY_BODY` | TEXT | SQL expression defining row filter logic |
| `COMMENT` | TEXT | Optional comment |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `LAST_ALTERED` | TIMESTAMP_LTZ | Last modification timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp |

---

## POLICY_REFERENCES

**Purpose:** Shows which columns and tables have masking or row access policies applied to them. Provides the mapping between policy objects and their assignments.

**Data Latency:** 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `POLICY_ID` | NUMBER | Policy internal ID |
| `POLICY_NAME` | TEXT | Name of the policy |
| `POLICY_KIND` | TEXT | MASKING_POLICY or ROW_ACCESS_POLICY |
| `POLICY_STATUS` | TEXT | ACTIVE or INACTIVE |
| `REF_DATABASE_NAME` | TEXT | Database of the referencing object |
| `REF_SCHEMA_NAME` | TEXT | Schema of the referencing object |
| `REF_ENTITY_NAME` | TEXT | Table or view name |
| `REF_ENTITY_DOMAIN` | TEXT | Object domain (TABLE, VIEW) |
| `REF_COLUMN_NAME` | TEXT | Column the policy is applied to (NULL for row policies) |
| `REF_ARG_COLUMN_NAMES` | VARIANT | Arguments passed to the policy |

**Sample Query:**
```sql
-- Find all tables/columns with masking policies applied
SELECT ref_database_name, ref_schema_name, ref_entity_name,
       ref_column_name, policy_name, policy_kind
FROM SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES
WHERE policy_kind = 'MASKING_POLICY'
  AND policy_status = 'ACTIVE'
ORDER BY ref_database_name, ref_schema_name, ref_entity_name, ref_column_name;
```

---

## TAG_REFERENCES

**Purpose:** Shows all tag assignments across objects and columns in your account. Tags are the foundation of Snowflake's metadata-based governance (classification, PII identification, cost attribution).

**Data Latency:** 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `TAG_DATABASE` | TEXT | Database where the tag is defined |
| `TAG_SCHEMA` | TEXT | Schema where the tag is defined |
| `TAG_NAME` | TEXT | Name of the tag |
| `TAG_VALUE` | TEXT | Value assigned to the object/column |
| `OBJECT_DATABASE` | TEXT | Database of the tagged object |
| `OBJECT_SCHEMA` | TEXT | Schema of the tagged object |
| `OBJECT_NAME` | TEXT | Name of the tagged object |
| `OBJECT_DOMAIN` | TEXT | Domain: TABLE, COLUMN, SCHEMA, DATABASE, WAREHOUSE, etc. |
| `COLUMN_NAME` | TEXT | Column name (only when tag is on a column) |
| `OBJECT_ID` | NUMBER | Internal object or column ID |
| `TAG_ID` | NUMBER | Internal tag ID |

**Sample Queries:**
```sql
-- Find all objects tagged as PII
SELECT object_database, object_schema, object_name,
       column_name, tag_name, tag_value
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
WHERE tag_name = 'PII'
ORDER BY object_database, object_schema, object_name;

-- Data classification coverage: which tables have no tags
SELECT t.table_catalog, t.table_schema, t.table_name
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
WHERE t.deleted IS NULL
  AND NOT EXISTS (
    SELECT 1
    FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
    WHERE tr.object_database = t.table_catalog
      AND tr.object_schema = t.table_schema
      AND tr.object_name = t.table_name
  )
ORDER BY t.table_catalog, t.table_schema, t.table_name;
```

---

## OBJECT_DEPENDENCIES

**Purpose:** Captures dependencies between Snowflake objects — showing which objects depend on which others. Essential for **impact analysis**, **change management**, and **data lineage**.

**Edition Required:** Enterprise or higher  
**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `REFERENCING_OBJECT_ID` | NUMBER | Internal ID of the dependent object |
| `REFERENCING_OBJECT_NAME` | TEXT | Name of the dependent object |
| `REFERENCING_OBJECT_DOMAIN` | TEXT | Domain of dependent object (TABLE, VIEW, FUNCTION, etc.) |
| `REFERENCING_DATABASE` | TEXT | Database of the dependent object |
| `REFERENCING_SCHEMA` | TEXT | Schema of the dependent object |
| `REFERENCED_OBJECT_ID` | NUMBER | Internal ID of the object being depended upon |
| `REFERENCED_OBJECT_NAME` | TEXT | Name of the referenced (upstream) object |
| `REFERENCED_OBJECT_DOMAIN` | TEXT | Domain of the referenced object |
| `REFERENCED_DATABASE` | TEXT | Database of the referenced object |
| `REFERENCED_SCHEMA` | TEXT | Schema of the referenced object |
| `DEPENDENCY_TYPE` | TEXT | Type of dependency (HARD, SOFT) |

**Sample Queries:**
```sql
-- Find all views that depend on a specific table
SELECT referencing_database, referencing_schema,
       referencing_object_name, referencing_object_domain
FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE referenced_database = 'MY_DB'
  AND referenced_schema = 'MY_SCHEMA'
  AND referenced_object_name = 'MY_TABLE'
ORDER BY referencing_object_name;

-- Find all objects with no dependents (potential orphans)
SELECT t.table_catalog, t.table_schema, t.table_name
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
WHERE t.deleted IS NULL
  AND NOT EXISTS (
    SELECT 1
    FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES d
    WHERE d.referenced_database = t.table_catalog
      AND d.referenced_schema = t.table_schema
      AND d.referenced_object_name = t.table_name
  )
ORDER BY t.table_catalog, t.table_schema, t.table_name;
```

---

## NETWORK_RULE_REFERENCES

**Purpose:** Shows network rules referenced in network policies, providing visibility into IP-based access controls.

---

## Governance Dashboard Queries

### Policy Coverage Scorecard
```sql
-- Summary: how many tables have each type of policy applied
SELECT
  COUNT(DISTINCT t.table_name) AS total_tables,
  COUNT(DISTINCT CASE WHEN pr.policy_kind = 'MASKING_POLICY' THEN t.table_name END) AS tables_with_masking,
  COUNT(DISTINCT CASE WHEN pr.policy_kind = 'ROW_ACCESS_POLICY' THEN t.table_name END) AS tables_with_row_policy,
  COUNT(DISTINCT CASE WHEN tr.tag_name IS NOT NULL THEN t.table_name END) AS tables_with_tags
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES pr
  ON pr.ref_database_name = t.table_catalog
  AND pr.ref_schema_name = t.table_schema
  AND pr.ref_entity_name = t.table_name
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
  ON tr.object_database = t.table_catalog
  AND tr.object_schema = t.table_schema
  AND tr.object_name = t.table_name
WHERE t.deleted IS NULL;
```
