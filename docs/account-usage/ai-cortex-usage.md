# Snowflake AI / Cortex Usage — Application-Ready Knowledge Base

> **Scope:** Snowflake **`ACCOUNT_USAGE`** (and related `ORGANIZATION_USAGE` /
> `READER_ACCOUNT_USAGE`) views relevant to **AI, Cortex, Cortex AISQL, Cortex AI
> functions, Cortex Code, Cortex Analyst, token/credit metering, and AI usage
> attribution**. Built to be consumed by Copilot-like platforms, AI coworker
> agents, app builders, FinOps, governance teams, and Snowflake admins.
>
> **Primary source:** Snowflake's official Account Usage reference
> (`https://docs.snowflake.com/en/sql-reference/account-usage`) and the per-view
> reference pages it links to.

---

## ⚠️ Accuracy and freshness contract (read first)

Snowflake adds, renames, and revises AI/Cortex Account Usage views **very
frequently** — faster than almost any other area of the platform. This document
distinguishes three confidence levels, and you (human or agent) must respect them:

- **[DOCUMENTED]** — corroborated by Snowflake's official documentation.
- **[BEST-EFFORT]** — consistent with Snowflake's documented patterns and how
  sibling views behave, but **not individually verified** for this exact view at
  time of writing. Treat as a strong hypothesis, not gospel.
- **[INTERPRETATION]** — a recommendation or design opinion, not a Snowflake fact.

**Golden rule for AI agents:** Before generating production SQL or asserting that a
column exists, prefer to confirm against the live account:

```sql
SHOW VIEWS IN SCHEMA SNOWFLAKE.ACCOUNT_USAGE;
DESC VIEW SNOWFLAKE.ACCOUNT_USAGE.<view_name>;
```

If a column or view named in this document is not present in `DESC`/`SHOW`
output, **trust the account, not this document.** Never invent column names.

---

# 1. Executive Summary

**What `ACCOUNT_USAGE` is.** `ACCOUNT_USAGE` is a read-only schema in the
system-shared `SNOWFLAKE` database. It exposes historical usage, metering,
metadata, and activity for the **current account**, including dropped objects,
with retention up to **365 days** and latency typically **minutes to a few
hours** (not real time). See [`concepts.md`](concepts.md).

**Why it matters for AI/Cortex.** Snowflake Cortex (LLM functions, AISQL,
Cortex Analyst, Cortex Search, Cortex Agents, and Cortex Code) is billed in
**credits derived from token (or page/request) consumption**, separate from
virtual-warehouse compute. To monitor, attribute, govern, and charge back AI
spend you must combine:

1. **Detailed AI usage views** (per-function / per-query / per-model token and
   credit detail) — the analytical source of truth for *what was used*.
2. **Metering views** (`METERING_HISTORY`, `METERING_DAILY_HISTORY`) — the
   **billed** source of truth for *what you pay*, where AI shows up under service
   types such as `AI_SERVICES` and the `CORTEX_CODE_*` family.
3. **Correlation views** (`QUERY_HISTORY`, `QUERY_ATTRIBUTION_HISTORY`,
   `WAREHOUSE_METERING_HISTORY`) — to tie AI usage to users, roles, warehouses,
   query tags, and applications.

**Which views matter most (today).**

