# User & Role Management Views

These views provide comprehensive information about users, roles, and privilege grants in your Snowflake account. They are essential for **access control auditing**, **compliance reporting**, and **security governance**.

---

## USERS

**Purpose:** Lists all users defined in your Snowflake account, including dropped users. Contains profile information, authentication settings, and account status.

**Data Latency:** Up to 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `USER_ID` | NUMBER | Internal unique identifier |
| `NAME` | TEXT | Snowflake login name (username) |
| `CREATED_ON` | TIMESTAMP_LTZ | When the user account was created |
| `DELETED_ON` | TIMESTAMP_LTZ | When the user was dropped (NULL if active) |
| `LOGIN_NAME` | TEXT | Login name (may differ from NAME) |
| `DISPLAY_NAME` | TEXT | Display name for UI |
| `FIRST_NAME` | TEXT | User's first name |
| `LAST_NAME` | TEXT | User's last name |
| `EMAIL` | TEXT | Email address |
| `MUST_CHANGE_PASSWORD` | BOOLEAN | Whether the user must change password on next login |
| `HAS_PASSWORD` | BOOLEAN | Whether a password is set |
| `COMMENT` | TEXT | Optional comment |
| `DISABLED` | BOOLEAN | Whether the user account is disabled |
| `SNOWFLAKE_LOCK` | BOOLEAN | Whether Snowflake has locked the account |
| `DEFAULT_WAREHOUSE` | TEXT | Default warehouse for the user |
| `DEFAULT_NAMESPACE` | TEXT | Default namespace (database.schema) |
| `DEFAULT_ROLE` | TEXT | Default role for the user |
| `DEFAULT_SECONDARY_ROLES` | TEXT | Secondary roles setting |
| `EXT_AUTHN_DUO` | BOOLEAN | Whether Duo MFA is enabled |
| `EXT_AUTHN_UID` | TEXT | External authentication UID |
| `BYPASS_MFA_UNTIL` | TIMESTAMP_LTZ | MFA bypass expiry |
| `LAST_SUCCESS_LOGIN` | TIMESTAMP_LTZ | Timestamp of last successful login |
| `EXPIRES_AT` | TIMESTAMP_LTZ | When the user account expires |
| `LOCKED_UNTIL_TIME` | TIMESTAMP_LTZ | Temporary lock expiry |
| `HAS_RSA_PUBLIC_KEY` | BOOLEAN | Whether an RSA public key is configured |
| `PASSWORD_LAST_SET_TIME` | TIMESTAMP_LTZ | When password was last changed |
| `OWNER` | TEXT | Role that owns this user |

**Sample Queries:**
```sql
-- Find all active users with their default roles and last login
SELECT name, email, default_role, last_success_login, disabled
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
ORDER BY last_success_login DESC NULLS LAST;

-- Find users who have never logged in
SELECT name, email, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND last_success_login IS NULL
ORDER BY created_on;

-- Find disabled user accounts
SELECT name, email, disabled
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND disabled = TRUE;
```

---

## ROLES

**Purpose:** Lists all roles defined in your Snowflake account, including dropped roles.

**Data Latency:** Up to 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `ROLE_ID` | NUMBER | Internal unique identifier |
| `NAME` | TEXT | Role name |
| `OWNER` | TEXT | Role that owns this role |
| `COMMENT` | TEXT | Optional comment |
| `CREATED_ON` | TIMESTAMP_LTZ | When the role was created |
| `DELETED_ON` | TIMESTAMP_LTZ | When the role was dropped (NULL if active) |

**Sample Query:**
```sql
-- List all active roles
SELECT name, owner, comment, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES
WHERE deleted_on IS NULL
ORDER BY name;
```

---

## GRANTS_TO_ROLES

**Purpose:** Lists all **object-level and account-level privileges** granted to roles. This is the primary view for auditing what access each role has.

**Data Latency:** Up to 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `CREATED_ON` | TIMESTAMP_LTZ | When the grant was made |
| `MODIFIED_ON` | TIMESTAMP_LTZ | When the grant was last modified |
| `PRIVILEGE` | TEXT | Privilege granted (e.g., SELECT, USAGE, MODIFY, MONITOR) |
| `GRANTED_ON` | TEXT | Object type the privilege applies to (e.g., TABLE, DATABASE, WAREHOUSE) |
| `NAME` | TEXT | Name of the object the privilege is on |
| `TABLE_CATALOG` | TEXT | Database name (for table-level grants) |
| `TABLE_SCHEMA` | TEXT | Schema name (for table-level grants) |
| `GRANTED_TO` | TEXT | Always "ROLE" |
| `GRANTEE_NAME` | TEXT | Name of the role receiving the privilege |
| `GRANT_OPTION` | BOOLEAN | Whether the role can grant this privilege further |
| `GRANTED_BY` | TEXT | Role that issued the grant |
| `DELETED_ON` | TIMESTAMP_LTZ | When the grant was revoked (NULL if still active) |

