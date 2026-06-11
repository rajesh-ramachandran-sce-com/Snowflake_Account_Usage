# Object Metadata Views

These views expose structural and descriptive metadata about all objects in your Snowflake account. Unlike `INFORMATION_SCHEMA`, these views include records for **dropped (deleted) objects** and retain history for up to 1 year.

---

## DATABASES

**Purpose:** Lists all databases in your Snowflake account, including dropped databases.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `DATABASE_ID` | NUMBER | Internal unique identifier |
| `DATABASE_NAME` | TEXT | Name of the database |
| `DATABASE_OWNER` | TEXT | Role that owns the database |
| `IS_TRANSIENT` | TEXT | Whether the database is transient (YES/NO) |
| `COMMENT` | TEXT | Optional comment/description |
| `CREATED` | TIMESTAMP_LTZ | When the database was created |
| `LAST_ALTERED` | TIMESTAMP_LTZ | When the database was last altered |
| `DELETED` | TIMESTAMP_LTZ | When the database was dropped (NULL if active) |
| `RETENTION_TIME` | NUMBER | Time Travel retention period in days |

**Sample Query:**
```sql
-- List all active databases
SELECT database_name, database_owner, created, retention_time
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASES
WHERE deleted IS NULL
ORDER BY database_name;
```

---

## SCHEMAS

**Purpose:** Lists all schemas in your account, across all databases, including dropped schemas.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `SCHEMA_ID` | NUMBER | Internal unique identifier |
| `SCHEMA_NAME` | TEXT | Name of the schema |
| `CATALOG_ID` | NUMBER | ID of the parent database |
| `CATALOG_NAME` | TEXT | Name of the parent database |
| `SCHEMA_OWNER` | TEXT | Role that owns the schema |
| `IS_TRANSIENT` | TEXT | Whether the schema is transient |
| `IS_MANAGED_ACCESS` | TEXT | Whether the schema uses managed access (YES/NO) |
| `RETENTION_TIME` | NUMBER | Time Travel retention period |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `LAST_ALTERED` | TIMESTAMP_LTZ | Last alteration timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp (NULL if active) |
| `COMMENT` | TEXT | Optional comment |

**Sample Query:**
```sql
-- List schemas not yet deleted, grouped by database
SELECT catalog_name AS database_name, schema_name, schema_owner, created
FROM SNOWFLAKE.ACCOUNT_USAGE.SCHEMAS
WHERE deleted IS NULL
ORDER BY catalog_name, schema_name;
```

---

## TABLES

**Purpose:** Lists all tables and external tables in your account, including dropped tables. Includes metadata like row counts, storage bytes, and clustering information.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `TABLE_ID` | NUMBER | Internal unique identifier |
| `TABLE_NAME` | TEXT | Name of the table |
| `TABLE_SCHEMA_ID` | NUMBER | ID of the schema |
| `TABLE_SCHEMA` | TEXT | Schema name |
| `TABLE_CATALOG_ID` | NUMBER | ID of the database |
| `TABLE_CATALOG` | TEXT | Database name |
| `TABLE_OWNER` | TEXT | Role that owns the table |
| `TABLE_TYPE` | TEXT | Type: BASE TABLE, EXTERNAL, TEMPORARY, TRANSIENT |
| `IS_TRANSIENT` | TEXT | Whether the table is transient |
| `CLUSTERING_KEY` | TEXT | Clustering key definition (if any) |
| `ROW_COUNT` | NUMBER | Approximate number of rows |
| `BYTES` | NUMBER | Total size in bytes |
| `RETENTION_TIME` | NUMBER | Time Travel retention period (days) |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `LAST_ALTERED` | TIMESTAMP_LTZ | Last alteration timestamp |
| `LAST_DDL` | TIMESTAMP_LTZ | Last DDL change timestamp |
| `LAST_DDL_BY` | TEXT | User who made the last DDL change |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp (NULL if active) |
| `COMMENT` | TEXT | Optional comment |
| `AUTO_CLUSTERING_ON` | BOOLEAN | Whether automatic clustering is enabled |

**Sample Query:**
```sql
-- Find the largest tables in the account
SELECT table_catalog, table_schema, table_name,
       row_count,
       ROUND(bytes / 1024 / 1024 / 1024, 2) AS size_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES
WHERE deleted IS NULL
ORDER BY bytes DESC NULLS LAST
LIMIT 20;
```

---

## VIEWS

**Purpose:** Lists all views (regular and materialized) in your account, including dropped views.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `VIEW_ID` | NUMBER | Internal unique identifier |
| `VIEW_NAME` | TEXT | Name of the view |
| `VIEW_SCHEMA_ID` | NUMBER | ID of the schema |
| `VIEW_SCHEMA` | TEXT | Schema name |
| `VIEW_CATALOG_ID` | NUMBER | ID of the database |
| `VIEW_CATALOG` | TEXT | Database name |
| `VIEW_OWNER` | TEXT | Role that owns the view |
| `VIEW_DEFINITION` | TEXT | The SQL text of the view definition |
| `IS_SECURE` | TEXT | Whether the view is secure (YES/NO) |
| `IS_UPDATABLE` | TEXT | Whether the view is updatable |
| `INSERTABLE_INTO` | TEXT | Whether rows can be inserted |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `LAST_ALTERED` | TIMESTAMP_LTZ | Last alteration timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp (NULL if active) |
| `COMMENT` | TEXT | Optional comment |