| Need | Go-to view |
|------|-----------|
| Hourly AI function token + credit detail | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` *(current)* |
| AISQL (AI SQL functions) detail | `CORTEX_AISQL_USAGE_HISTORY` *(current)* |
| Legacy per-function aggregate | `CORTEX_FUNCTIONS_USAGE_HISTORY` *(legacy/superseded)* |
| Legacy per-query AI detail | `CORTEX_FUNCTIONS_QUERY_USAGE_HISTORY` *(legacy/superseded)* |
| Cortex Code (CLI / Snowsight) | `CORTEX_CODE_CLI_USAGE_HISTORY`, `CORTEX_CODE_SNOWSIGHT_USAGE_HISTORY` |
| **Billed** AI credits | `METERING_DAILY_HISTORY` / `METERING_HISTORY` (`SERVICE_TYPE` AI families) |
| Query-level cost attribution | `QUERY_ATTRIBUTION_HISTORY` |
| Warehouse compute cost | `WAREHOUSE_METERING_HISTORY` |
| Org-wide rollups + currency | `ORGANIZATION_USAGE.METERING_DAILY_HISTORY`, `USAGE_IN_CURRENCY_DAILY` |

**What changed over time.** Snowflake's first AI usage views were
`CORTEX_FUNCTIONS_USAGE_HISTORY` and `CORTEX_FUNCTIONS_QUERY_USAGE_HISTORY`.
As Cortex expanded (AISQL, Analyst, Search, Agents, Cortex Code), Snowflake
introduced broader, richer views — notably `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`
(with user/role/query-tag attribution and a flexible `METRICS` structure) and
`CORTEX_AISQL_USAGE_HISTORY` — and dedicated Cortex Code views. New views carry
more attribution dimensions and a unit-agnostic metric model (tokens *or* pages
*or* requests). **[DOCUMENTED]** that `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` and
`CORTEX_AISQL_USAGE_HISTORY` exist as current views.

**How AI usage should generally be monitored.**

- **Reconcile, don't pick one.** Use detailed Cortex views for *analysis* and the
  metering views for *billing totals*; expect small reconciliation differences.
- **Filter metering by AI `SERVICE_TYPE`s** for the billed AI number.
- **Attribute with the current views** (`CORTEX_AI_FUNCTIONS_USAGE_HISTORY`)
  because they carry `USER`, `ROLE_NAMES`, `QUERY_TAG`, and `QUERY_ID`.
- **Join to `QUERY_HISTORY` on `QUERY_ID`** for query text, app, and session.

**What downstream AI agents should rely on.** Treat **metering views as the
billed source of truth**, **`CORTEX_AI_FUNCTIONS_USAGE_HISTORY` / `CORTEX_AISQL_USAGE_HISTORY`
as the detailed source of truth**, and **`QUERY_HISTORY` as the correlation
backbone**. Treat the legacy `CORTEX_FUNCTIONS_*` views as fallback only.

---

# 2. Audience and Intended Consumers

| Consumer | How they use this KB | Example questions they should answer |
|----------|----------------------|--------------------------------------|
| **Copilot platforms** | Ground generated SQL; pick correct view; avoid hallucinated columns | "Which view gives token counts per model?" |
| **AI coworker / agent platforms** | Multi-turn reasoning over usage; safe joins; semantic aliases | "Show AI spend by team last month and explain the source." |
| **App developers** | Decide which views to expose; build semantic models | "Which fields are safe for a user-facing dashboard?" |
| **Analytics engineers** | Model marts; reconcile detail vs billed | "How do I reconcile Cortex detail with `METERING_DAILY_HISTORY`?" |
| **FinOps analysts** | Cost trends, chargeback/showback, anomalies | "Top users by AI credits this week." |
| **Governance / audit teams** | Attribution, policy, abnormal activity | "Who ran the most expensive AI queries, under which role?" |
| **Snowflake admins** | Operational monitoring, capacity, guardrails | "Is AI spend trending above budget?" |

**Question classes this KB supports:** *what was used* (functions/models/tokens),
*who used it* (user/role/query-tag/app), *where it ran* (warehouse/interface),
*what it cost* (credits, billed credits, currency), *when* (hourly/daily trend),
and *is it anomalous* (spikes, new models, runaway query tags).

---

# 3. Inventory of Relevant Views

> Column-level specifics labeled **[BEST-EFFORT]** must be verified with `DESC VIEW`.
> All views below live in `SNOWFLAKE.ACCOUNT_USAGE` unless stated; most have an
> `ORGANIZATION_USAGE` counterpart and (where applicable) a `READER_ACCOUNT_USAGE` one.

### 3.1 `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`  *(current — recommended detail SOT)*
- **Schema:** `ACCOUNT_USAGE`
- **Category:** AI / Cortex detail
- **Purpose:** Token/credit and attribution detail for Cortex **AI functions**
  across surfaces (SQL and other interfaces). **[DOCUMENTED]** view exists.
- **Row grain [BEST-EFFORT]:** one row per (time window × function × model ×
  query × warehouse), typically aggregated to an hourly window.
- **Time grain [BEST-EFFORT]:** hourly window (`START_TIME`/`END_TIME`).
- **Primary usage metrics [BEST-EFFORT]:** `METRICS` (array of `{metric, unit, value}`,
  e.g. input/output tokens, pages); derived token totals.
- **Primary cost metrics [BEST-EFFORT]:** `CREDITS` (AI-service credits; excludes
  warehouse compute).
- **Important dimensions [BEST-EFFORT]:** `FUNCTION_NAME`, `MODEL_NAME`, `QUERY_ID`,
  `WAREHOUSE_ID`, `USER_NAME`/`USER_ID`, `ROLE_NAMES`, `QUERY_TAG`.
- **Join keys:** `QUERY_ID` → `QUERY_HISTORY`/`QUERY_ATTRIBUTION_HISTORY`;
  `WAREHOUSE_ID` → `WAREHOUSE_METERING_HISTORY`.
- **Retention:** up to 365 days **[BEST-EFFORT]**.
- **Latency:** a few hours **[BEST-EFFORT]**.
- **Billing semantics:** `CREDITS` here is AI-service consumption, **not** the
  10%-adjusted billed number — reconcile against `METERING_DAILY_HISTORY`.
- **Status:** **Current / recommended.**
- **Best used for:** model/function/user/role/tag attribution; detailed AI analytics.
- **Not recommended for:** final billed totals; warehouse compute cost.
- **Caveats:** unit-agnostic `METRICS` requires careful flattening; data only
  exists from the view's introduction date forward (no full backfill).

### 3.2 `CORTEX_AISQL_USAGE_HISTORY`  *(current)*
- **Schema:** `ACCOUNT_USAGE` — **Category:** AI / Cortex (AISQL) detail
- **Purpose:** Usage history for **Cortex AISQL** (AI SQL functions like
  `AI_COMPLETE`, `AI_CLASSIFY`, `AI_FILTER`, `AI_AGG`, etc.). **[DOCUMENTED]**.
- **Row grain [BEST-EFFORT]:** hourly aggregate by function/model/query/warehouse.
- **Primary usage metrics [BEST-EFFORT]:** `TOKENS`, `TOKENS_GRANULAR`.
- **Primary cost metrics [BEST-EFFORT]:** `TOKEN_CREDITS`, `TOKEN_CREDITS_GRANULAR`.
- **Important dimensions [BEST-EFFORT]:** `MODEL_NAME`, `FUNCTION_NAME`, `QUERY_ID`,
  `QUERY_TAG`, `USER_*`, `WAREHOUSE_ID`, `USAGE_TIME`.
- **Status:** **Current.** **[BEST-EFFORT]** data starts on/after its launch
  (community reports usage from ~Nov 2025 onward) — older AISQL may not appear.
- **Best used for:** AISQL function token/credit analysis with attribution.
- **Caveats:** overlaps conceptually with `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`;
  do **not** sum the two together without de-duplicating scope.

### 3.3 `CORTEX_FUNCTIONS_USAGE_HISTORY`  *(legacy / superseded)*
- **Schema:** `ACCOUNT_USAGE` — **Category:** AI / Cortex detail (legacy)
- **Purpose:** Token + credit usage for Cortex LLM functions, aggregated.
- **Row grain [BEST-EFFORT]:** per time window × function × model (limited
  attribution — typically **no** user/role/query-tag).
- **Primary metrics [BEST-EFFORT]:** `TOKENS`, `TOKEN_CREDITS`.
- **Dimensions [BEST-EFFORT]:** `FUNCTION_NAME`, `MODEL_NAME`, `WAREHOUSE_ID`,
  `START_TIME`/`END_TIME`.
- **Status:** **Legacy / superseded** by `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`.
  **[BEST-EFFORT]** still queryable; may not capture newer AI functions/surfaces.
- **Best used for:** historical continuity for older dashboards.
- **Not recommended for:** user/role attribution; new builds.

### 3.4 `CORTEX_FUNCTIONS_QUERY_USAGE_HISTORY`  *(legacy / superseded)*
- **Schema:** `ACCOUNT_USAGE` — **Category:** AI / Cortex per-query (legacy)
- **Purpose:** Per-**query** Cortex function token/credit detail.
- **Row grain [BEST-EFFORT]:** one row per query (× function/model) that invoked
  Cortex.
- **Primary metrics [BEST-EFFORT]:** `TOKENS`, `TOKEN_CREDITS`.
- **Dimensions [BEST-EFFORT]:** `QUERY_ID`, `FUNCTION_NAME`, `MODEL_NAME`,
  `WAREHOUSE_ID`.
- **Join keys:** `QUERY_ID`.
- **Status:** **Legacy / superseded** by `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`
  (which carries `QUERY_ID` plus richer attribution).
- **Best used for:** legacy query-level AI cost drill-down.

### 3.5 `CORTEX_CODE_CLI_USAGE_HISTORY`  *(current)*
- **Schema:** `ACCOUNT_USAGE` — **Category:** Cortex Code
- **Purpose:** Per-request Cortex **Code** usage via the **CLI**. **[DOCUMENTED]**.
- **Row grain [BEST-EFFORT]:** per request (user × model × time).
- **Metrics [BEST-EFFORT]:** token counts + granular breakdown, `TOKEN_CREDITS`/`CREDITS`.
- **Dimensions [BEST-EFFORT]:** `USER_*`, `MODEL_NAME`, `USAGE_TIME`/time window.
- **Status:** **Current.** **Best for:** Cortex Code spend by developer/model.

### 3.6 `CORTEX_CODE_SNOWSIGHT_USAGE_HISTORY`  *(current)*
- Same shape as 3.5 but for Cortex **Code in Snowsight (UI)**. **[BEST-EFFORT].**
- **Best for:** UI-based Cortex Code spend; pair with the CLI view for total
  Cortex Code usage (union, not join).

### 3.7 `METERING_HISTORY`  *(current — billed, hourly)*
- **Schema:** `ACCOUNT_USAGE` — **Category:** Metering / billing
- **Purpose:** Hourly credit consumption for the whole account **by `SERVICE_TYPE`**.
  AI shows up under `AI_SERVICES` and `CORTEX_CODE_*` (and related) types.
- **Row grain:** one row per (`SERVICE_TYPE` × `NAME` × hour).
- **Metrics:** `CREDITS_USED_COMPUTE`, `CREDITS_USED_CLOUD_SERVICES`, `CREDITS_USED`.
- **Status:** **Current.** **Best for:** total credits trend; AI vs non-AI split.
- **Not for:** token/model detail; per-user attribution.

### 3.8 `METERING_DAILY_HISTORY`  *(current — billed, daily SOT)*
- **Schema:** `ACCOUNT_USAGE` — **Category:** Metering / billing
- **Purpose:** Daily rollup by `SERVICE_TYPE`, including the **10% cloud-services
  adjustment** and the **net billed** number.
- **Row grain:** one row per (`SERVICE_TYPE` × day).
- **Metrics:** `CREDITS_USED`, `CREDITS_ADJUSTMENT_CLOUD_SERVICES`, `CREDITS_BILLED`.
- **Status:** **Current.** **Best for:** billed AI credits, reconciliation, daily charts.
- **Not for:** intra-day, per-query, or token detail.

### 3.9 `WAREHOUSE_METERING_HISTORY`  *(current)*
- One row per warehouse per hour; `CREDITS_USED*`. **Best for:** warehouse compute
  cost that *runs alongside* AISQL (the SQL that calls AI functions still uses a
  warehouse). See [`cost-and-usage.md`](cost-and-usage.md).

### 3.10 `QUERY_HISTORY`  *(current — correlation backbone)*
- One row per query; carries `QUERY_ID`, `QUERY_TEXT`, `USER_NAME`, `ROLE_NAME`,
  `WAREHOUSE_NAME`, `QUERY_TAG`, timing. **Best for:** correlating AI usage to the
  full query context. See [`query-and-performance.md`](query-and-performance.md).

### 3.11 `QUERY_ATTRIBUTION_HISTORY`  *(current)*
- **Purpose:** Per-query **compute credit attribution**. **[DOCUMENTED]**.
- **Metrics [BEST-EFFORT]:** `CREDITS_ATTRIBUTED_COMPUTE`, plus query/parent IDs.
- **Best for:** attributing **warehouse compute** of AI-bearing queries to query/user.
- **Caveat:** attributes **compute**, not AI-service credits — do not confuse the two.

### 3.12 Organization-wide counterparts (`ORGANIZATION_USAGE`)
- `METERING_DAILY_HISTORY`, `METERING_HISTORY`, `USAGE_IN_CURRENCY_DAILY`, and (as
  they roll out) Cortex usage views. **Best for:** multi-account rollups and
  **currency** conversion. See [`organization-usage.md`](organization-usage.md).

### 3.13 `READER_ACCOUNT_USAGE`
- Mirror of selected views for **reader (managed) accounts**. Relevant only if you
  provision reader accounts that run Cortex. See [`reader-account-usage.md`](reader-account-usage.md).

---

# 4. Detailed View Profiles

> Where a column is **[BEST-EFFORT]**, confirm with `DESC VIEW` before relying on it.

## 4.1 `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`

**Official purpose (plain English):** The current, attribution-rich record of
Cortex AI function consumption — *which* function and model ran, *who* ran it,
*under which role and query tag*, in *which query and warehouse*, and *how many
credits / tokens* it consumed.

**What one row represents [BEST-EFFORT]:** an aggregated slice of AI-function usage
for a time window (commonly hourly) for a given function/model/query/warehouse.
A single heavy query can therefore produce one or more rows.

**Important columns (plain English) [BEST-EFFORT]:**
- `START_TIME` / `END_TIME` — the aggregation window (local time zone, `_LTZ`).
- `FUNCTION_NAME` — the Cortex AI function (e.g., `COMPLETE`, `AI_COMPLETE`).
- `MODEL_NAME` — the model invoked; can be blank when no model applies.
- `QUERY_ID` — the calling query; join key to `QUERY_HISTORY`.
- `WAREHOUSE_ID` — the warehouse that hosted the calling SQL.
- `USER_NAME` / `USER_ID` — the invoker (attribution).
- `ROLE_NAMES` — array of roles active for the query (attribution/governance).
- `QUERY_TAG` — application/workload tag carried from the session/query.
- `METRICS` — array of `{key:{metric, unit}, value}` entries (tokens, pages, etc.).
- `CREDITS` — AI-service credits consumed (NOT warehouse compute, NOT billed-net).

**For app builders:** This is your detail table. Expose `FUNCTION_NAME`,
`MODEL_NAME`, derived token totals, `CREDITS`, and attribution dims. Flatten
`METRICS` into typed columns (input_tokens, output_tokens, …) behind a view.

**For Copilot/agent systems:** Prefer this view for "who/what/which model"
questions. **Do not present `CREDITS` as the bill** — label it "AI-service credits
(unbilled)" and direct billed totals to `METERING_DAILY_HISTORY`.

**Differs from siblings:**
- vs `CORTEX_FUNCTIONS_USAGE_HISTORY`: adds user/role/query-tag and a unit-agnostic
  `METRICS` model; broader surface coverage.
- vs `CORTEX_AISQL_USAGE_HISTORY`: AISQL view is scoped to AI **SQL** functions;
  this one is broader. Scopes overlap — don't double count.

**When to use:** attribution, model/function analytics, chargeback inputs.
**When not to use:** final billed numbers; pure warehouse cost.

**Sample business questions:** top users by AI credits; most-used models; spend by
query tag; role-based AI adoption.

**Sample agent prompts:** *"Break down Cortex AI credits by model and user for last
30 days."* / *"Which query tags drove the most AI tokens this week?"*

**Recommended joins:** `QUERY_ID` → `QUERY_HISTORY` (text/app/session);
`WAREHOUSE_ID` → `WAREHOUSE_METERING_HISTORY` (compute alongside).

**Warnings:** unit-agnostic `METRICS` (tokens vs pages); no full historical
backfill; `CREDITS` ≠ billed credits.

## 4.2 `CORTEX_AISQL_USAGE_HISTORY`

**Purpose:** Token/credit detail specific to **AISQL** functions.
**One row [BEST-EFFORT]:** hourly aggregate per function/model/query/warehouse.
**Key columns [BEST-EFFORT]:** `USAGE_TIME`, `MODEL_NAME`, `FUNCTION_NAME`,
`TOKENS`, `TOKENS_GRANULAR`, `TOKEN_CREDITS`, `TOKEN_CREDITS_GRANULAR`, `QUERY_ID`,
`QUERY_TAG`, `USER_*`, `WAREHOUSE_ID`.
**App builders:** great for token-economics dashboards (input vs output via the
`*_GRANULAR` objects). **Agents:** use for AISQL-specific token analysis; clarify
it may not cover non-SQL Cortex surfaces.
**When not to use:** as the single AI total (combine deliberately with other AI
surfaces); for billed totals.
**Joins:** `QUERY_ID` → `QUERY_HISTORY`. **Warnings:** recent launch ⇒ short
history; overlaps with `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`.

## 4.3 `CORTEX_FUNCTIONS_USAGE_HISTORY` (legacy)

**Purpose:** original aggregate token/credit view for Cortex LLM functions.
**One row [BEST-EFFORT]:** time window × function × model. **Key columns:**
`START_TIME`/`END_TIME`, `FUNCTION_NAME`, `MODEL_NAME`, `TOKENS`, `TOKEN_CREDITS`,
`WAREHOUSE_ID`. **Use:** legacy dashboards / historical continuity. **Avoid:** new
builds needing user/role/tag attribution (it lacks them). **Migration:** move to
`CORTEX_AI_FUNCTIONS_USAGE_HISTORY` (see §5).

## 4.4 `CORTEX_FUNCTIONS_QUERY_USAGE_HISTORY` (legacy)

**Purpose:** per-query Cortex function detail. **One row [BEST-EFFORT]:** per query
(× function/model). **Key columns:** `QUERY_ID`, `FUNCTION_NAME`, `MODEL_NAME`,
`TOKENS`, `TOKEN_CREDITS`, `WAREHOUSE_ID`. **Use:** legacy per-query drill-down.
**Migration:** `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` (carries `QUERY_ID` + more).

## 4.5 `CORTEX_CODE_CLI_USAGE_HISTORY` / `CORTEX_CODE_SNOWSIGHT_USAGE_HISTORY`

**Purpose:** Cortex **Code** usage/billing for CLI and Snowsight respectively.
**One row [BEST-EFFORT]:** per request (user × model × time). **Key columns:**
`USAGE_TIME`/window, `USER_*`, `MODEL_NAME`, token counts (+ granular),
`TOKEN_CREDITS`/`CREDITS`. **App builders:** developer-level Cortex Code cost
dashboards, per-model token efficiency. **Agents:** answer "who is spending on
Cortex Code and on which model." **Combine the two** by **UNION** (they are
different surfaces, not joinable rows). **Migration note:** these replace ad-hoc
reliance on raw metering for Cortex Code line items.

## 4.6 `METERING_HISTORY` / `METERING_DAILY_HISTORY`

**Purpose:** **billed** credit consumption by `SERVICE_TYPE`. AI families appear as
`AI_SERVICES` and `CORTEX_CODE_CLI` / `CORTEX_CODE_SNOWSIGHT` (and other
AI/agent/intelligence service types as they ship). **One row:** per service type
per hour (history) or per day (daily). **Use:** total billed AI credits, daily
charts, reconciliation. **Avoid:** per-user/model/token detail. **Daily view** is
the place for the **10% cloud-services adjustment** and **`CREDITS_BILLED`**.

## 4.7 `QUERY_HISTORY` (correlation)

Join target for `QUERY_ID`. Adds `QUERY_TEXT`, `USER_NAME`, `ROLE_NAME`,
`WAREHOUSE_NAME`, `QUERY_TAG`, timing, error status. **Use:** enrich AI usage with
full query context, app detection (via `QUERY_TAG`), and troubleshooting.

## 4.8 `QUERY_ATTRIBUTION_HISTORY`

Attributes **warehouse compute credits** to individual queries
(`CREDITS_ATTRIBUTED_COMPUTE`). **Use:** the *compute* portion of AI-bearing
queries (the SQL around the AI call), distinct from AI-service credits.

---

# 5. Current vs Deprecated View Mapping

> **[BEST-EFFORT] / [INTERPRETATION]** for retirement timing — Snowflake has not
> always published hard end-of-life dates for these. **Confirm in release notes.**

| Legacy view | Still queryable? | Still updated? | Replacement | What changed | Migration guidance |
|-------------|------------------|----------------|-------------|--------------|--------------------|
| `CORTEX_FUNCTIONS_USAGE_HISTORY` | Yes **[BEST-EFFORT]** | Likely yes for legacy functions; may miss new surfaces **[BEST-EFFORT]** | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` | + user/role/query-tag attribution; unit-agnostic `METRICS`; broader coverage | Repoint dashboards; map `TOKENS`→derived from `METRICS`; add attribution dims |
| `CORTEX_FUNCTIONS_QUERY_USAGE_HISTORY` | Yes **[BEST-EFFORT]** | As above **[BEST-EFFORT]** | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` (has `QUERY_ID`) | per-query detail now inside the richer view | Replace `QUERY_ID` joins; recompute token totals from `METRICS` |
| Raw metering for Cortex Code line items | Yes | Yes | `CORTEX_CODE_CLI_USAGE_HISTORY` / `CORTEX_CODE_SNOWSIGHT_USAGE_HISTORY` | dedicated per-request detail + attribution | Use dedicated views for detail; keep metering for billed totals |

**Migration principles [INTERPRETATION]:**
1. Don't hard-cut; run **legacy and current in parallel** and reconcile for ≥1
   billing cycle.
2. Build a **compatibility view** that exposes a stable column contract
   (`usage_time, function_name, model_name, tokens, credits, user, role, query_tag,
   query_id`) so downstream apps don't break when the underlying source changes.
3. Prefer current views for any new attribution or governance requirement.

---

# 6. Semantic Layer for AI Agents

**Canonical metric meanings:**
- **tokens** — count of model input/output units. *Usage*, not money. May be split
  into input/output/cached (granular). For page-based functions the unit is
  **pages**, not tokens — always read the metric's `unit`.
- **token_credits / credits (Cortex detail views)** — Snowflake **AI-service
  credits** consumed by the AI operation. *Approximate spend*, **not** the final
  billed number.
- **credits_used (metering)** — credits consumed by a `SERVICE_TYPE`.
- **credits_billed (`METERING_DAILY_HISTORY`)** — **net billed** credits after the
  cloud-services adjustment. **This is the money number.**
- **credits_attributed_compute (`QUERY_ATTRIBUTION_HISTORY`)** — warehouse
  **compute** credits attributed to a query. Distinct from AI-service credits.

**Canonical dimension meanings:**
- **function_name** — the AI function invoked.
- **model_name** — the LLM/model; can be blank for non-model functions.
- **query_id** — stable join key to `QUERY_HISTORY`.
- **warehouse_id / warehouse_name** — compute that hosted the calling SQL.
- **user_name / user_id** — invoker identity (attribution).
- **role_names** — roles active during the query (array; governance).
- **query_tag** — free-text workload/app label; only as reliable as your tagging.
- **service_type** — billing bucket (e.g., `AI_SERVICES`, `CORTEX_CODE_*`).

**Trusted interpretation rules:**
1. **Billed totals → metering views.** Detail-view credits are estimates.
2. **Attribution → current Cortex views** (`USER`, `ROLE_NAMES`, `QUERY_TAG`).
3. **One AI query touches multiple views** — never sum AI-service credits and
   warehouse compute credits into one "AI cost" without labeling each component.
4. **Respect the metric `unit`** in `METRICS` (tokens vs pages vs requests).

**Ambiguous fields & handling:**
- `CREDITS` vs `CREDITS_BILLED` — never treat detail `CREDITS` as the invoice.
- `QUERY_TAG` — may be null/duplicated/free-form; never assume a clean app key.
- `ROLE_NAMES` (array) — multiple roles; don't pick one silently.
- `MODEL_NAME` blank — legitimate for non-model functions; don't drop the row.

**Fields not to over-interpret:** `QUERY_TAG` (governance hygiene varies),
granular token objects (schema may evolve), AI-service `CREDITS` as money.

**Time-grain caution:** Cortex detail windows are usually **hourly**; metering
daily view is **daily**. Don't compare an hourly slice to a daily billed number
without aggregation.

**Billing caution:** the **10% cloud-services adjustment** lives in
`METERING_DAILY_HISTORY`; detail views don't apply it.

**Safe vs unsafe assumptions:**

| Safe ✅ | Unsafe ❌ |
|--------|----------|
| `METERING_DAILY_HISTORY.CREDITS_BILLED` is the billed AI number (filtered to AI service types) | Detail-view `CREDITS` equals the invoice |
| `QUERY_ID` joins detail → `QUERY_HISTORY` | Every AI row has a non-null `QUERY_ID` for non-SQL surfaces |
| Current views carry user/role/tag | Legacy `CORTEX_FUNCTIONS_*` carry user/role |
| `METRICS.unit` tells you tokens vs pages | All `METRICS` values are tokens |

---

# 7. Relationship Map

```
                         ┌─────────────────────────────────────────┐
                         │  BILLED SOURCE OF TRUTH (money)          │
                         │  METERING_DAILY_HISTORY  (daily, net)    │
                         │  METERING_HISTORY        (hourly)        │
                         │  → filter SERVICE_TYPE in AI families    │
                         └───────────────▲─────────────────────────┘
                                         │ reconcile (not exact)
   DETAILED AI SOURCE OF TRUTH           │
   CORTEX_AI_FUNCTIONS_USAGE_HISTORY ────┤
   CORTEX_AISQL_USAGE_HISTORY            │
   CORTEX_CODE_CLI / _SNOWSIGHT          │
   (legacy: CORTEX_FUNCTIONS_*)          │
            │ QUERY_ID                   │ WAREHOUSE_ID
            ▼                            ▼
   QUERY_HISTORY  ◄── QUERY_ATTRIBUTION_HISTORY     WAREHOUSE_METERING_HISTORY
   (context, tag,     (per-query compute credits)   (warehouse compute by hour)
    user, role)
            │
            ▼
   ORGANIZATION_USAGE.* (multi-account rollups, USAGE_IN_CURRENCY_DAILY)
