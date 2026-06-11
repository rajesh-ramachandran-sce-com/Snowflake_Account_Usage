# Replication & Sharing Views

These views track **Snowflake's data replication** (for high availability, disaster recovery, and multi-region deployments) and **Secure Data Sharing** (for sharing data with other Snowflake accounts). Both are billable features and these views enable cost tracking, operational monitoring, and governance.

---

## REPLICATION_GROUP_USAGE_HISTORY

**Purpose:** Tracks the compute credits and data transfer bytes consumed by **replication group** operations (failover groups and replication groups) over time.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the measurement window |
| `END_TIME` | TIMESTAMP_LTZ | End of the measurement window |
| `REPLICATION_GROUP_NAME` | TEXT | Name of the replication or failover group |
| `REPLICATION_GROUP_ID` | NUMBER | Internal group identifier |
| `PHASE_TYPE` | TEXT | Type of replication phase (e.g., PRIMARY, SECONDARY) |
| `CREDITS_USED` | NUMBER | Compute credits consumed for replication |
| `BYTES_TRANSFERRED` | NUMBER | Data bytes transferred during replication |

**Sample Queries:**
```sql
-- Replication cost by group (last 30 days)
SELECT replication_group_name,
       SUM(credits_used) AS total_credits,
       SUM(bytes_transferred) / 1024 / 1024 / 1024 AS total_gb_transferred
FROM SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUP_USAGE_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY total_credits DESC;

-- Daily replication cost trend
SELECT DATE(start_time) AS usage_date,
       replication_group_name,
       SUM(credits_used) AS daily_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUP_USAGE_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 1 DESC, 3 DESC;
```

---

## DATABASE_REPLICATION_USAGE_HISTORY

**Purpose:** Provides detailed replication usage statistics at the **individual database** level. Useful when multiple databases are part of a replication group and you need per-database cost visibility.

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the measurement window |
| `END_TIME` | TIMESTAMP_LTZ | End of the measurement window |
| `DATABASE_NAME` | TEXT | Name of the database being replicated |
| `DATABASE_ID` | NUMBER | Internal database identifier |
| `CREDITS_USED` | NUMBER | Compute credits used for this database's replication |
| `BYTES_TRANSFERRED` | NUMBER | Bytes replicated for this database |

**Sample Query:**
```sql
-- Per-database replication cost
SELECT database_name,
       SUM(credits_used) AS total_credits,
       SUM(bytes_transferred) / 1024 / 1024 / 1024 AS gb_transferred
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_REPLICATION_USAGE_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY total_credits DESC;
```

---

## REPLICATION_USAGE_HISTORY

**Purpose:** Historical record of replication operations at a high level (older/legacy view for database-level replication before Replication Groups were introduced).

**Data Latency:** 3 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the replication operation |
| `END_TIME` | TIMESTAMP_LTZ | End of the replication operation |
| `DATABASE_NAME` | TEXT | Database being replicated |
| `CREDITS_USED` | NUMBER | Credits consumed |
| `BYTES_TRANSFERRED` | NUMBER | Bytes transferred |
| `REPLICATION_ROLE` | TEXT | PRIMARY or SECONDARY |

---

## SHARES

**Purpose:** Lists all database shares in your Snowflake account — both **outbound shares** (data you share with others) and **inbound shares** (data shared with you from other accounts).

**Data Latency:** 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `SHARE_ID` | NUMBER | Internal share identifier |
| `SHARE_NAME` | TEXT | Name of the share |
| `SHARE_OWNER` | TEXT | Role that owns the share |
| `CREATED_ON` | TIMESTAMP_LTZ | When the share was created |
| `KIND` | TEXT | OUTBOUND (you are sharing) or INBOUND (shared with you) |
| `OWNER_ACCOUNT` | TEXT | Account that owns the share |
| `LISTING_GLOBAL_NAME` | TEXT | Global name if listed on Snowflake Marketplace |
| `COMMENT` | TEXT | Optional comment |
| `DELETED_ON` | TIMESTAMP_LTZ | When the share was dropped (NULL if active) |