**Sample Query:**
```sql
-- Find all secure views
SELECT view_catalog, view_schema, view_name, view_owner
FROM SNOWFLAKE.ACCOUNT_USAGE.VIEWS
WHERE is_secure = 'YES'
  AND deleted IS NULL
ORDER BY view_catalog, view_schema, view_name;
```

---

## COLUMNS

**Purpose:** Lists all columns for all tables and views in your account, including columns of dropped objects.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `COLUMN_ID` | NUMBER | Internal unique identifier |
| `COLUMN_NAME` | TEXT | Name of the column |
| `TABLE_ID` | NUMBER | ID of the parent table |
| `TABLE_NAME` | TEXT | Name of the parent table |
| `TABLE_SCHEMA_ID` | NUMBER | Schema ID |
| `TABLE_SCHEMA` | TEXT | Schema name |
| `TABLE_CATALOG_ID` | NUMBER | Database ID |
| `TABLE_CATALOG` | TEXT | Database name |
| `ORDINAL_POSITION` | NUMBER | Position of the column in the table |
| `COLUMN_DEFAULT` | TEXT | Default expression for the column |
| `IS_NULLABLE` | TEXT | Whether the column can contain NULL |
| `DATA_TYPE` | TEXT | Data type of the column |
| `CHARACTER_MAXIMUM_LENGTH` | NUMBER | Max length for character types |
| `NUMERIC_PRECISION` | NUMBER | Precision for numeric types |
| `NUMERIC_SCALE` | NUMBER | Scale for numeric types |
| `COMMENT` | TEXT | Optional column-level comment |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp (NULL if active) |

**Sample Query:**
```sql
-- Find all columns with PII-related names
SELECT table_catalog, table_schema, table_name, column_name, data_type
FROM SNOWFLAKE.ACCOUNT_USAGE.COLUMNS
WHERE deleted IS NULL
  AND LOWER(column_name) REGEXP '.*(email|ssn|phone|dob|birth|address|salary|credit_card).*'
ORDER BY table_catalog, table_schema, table_name;
```

---

## STAGES

**Purpose:** Lists all internal and external stages in your account.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `STAGE_ID` | NUMBER | Internal unique identifier |
| `STAGE_NAME` | TEXT | Name of the stage |
| `STAGE_SCHEMA_ID` | NUMBER | Schema ID |
| `STAGE_SCHEMA` | TEXT | Schema name |
| `STAGE_CATALOG_ID` | NUMBER | Database ID |
| `STAGE_CATALOG` | TEXT | Database name |
| `STAGE_URL` | TEXT | URL of external stage (NULL for internal) |
| `STAGE_REGION` | TEXT | Cloud region of the stage |
| `STAGE_TYPE` | TEXT | Type: Internal Named, External Named, User, Table |
| `STAGE_OWNER` | TEXT | Role that owns the stage |
| `COMMENT` | TEXT | Optional comment |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `LAST_ALTERED` | TIMESTAMP_LTZ | Last alteration timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp (NULL if active) |

---

## FILE_FORMATS

**Purpose:** Lists all file formats defined in your account.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `FILE_FORMAT_ID` | NUMBER | Internal unique identifier |
| `FILE_FORMAT_NAME` | TEXT | Name of the file format |
| `FILE_FORMAT_SCHEMA` | TEXT | Schema name |
| `FILE_FORMAT_CATALOG` | TEXT | Database name |
| `FILE_FORMAT_TYPE` | TEXT | Format type: CSV, JSON, AVRO, ORC, PARQUET, XML |
| `FILE_FORMAT_OWNER` | TEXT | Owning role |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp |

---

## PIPES

**Purpose:** Lists all Snowpipe objects in your account, including dropped pipes.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `PIPE_ID` | NUMBER | Internal unique identifier |
| `PIPE_NAME` | TEXT | Name of the pipe |
| `PIPE_SCHEMA_ID` | NUMBER | Schema ID |
| `PIPE_SCHEMA` | TEXT | Schema name |
| `PIPE_CATALOG_ID` | NUMBER | Database ID |
| `PIPE_CATALOG` | TEXT | Database name |
| `IS_AUTOINGEST_ENABLED` | BOOLEAN | Whether auto-ingest is enabled |
| `DEFINITION` | TEXT | COPY INTO SQL of the pipe |
| `PIPE_OWNER` | TEXT | Owning role |
| `COMMENT` | TEXT | Optional comment |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `LAST_ALTERED` | TIMESTAMP_LTZ | Last alteration timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp |

---

## FUNCTIONS

**Purpose:** Lists all user-defined functions (UDFs) and external functions in your account.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `FUNCTION_ID` | NUMBER | Internal unique identifier |
| `FUNCTION_NAME` | TEXT | Name of the function |
| `FUNCTION_SCHEMA_ID` | NUMBER | Schema ID |
| `FUNCTION_SCHEMA` | TEXT | Schema name |
| `FUNCTION_CATALOG_ID` | NUMBER | Database ID |
| `FUNCTION_CATALOG` | TEXT | Database name |
| `FUNCTION_OWNER` | TEXT | Owning role |
| `FUNCTION_LANGUAGE` | TEXT | Implementation language (SQL, JavaScript, Python, etc.) |
| `FUNCTION_DEFINITION` | TEXT | Function body |
| `DATA_TYPE` | TEXT | Return type |
| `IS_SECURE` | TEXT | Whether the function is secure |
| `IS_EXTERNAL` | TEXT | Whether it is an external function |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp |

---

## PROCEDURES

**Purpose:** Lists all stored procedures in your account.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `PROCEDURE_ID` | NUMBER | Internal unique identifier |
| `PROCEDURE_NAME` | TEXT | Name of the stored procedure |
| `PROCEDURE_SCHEMA` | TEXT | Schema name |
| `PROCEDURE_CATALOG` | TEXT | Database name |
| `PROCEDURE_OWNER` | TEXT | Owning role |
| `PROCEDURE_LANGUAGE` | TEXT | Language: JavaScript, SQL, Python, Java, Scala |
| `PROCEDURE_DEFINITION` | TEXT | Procedure body |
| `ARGUMENT_SIGNATURE` | TEXT | Argument types |
| `DATA_TYPE` | TEXT | Return type |
| `IS_SECURE` | TEXT | Whether the procedure is secure |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp |

---

## SEQUENCES

**Purpose:** Lists all sequences in your account.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `SEQUENCE_ID` | NUMBER | Internal unique identifier |
| `SEQUENCE_NAME` | TEXT | Name of the sequence |
| `SEQUENCE_SCHEMA` | TEXT | Schema name |
| `SEQUENCE_CATALOG` | TEXT | Database name |
| `SEQUENCE_OWNER` | TEXT | Owning role |
| `START_VALUE` | NUMBER | Starting value |
| `INCREMENT` | NUMBER | Increment amount |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp |

---

## TASKS

**Purpose:** Lists all task objects (scheduled SQL jobs) in your account.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `TASK_ID` | NUMBER | Internal unique identifier |
| `TASK_NAME` | TEXT | Name of the task |
| `TASK_SCHEMA` | TEXT | Schema name |
| `TASK_CATALOG` | TEXT | Database name |
| `TASK_OWNER` | TEXT | Owning role |
| `DEFINITION` | TEXT | SQL statement the task executes |
| `SCHEDULE` | TEXT | CRON or interval schedule |
| `WAREHOUSE` | TEXT | Warehouse used by the task (NULL for serverless) |
| `STATE` | TEXT | Task state: started, suspended |
| `CONDITION` | TEXT | WHEN condition for conditional execution |
| `ALLOW_OVERLAPPING_EXECUTION` | BOOLEAN | Whether concurrent runs are allowed |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `LAST_ALTERED` | TIMESTAMP_LTZ | Last alteration timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp |

---

## WAREHOUSES

**Purpose:** Lists all virtual warehouses in your account.

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `WAREHOUSE_ID` | NUMBER | Internal unique identifier |
| `WAREHOUSE_NAME` | TEXT | Name of the warehouse |
| `WAREHOUSE_SIZE` | TEXT | Size: X-Small, Small, Medium, Large, X-Large, etc. |
| `WAREHOUSE_TYPE` | TEXT | Type: STANDARD, SNOWPARK-OPTIMIZED |
| `AUTO_SUSPEND` | NUMBER | Seconds of inactivity before auto-suspend |
| `AUTO_RESUME` | BOOLEAN | Whether the warehouse auto-resumes |
| `MAX_CLUSTER_COUNT` | NUMBER | Maximum clusters (multi-cluster) |
| `MIN_CLUSTER_COUNT` | NUMBER | Minimum clusters |
| `SCALING_POLICY` | TEXT | Multi-cluster scaling policy |
| `OWNER` | TEXT | Owning role |
| `COMMENT` | TEXT | Optional comment |
| `CREATED` | TIMESTAMP_LTZ | Creation timestamp |
| `DELETED` | TIMESTAMP_LTZ | Drop timestamp |
| `ENABLE_QUERY_ACCELERATION` | BOOLEAN | Whether Query Acceleration Service is enabled |

**Sample Query:**
```sql
-- List all active warehouses with their sizes
SELECT warehouse_name, warehouse_size, warehouse_type, auto_suspend, auto_resume, owner
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSES
WHERE deleted IS NULL
ORDER BY warehouse_name;
```