```

- **Detailed AI/Cortex views**: what/who/which model — analysis.
- **Metering views**: billed totals — finance.
- **Warehouse cost views**: compute that runs the AI SQL — efficiency.
- **Query correlation views**: context, tag, troubleshooting.
- **Org views**: enterprise rollups + currency.

**Use together when:** building chargeback (detail attribution × billed total),
or executive views (billed trend + top drivers from detail).
**Don't combine when:** summing AI-service credits + compute credits into a single
"AI cost" without labels; or stacking AISQL and AI-functions views that overlap.

---

# 8. Join Patterns and Query Design Rules

> **General rule:** AI detail rows are often hourly aggregates; `QUERY_HISTORY` is
> per query. Joining on `QUERY_ID` can **fan out** if a query spans windows.
> De-duplicate or aggregate before/after the join.

### 8.1 AI usage ↔ `QUERY_HISTORY`
- **Purpose:** add query text, user, role, warehouse name, tag, timing.
- **Join keys:** `QUERY_ID`.
- **Cardinality:** detail → query is **many-to-one** (multiple AI rows per query).
- **Grain:** aggregate detail to query before joining for per-query totals.
- **Latency risk:** views land at different times; a fresh AI row may not yet have
  its `QUERY_HISTORY` partner — use `LEFT JOIN` and tolerate nulls.
- **Double-counting:** don't multiply tokens by query rows.
- **Safe filter:** bound both sides by the same `START_TIME`/window range.
- **Anti-pattern:** `INNER JOIN` that silently drops not-yet-landed queries.

### 8.2 AI usage ↔ `WAREHOUSE_METERING_HISTORY`
- **Purpose:** compare AI-service credits to warehouse compute credits.
- **Join keys:** `WAREHOUSE_ID` + overlapping time window.
- **Cardinality:** many-to-many across hours — **aggregate to (warehouse, hour)
  first**, then join.
- **Double-counting:** these are **different credit pools** — present side by side,
  never add into one figure.

### 8.3 AI usage ↔ `METERING_DAILY_HISTORY`
- **Purpose:** reconcile detailed AI credits vs billed AI credits.
- **Join keys:** **day** (aggregate detail to date) + AI `SERVICE_TYPE`.
- **Grain:** daily on both sides. **Expect small variances** (adjustment, rounding,
  surface coverage) — reconcile, don't equate.

### 8.4 AI usage ↔ user / role attribution
- **Purpose:** spend by user/role.
- **Keys:** prefer native `USER_*` / `ROLE_NAMES` in current views; else go via
  `QUERY_ID` → `QUERY_HISTORY.USER_NAME`/`ROLE_NAME`.
- **Caveat:** `ROLE_NAMES` is an array — `FLATTEN` carefully (can multiply credits;
  attribute, don't duplicate).

### 8.5 AI usage ↔ query-tag / application attribution
- **Purpose:** app/workload chargeback.
- **Keys:** `QUERY_TAG` (native or via `QUERY_HISTORY`).
- **Caveat:** depends entirely on disciplined tagging; bucket null/blank tags as
  `UNTAGGED` rather than dropping.

### 8.6 ACCOUNT_USAGE ↔ ORGANIZATION_USAGE
- **Purpose:** multi-account rollups + currency.
- **Keys:** date + `SERVICE_TYPE` (+ account).
- **Caveat:** org views have **higher latency** and may lag account views; don't
  expect same-hour parity.

---

# 9. Application Builder Guidance

**Expose to apps (curated):** `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`,
`CORTEX_AISQL_USAGE_HISTORY`, `CORTEX_CODE_*_USAGE_HISTORY`,
`METERING_DAILY_HISTORY` (AI filtered). Wrap each behind a semantic view with a
**stable column contract**.

**Abstract behind a semantic model:** the legacy `CORTEX_FUNCTIONS_*` views and any
raw `METRICS`/granular objects — present typed, flattened columns instead.

**Safe for user-facing analytics:** time, `FUNCTION_NAME`, `MODEL_NAME`, token
totals, AI-service `CREDITS` (clearly labeled "estimated/unbilled"),
`CREDITS_BILLED` (billed), `QUERY_TAG`, warehouse name.

**Keep internal / handle carefully:** raw `USER_ID`/`USER_NAME` (PII —
pseudonymize for broad audiences), full `QUERY_TEXT` (may contain sensitive data),
`ROLE_NAMES` arrays (governance-sensitive).

**Modeling for retrieval/grounding:**
- Create one **fact view per grain**: `fct_ai_usage_hourly` (detail),
  `fct_ai_billed_daily` (metering), `fct_warehouse_hourly`.
- Add explicit **`metric_unit`** and **`credit_type`** ('ai_service' | 'billed' |
  'compute') columns so agents can't conflate pools.
- Pre-flatten `METRICS`/granular objects into `input_tokens`, `output_tokens`, etc.

**Document trusted SQL patterns:** ship a vetted query library (see §12) and pin
agents to it; prefer parameterized templates over free-form generation.

**Multi-turn AI questions:** maintain a session glossary (metric → view → grain) so
follow-ups ("now by model", "now billed") route to the correct source.

**Reduce hallucination risk:** (1) ground on `DESC VIEW` output, (2) restrict
generation to whitelisted views/columns, (3) force the model to state grain and
credit-type, (4) reject queries that add `CREDITS` + `CREDITS_ATTRIBUTED_COMPUTE`.

**Semantic-layer metadata to add:** synonyms (e.g., "AI spend" → billed AI
credits), grain, default time filters, join-key registry, and "do-not-sum" rules.

---

# 10. Monitoring / FinOps Use Cases

| Use case | Recommended view(s) | Notes |
|----------|--------------------|-------|
| Total AI credits over time | `METERING_DAILY_HISTORY` (AI service types) | Billed truth |
| Total AI tokens over time | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` / `CORTEX_AISQL_USAGE_HISTORY` | Sum tokens from `METRICS`/`TOKENS` |
| Top users by AI spend | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` | Native `USER_*`; else via `QUERY_HISTORY` |
| Top warehouses by AI spend | AI detail + `WAREHOUSE_METERING_HISTORY` | Show AI-service vs compute separately |
| Top models by usage | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` / AISQL | Group by `MODEL_NAME` |
| Top functions by usage | same | Group by `FUNCTION_NAME` |
| Top apps by query tag | AI detail + `QUERY_TAG` | Bucket `UNTAGGED` |
| AI spend trending | `METERING_DAILY_HISTORY` | Daily; smooth weekly |
| Billed AI vs warehouse spend | `METERING_DAILY_HISTORY` by `SERVICE_TYPE` | Compare AI families vs `WAREHOUSE_METERING` |
| Showback / chargeback | AI detail (attribution) × billed total | Allocate billed by detail share |
| Anomaly detection | `METERING_HISTORY` (hourly) + detail | Spike in service type / model |
| Executive dashboard | `METERING_DAILY_HISTORY` + `ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY` | Currency for finance |
| Cortex Code spend | `CORTEX_CODE_CLI/_SNOWSIGHT_USAGE_HISTORY` | UNION CLI + Snowsight |