**Sample Queries:**
```sql
-- List all active shares
SELECT share_name, kind, share_owner, created_on, comment
FROM SNOWFLAKE.ACCOUNT_USAGE.SHARES
WHERE deleted_on IS NULL
ORDER BY kind, share_name;

-- Audit who the outbound shares are shared with (using SHOW SHARES)
-- For detailed consumer account info, combine with SHOW SHARES:
SHOW SHARES;
```

---

## DATA_TRANSFER_HISTORY

**Purpose:** Records all data transfer operations between Snowflake and external locations (such as uploads to/downloads from external stages in AWS S3, Azure Blob Storage, or Google Cloud Storage). Tracks **transfer credits and bytes** for billing visibility.

**Data Latency:** 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `START_TIME` | TIMESTAMP_LTZ | Start of the transfer operation |
| `END_TIME` | TIMESTAMP_LTZ | End of the transfer operation |
| `SOURCE_CLOUD` | TEXT | Source cloud provider (AWS, AZURE, GCP) |
| `SOURCE_REGION` | TEXT | Source cloud region |
| `TARGET_CLOUD` | TEXT | Target cloud provider |
| `TARGET_REGION` | TEXT | Target cloud region |
| `BYTES_TRANSFERRED` | NUMBER | Bytes transferred in this operation |
| `TRANSFER_TYPE` | TEXT | UPLOAD, DOWNLOAD, or COPY |

> **Note:** Data transfer fees apply when data moves between cloud providers or regions. Same-region transfers within the same cloud provider are typically free.

**Sample Queries:**
```sql
-- Data transfer volume by source and target (last 30 days)
SELECT source_cloud, source_region,
       target_cloud, target_region,
       transfer_type,
       COUNT(*) AS operations,
       SUM(bytes_transferred) / 1024 / 1024 / 1024 AS total_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.DATA_TRANSFER_HISTORY
WHERE start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY 1, 2, 3, 4, 5
ORDER BY total_gb DESC;

-- Cross-region transfers (potential cost alert)
SELECT start_time, source_cloud, source_region, target_cloud, target_region,
       bytes_transferred / 1024 / 1024 AS mb_transferred
FROM SNOWFLAKE.ACCOUNT_USAGE.DATA_TRANSFER_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND (source_cloud != target_cloud OR source_region != target_region)
ORDER BY bytes_transferred DESC;
```

---

## EXTERNAL_ACCESS_HISTORY

**Purpose:** Records data access events from external locations (e.g., queries that accessed external tables or stages), supporting governance and lineage for externally-sourced data.

---

## Combined Replication + Sharing Cost Dashboard

```sql
-- Total replication and data transfer cost this month
SELECT 'Replication Groups' AS service,
       SUM(credits_used) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.REPLICATION_GROUP_USAGE_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP())

UNION ALL

SELECT 'Database Replication (Legacy)' AS service,
       SUM(credits_used) AS credits
FROM SNOWFLAKE.ACCOUNT_USAGE.DATABASE_REPLICATION_USAGE_HISTORY
WHERE start_time >= DATE_TRUNC('month', CURRENT_TIMESTAMP())

ORDER BY credits DESC;
```

---

## Key Concepts

### Replication Groups vs. Failover Groups
| Feature | Replication Group | Failover Group |
|---------|-----------------|----------------|
| **Purpose** | Read-only replicas for analytics in other regions | DR/HA with primary/secondary failover capability |
| **Failover support** | No | Yes |
| **Use case** | Global analytics distribution | Business continuity |

### Secure Data Sharing
- **No data copying** — consumers query your data directly from your Snowflake storage
- **No egress costs** for the data provider when within the same cloud/region
- Supported objects: Tables, External tables, Secure views, Secure UDFs
- Monitor active consumers via: `SHOW GRANTS OF SHARE <share_name>;`
