# Security & Access Views

These views support security monitoring, access reviews, and audit dashboards:
authentication, sessions, identities, role/grant structure, and security policies.

---

## `LOGIN_HISTORY`

Authentication attempts — successes and failures. Core source for security
dashboards (failed‑login spikes, MFA usage, IP anomalies).

| Column | Notes |
|--------|-------|
| `EVENT_ID` | Unique login event (join to `SESSIONS.LOGIN_EVENT_ID`). |
| `EVENT_TIMESTAMP` | When. |
| `EVENT_TYPE` | e.g. `LOGIN`. |
| `USER_NAME` | User. |
| `CLIENT_IP` | Source IP. |
| `REPORTED_CLIENT_TYPE` / `REPORTED_CLIENT_VERSION` | Driver/client. |
| `FIRST_AUTHENTICATION_FACTOR` | Primary auth method (password, SAML, OAuth, key‑pair). |
| `SECOND_AUTHENTICATION_FACTOR` | MFA factor if used. |
| `IS_SUCCESS` | `YES` / `NO`. |
| `ERROR_CODE` / `ERROR_MESSAGE` | Failure reason. |
| `CONNECTION` | Connection name (where applicable). |

---

## `SESSIONS`

Sessions and their authentication context. (Also documented under
[query-and-performance.md](query-and-performance.md).) Join `LOGIN_EVENT_ID` to
`LOGIN_HISTORY` to connect a session back to its authentication event.

---

## `USERS`

All users, **including dropped ones** (with `DELETED_ON` set). Source of truth for
identity inventory and offboarding/audit checks.

| Column | Notes |
|--------|-------|
| `USER_ID` | Stable identifier. |
| `NAME` | User name. |
| `LOGIN_NAME` | Login name. |
| `DISPLAY_NAME` / `FIRST_NAME` / `LAST_NAME` / `EMAIL` | Profile. |
| `DISABLED` | Whether disabled. |
| `MUST_CHANGE_PASSWORD` | Password reset flag. |
| `HAS_PASSWORD` / `HAS_RSA_PUBLIC_KEY` | Credential types configured. |
| `DEFAULT_ROLE` / `DEFAULT_WAREHOUSE` / `DEFAULT_NAMESPACE` | Defaults. |
| `EXT_AUTHN_DUO` / `EXT_AUTHN_UID` | External MFA config. |
| `LAST_SUCCESS_LOGIN` | Last successful login (dormant‑account detection). |
| `CREATED_ON` / `DELETED_ON` | Lifecycle. |
| `OWNER` | Owning role. |
| `TYPE` | User type (e.g. `PERSON`, `SERVICE`, `LEGACY_SERVICE`). |

---

## `ROLES`

All roles, including dropped ones.

| Column | Notes |
|--------|-------|
| `ROLE_ID` | Stable identifier. |
| `NAME` | Role name. |
| `ROLE_TYPE` | e.g. `ROLE`, `DATABASE_ROLE`, `INSTANCE_ROLE`. |
| `COMMENT` | Description. |
| `OWNER` | Owning role. |
| `CREATED_ON` / `DELETED_ON` | Lifecycle. |

---

## `GRANTS_TO_USERS`

Role → user assignments. Use it to see which roles a user holds.

| Column | Notes |
|--------|-------|
| `ROLE` | Role granted. |
| `GRANTED_TO` | Grantee type (`USER`). |
| `GRANTEE_NAME` | The user. |
| `GRANTED_BY` | Who granted it. |
| `CREATED_ON` | When granted. |
| `DELETED_ON` | When revoked (NULL if active). |

---

## `GRANTS_TO_ROLES`

Privilege → role and role → role grants. The basis for entitlement reviews and
building an effective‑access graph.

| Column | Notes |
|--------|-------|
| `PRIVILEGE` | Privilege granted (e.g. `SELECT`, `USAGE`, `OWNERSHIP`). |
| `GRANTED_ON` | Object type the privilege is on (e.g. `TABLE`, `SCHEMA`, `ROLE`). |
| `NAME` | Name of the object/role granted on. |
| `TABLE_CATALOG` / `TABLE_SCHEMA` | Qualifier for object grants. |
| `GRANTED_TO` | Grantee type (`ROLE`). |
| `GRANTEE_NAME` | Role receiving the grant. |
| `GRANT_OPTION` | Whether the grantee can re‑grant. |
| `GRANTED_BY` | Granting role. |
| `CREATED_ON` / `DELETED_ON` | Lifecycle. |

---

## `GRANTS_TO_SHARE`

Privileges granted to outbound shares — useful for data‑sharing security reviews.

| Column | Notes |
|--------|-------|
| `PRIVILEGE` | Privilege. |
| `GRANTED_ON` | Object type. |
| `NAME` | Object name. |
| `SHARE_NAME` | The share. |
| `CREATED_ON` / `DELETED_ON` | Lifecycle. |

---

## Security policy views

Definitions of configured policies. Pair with `POLICY_REFERENCES`
([governance-and-data-quality.md](governance-and-data-quality.md)) to see where
each policy is applied.

| View | What it lists |
|------|---------------|
| `MASKING_POLICIES` | Column data‑masking policy definitions. |
| `ROW_ACCESS_POLICIES` | Row access policy definitions. |
| `PASSWORD_POLICIES` | Password complexity/expiry policies. |
| `SESSION_POLICIES` | Session timeout/idle policies. |
| `AUTHENTICATION_POLICIES` | Allowed authentication methods/clients. |
| `NETWORK_POLICIES` | Allowed/blocked IP ranges. |
| `NETWORK_RULES` | Network rule definitions used by policies. |
| `DELEGATED_AUTHORIZATIONS` | OAuth/delegated authorization grants. |

Typical columns across policy definition views: `POLICY_ID`, `POLICY_NAME`,
`POLICY_SCHEMA`, `POLICY_CATALOG`, `POLICY_BODY` / signature, `OWNER`,
`CREATED`, `LAST_ALTERED`, `DELETED`, `COMMENT`.

---

## Common security dashboard checks

- **Failed login spikes / brute force:** `LOGIN_HISTORY` where `IS_SUCCESS='NO'`,
  grouped by `USER_NAME` / `CLIENT_IP`.
- **Dormant or stale accounts:** `USERS` where `LAST_SUCCESS_LOGIN` is old and
  `DELETED_ON IS NULL`.
- **Over‑privileged roles:** `GRANTS_TO_ROLES` filtered to high‑risk privileges
  (`OWNERSHIP`, `ACCOUNTADMIN` membership, `MANAGE GRANTS`).
- **Weak authentication:** `LOGIN_HISTORY` / `SESSIONS` for password‑only logins
  without a second factor.
- **Policy coverage gaps:** policies in `MASKING_POLICIES` /
  `ROW_ACCESS_POLICIES` not present in `POLICY_REFERENCES`.