---

# 11. Governance / Risk / Audit Use Cases

- **Accountability / user attribution:** `CORTEX_AI_FUNCTIONS_USAGE_HISTORY.USER_*`
  (or via `QUERY_HISTORY`). Who used which model and how much.
- **Application attribution:** `QUERY_TAG` (enforce tagging policy).
- **Query-tag governance:** report % of AI usage that is `UNTAGGED`; trend it down.
- **Role-based analysis:** `ROLE_NAMES` / `QUERY_HISTORY.ROLE_NAME` — detect
  privileged roles running AI, or shared roles obscuring attribution.
- **Policy / compliance review:** correlate `QUERY_HISTORY.QUERY_TEXT` with AI usage
  to verify approved models/functions only (watch for sensitive data in prompts).
- **Cost guardrails:** alert when daily AI `CREDITS_BILLED` exceeds budget; use
  Snowflake budgets/resource monitors as the enforcement layer.
- **AI adoption oversight:** distinct users/models/functions over time.
- **Abnormal activity investigation:** join hourly metering spikes → top
  query/user/model in detail → `QUERY_HISTORY` for full context.

---

# 12. SQL Query Library

> **Conventions:** `_LTZ` time columns; bound by `START_TIME`/`USAGE_DATE`. Replace
> view/column names if your account's `DESC VIEW` differs. AI service-type lists are
> **[BEST-EFFORT]** — adjust to the service types actually present in your account.