**Sample Queries:**
```sql
-- Find all active privileges granted to a specific role
SELECT privilege, granted_on, name, table_catalog, table_schema, grant_option
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE grantee_name = 'MY_ANALYST_ROLE'
  AND deleted_on IS NULL
ORDER BY granted_on;

-- Audit who has SELECT on production tables
SELECT grantee_name, table_catalog, table_schema, name AS table_name, granted_by
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE privilege = 'SELECT'
  AND granted_on = 'TABLE'
  AND deleted_on IS NULL
  AND table_catalog = 'PROD_DB'
ORDER BY grantee_name, table_schema, table_name;

-- Find roles with ACCOUNTADMIN-like broad access
SELECT grantee_name, COUNT(*) AS privilege_count
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE deleted_on IS NULL
GROUP BY grantee_name
ORDER BY privilege_count DESC;
```

---

## GRANTS_TO_USERS

**Purpose:** Lists all **role grants to users** — showing which users have been assigned which roles.

**Data Latency:** Up to 2 hours  
**Retention:** 1 year

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `CREATED_ON` | TIMESTAMP_LTZ | When the grant was made |
| `ROLE` | TEXT | Name of the role granted to the user |
| `GRANTED_TO` | TEXT | Always "USER" |
| `GRANTEE_NAME` | TEXT | Username receiving the role |
| `GRANTED_BY` | TEXT | Role that issued the grant |
| `DELETED_ON` | TIMESTAMP_LTZ | When the grant was revoked (NULL if active) |

**Sample Queries:**
```sql
-- Find all roles assigned to a specific user
SELECT role, granted_by, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE grantee_name = 'JOHN.DOE'
  AND deleted_on IS NULL
ORDER BY role;

-- Find all users with the ACCOUNTADMIN role
SELECT grantee_name, granted_by, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS
WHERE role = 'ACCOUNTADMIN'
  AND deleted_on IS NULL;

-- Generate a full user-to-role mapping
SELECT u.name AS username, u.email, g.role
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS u
JOIN SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_USERS g
  ON u.name = g.grantee_name
WHERE u.deleted_on IS NULL
  AND g.deleted_on IS NULL
ORDER BY u.name, g.role;
```

---

## ROLE_GRANTS

**Purpose:** Lists all **role-to-role grants** (i.e., which roles are granted to other roles, forming role hierarchies).

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `CREATED_ON` | TIMESTAMP_LTZ | When the grant was made |
| `ROLE` | TEXT | Name of the child role being granted |
| `GRANTED_TO` | TEXT | Always "ROLE" |
| `GRANTEE_NAME` | TEXT | Name of the parent role receiving the child role |
| `GRANTED_BY` | TEXT | Role that issued the grant |
| `DELETED_ON` | TIMESTAMP_LTZ | When revoked (NULL if active) |

**Sample Query:**
```sql
-- Visualize role hierarchy
SELECT grantee_name AS parent_role, role AS child_role, granted_by
FROM SNOWFLAKE.ACCOUNT_USAGE.ROLE_GRANTS
WHERE deleted_on IS NULL
ORDER BY parent_role, child_role;
```

---

## Governance Use Cases

### 1. Unused Role Detection
```sql
-- Find roles that haven't been used in the last 90 days
SELECT r.name AS role_name
FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES r
WHERE r.deleted_on IS NULL
  AND r.name NOT IN (
    SELECT DISTINCT role_name
    FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
    WHERE start_time >= DATEADD('day', -90, CURRENT_TIMESTAMP())
  )
ORDER BY r.name;
```

### 2. Privilege Drift Detection
```sql
-- Find recently added high-risk grants (last 7 days)
SELECT grantee_name, privilege, granted_on, name, granted_by, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE deleted_on IS NULL
  AND privilege IN ('OWNERSHIP', 'ALL', 'MANAGE GRANTS')
  AND created_on >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY created_on DESC;
```

### 3. Users with No Recent Login
```sql
-- Active users who have not logged in for 60+ days
SELECT name, email, last_success_login, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND disabled = FALSE
  AND (last_success_login IS NULL
       OR last_success_login < DATEADD('day', -60, CURRENT_TIMESTAMP()))
ORDER BY last_success_login NULLS FIRST;
```