### 12.1 Billed AI credits — weekly & monthly
- **Description:** net billed AI credits by week/month. **Grain:** daily→weekly.
- **Assumptions:** AI shows up under the listed service types. **Caveat:** billed,
  post-adjustment. **Trustworthy because:** uses `CREDITS_BILLED` (the money number).
```sql
SELECT DATE_TRUNC('week', usage_date) AS wk,
       service_type,
       SUM(credits_billed)            AS ai_credits_billed
FROM   SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE  service_type IN ('AI_SERVICES','CORTEX_CODE_CLI','CORTEX_CODE_SNOWSIGHT')
  AND  usage_date >= DATEADD('month', -3, CURRENT_DATE)
GROUP  BY 1,2
ORDER  BY 1 DESC, ai_credits_billed DESC;
```

### 12.2 Token consumption by model
- **Grain:** hourly→period. **Caveat:** AI-service tokens, not billed. **Trust:**
  groups native attribution columns; respects metric unit (filter tokens only).
```sql
SELECT model_name,
       SUM(m.value::NUMBER) AS total_tokens
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY,
       LATERAL FLATTEN(input => metrics) m
WHERE  start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP)
  AND  m.value:"key":"unit"::STRING = 'tokens'   -- [BEST-EFFORT] adapt to actual METRICS shape
GROUP  BY model_name
ORDER  BY total_tokens DESC;
```
> If your account exposes `CORTEX_AISQL_USAGE_HISTORY` with a flat `TOKENS` column,
> prefer: `SELECT model_name, SUM(tokens) ... GROUP BY model_name`.

### 12.3 AI credits by function
```sql
SELECT function_name, SUM(credits) AS ai_service_credits
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
WHERE  start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP)
GROUP  BY function_name
ORDER  BY ai_service_credits DESC;
```

### 12.4 AI credits by user (attribution)
```sql
SELECT user_name, SUM(credits) AS ai_service_credits
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
WHERE  start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP)
GROUP  BY user_name
ORDER  BY ai_service_credits DESC
LIMIT  50;
```

### 12.5 AI usage by warehouse (with compute side-by-side)
- **Caveat:** two different credit pools — reported in separate columns, not summed.
```sql
WITH ai AS (
  SELECT warehouse_id, SUM(credits) AS ai_service_credits
  FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
  WHERE  start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP)
  GROUP  BY warehouse_id),
wh AS (
  SELECT warehouse_id, warehouse_name, SUM(credits_used) AS compute_credits
  FROM   SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
  WHERE  start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP)
  GROUP  BY warehouse_id, warehouse_name)
SELECT wh.warehouse_name, ai.ai_service_credits, wh.compute_credits
FROM   wh LEFT JOIN ai USING (warehouse_id)
ORDER  BY ai.ai_service_credits DESC NULLS LAST;
```

### 12.6 AI usage by query tag (application/workload)
```sql
SELECT COALESCE(NULLIF(query_tag,''),'UNTAGGED') AS app_tag,
       SUM(credits) AS ai_service_credits
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
WHERE  start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP)
GROUP  BY 1
ORDER  BY ai_service_credits DESC;
```

### 12.7 AI usage by role (array flatten — attribute, don't duplicate)
- **Caveat:** flattening an array can multiply credits; use only for *adoption*
  counts, not for summing total spend.
```sql
SELECT r.value::STRING AS role_name,
       COUNT(*)        AS ai_invocation_rows
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY,
       LATERAL FLATTEN(input => role_names) r
WHERE  start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP)
GROUP  BY 1 ORDER BY ai_invocation_rows DESC;
```

### 12.8 Most expensive AI queries
```sql
SELECT a.query_id, SUM(a.credits) AS ai_service_credits,
       ANY_VALUE(q.user_name) AS user_name,
       ANY_VALUE(q.query_tag) AS query_tag
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY a
LEFT   JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY q USING (query_id)
WHERE  a.start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP)
GROUP  BY a.query_id
ORDER  BY ai_service_credits DESC
LIMIT  25;
```

### 12.9 Daily trend (dashboard)
```sql
SELECT DATE_TRUNC('day', start_time) AS d,
       SUM(credits) AS ai_service_credits
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
WHERE  start_time >= DATEADD('day', -90, CURRENT_TIMESTAMP)
GROUP  BY 1 ORDER BY 1;
```

### 12.10 Reconciliation: detail vs billed
- **Description:** compare AI-service credits (detail) to billed AI credits.
- **Caveat:** expect variance (adjustment, coverage, latency) — investigate gaps,
  don't expect zero.
```sql
WITH detail AS (
  SELECT DATE_TRUNC('day', start_time) AS d, SUM(credits) AS detail_credits
  FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
  WHERE  start_time >= DATEADD('day', -30, CURRENT_TIMESTAMP)
  GROUP  BY 1),
billed AS (
  SELECT usage_date AS d, SUM(credits_billed) AS billed_credits
  FROM   SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
  WHERE  service_type = 'AI_SERVICES'
    AND  usage_date >= DATEADD('day', -30, CURRENT_DATE)
  GROUP  BY 1)
SELECT d, detail_credits, billed_credits,
       billed_credits - detail_credits AS variance
FROM   detail FULL OUTER JOIN billed USING (d)
ORDER  BY d;
```

### 12.11 Deprecated → current migration example
```sql
-- Legacy
SELECT function_name, model_name, SUM(token_credits) AS credits
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY
WHERE  start_time >= DATEADD('day',-30,CURRENT_TIMESTAMP)
GROUP  BY 1,2;
-- Current (adds user/role/tag attribution)
SELECT function_name, model_name, user_name, query_tag, SUM(credits) AS credits
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AI_FUNCTIONS_USAGE_HISTORY
WHERE  start_time >= DATEADD('day',-30,CURRENT_TIMESTAMP)
GROUP  BY 1,2,3,4;
```

### 12.12 Cortex Code monitoring (CLI + Snowsight)
```sql
SELECT 'CLI' AS surface, user_name, model_name, SUM(token_credits) AS credits
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_CODE_CLI_USAGE_HISTORY
WHERE  usage_time >= DATEADD('day',-30,CURRENT_TIMESTAMP)
GROUP  BY 1,2,3
UNION ALL
SELECT 'SNOWSIGHT', user_name, model_name, SUM(token_credits)
FROM   SNOWFLAKE.ACCOUNT_USAGE.CORTEX_CODE_SNOWSIGHT_USAGE_HISTORY
WHERE  usage_time >= DATEADD('day',-30,CURRENT_TIMESTAMP)
GROUP  BY 1,2,3
ORDER  BY credits DESC;
```

### 12.13 Org-wide AI spend in currency
```sql
SELECT usage_date, SUM(usage_in_currency) AS spend
FROM   SNOWFLAKE.ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY
WHERE  usage_type ILIKE '%ai%'      -- [BEST-EFFORT] confirm usage_type values
  AND  usage_date >= DATEADD('month',-3,CURRENT_DATE)
GROUP  BY usage_date ORDER BY usage_date;
```

---

# 13. Canonical Enterprise Monitoring Model

| Role | Canonical view |
|------|----------------|
| **Detailed source of truth** | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` (+ `CORTEX_AISQL_USAGE_HISTORY`, `CORTEX_CODE_*`) |
| **Billed source of truth** | `METERING_DAILY_HISTORY` (AI service types) |
| **Dashboard source** | `METERING_DAILY_HISTORY` (totals) + detail views (drill-down) |
| **Troubleshooting source** | `QUERY_HISTORY` + detail (`QUERY_ID`) |
| **Governance source** | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` (user/role/tag) + `QUERY_HISTORY` |
| **Executive reporting** | `ORGANIZATION_USAGE.METERING_DAILY_HISTORY` + `USAGE_IN_CURRENCY_DAILY` |

**Canonical dimensions:** `date/time` (usage_date / start_time), `user`
(user_name/id), `role` (role_names / role_name), `warehouse` (warehouse_id/name),
`model` (model_name), `function` (function_name), `query` (query_id),
`application/query_tag` (query_tag), `service_type`, `environment`
(derive from query_tag/warehouse naming — **[INTERPRETATION]**).

---

# 14. Machine-Readable Summary

```json
[
  {
    "view_name": "CORTEX_AI_FUNCTIONS_USAGE_HISTORY",
    "status": "current",
    "purpose": "Attribution-rich Cortex AI function token/credit detail.",
    "row_grain": "hourly window x function x model x query x warehouse [BEST-EFFORT]",
    "best_for": ["model attribution","user/role/tag attribution","AI analytics"],
    "join_keys": ["QUERY_ID","WAREHOUSE_ID"],
    "pitfalls": ["CREDITS is AI-service not billed","unit-agnostic METRICS","no full backfill"],
    "replacement_view_if_any": null
  },
  {
    "view_name": "CORTEX_AISQL_USAGE_HISTORY",
    "status": "current",
    "purpose": "AISQL (AI SQL functions) token/credit detail.",
    "row_grain": "hourly aggregate by function/model/query/warehouse [BEST-EFFORT]",
    "best_for": ["AISQL token economics","model usage"],
    "join_keys": ["QUERY_ID","WAREHOUSE_ID"],
    "pitfalls": ["short history","overlaps CORTEX_AI_FUNCTIONS_USAGE_HISTORY"],
    "replacement_view_if_any": null
  },
  {
    "view_name": "CORTEX_FUNCTIONS_USAGE_HISTORY",
    "status": "legacy",
    "purpose": "Original aggregate Cortex LLM function token/credit usage.",
    "row_grain": "time window x function x model [BEST-EFFORT]",
    "best_for": ["legacy dashboard continuity"],
    "join_keys": ["WAREHOUSE_ID"],
    "pitfalls": ["no user/role/tag attribution","may miss new surfaces"],
    "replacement_view_if_any": "CORTEX_AI_FUNCTIONS_USAGE_HISTORY"
  },
  {
    "view_name": "CORTEX_FUNCTIONS_QUERY_USAGE_HISTORY",
    "status": "legacy",
    "purpose": "Per-query Cortex function token/credit detail.",
    "row_grain": "per query x function/model [BEST-EFFORT]",
    "best_for": ["legacy per-query drill-down"],
    "join_keys": ["QUERY_ID"],
    "pitfalls": ["limited attribution"],
    "replacement_view_if_any": "CORTEX_AI_FUNCTIONS_USAGE_HISTORY"
  },
  {
    "view_name": "CORTEX_CODE_CLI_USAGE_HISTORY",
    "status": "current",
    "purpose": "Cortex Code usage/billing via CLI.",
    "row_grain": "per request (user x model x time) [BEST-EFFORT]",
    "best_for": ["Cortex Code spend by developer/model (CLI)"],
    "join_keys": ["USER_NAME"],
    "pitfalls": ["UNION with Snowsight view; do not join"],
    "replacement_view_if_any": null
  },
  {
    "view_name": "CORTEX_CODE_SNOWSIGHT_USAGE_HISTORY",
    "status": "current",
    "purpose": "Cortex Code usage/billing via Snowsight UI.",
    "row_grain": "per request (user x model x time) [BEST-EFFORT]",
    "best_for": ["Cortex Code spend by developer/model (UI)"],
    "join_keys": ["USER_NAME"],
    "pitfalls": ["UNION with CLI view; do not join"],
    "replacement_view_if_any": null
  },
  {
    "view_name": "METERING_HISTORY",
    "status": "current",
    "purpose": "Hourly billed credits by service type (incl. AI families).",
    "row_grain": "service_type x name x hour",
    "best_for": ["total credits trend","AI vs non-AI split","hourly anomalies"],
    "join_keys": ["SERVICE_TYPE","NAME"],
    "pitfalls": ["no token/model/user detail"],
    "replacement_view_if_any": null
  },
  {
    "view_name": "METERING_DAILY_HISTORY",
    "status": "current",
    "purpose": "Daily billed credits by service type with cloud-services adjustment.",
    "row_grain": "service_type x day",
    "best_for": ["billed AI totals","reconciliation","daily charts"],
    "join_keys": ["SERVICE_TYPE","USAGE_DATE"],
    "pitfalls": ["no intra-day or token detail"],
    "replacement_view_if_any": null
  },
  {
    "view_name": "QUERY_ATTRIBUTION_HISTORY",
    "status": "current",
    "purpose": "Per-query warehouse compute credit attribution.",
    "row_grain": "per query",
    "best_for": ["attributing compute of AI-bearing queries"],
    "join_keys": ["QUERY_ID","WAREHOUSE_ID"],
    "pitfalls": ["compute credits, not AI-service credits"],
    "replacement_view_if_any": null
  },
  {
    "view_name": "QUERY_HISTORY",
    "status": "current",
    "purpose": "Correlation backbone: query text, user, role, tag, timing.",
    "row_grain": "per query",
    "best_for": ["enrichment","troubleshooting","app/tag detection"],
    "join_keys": ["QUERY_ID","WAREHOUSE_ID"],
    "pitfalls": ["latency differs from detail views"],
    "replacement_view_if_any": null
  }
]
```

---

# 15. Data Dictionary

> Semantic nuance per field; verify exact presence/types per view with `DESC VIEW`.

| Field | Meaning & cross-view nuance |
|-------|------------------------------|
| `start_time` / `end_time` | Aggregation window (TIMESTAMP_LTZ) in Cortex/metering hourly views. **Local time zone.** Window edges, not exact event timestamps. |
| `usage_time` | Single window-start timestamp used by some Cortex/AISQL/Code views in place of start/end pairs. **[BEST-EFFORT]** — confirm which your view uses. |
| `usage_date` | DATE in daily views (`METERING_DAILY_HISTORY`, org currency views). Day-grain only. |
| `function_name` | Cortex function invoked (`COMPLETE`, `AI_COMPLETE`, `AI_CLASSIFY`, …). Spelling differs across function families. |
| `model_name` | Model used; **blank/NULL is valid** for non-model functions. |
| `query_id` | Stable join key to `QUERY_HISTORY`/`QUERY_ATTRIBUTION_HISTORY`. **May be NULL** for non-SQL surfaces (e.g., some Cortex Code/Analyst paths). |
| `warehouse_id` | Stable numeric warehouse key (join-safe across renames). |
| `warehouse_name` | Display name; can be reused/renamed — prefer `warehouse_id` for joins. |
| `user_id` / `user_name` | Invoker. **PII** — pseudonymize for broad audiences. Present natively in current views; otherwise via `QUERY_HISTORY`. |
| `role_names` | **Array** of roles active for the query (current views). `QUERY_HISTORY.role_name` is the single primary role. |
| `query_tag` | Free-text workload/app label; reliability depends on tagging discipline. Bucket blanks as `UNTAGGED`. |
| `tokens` | Count of model token units (usage). Flat column in some views. |
| `tokens_granular` | OBJECT/array splitting tokens by type (input/output/cached). Schema may evolve. |
| `token_credits` | AI-service credits derived from tokens (legacy/AISQL views). **Estimate, not billed.** |
| `token_credits_granular` | Credit breakdown by token type. **[BEST-EFFORT].** |
| `credits` | AI-service credits in `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`/Code views. **Not** billed-net; **not** warehouse compute. |
| `credits_used` | Metering views: credits for a service type (compute + cloud services). |
| `credits_billed` | `METERING_DAILY_HISTORY`: **net billed** after cloud-services adjustment. **The money number.** |
| `credits_attributed_compute` | `QUERY_ATTRIBUTION_HISTORY`: warehouse **compute** credits for a query. Distinct pool from AI-service credits. |
| `metrics` | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`: array of `{key:{metric,unit},value}`. **Always read `unit`** (tokens vs pages vs requests). |
| `service_type` | Metering bucket: `WAREHOUSE_METERING`, `AI_SERVICES`, `CORTEX_CODE_CLI`, `CORTEX_CODE_SNOWSIGHT`, and others (e.g., agents/intelligence) **[BEST-EFFORT]**. |
| `entity_id` / `entity_type` | Where present, identify the metered object/feature. **[BEST-EFFORT]** — confirm per view. |
| `name` | Metering views: object name for the service type (e.g., warehouse). Meaning is service-type-dependent. |

---

# 16. Caveats and Gotchas

- **Retention:** generally up to **365 days**, but **newer AI views only contain
  data from their launch date forward** — no historical backfill. Long trends may
  be truncated.
- **Latency:** Cortex detail and metering views lag (minutes to hours); **org**
  views lag more. Same-hour parity across views is not guaranteed.
- **Daily vs hourly grain:** never compare an hourly detail slice to a daily billed
  number without aggregating to a common grain.
- **Query completion vs execution window:** AI usage is bucketed by usage window,
  while `QUERY_HISTORY` keys on query timing — a long query can straddle windows.
- **Local time zone (`_LTZ`):** detail/metering timestamps are account-local;
  normalize before cross-source joins or UTC reporting.
- **Org-level vs account-level:** org views aggregate accounts and add currency;
  numbers won't match a single account view exactly.
- **Deprecated-view traps:** legacy `CORTEX_FUNCTIONS_*` lack user/role/tag and may
  miss new AI surfaces — don't treat them as complete.
- **New features not in old views:** AISQL, Analyst, Agents, Cortex Code, and new
  models may not appear in legacy views at all.
- **Missing attribution edge cases:** non-SQL surfaces may have NULL `QUERY_ID` /
  blank `MODEL_NAME`; don't drop those rows.
- **Billing interpretation pitfalls:** detail `CREDITS`/`TOKEN_CREDITS` are
  estimates; only `METERING_DAILY_HISTORY.CREDITS_BILLED` is billed-net.
- **Warehouse vs AI-service cost:** AISQL runs *inside* a SQL query on a warehouse —
  you pay **both** warehouse compute (`WAREHOUSE_METERING_HISTORY`) **and**
  AI-service credits (`AI_SERVICES`). Report them separately.
- **Double-counting traps:** (1) summing AI-service + compute credits; (2) stacking
  overlapping AISQL and AI-functions views; (3) `FLATTEN` of `role_names`/`metrics`
  multiplying credit sums; (4) joining hourly detail to per-query history without
  aggregation.

---

# 17. Final Recommendations

- **Best starting views:** `METERING_DAILY_HISTORY` (billed AI totals) +
  `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` (detail/attribution).
- **Best for apps:** curated semantic views over
  `CORTEX_AI_FUNCTIONS_USAGE_HISTORY`, `CORTEX_AISQL_USAGE_HISTORY`,
  `CORTEX_CODE_*`, and AI-filtered `METERING_DAILY_HISTORY`.
- **Best for agentic platforms:** the same current detail views **plus an explicit
  semantic layer** encoding grain, credit-type, units, join keys, and do-not-sum
  rules (§6, §9).
- **Best for enterprise dashboarding:** `METERING_DAILY_HISTORY` for totals,
  detail views for drill-down, `ORGANIZATION_USAGE` for multi-account + currency.
- **Best for FinOps:** `METERING_DAILY_HISTORY` (billed) reconciled with detail for
  showback/chargeback; `WAREHOUSE_METERING_HISTORY` for the compute side.
- **Best for governance:** `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` (user/role/tag) +
  `QUERY_HISTORY`.
- **Recommended reporting architecture [INTERPRETATION]:** land detail and metering
  into per-grain fact views (`fct_ai_usage_hourly`, `fct_ai_billed_daily`,
  `fct_warehouse_hourly`), expose a conformed dimension set (date/user/role/
  warehouse/model/function/tag/service_type), and reconcile detail↔billed daily.
- **Recommended semantic abstraction for Copilot/Cowork [INTERPRETATION]:** a thin
  metric layer with synonyms ("AI spend"→billed AI credits), enforced credit-type
  labels, whitelisted views/columns, and pinned vetted SQL templates (§12).

---

# 18. One-Page Cheat Sheet

| If you want… | Use |
|--------------|-----|
| **Billed AI totals** (the money number) | `METERING_DAILY_HISTORY` → `CREDITS_BILLED`, filter AI `SERVICE_TYPE`s |
| **Hourly AI billed trend / anomalies** | `METERING_HISTORY`, filter AI `SERVICE_TYPE`s |
| **Token-level AI detail + attribution** | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` |
| **AISQL token economics** | `CORTEX_AISQL_USAGE_HISTORY` |
| **Cortex Code spend (CLI + UI)** | `CORTEX_CODE_CLI_USAGE_HISTORY` ∪ `CORTEX_CODE_SNOWSIGHT_USAGE_HISTORY` |
| **Query-level AI troubleshooting** | detail view `QUERY_ID` → `QUERY_HISTORY` |
| **Per-query compute attribution** | `QUERY_ATTRIBUTION_HISTORY` |
| **Warehouse correlation / compute cost** | `WAREHOUSE_METERING_HISTORY` (by `WAREHOUSE_ID`) |
| **Spend by user / role / app** | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` (`USER_*`, `ROLE_NAMES`, `QUERY_TAG`) |
| **Organization-wide rollups + currency** | `ORGANIZATION_USAGE.METERING_DAILY_HISTORY`, `USAGE_IN_CURRENCY_DAILY` |
| **Migrating legacy dashboards** | `CORTEX_FUNCTIONS[_QUERY]_USAGE_HISTORY` → `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` |
| **Top models / functions** | `CORTEX_AI_FUNCTIONS_USAGE_HISTORY` group by `MODEL_NAME` / `FUNCTION_NAME` |

**Three rules to never break:**
1. **Billed ≠ detail.** Money = `METERING_DAILY_HISTORY.CREDITS_BILLED`; detail
   `CREDITS`/`TOKEN_CREDITS` are estimates.
2. **AI-service credits ≠ warehouse compute credits.** Report separately.
3. **Verify against the live account** (`DESC VIEW`) before trusting any column
   name in this document.

---

> **Maintenance note:** Because Snowflake evolves AI/Cortex Account Usage views
> rapidly, re-validate this document against
> `https://docs.snowflake.com/en/sql-reference/account-usage` and your account's
> `SHOW VIEWS IN SCHEMA SNOWFLAKE.ACCOUNT_USAGE` output at least quarterly.
