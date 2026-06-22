# Snowflake Implementation Patterns for DAMA DMBOK2 Knowledge Areas

> A technical companion mapping DAMA's 11 knowledge areas to concrete Snowflake implementation patterns — SQL examples, architecture decisions, failure modes, and cost-aware checklists. This is not a governance program guide; it assumes organizational governance foundations are already in place.

---

## Table of Contents

- [Prerequisites & Scope](#prerequisites--scope)

1. [Data Governance](#1-data-governance)
2. [Data Architecture](#2-data-architecture)
3. [Data Modeling & Design](#3-data-modeling--design)
4. [Data Storage & Operations](#4-data-storage--operations)
5. [Data Security](#5-data-security)
6. [Data Integration & Interoperability](#6-data-integration--interoperability)
7. [Documents & Content Management](#7-documents--content-management)
8. [Reference & Master Data](#8-reference--master-data)
9. [Data Warehousing & Business Intelligence](#9-data-warehousing--business-intelligence)
10. [Metadata Management](#10-metadata-management)
11. [Data Quality](#11-data-quality)

- [Implementation Maturity Model](#implementation-maturity-model)
- [Summary: Quick Reference](#summary-dama--snowflake-quick-reference)

---

## Prerequisites & Scope

**What this guide IS:**
- A technical implementation companion for teams that already understand DAMA principles
- Runnable SQL patterns organized by governance purpose
- A bridge between "what governance requires" and "how Snowflake enables it"

**What this guide is NOT:**
- A governance program guide (no operating models, councils, or change management)
- A replacement for DMBOK2 (read the book for theory, use this for implementation)
- A complete solution (features without organizational scaffolding produce shelfware)

**Before using this guide, you should already have:**
- A data governance operating model (centralized, federated, or hybrid)
- Defined data ownership and stewardship roles (people, not Snowflake roles)
- Policy frameworks for classification, retention, access, and quality
- A prioritized list of data domains to govern (don't boil the ocean)
- Executive sponsorship and a governance council or equivalent decision body

**If you don't have those yet**, start with DMBOK2 Chapters 3 (Governance) and 4 (Architecture), establish organizational foundations, then return here for implementation.

---

## 1. Data Governance

### DAMA Principle

Data Governance is the foundational discipline — the exercise of authority, control, and shared decision-making over data assets. It establishes who can take what actions, with what data, in what situations, using what methods. Governance operates at strategic (policies/standards), tactical (processes/procedures), and operational (day-to-day stewardship) levels.

Effective governance requires a Data Governance Council, data stewards, data owners, and data custodians collaborating to define policies, resolve issues, and ensure compliance across all other DAMA knowledge areas.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Governance org structure | ORGADMIN, ACCOUNTADMIN, SYSADMIN, custom roles | `CREATE ROLE`, `GRANT ROLE` |
| RBAC model | Role-based access control (hierarchical) | `GRANT PRIVILEGES ON ... TO ROLE` |
| DAC model | Discretionary access via ownership | `GRANT OWNERSHIP ON ... TO ROLE` |
| Policy management | Masking policies, row access policies, aggregation policies | `CREATE MASKING POLICY` |
| Governance dashboards | ACCESS_HISTORY, QUERY_HISTORY, LOGIN_HISTORY | `SELECT * FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` |
| Data stewardship | Object tagging, TAG references | `CREATE TAG`, `ALTER TABLE SET TAG` |
| Horizon governance | Unified governance plane | Snowsight → Governance tab |
| Data classification | CLASSIFY function, sensitivity tags | `SELECT SYSTEM$CLASSIFY(...)` |

### Implementation Best Practices

**Role Hierarchy Design:**

```sql
-- Functional roles (what you can DO)
CREATE ROLE DATA_ENGINEER;
CREATE ROLE DATA_ANALYST;
CREATE ROLE DATA_SCIENTIST;
CREATE ROLE DATA_STEWARD;

-- Access roles (what you can ACCESS)
CREATE ROLE RAW_READ;
CREATE ROLE ANALYTICS_READ;
CREATE ROLE ANALYTICS_WRITE;
CREATE ROLE PII_ACCESS;

-- Hierarchy
GRANT ROLE RAW_READ TO ROLE DATA_ENGINEER;
GRANT ROLE ANALYTICS_WRITE TO ROLE DATA_ENGINEER;
GRANT ROLE ANALYTICS_READ TO ROLE DATA_ANALYST;
GRANT ROLE ANALYTICS_READ TO ROLE DATA_SCIENTIST;
GRANT ROLE PII_ACCESS TO ROLE DATA_STEWARD;

-- Functional to SYSADMIN chain
GRANT ROLE DATA_ENGINEER TO ROLE SYSADMIN;
GRANT ROLE DATA_ANALYST TO ROLE SYSADMIN;
GRANT ROLE DATA_SCIENTIST TO ROLE SYSADMIN;
GRANT ROLE DATA_STEWARD TO ROLE SYSADMIN;
```

**Tag-Based Governance:**

```sql
-- Create governance tags
CREATE TAG IF NOT EXISTS GOVERNANCE.TAGS.DATA_DOMAIN
  ALLOWED_VALUES 'FINANCE', 'HR', 'SALES', 'MARKETING', 'PRODUCT';

CREATE TAG IF NOT EXISTS GOVERNANCE.TAGS.DATA_OWNER
  COMMENT = 'Team or individual owning this data asset';

CREATE TAG IF NOT EXISTS GOVERNANCE.TAGS.SENSITIVITY
  ALLOWED_VALUES 'PUBLIC', 'INTERNAL', 'CONFIDENTIAL', 'RESTRICTED';

CREATE TAG IF NOT EXISTS GOVERNANCE.TAGS.RETENTION_DAYS
  COMMENT = 'Data retention period in days';

-- Apply tags
ALTER TABLE SALES.ORDERS SET TAG
  GOVERNANCE.TAGS.DATA_DOMAIN = 'SALES',
  GOVERNANCE.TAGS.DATA_OWNER = 'revenue-team',
  GOVERNANCE.TAGS.SENSITIVITY = 'CONFIDENTIAL';
```

**Access Monitoring:**

```sql
-- Who accessed PII-tagged objects in last 7 days
SELECT
  qh.USER_NAME,
  ah.QUERY_ID,
  ah.OBJECT_NAME,
  ah.OBJECT_DOMAIN,
  ah.COLUMNS,
  qh.START_TIME
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY ah
JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY qh
  ON ah.QUERY_ID = qh.QUERY_ID
WHERE ah.OBJECT_NAME IN (
  SELECT OBJECT_NAME
  FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
  WHERE TAG_NAME = 'SENSITIVITY'
    AND TAG_VALUE = 'RESTRICTED'
)
AND qh.START_TIME >= DATEADD('day', -7, CURRENT_TIMESTAMP());
```

**Dynamic Masking Policy:**

```sql
CREATE OR REPLACE MASKING POLICY GOVERNANCE.POLICIES.MASK_PII AS
(val STRING) RETURNS STRING ->
  CASE
    WHEN CURRENT_ROLE() IN ('DATA_STEWARD', 'SYSADMIN') THEN val
    WHEN CURRENT_ROLE() IN ('DATA_ANALYST') THEN SHA2(val)
    ELSE '***MASKED***'
  END;

-- Apply to column
ALTER TABLE HR.EMPLOYEES MODIFY COLUMN SSN
  SET MASKING POLICY GOVERNANCE.POLICIES.MASK_PII;
```

### Anti-Patterns

- Using ACCOUNTADMIN for day-to-day operations
- Flat role structure (no hierarchy) — becomes unmanageable at scale
- Granting privileges directly to users instead of roles
- No tagging strategy — governance becomes reactive instead of systematic
- Masking policies with hardcoded role names (use tag-based instead)
- Ignoring ACCOUNT_USAGE views — flying blind on who accesses what
- Single monolithic role per team (no separation of read vs write)

### What Goes Wrong

- **RBAC without periodic access reviews** → role grants accumulate over months; nobody loses access; least-privilege erodes silently
- **Tags without enforcement** → teams tag at creation, never update; tags become decoration, not governance instruments
- **No break-glass procedure** → production incident at 2am, someone uses ACCOUNTADMIN directly, never reverts the escalation
- **Governance without organizational backing** → perfectly configured policies that nobody follows because there's no stewardship council enforcing accountability
- **Classification without downstream action** → you know where PII lives but haven't attached masking policies, so knowledge is inert

### Implementation Checklist

- [ ] Define role hierarchy (functional + access roles)
- [ ] Create tagging taxonomy (domain, owner, sensitivity, retention)
- [ ] Tag all existing objects with at least domain and sensitivity
- [ ] Implement masking policies for PII/sensitive columns
- [ ] Set up ACCESS_HISTORY monitoring queries or dashboards
- [ ] Document governance policies and communicate to teams
- [ ] Schedule quarterly access reviews using LOGIN_HISTORY + GRANTS_TO_USERS
- [ ] Enable Horizon governance features in Snowsight
- [ ] Create a GOVERNANCE database for policies, tags, and audit objects

---

## 2. Data Architecture

### DAMA Principle

Data Architecture defines the blueprint for managing data assets — the specifications that describe existing state, define requirements, guide integration, and control data assets aligned with business strategy. It encompasses enterprise data models, data flow diagrams, technology standards, and naming conventions.

Architecture determines how data flows from sources through integration layers to consumption, how systems share data, how data replicates across regions, and how new sources are onboarded.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Enterprise data model | Database/schema design patterns | Multi-DB medallion layout |
| Data flow architecture | Multi-cluster shared data architecture | Separation of storage & compute |
| Integration architecture | Data Sharing, Snowgrid, Listings | `CREATE SHARE` |
| Technology architecture | Account topology, regions, replication | `CREATE FAILOVER GROUP` |
| Medallion pattern | RAW → STAGING → ANALYTICS databases | Layered DB design |
| Data Vault | Hub/Link/Satellite in staging layer | Schema per pattern |
| Modern formats | Iceberg tables, hybrid tables, external tables | `CREATE ICEBERG TABLE` |
| Data mesh | Per-domain accounts + sharing | Cross-account shares |

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SNOWFLAKE ACCOUNT TOPOLOGY                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │   RAW DB     │    │  STAGING DB  │    │ ANALYTICS DB │          │
│  ├──────────────┤    ├──────────────┤    ├──────────────┤          │
│  │ ERP_SCHEMA   │───▶│ CLEANSED     │───▶│ FINANCE      │          │
│  │ CRM_SCHEMA   │    │ CONFORMED    │    │ SALES        │──┐       │
│  │ WEB_SCHEMA   │    │ VAULT        │    │ MARKETING    │  │       │
│  │ FILES_SCHEMA │    │              │    │ PRODUCT      │  │       │
│  └──────────────┘    └──────────────┘    └──────────────┘  │       │
│                                                             │       │
│  ┌──────────────┐    ┌──────────────┐                      │       │
│  │ GOVERNANCE   │    │  SANDBOX DB  │    ┌──────────────┐  │       │
│  ├──────────────┤    ├──────────────┤    │   SHARES     │◀─┘       │
│  │ POLICIES     │    │ DEV_<USER>   │    ├──────────────┤          │
│  │ TAGS         │    │ (clones)     │    │ → Partner A  │          │
│  │ AUDIT        │    │              │    │ → Partner B  │          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│                                                                     │
│  Warehouses: INGEST_WH │ TRANSFORM_WH │ ANALYTICS_WH │ BI_WH     │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation Best Practices

**Medallion Architecture Setup:**

```sql
-- Bronze (raw ingestion)
CREATE DATABASE RAW;
CREATE SCHEMA RAW.ERP;
CREATE SCHEMA RAW.CRM;
CREATE SCHEMA RAW.WEB_EVENTS;
CREATE SCHEMA RAW.FILES;

-- Silver (cleansed, conformed)
CREATE DATABASE STAGING;
CREATE SCHEMA STAGING.CLEANSED;
CREATE SCHEMA STAGING.CONFORMED;
CREATE SCHEMA STAGING.VAULT;  -- Data Vault hubs/links/satellites

-- Gold (business-ready)
CREATE DATABASE ANALYTICS;
CREATE SCHEMA ANALYTICS.FINANCE;
CREATE SCHEMA ANALYTICS.SALES;
CREATE SCHEMA ANALYTICS.MARKETING;

-- Governance
CREATE DATABASE GOVERNANCE;
CREATE SCHEMA GOVERNANCE.POLICIES;
CREATE SCHEMA GOVERNANCE.TAGS;
CREATE SCHEMA GOVERNANCE.AUDIT;

-- Sandbox (ephemeral dev)
CREATE DATABASE SANDBOX;
```

**Naming Conventions:**

```
Databases:   UPPER_SNAKE (RAW, STAGING, ANALYTICS)
Schemas:     UPPER_SNAKE by domain (FINANCE, HR, SALES)
Tables:      UPPER_SNAKE (DIM_CUSTOMER, FCT_ORDERS, STG_CRM_CONTACTS)
Views:       VW_ prefix (VW_ACTIVE_CUSTOMERS)
Dynamic Tbl: DT_ prefix (DT_DAILY_REVENUE)
Stages:      STG_ prefix (STG_S3_LANDING)
Pipes:       PIPE_ prefix (PIPE_CRM_INGEST)
Tasks:       TSK_ prefix (TSK_DAILY_TRANSFORM)
Streams:     STRM_ prefix (STRM_ORDERS_CDC)
```

**Iceberg Tables:**

```sql
CREATE ICEBERG TABLE ANALYTICS.SALES.HISTORICAL_ORDERS (
  ORDER_ID STRING,
  ORDER_DATE DATE,
  CUSTOMER_ID STRING,
  TOTAL_AMOUNT NUMBER(12,2)
)
  CATALOG = 'SNOWFLAKE'
  EXTERNAL_VOLUME = 'S3_DATA_LAKE'
  BASE_LOCATION = 'orders/historical/';
```

**Replication for DR:**

```sql
-- Primary account
CREATE FAILOVER GROUP PRIMARY_FG
  OBJECT_TYPES = DATABASES, ROLES, WAREHOUSES, INTEGRATIONS
  ALLOWED_DATABASES = RAW, STAGING, ANALYTICS
  ALLOWED_ACCOUNTS = ORG.DR_ACCOUNT
  REPLICATION_SCHEDULE = '10 MINUTE';

-- Secondary account
CREATE FAILOVER GROUP PRIMARY_FG
  AS REPLICA OF ORG.PRIMARY_ACCOUNT.PRIMARY_FG;
```

**Data Sharing:**

```sql
CREATE SHARE PARTNER_ANALYTICS;
GRANT USAGE ON DATABASE ANALYTICS TO SHARE PARTNER_ANALYTICS;
GRANT USAGE ON SCHEMA ANALYTICS.SALES TO SHARE PARTNER_ANALYTICS;
GRANT SELECT ON VIEW ANALYTICS.SALES.VW_PARTNER_METRICS TO SHARE PARTNER_ANALYTICS;

ALTER SHARE PARTNER_ANALYTICS ADD ACCOUNTS = ORG.PARTNER_ACCOUNT;
```

### Anti-Patterns

- Single database for everything (no separation of concerns)
- No naming conventions — chaos at scale
- Granting direct table access in shares (use secure views instead)
- Skipping the staging layer (raw → analytics direct)
- Per-user databases instead of schema-level isolation
- No DR/replication strategy until it's too late
- Mixing workloads on a single warehouse

### What Goes Wrong

- **No shared dimension ownership** → FINANCE and SALES each build their own `DIM_CUSTOMER`; numbers diverge; executives lose trust
- **Iceberg without understanding consistency model** → external catalog updates lag behind writes; queries return stale/partial data silently
- **Data Sharing without cross-cloud planning** → discover at go-live that consumers are in a different cloud/region; forced into expensive replication
- **RAW → ANALYTICS with no TRANSFORM layer** → analysts write spaghetti SQL against raw schemas; logic duplicated across 50 dashboards
- **No RTO/RPO targets defined** → failover groups exist but nobody tested recovery; actual failover takes 4 hours instead of expected 15 minutes

### Implementation Checklist

- [ ] Design database/schema topology (medallion or domain-based)
- [ ] Document and enforce naming conventions
- [ ] Create dedicated warehouses per workload type
- [ ] Set up replication/failover groups for critical databases
- [ ] Implement data sharing for cross-account consumption
- [ ] Create Iceberg tables for open-format interoperability
- [ ] Establish sandbox/dev environment using zero-copy clones
- [ ] Document data flow diagrams source → raw → staging → analytics

---

## 3. Data Modeling & Design

### DAMA Principle

Data Modeling operates at three levels: conceptual (business entities and relationships), logical (attributes, keys, normalization), and physical (platform-specific storage). Models are communication tools bridging business understanding and technical implementation. Schema patterns — dimensional, normalized, Data Vault — each serve specific use cases.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Conceptual models | Schema design, VARIANT for flexible schemas | Iterative schema evolution |
| Logical models | Views, Secure Views, Dynamic Tables | `CREATE DYNAMIC TABLE` |
| Physical models | Clustering keys, search optimization, micro-partitions | `CLUSTER BY` |
| Star schema | Fact + dimension tables | Standard DDL |
| Data Vault | Hubs, Links, Satellites | Hash keys + load timestamps |
| Semi-structured | VARIANT, ARRAY, OBJECT types | `col VARIANT` |
| Flattening | LATERAL FLATTEN | `FLATTEN(input => col)` |
| Temporal models | Streams, Time Travel | `CREATE STREAM ON TABLE` |
| SCD Type 2 | MERGE + Streams | `MERGE INTO ... WHEN MATCHED AND ...` |
| Search optimization | Search Optimization Service | `ALTER TABLE ADD SEARCH OPTIMIZATION` |

### Implementation Best Practices

**Star Schema DDL:**

```sql
-- Dimension
CREATE TABLE ANALYTICS.SALES.DIM_CUSTOMER (
  CUSTOMER_SK NUMBER IDENTITY PRIMARY KEY,
  CUSTOMER_BK STRING NOT NULL,        -- business key
  CUSTOMER_NAME STRING,
  EMAIL STRING,
  SEGMENT STRING,
  REGION STRING,
  EFFECTIVE_FROM TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  EFFECTIVE_TO TIMESTAMP_NTZ DEFAULT '9999-12-31',
  IS_CURRENT BOOLEAN DEFAULT TRUE
)
CLUSTER BY (REGION, SEGMENT);

-- Fact
CREATE TABLE ANALYTICS.SALES.FCT_ORDERS (
  ORDER_SK NUMBER IDENTITY PRIMARY KEY,
  ORDER_DATE DATE NOT NULL,
  CUSTOMER_SK NUMBER REFERENCES DIM_CUSTOMER(CUSTOMER_SK),
  PRODUCT_SK NUMBER,
  QUANTITY NUMBER,
  UNIT_PRICE NUMBER(12,2),
  DISCOUNT_PCT NUMBER(5,2),
  TOTAL_AMOUNT NUMBER(12,2)
)
CLUSTER BY (ORDER_DATE);
```

**Data Vault Pattern:**

> **When to use Data Vault:** Multiple source systems feeding the same business entities; regulatory audit requirements demanding full load traceability; high-change environments where schema evolution is frequent. Data Vault shines when you need to integrate, not just store.
>
> **When NOT to use Data Vault:** Single-source analytics, small teams without dedicated modeling expertise, or when query performance matters more than auditability. The join complexity of Hub→Link→Satellite is real — in Snowflake's MPP architecture, these multi-way joins on hash keys can be expensive without proper clustering. If your team will abandon it within 6 months, start with dimensional modeling.

```sql
-- Hub (business keys)
CREATE TABLE STAGING.VAULT.HUB_CUSTOMER (
  HUB_CUSTOMER_HK BINARY(20) PRIMARY KEY,  -- SHA1 hash of BK
  CUSTOMER_BK STRING NOT NULL,
  LOAD_DTS TIMESTAMP_NTZ NOT NULL,
  RECORD_SOURCE STRING NOT NULL
);

-- Link (relationships)
CREATE TABLE STAGING.VAULT.LNK_ORDER_CUSTOMER (
  LNK_ORDER_CUSTOMER_HK BINARY(20) PRIMARY KEY,
  HUB_ORDER_HK BINARY(20) NOT NULL,
  HUB_CUSTOMER_HK BINARY(20) NOT NULL,
  LOAD_DTS TIMESTAMP_NTZ NOT NULL,
  RECORD_SOURCE STRING NOT NULL
);

-- Satellite (descriptive attributes)
CREATE TABLE STAGING.VAULT.SAT_CUSTOMER_DETAILS (
  HUB_CUSTOMER_HK BINARY(20) NOT NULL,
  LOAD_DTS TIMESTAMP_NTZ NOT NULL,
  LOAD_END_DTS TIMESTAMP_NTZ DEFAULT '9999-12-31',
  HASH_DIFF BINARY(20) NOT NULL,  -- hash of payload for change detection
  CUSTOMER_NAME STRING,
  EMAIL STRING,
  PHONE STRING,
  RECORD_SOURCE STRING NOT NULL,
  PRIMARY KEY (HUB_CUSTOMER_HK, LOAD_DTS)
)
CLUSTER BY (HUB_CUSTOMER_HK);
-- Clustering on hub hash key is critical for Vault join performance.
-- Without it, satellite lookups full-scan on large tables.
```

**Dynamic Tables (layered transformation):**

```sql
CREATE DYNAMIC TABLE ANALYTICS.SALES.DT_DAILY_REVENUE
  TARGET_LAG = '1 hour'
  WAREHOUSE = TRANSFORM_WH
AS
SELECT
  ORDER_DATE,
  d.REGION,
  d.SEGMENT,
  COUNT(*) AS ORDER_COUNT,
  SUM(f.TOTAL_AMOUNT) AS TOTAL_REVENUE,
  AVG(f.TOTAL_AMOUNT) AS AVG_ORDER_VALUE
FROM STAGING.CLEANSED.ORDERS f
JOIN STAGING.CLEANSED.CUSTOMERS d ON f.CUSTOMER_BK = d.CUSTOMER_BK
GROUP BY 1, 2, 3;
```

> **Cost note — TARGET_LAG:** Lower lag = more frequent refreshes = higher compute cost. A `TARGET_LAG = '1 minute'` table costs ~60x more compute than `'1 hour'`. Match lag to actual business consumption frequency. If a dashboard refreshes hourly, a 5-minute lag wastes money. Use `DOWNSTREAM` only when a consuming Dynamic Table exists; otherwise the table never refreshes.

**Semi-Structured with FLATTEN:**

```sql
-- Table with VARIANT
CREATE TABLE RAW.WEB_EVENTS.CLICKSTREAM (
  EVENT_ID STRING,
  RECEIVED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  PAYLOAD VARIANT
);

-- Query nested arrays
SELECT
  EVENT_ID,
  PAYLOAD:user_id::STRING AS USER_ID,
  PAYLOAD:event_type::STRING AS EVENT_TYPE,
  f.value:product_id::STRING AS PRODUCT_ID,
  f.value:quantity::NUMBER AS QUANTITY
FROM RAW.WEB_EVENTS.CLICKSTREAM,
LATERAL FLATTEN(input => PAYLOAD:items) f
WHERE PAYLOAD:event_type = 'purchase';
```

**SCD Type 2 MERGE:**

```sql
-- Stream captures changes
CREATE STREAM STRM_CUSTOMER_CDC ON TABLE STAGING.CLEANSED.CUSTOMERS;

-- MERGE for SCD2
MERGE INTO ANALYTICS.SALES.DIM_CUSTOMER tgt
USING (
  SELECT * FROM STRM_CUSTOMER_CDC
  WHERE METADATA$ACTION = 'INSERT'
) src
ON tgt.CUSTOMER_BK = src.CUSTOMER_BK AND tgt.IS_CURRENT = TRUE
WHEN MATCHED AND tgt.CUSTOMER_NAME != src.CUSTOMER_NAME THEN UPDATE SET
  EFFECTIVE_TO = CURRENT_TIMESTAMP(),
  IS_CURRENT = FALSE
WHEN NOT MATCHED THEN INSERT (
  CUSTOMER_BK, CUSTOMER_NAME, EMAIL, SEGMENT, REGION
) VALUES (
  src.CUSTOMER_BK, src.CUSTOMER_NAME, src.EMAIL, src.SEGMENT, src.REGION
);
```

**Search Optimization:**

```sql
ALTER TABLE RAW.WEB_EVENTS.CLICKSTREAM
  ADD SEARCH OPTIMIZATION ON EQUALITY(EVENT_ID),
  ADD SEARCH OPTIMIZATION ON SUBSTRING(PAYLOAD:user_id),
  ADD SEARCH OPTIMIZATION ON GEO(PAYLOAD:location);
```

### Anti-Patterns

- Storing everything in VARIANT without extracting high-cardinality filter columns
- Clustering on columns with very low cardinality (< 10 distinct values alone)
- No business keys in dimensional models (only surrogate keys, losing traceability)
- Skipping SCD handling — overwriting history silently
- Dynamic tables with `TARGET_LAG = '0 seconds'` (defeats the purpose, use streams/tasks)
- Deeply nested FLATTEN chains without materializing intermediate results

### What Goes Wrong

- **Data Vault without the team to maintain it** → hubs/links/satellites proliferate; nobody understands the joins; query performance degrades; team abandons it for direct-to-dimensional
- **Star schema without clustering** → large fact tables full-scan on every dashboard query; users complain about latency; you add warehouse size instead of fixing the model
- **Dynamic Tables with aggressive lag on low-value data** → `TARGET_LAG = '1 minute'` on a table queried once daily; paying for continuous compute that delivers no business value
- **SCD Type 2 without hash-based change detection** → comparing every column is fragile; schema changes break the pipeline silently
- **No conceptual model alignment** → physical model diverges from business language; "order" means different things in 3 different schemas

### Implementation Checklist

- [ ] Define conceptual data model with business stakeholders
- [ ] Choose schema pattern per layer (vault for staging, dimensional for analytics)
- [ ] Implement clustering keys on large fact tables (by date + high-filter columns)
- [ ] Set up dynamic tables for automated transformation chains
- [ ] Create streams for CDC on source-of-truth tables
- [ ] Enable search optimization on point-lookup tables
- [ ] Document SCD strategy (Type 1 vs 2) per dimension
- [ ] Validate semi-structured schema evolution strategy

---

## 4. Data Storage & Operations

### DAMA Principle

Data Storage and Operations covers the design, implementation, and management of stored data to maximize value throughout the lifecycle. This includes technology selection, physical storage management, backup/recovery, archival/retention, and ongoing operational support. The goal: data is available, performant, protected, and cost-efficient.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Storage management | Micro-partitions, auto compression, columnar | Fully managed |
| Data retention | Time Travel (0–90 days) | `DATA_RETENTION_TIME_IN_DAYS` |
| Fail-safe | 7-day fail-safe (non-configurable) | Snowflake-managed recovery |
| Backup & recovery | UNDROP, zero-copy CLONE, replication | `UNDROP TABLE` |
| Archival | External stages, Iceberg for cold data | `CREATE STAGE` |
| Storage monitoring | STORAGE_USAGE, TABLE_STORAGE_METRICS | ACCOUNT_USAGE views |
| Lifecycle management | Transient/temporary tables, retention settings | `CREATE TRANSIENT TABLE` |
| Capacity planning | WAREHOUSE_METERING_HISTORY | Cost views |

### Implementation Best Practices

**Time Travel Configuration:**

```sql
-- Production: max retention
ALTER DATABASE ANALYTICS SET DATA_RETENTION_TIME_IN_DAYS = 90;

-- Staging: moderate retention
ALTER DATABASE STAGING SET DATA_RETENTION_TIME_IN_DAYS = 14;

-- Raw landing: minimal (data can be re-ingested)
ALTER DATABASE RAW SET DATA_RETENTION_TIME_IN_DAYS = 1;

-- Transient tables for ephemeral workloads (0 fail-safe cost)
CREATE TRANSIENT TABLE SANDBOX.DEV.TEMP_ANALYSIS (
  ...
) DATA_RETENTION_TIME_IN_DAYS = 0;
```

> **Cost note — Time Travel & Fail-safe storage:** Every retained change consumes storage. A 10TB table with 90-day retention + 7-day Fail-safe can accumulate 2-5x its active size in historical storage. Use 90 days only for critical analytics tables. Use 1 day for re-ingestable raw data. Use TRANSIENT (0 Fail-safe) for staging/scratch. Monitor `TABLE_STORAGE_METRICS` monthly — storage creep is the #1 surprise on Snowflake invoices.

**Zero-Copy Clone for Dev/Test:**

```sql
-- Full database clone (instant, no extra storage until divergence)
CREATE DATABASE STAGING_DEV CLONE STAGING;

-- Schema-level clone
CREATE SCHEMA ANALYTICS.SALES_TEST CLONE ANALYTICS.SALES;

-- Table-level clone at a point in time
CREATE TABLE ANALYTICS.SALES.ORDERS_SNAPSHOT
  CLONE ANALYTICS.SALES.FCT_ORDERS
  AT (TIMESTAMP => '2024-01-15 08:00:00'::TIMESTAMP);
```

**Time Travel Queries:**

```sql
-- Query data as it was 30 minutes ago
SELECT * FROM ANALYTICS.SALES.FCT_ORDERS
  AT (OFFSET => -1800);

-- Query before a specific statement (undo accidental UPDATE)
SELECT * FROM ANALYTICS.SALES.DIM_CUSTOMER
  BEFORE (STATEMENT => '01abc123-0000-0000-0000-000000000000');

-- Restore accidentally dropped table
UNDROP TABLE ANALYTICS.SALES.FCT_ORDERS;
UNDROP SCHEMA ANALYTICS.SALES;
UNDROP DATABASE ANALYTICS;
```

**Storage Monitoring:**

```sql
-- Top 20 tables by storage
SELECT
  TABLE_CATALOG || '.' || TABLE_SCHEMA || '.' || TABLE_NAME AS FQN,
  ACTIVE_BYTES / POWER(1024,3) AS ACTIVE_GB,
  TIME_TRAVEL_BYTES / POWER(1024,3) AS TIME_TRAVEL_GB,
  FAILSAFE_BYTES / POWER(1024,3) AS FAILSAFE_GB,
  (ACTIVE_BYTES + TIME_TRAVEL_BYTES + FAILSAFE_BYTES) / POWER(1024,3) AS TOTAL_GB
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
WHERE ACTIVE_BYTES > 0
ORDER BY TOTAL_GB DESC
LIMIT 20;

-- Monthly storage trend
SELECT
  DATE_TRUNC('month', USAGE_DATE) AS MONTH,
  AVG(STORAGE_BYTES) / POWER(1024,4) AS AVG_STORAGE_TB,
  AVG(STAGE_BYTES) / POWER(1024,4) AS AVG_STAGE_TB,
  AVG(FAILSAFE_BYTES) / POWER(1024,4) AS AVG_FAILSAFE_TB
FROM SNOWFLAKE.ACCOUNT_USAGE.STORAGE_USAGE
GROUP BY 1
ORDER BY 1;
```

**Replication for DR:**

```sql
CREATE FAILOVER GROUP DR_CRITICAL
  OBJECT_TYPES = DATABASES, ROLES, USERS, WAREHOUSES
  ALLOWED_DATABASES = ANALYTICS, GOVERNANCE
  ALLOWED_ACCOUNTS = ORG.DR_ACCOUNT
  REPLICATION_SCHEDULE = '10 MINUTE';
```

### Anti-Patterns

- 90-day retention on large transient/staging tables (massive cost for no benefit)
- Never monitoring TABLE_STORAGE_METRICS (cost surprises)
- Using CLONE as a "backup" without lifecycle management (clones diverge, consuming storage)
- Storing large temporary datasets in permanent tables (use transient)
- No DR strategy — single account, single region
- Dropping and recreating tables instead of TRUNCATE (loses time travel on original)

### What Goes Wrong

- **Time Travel at 1 day on critical tables** → someone drops a table Saturday; discovered Monday; UNDROP window expired; data gone permanently
- **Clone sprawl without lifecycle management** → dev clones from 6 months ago consuming storage; nobody tracks them; surprise bills
- **No storage monitoring** → FAIL_SAFE + TIME_TRAVEL storage for large transient workloads costs 3x what teams expect
- **Transient tables for important data** → saves storage cost until someone accidentally drops it; no Fail-safe recovery possible
- **No tested restore procedure** → Time Travel and UNDROP exist but nobody has practiced; actual recovery during incident takes hours of trial-and-error

### Implementation Checklist

- [ ] Set retention policies per database/schema based on criticality
- [ ] Implement zero-copy clone workflow for dev/test environments
- [ ] Create storage monitoring dashboard (TABLE_STORAGE_METRICS)
- [ ] Define and document backup/recovery procedures using Time Travel + UNDROP
- [ ] Set up failover groups for critical databases
- [ ] Schedule monthly storage cost reviews
- [ ] Use transient tables for staging/ephemeral workloads
- [ ] Document table lifecycle (when to archive, when to drop)

---

## 5. Data Security

### DAMA Principle

Data Security ensures privacy, confidentiality, integrity, and availability through authentication (identity), authorization (access control), encryption (protection), auditing (monitoring), and compliance (regulatory). Defense in depth — multiple layered controls so no single failure compromises data.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Authentication | MFA, SSO (SAML 2.0), OAuth, key-pair | `ALTER USER SET RSA_PUBLIC_KEY` |
| Authorization | RBAC, DAC, database roles, future grants | `GRANT ... TO ROLE` |
| Encryption at rest | AES-256, automatic, always-on | Managed (no config needed) |
| Encryption in transit | TLS 1.2+ enforced | Always enabled |
| Customer-managed keys | Tri-Secret Secure | Business Critical+ edition |
| Network security | Network policies, AWS PrivateLink, Azure Private Link | `CREATE NETWORK POLICY` |
| Data classification | CLASSIFY function, system tags | `SYSTEM$CLASSIFY(...)` |
| Dynamic masking | Masking policies on columns | `CREATE MASKING POLICY` |
| Row-level security | Row access policies | `CREATE ROW ACCESS POLICY` |
| Column-level security | Projection policies | `CREATE PROJECTION POLICY` |
| Audit & compliance | ACCESS_HISTORY, LOGIN_HISTORY, Trust Center | ACCOUNT_USAGE views |
| Privacy | Aggregation policies | `CREATE AGGREGATION POLICY` |
| Identity provisioning | SCIM integration | SCIM API endpoint |

### Implementation Best Practices

**Network Policy:**

```sql
CREATE NETWORK POLICY CORP_ONLY
  ALLOWED_IP_LIST = ('10.0.0.0/8', '172.16.0.0/12', '203.0.113.0/24')
  BLOCKED_IP_LIST = ('10.0.0.1')  -- specific blocked IPs
  COMMENT = 'Allow only corporate network ranges';

-- Apply at account level
ALTER ACCOUNT SET NETWORK_POLICY = CORP_ONLY;

-- Or per-user for service accounts
ALTER USER SVC_ETL SET NETWORK_POLICY = SVC_NETWORK_POLICY;
```

**Tag-Based Masking (classify → tag → mask):**

```sql
-- 1. Classify columns automatically
SELECT SYSTEM$CLASSIFY('ANALYTICS.SALES.DIM_CUSTOMER', {'auto_tag': true});

-- 2. Create tag-based masking policy
CREATE MASKING POLICY GOVERNANCE.POLICIES.TAG_BASED_MASK AS
(val STRING) RETURNS STRING ->
  CASE
    WHEN SYSTEM$GET_TAG_ON_CURRENT_COLUMN('GOVERNANCE.TAGS.SENSITIVITY') = 'RESTRICTED'
      AND NOT IS_ROLE_IN_SESSION('PII_ACCESS')
    THEN '***MASKED***'
    ELSE val
  END;

-- 3. Apply policy via tag reference
ALTER TAG GOVERNANCE.TAGS.SENSITIVITY SET
  MASKING POLICY GOVERNANCE.POLICIES.TAG_BASED_MASK;
```

**Row Access Policy:**

```sql
CREATE ROW ACCESS POLICY GOVERNANCE.POLICIES.REGION_FILTER AS
(region_col STRING) RETURNS BOOLEAN ->
  CURRENT_ROLE() IN ('SYSADMIN', 'DATA_STEWARD')
  OR region_col = CURRENT_SESSION()::VARIANT:region::STRING
  OR EXISTS (
    SELECT 1 FROM GOVERNANCE.AUDIT.USER_REGION_ACCESS
    WHERE USER_NAME = CURRENT_USER()
      AND ALLOWED_REGION = region_col
  );

ALTER TABLE ANALYTICS.SALES.FCT_ORDERS
  ADD ROW ACCESS POLICY GOVERNANCE.POLICIES.REGION_FILTER ON (REGION);
```

**Auditing — Who Accessed What:**

```sql
-- Sensitive data access report
SELECT
  USER_NAME,
  QUERY_ID,
  QUERY_START_TIME,
  DIRECT_OBJECTS_ACCESSED,
  BASE_OBJECTS_ACCESSED,
  POLICIES_REFERENCED
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE ARRAY_SIZE(POLICIES_REFERENCED) > 0  -- masking was applied
  AND QUERY_START_TIME >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY QUERY_START_TIME DESC;

-- Failed login attempts
SELECT
  USER_NAME,
  CLIENT_IP,
  REPORTED_CLIENT_TYPE,
  ERROR_CODE,
  ERROR_MESSAGE,
  EVENT_TIMESTAMP
FROM SNOWFLAKE.ACCOUNT_USAGE.LOGIN_HISTORY
WHERE IS_SUCCESS = 'NO'
  AND EVENT_TIMESTAMP >= DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY EVENT_TIMESTAMP DESC;
```

**Key-Pair Authentication (service accounts):**

```sql
ALTER USER SVC_ETL SET
  RSA_PUBLIC_KEY = 'MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...'
  RSA_PUBLIC_KEY_2 = 'MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA...'  -- rotation
  COMMENT = 'ETL service account - key-pair auth only';

-- Disable password auth for service accounts
ALTER USER SVC_ETL SET DISABLED = FALSE;
ALTER USER SVC_ETL SET PASSWORD = NULL;  -- no password login possible
```

**Data Exfiltration Detection:**

```sql
-- Detect bulk reads from sensitive tables (potential exfiltration)
SELECT
  USER_NAME,
  QUERY_ID,
  QUERY_START_TIME,
  ROWS_PRODUCED,
  BYTES_SCANNED,
  f.value:objectName::STRING AS TABLE_ACCESSED
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY q,
  LATERAL FLATTEN(input => PARSE_JSON(q.ACCESS_HISTORY):objectsAccessed) f
WHERE ROWS_PRODUCED > 1000000  -- threshold: 1M+ rows extracted
  AND QUERY_START_TIME >= DATEADD('day', -1, CURRENT_TIMESTAMP())
  AND f.value:objectName::STRING ILIKE '%PII%' OR f.value:objectName::STRING ILIKE '%CUSTOMER%'
ORDER BY ROWS_PRODUCED DESC;

-- Detect COPY INTO stage operations (data leaving Snowflake)
SELECT
  USER_NAME,
  QUERY_TEXT,
  QUERY_START_TIME,
  EXECUTION_STATUS
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE QUERY_TEXT ILIKE '%COPY INTO @%'
  AND QUERY_START_TIME >= DATEADD('day', -1, CURRENT_TIMESTAMP())
ORDER BY QUERY_START_TIME DESC;
```

**Privilege Escalation Audit:**

```sql
-- Detect MANAGE GRANTS usage (escalation risk)
SELECT
  USER_NAME,
  ROLE_NAME,
  QUERY_TEXT,
  EXECUTION_STATUS,
  START_TIME
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE QUERY_TEXT ILIKE '%GRANT%ACCOUNTADMIN%'
   OR QUERY_TEXT ILIKE '%MANAGE GRANTS%'
   AND START_TIME >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY START_TIME DESC;

-- Roles with dangerous privilege combinations
SELECT
  GRANTEE_NAME AS ROLE_NAME,
  PRIVILEGE,
  GRANTED_ON,
  NAME AS OBJECT_NAME
FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
WHERE PRIVILEGE IN ('MANAGE GRANTS', 'CREATE ROLE', 'OWNERSHIP')
  AND DELETED_ON IS NULL
ORDER BY GRANTEE_NAME;
```

**External Integration Attack Surface:**

```sql
-- Audit all external stages (data egress points)
SELECT STAGE_NAME, STAGE_TYPE, STAGE_URL, STAGE_OWNER
FROM SNOWFLAKE.ACCOUNT_USAGE.STAGES
WHERE STAGE_TYPE != 'Internal Named'
  AND DELETED IS NULL;

-- Audit external functions (code execution outside Snowflake)
SELECT FUNCTION_NAME, FUNCTION_SCHEMA, FUNCTION_OWNER, FUNCTION_LANGUAGE
FROM SNOWFLAKE.ACCOUNT_USAGE.FUNCTIONS
WHERE FUNCTION_LANGUAGE = 'EXTERNAL'
  AND DELETED IS NULL;

-- Audit active shares (data leaving your account)
SELECT DATABASE_NAME, OWNER, CONSUMER_ACCOUNT, CONSUMER_REGION
FROM SNOWFLAKE.ACCOUNT_USAGE.DATA_TRANSFER_HISTORY
WHERE TRANSFER_TYPE = 'REPLICATION'
  AND START_TIME >= DATEADD('day', -30, CURRENT_TIMESTAMP());
```

> **Supply chain risk:** Every UDF, stored procedure, and external function is a code execution vector. Restrict `CREATE FUNCTION` and `CREATE PROCEDURE` privileges to vetted roles. Audit `FUNCTIONS` and `PROCEDURES` views regularly. Consider a deployment pipeline that reviews procedure code before granting USAGE.

### Anti-Patterns

- Using ACCOUNTADMIN for service accounts
- No MFA enforcement for human users
- Network policies that allow 0.0.0.0/0
- Masking policies with `WHEN CURRENT_USER() = 'john'` (use roles, not users)
- Row access policies that scan large lookup tables without optimization
- No separation between security admin and data admin roles
- Skipping CLASSIFY — manually tracking PII is error-prone and incomplete
- Granting future privileges too broadly (`GRANT ALL ON FUTURE TABLES IN DATABASE`)

### What Goes Wrong

- **Masking without COPY INTO controls** → data is masked in SELECT but analysts export unmasked data via `COPY INTO @stage`; exfiltration through the side door
- **Network policies without service account coverage** → human access is locked to VPN but ETL service accounts connect from anywhere
- **No privilege escalation auditing** → SYSADMIN with MANAGE GRANTS can escalate to ACCOUNTADMIN; nobody monitors for this pattern in ACCESS_HISTORY
- **Row access policies on hot tables without optimization** → policy joins a large lookup table on every query; p95 latency doubles; users bypass with direct grants
- **Security theater without exfiltration detection** → policies exist on paper, but nobody alerts on bulk `SELECT *` from PII tables followed by stage writes

### Implementation Checklist

- [ ] Enforce MFA for all human users
- [ ] Implement network policies (account-level + user-level for service accounts)
- [ ] Set up key-pair authentication for all service accounts
- [ ] Run CLASSIFY on all databases to identify sensitive data
- [ ] Implement tag-based masking policies
- [ ] Create row access policies for multi-tenant/regional data
- [ ] Set up audit dashboards on ACCESS_HISTORY and LOGIN_HISTORY
- [ ] Configure SCIM for automated user provisioning
- [ ] Document security incident response using UNDROP + Time Travel
- [ ] Review Trust Center findings monthly

---

## 6. Data Integration & Interoperability

### DAMA Principle

Data Integration encompasses the processes to move and consolidate data within and between systems. The goal: data flows reliably from source to target while preserving meaning, quality, and lineage. ELT (Extract, Load, Transform) leverages cloud compute for schema-on-read. Interoperability ensures semantic consistency and standards adherence across systems.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Batch integration | COPY INTO, external stages | `COPY INTO table FROM @stage` |
| Real-time/streaming | Snowpipe, Snowpipe Streaming | `CREATE PIPE` |
| Change data capture | Streams + Tasks | `CREATE STREAM ON TABLE` |
| ELT transforms | Dynamic tables | `CREATE DYNAMIC TABLE` |
| API integration | External functions, external access integration | `CREATE EXTERNAL FUNCTION` |
| Data virtualization | External tables (no data movement) | `CREATE EXTERNAL TABLE` |
| Data exchange | Marketplace, private listings, clean rooms | Snowsight UI |
| Connectors | Kafka, Spark, JDBC/ODBC | Snowflake connector configs |
| File formats | Parquet, Avro, ORC, JSON, CSV | `CREATE FILE FORMAT` |
| Error handling | VALIDATION_MODE, COPY error options | `ON_ERROR = CONTINUE` |

### Implementation Best Practices

**Snowpipe Setup:**

```sql
-- File format
CREATE FILE FORMAT RAW.PUBLIC.JSON_FORMAT
  TYPE = JSON
  STRIP_OUTER_ARRAY = TRUE
  COMPRESSION = AUTO;

-- External stage
CREATE STAGE RAW.PUBLIC.STG_S3_EVENTS
  URL = 's3://company-data-lake/events/'
  STORAGE_INTEGRATION = S3_INTEGRATION
  FILE_FORMAT = RAW.PUBLIC.JSON_FORMAT;

-- Pipe (auto-ingest with SQS notification)
CREATE PIPE RAW.PUBLIC.PIPE_EVENTS AUTO_INGEST = TRUE AS
  COPY INTO RAW.WEB_EVENTS.CLICKSTREAM (EVENT_ID, RECEIVED_AT, PAYLOAD)
  FROM (
    SELECT
      $1:event_id::STRING,
      CURRENT_TIMESTAMP(),
      $1
    FROM @RAW.PUBLIC.STG_S3_EVENTS
  );
```

**Stream + Task CDC Pipeline:**

```sql
-- Stream on source
CREATE STREAM STAGING.CLEANSED.STRM_ORDERS
  ON TABLE RAW.ERP.ORDERS
  SHOW_INITIAL_ROWS = TRUE;

-- Task to process changes every 5 minutes
CREATE TASK STAGING.CLEANSED.TSK_PROCESS_ORDERS
  WAREHOUSE = TRANSFORM_WH
  SCHEDULE = '5 MINUTE'
  WHEN SYSTEM$STREAM_HAS_DATA('STAGING.CLEANSED.STRM_ORDERS')
AS
MERGE INTO STAGING.CLEANSED.ORDERS tgt
USING STAGING.CLEANSED.STRM_ORDERS src
ON tgt.ORDER_ID = src.ORDER_ID
WHEN MATCHED AND src.METADATA$ACTION = 'INSERT' AND src.METADATA$ISUPDATE = TRUE
  THEN UPDATE SET
    tgt.STATUS = src.STATUS,
    tgt.UPDATED_AT = CURRENT_TIMESTAMP()
WHEN NOT MATCHED AND src.METADATA$ACTION = 'INSERT'
  THEN INSERT (ORDER_ID, CUSTOMER_ID, STATUS, AMOUNT, CREATED_AT)
  VALUES (src.ORDER_ID, src.CUSTOMER_ID, src.STATUS, src.AMOUNT, CURRENT_TIMESTAMP())
WHEN MATCHED AND src.METADATA$ACTION = 'DELETE'
  THEN DELETE;

ALTER TASK STAGING.CLEANSED.TSK_PROCESS_ORDERS RESUME;
```

**Dynamic Table Chain (bronze → silver → gold):**

```sql
-- Silver: cleansed
CREATE DYNAMIC TABLE STAGING.CLEANSED.DT_ORDERS
  TARGET_LAG = '5 minutes'
  WAREHOUSE = TRANSFORM_WH
AS
SELECT
  ORDER_ID,
  TRIM(CUSTOMER_ID) AS CUSTOMER_ID,
  UPPER(STATUS) AS STATUS,
  AMOUNT::NUMBER(12,2) AS AMOUNT,
  TRY_TO_TIMESTAMP(ORDER_DATE) AS ORDER_DATE
FROM RAW.ERP.ORDERS
WHERE ORDER_ID IS NOT NULL;

-- Gold: aggregated
CREATE DYNAMIC TABLE ANALYTICS.SALES.DT_DAILY_SUMMARY
  TARGET_LAG = '30 minutes'
  WAREHOUSE = TRANSFORM_WH
AS
SELECT
  ORDER_DATE::DATE AS ORDER_DATE,
  COUNT(*) AS ORDER_COUNT,
  SUM(AMOUNT) AS TOTAL_REVENUE,
  COUNT(DISTINCT CUSTOMER_ID) AS UNIQUE_CUSTOMERS
FROM STAGING.CLEANSED.DT_ORDERS
WHERE STATUS != 'CANCELLED'
GROUP BY 1;
```

**COPY INTO with Error Handling:**

```sql
COPY INTO RAW.CRM.CONTACTS
FROM @RAW.PUBLIC.STG_S3_CRM/contacts/
  FILE_FORMAT = (TYPE = CSV SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"')
  ON_ERROR = CONTINUE
  VALIDATION_MODE = RETURN_ERRORS  -- dry run: check errors without loading
;

-- After loading, check rejected records
SELECT * FROM TABLE(VALIDATE(RAW.CRM.CONTACTS, JOB_ID => '_last'));
```

### Anti-Patterns

- Using COPY INTO in a loop with single files (use Snowpipe for continuous)
- Streams without tasks consuming them (stream offset never advances, grows unbounded)
- Dynamic tables with TARGET_LAG = DOWNSTREAM but no downstream consumer (never refreshes)
- No error handling on COPY INTO — silent data loss with `ON_ERROR = SKIP_FILE`
- External functions without timeout/retry logic
- Loading all file formats as CSV when Parquet is available (lose schema + types)

### What Goes Wrong

- **Snowpipe without dead-letter handling** → malformed files silently land in `COPY_HISTORY` errors; nobody checks; data gaps accumulate for weeks before discovery
- **Dynamic Tables as universal solution** → team converts all Streams+Tasks to Dynamic Tables; loses exactly-once processing guarantees; discovers duplicates in production
- **TARGET_LAG = DOWNSTREAM with no consumer** → Dynamic Table never refreshes because nothing reads from it; team thinks data is flowing; it isn't
- **ON_ERROR = 'CONTINUE' without monitoring** → 5% of rows fail parsing every load; over 6 months, millions of records silently missing
- **No backfill strategy** → pipeline handles incremental perfectly but cannot replay historical data; source system changes require full rebuild with no tooling

### Implementation Checklist

- [ ] Set up storage integrations for all external cloud stages
- [ ] Implement Snowpipe for continuous ingestion workloads
- [ ] Create streams on all source-of-truth tables for CDC
- [ ] Build dynamic table chains for transformation layers
- [ ] Define file formats and standardize across pipelines
- [ ] Implement error handling (VALIDATION_MODE, error tables)
- [ ] Monitor pipe status with PIPE_USAGE_HISTORY
- [ ] Document integration patterns and data flow diagrams

---

## 7. Documents & Content Management

### DAMA Principle

Document and Content Management governs unstructured and semi-structured data throughout its lifecycle — from creation through storage, retrieval, and disposition. This includes PDFs, images, audio, video, and office files requiring specialized handling for storage, indexing, access control, version management, and compliance with retention policies.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Document storage | Internal/external stages | `CREATE STAGE` |
| Document catalog | Directory tables | `DIRECTORY = (ENABLE = TRUE)` |
| Document processing | Document AI, PARSE_DOCUMENT | Cortex functions |
| Content search | Cortex Search Service | `CREATE CORTEX SEARCH SERVICE` |
| Metadata extraction | Directory table columns (size, md5, last_modified) | `SELECT * FROM DIRECTORY(@stage)` |
| Content classification | Cortex CLASSIFY_TEXT | `SNOWFLAKE.CORTEX.CLASSIFY_TEXT(...)` |
| Summarization | Cortex SUMMARIZE | `SNOWFLAKE.CORTEX.SUMMARIZE(...)` |
| Secure access | Pre-signed URLs | `GET_PRESIGNED_URL(@stage, path)` |
| Lifecycle management | Stage retention, external stage lifecycle rules | Cloud provider lifecycle |

### Implementation Best Practices

**Directory Table for Document Catalog:**

```sql
-- Internal stage with directory table
CREATE STAGE RAW.FILES.DOCUMENTS
  DIRECTORY = (ENABLE = TRUE)
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');

-- Upload files (via SnowSQL or PUT)
-- PUT file:///local/path/contract.pdf @RAW.FILES.DOCUMENTS/contracts/;

-- Query document catalog
SELECT
  RELATIVE_PATH,
  SIZE,
  LAST_MODIFIED,
  MD5,
  FILE_URL
FROM DIRECTORY(@RAW.FILES.DOCUMENTS)
ORDER BY LAST_MODIFIED DESC;
```

**Document AI — PDF Extraction:**

```sql
-- Extract structured data from invoices
SELECT
  RELATIVE_PATH,
  SNOWFLAKE.CORTEX.PARSE_DOCUMENT(
    @RAW.FILES.DOCUMENTS,
    RELATIVE_PATH,
    {'mode': 'LAYOUT'}
  ) AS PARSED_CONTENT
FROM DIRECTORY(@RAW.FILES.DOCUMENTS)
WHERE RELATIVE_PATH LIKE '%.pdf';

-- Extract specific fields from parsed output
SELECT
  RELATIVE_PATH,
  PARSED_CONTENT:content[0]:text::STRING AS FIRST_PAGE_TEXT,
  ARRAY_SIZE(PARSED_CONTENT:content) AS PAGE_COUNT
FROM (
  SELECT
    RELATIVE_PATH,
    SNOWFLAKE.CORTEX.PARSE_DOCUMENT(
      @RAW.FILES.DOCUMENTS,
      RELATIVE_PATH,
      {'mode': 'LAYOUT'}
    ) AS PARSED_CONTENT
  FROM DIRECTORY(@RAW.FILES.DOCUMENTS)
  WHERE RELATIVE_PATH LIKE 'contracts/%.pdf'
);
```

**Content Summarization:**

```sql
-- Summarize document text
SELECT
  RELATIVE_PATH,
  SNOWFLAKE.CORTEX.SUMMARIZE(extracted_text) AS SUMMARY
FROM ANALYTICS.CONTENT.DOCUMENT_TEXT
WHERE CHAR_LENGTH(extracted_text) > 500;
```

**Pre-signed URLs for Secure Sharing:**

```sql
-- Generate time-limited URL for document access
SELECT
  RELATIVE_PATH,
  GET_PRESIGNED_URL(@RAW.FILES.DOCUMENTS, RELATIVE_PATH, 3600) AS DOWNLOAD_URL
FROM DIRECTORY(@RAW.FILES.DOCUMENTS)
WHERE RELATIVE_PATH LIKE 'reports/2024/%';
```

### Anti-Patterns

- Storing document content as BLOBs in regular tables (use stages)
- No directory table enabled — can't catalog or query file metadata
- Pre-signed URLs with very long expiry (security risk)
- Not using PARSE_DOCUMENT when structured extraction is needed (manual regex instead)
- Mixing document types in a single flat stage path (no organization)

### What Goes Wrong

- **PARSE_DOCUMENT on unstructured garbage** → OCR quality varies wildly; extraction pipeline produces confident-looking but wrong data; no validation layer catches it
- **Pre-signed URLs with long expiry** → URL leaked via email/Slack; anyone with the link downloads sensitive documents for days/weeks
- **No document lifecycle management** → stage grows indefinitely; storage costs compound; nobody knows which files are still needed
- **Treating stage as a database** → querying directory tables for analytics instead of materializing file metadata into proper tables; performance degrades as file count grows

### Implementation Checklist

- [ ] Create stages with directory tables for all document repositories
- [ ] Organize stage paths by document type and date
- [ ] Implement PARSE_DOCUMENT pipelines for PDFs requiring extraction
- [ ] Set up Cortex Search for full-text document discovery
- [ ] Create pre-signed URL generation for secure external access
- [ ] Define retention policies on stages (or external lifecycle rules)
- [ ] Tag document stages with governance tags (sensitivity, owner)
- [ ] Build document catalog views joining directory tables with extracted metadata

---

## 8. Reference & Master Data

### DAMA Principle

Reference and Master Data Management (RDM/MDM) establishes and maintains authoritative shared data — customers, products, locations — and standard reference values (country codes, currency codes). The goal: a single "golden record" per entity serving as the trusted source. MDM involves matching, merging, survivorship logic, cross-reference maintenance, and SCD handling.

> **Honest framing:** Snowflake is not an MDM platform. It lacks stewardship UIs, hierarchy management, survivorship rule engines, and workflow orchestration that dedicated MDM tools (Informatica MDM, Reltio, Tamr) provide. What Snowflake CAN do well: store golden records, distribute reference data via Shares, perform batch match/merge with SQL, and track changes via Streams. For organizations with <100K entities and simple match rules, Snowflake-native MDM patterns below may suffice. For complex entity resolution across millions of records with manual stewardship needs, invest in a dedicated tool and use Snowflake as the persistence/distribution layer.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Golden record | Secure views over master tables | `CREATE SECURE VIEW` |
| Reference data distribution | Data Sharing across accounts | `CREATE SHARE` |
| Master data matching | JAROWINKLER_SIMILARITY, EDITDISTANCE, Cortex ML | UDFs + built-in functions |
| SCD management | MERGE + Streams | `MERGE INTO ... WHEN MATCHED` |
| Surrogate keys | IDENTITY, SEQUENCES, UUID_STRING() | `col NUMBER IDENTITY` |
| Standardization | UDFs, REGEXP, INITCAP, TRIM | Cleansing functions |
| Cross-references | Mapping tables | FK relationships |
| Version control | Time Travel for point-in-time reference | `AT (TIMESTAMP => ...)` |

### Implementation Best Practices

**MDM Hub Pattern:**

```sql
-- Master customer table
CREATE TABLE ANALYTICS.MDM.MST_CUSTOMER (
  MASTER_CUSTOMER_ID NUMBER IDENTITY PRIMARY KEY,
  GOLDEN_NAME STRING NOT NULL,
  GOLDEN_EMAIL STRING,
  GOLDEN_PHONE STRING,
  GOLDEN_ADDRESS STRING,
  CONFIDENCE_SCORE NUMBER(5,2),
  CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  UPDATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  IS_ACTIVE BOOLEAN DEFAULT TRUE
);

-- Cross-reference table (maps source system IDs to master)
CREATE TABLE ANALYTICS.MDM.XREF_CUSTOMER (
  XREF_ID NUMBER IDENTITY PRIMARY KEY,
  MASTER_CUSTOMER_ID NUMBER REFERENCES MST_CUSTOMER(MASTER_CUSTOMER_ID),
  SOURCE_SYSTEM STRING NOT NULL,      -- 'CRM', 'ERP', 'WEB'
  SOURCE_CUSTOMER_ID STRING NOT NULL,
  MATCH_CONFIDENCE NUMBER(5,2),
  LINKED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
  UNIQUE (SOURCE_SYSTEM, SOURCE_CUSTOMER_ID)
);
```

**Deduplication with Blocking Strategy + QUALIFY:**

```sql
-- Find and resolve duplicates using blocking + fuzzy matching
-- Blocking key (first 3 chars of name + zip) reduces O(n²) to manageable pairs
WITH BLOCKED_PAIRS AS (
  SELECT
    a.MASTER_CUSTOMER_ID AS ID_A,
    b.MASTER_CUSTOMER_ID AS ID_B,
    JAROWINKLER_SIMILARITY(a.GOLDEN_NAME, b.GOLDEN_NAME) AS NAME_SIMILARITY,
    JAROWINKLER_SIMILARITY(a.GOLDEN_EMAIL, b.GOLDEN_EMAIL) AS EMAIL_SIMILARITY
  FROM ANALYTICS.MDM.MST_CUSTOMER a
  JOIN ANALYTICS.MDM.MST_CUSTOMER b
    ON a.MASTER_CUSTOMER_ID < b.MASTER_CUSTOMER_ID
    AND LEFT(a.GOLDEN_NAME, 3) = LEFT(b.GOLDEN_NAME, 3)  -- blocking key
  WHERE JAROWINKLER_SIMILARITY(a.GOLDEN_NAME, b.GOLDEN_NAME) > 85
    OR a.GOLDEN_EMAIL = b.GOLDEN_EMAIL
)
SELECT * FROM BLOCKED_PAIRS
WHERE NAME_SIMILARITY > 90 OR EMAIL_SIMILARITY = 100;

-- Deduplicate keeping best record
SELECT *
FROM RAW.CRM.CONTACTS
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY LOWER(TRIM(EMAIL))
  ORDER BY UPDATED_AT DESC, COMPLETENESS_SCORE DESC
) = 1;
```

**SCD Type 2 with Streams:**

```sql
CREATE STREAM ANALYTICS.MDM.STRM_CUSTOMER_CHANGES
  ON TABLE STAGING.CLEANSED.CUSTOMERS;

-- Merge: expire old, insert new version
MERGE INTO ANALYTICS.MDM.DIM_CUSTOMER tgt
USING (
  SELECT *, SHA2(CONCAT(NAME, EMAIL, PHONE, ADDRESS)) AS HASH_DIFF
  FROM ANALYTICS.MDM.STRM_CUSTOMER_CHANGES
  WHERE METADATA$ACTION = 'INSERT'
) src
ON tgt.CUSTOMER_BK = src.CUSTOMER_ID AND tgt.IS_CURRENT = TRUE
-- Changed record: expire existing
WHEN MATCHED AND tgt.HASH_DIFF != src.HASH_DIFF THEN UPDATE SET
  tgt.EFFECTIVE_TO = CURRENT_TIMESTAMP(),
  tgt.IS_CURRENT = FALSE
-- New record
WHEN NOT MATCHED THEN INSERT (
  CUSTOMER_BK, NAME, EMAIL, PHONE, ADDRESS, HASH_DIFF,
  EFFECTIVE_FROM, EFFECTIVE_TO, IS_CURRENT
) VALUES (
  src.CUSTOMER_ID, src.NAME, src.EMAIL, src.PHONE, src.ADDRESS, src.HASH_DIFF,
  CURRENT_TIMESTAMP(), '9999-12-31'::TIMESTAMP, TRUE
);
```

**Reference Data Distribution via Sharing:**

```sql
-- Create share for reference data
CREATE SHARE REFERENCE_DATA_SHARE;

-- Grant access to reference tables through secure view
CREATE SECURE VIEW ANALYTICS.MDM.VW_COUNTRY_CODES AS
SELECT CODE, NAME, REGION, CURRENCY FROM ANALYTICS.MDM.REF_COUNTRIES;

GRANT USAGE ON DATABASE ANALYTICS TO SHARE REFERENCE_DATA_SHARE;
GRANT USAGE ON SCHEMA ANALYTICS.MDM TO SHARE REFERENCE_DATA_SHARE;
GRANT SELECT ON VIEW ANALYTICS.MDM.VW_COUNTRY_CODES TO SHARE REFERENCE_DATA_SHARE;

-- Share to consumer accounts
ALTER SHARE REFERENCE_DATA_SHARE ADD ACCOUNTS = ORG.SUBSIDIARY_A, ORG.SUBSIDIARY_B;
```

### Anti-Patterns

- No cross-reference table — losing source system traceability
- Fuzzy matching on full dataset every run (use blocking/bucketing first)
- SCD Type 2 without hash_diff — comparing every column is fragile and slow
- Reference data as hardcoded values in application code (should be tables)
- No golden record confidence score — can't prioritize manual review
- Distributing master data via file exports instead of data sharing

### What Goes Wrong

- **MERGE-as-MDM delusion** → team thinks a MERGE statement is master data management; no survivorship rules, no stewardship workflow, no conflict resolution; golden record is just "last write wins"
- **Fuzzy matching without blocking strategy** → JAROWINKLER on every pair = O(n²); 1M records = 1 trillion comparisons; query never finishes
- **No confidence scoring** → automated match/merge runs unsupervised; false positives merge distinct entities; customers receive wrong invoices; reversing merges is manual nightmare
- **Reference data without versioning** → lookup table changes without history; downstream reports shift retroactively; nobody can explain why last month's numbers changed
- **Snowflake as MDM platform** → trying to build survivorship, stewardship UI, hierarchy management in SQL when a dedicated MDM tool is needed

### Implementation Checklist

- [ ] Design MDM hub + cross-reference table structure
- [ ] Implement deduplication logic with fuzzy matching + blocking strategy
- [ ] Set up SCD Type 2 with streams and hash-based change detection
- [ ] Create secure views as the golden record API
- [ ] Distribute reference data via Snowflake data shares
- [ ] Define survivorship rules (which source wins per attribute)
- [ ] Build confidence scoring for match quality
- [ ] Schedule periodic duplicate scans and quality reports

---

## 9. Data Warehousing & Business Intelligence

### DAMA Principle

DW/BI integrates data from multiple sources into a unified repository optimized for analytical queries, combined with tools that transform data into actionable insights. The warehouse serves as the single source of truth for reporting, ad-hoc analysis, and advanced analytics. Key concerns: performance, scalability, cost management, and BI delivery.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Warehouse architecture | Virtual warehouses (elastic compute) | `CREATE WAREHOUSE` |
| Concurrency | Multi-cluster warehouses | `MAX_CLUSTER_COUNT = 10` |
| Query performance | Result cache, metadata cache, warehouse cache | Automatic |
| Query acceleration | Query Acceleration Service (QAS) | `ENABLE_QUERY_ACCELERATION = TRUE` |
| Materialization | Materialized views | `CREATE MATERIALIZED VIEW` |
| Semantic transform | Dynamic tables | `CREATE DYNAMIC TABLE` |
| Workload management | Resource monitors, warehouse scheduling | `CREATE RESOURCE MONITOR` |
| BI serving | Snowsight dashboards, Streamlit in Snowflake | Snowsight UI |
| Semantic layer | Cortex Analyst, semantic views | YAML model definitions |
| Data products | Marketplace, native apps | Listings |
| Cost management | Resource monitors, credit quotas | `ALTER RESOURCE MONITOR` |

### Implementation Best Practices

**Warehouse Sizing Strategy:**

```sql
-- Analytics: medium, multi-cluster for concurrency
CREATE WAREHOUSE ANALYTICS_WH
  WAREHOUSE_SIZE = 'MEDIUM'
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 5
  SCALING_POLICY = 'STANDARD'
  AUTO_SUSPEND = 60          -- 1 minute idle
  AUTO_RESUME = TRUE
  ENABLE_QUERY_ACCELERATION = TRUE
  QUERY_ACCELERATION_MAX_SCALE_FACTOR = 8
  COMMENT = 'BI and ad-hoc analytics workloads';

-- Ingestion: small, single cluster (predictable load)
CREATE WAREHOUSE INGEST_WH
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 30
  AUTO_RESUME = TRUE
  COMMENT = 'Snowpipe and COPY INTO workloads';

-- Transform: large, single cluster (complex joins)
CREATE WAREHOUSE TRANSFORM_WH
  WAREHOUSE_SIZE = 'LARGE'
  AUTO_SUSPEND = 120
  AUTO_RESUME = TRUE
  COMMENT = 'Dynamic tables and scheduled transforms';
```

> **Cost note — Multi-cluster warehouses:** Each additional cluster costs the same as the base warehouse. A MEDIUM with MAX_CLUSTER_COUNT=5 can burn 5x MEDIUM credits simultaneously. Always set an explicit MAX. Use STANDARD scaling policy (conservative) unless you have measured queuing. ECONOMY scaling adds clusters only when queries queue for 6+ minutes — often sufficient, much cheaper.

**Resource Monitor:**

```sql
CREATE RESOURCE MONITOR MONTHLY_BUDGET
  CREDIT_QUOTA = 5000
  FREQUENCY = MONTHLY
  START_TIMESTAMP = IMMEDIATELY
  TRIGGERS
    ON 75 PERCENT DO NOTIFY
    ON 90 PERCENT DO NOTIFY
    ON 100 PERCENT DO SUSPEND
    ON 110 PERCENT DO SUSPEND_IMMEDIATE;

ALTER WAREHOUSE ANALYTICS_WH SET RESOURCE_MONITOR = MONTHLY_BUDGET;
ALTER WAREHOUSE TRANSFORM_WH SET RESOURCE_MONITOR = MONTHLY_BUDGET;
```

**Materialized View for BI Aggregation:**

```sql
CREATE MATERIALIZED VIEW ANALYTICS.SALES.MV_MONTHLY_REVENUE AS
SELECT
  DATE_TRUNC('month', ORDER_DATE) AS MONTH,
  REGION,
  SEGMENT,
  COUNT(*) AS ORDER_COUNT,
  SUM(TOTAL_AMOUNT) AS REVENUE,
  AVG(TOTAL_AMOUNT) AS AVG_ORDER_VALUE,
  COUNT(DISTINCT CUSTOMER_SK) AS UNIQUE_CUSTOMERS
FROM ANALYTICS.SALES.FCT_ORDERS f
JOIN ANALYTICS.SALES.DIM_CUSTOMER d ON f.CUSTOMER_SK = d.CUSTOMER_SK
WHERE d.IS_CURRENT = TRUE
GROUP BY 1, 2, 3;
```

**Performance Monitoring:**

```sql
-- Top 20 slowest queries this week
SELECT
  QUERY_ID,
  USER_NAME,
  WAREHOUSE_NAME,
  EXECUTION_STATUS,
  TOTAL_ELAPSED_TIME / 1000 AS ELAPSED_SEC,
  BYTES_SCANNED / POWER(1024,3) AS GB_SCANNED,
  PARTITIONS_SCANNED,
  PARTITIONS_TOTAL,
  ROUND(PARTITIONS_SCANNED / NULLIF(PARTITIONS_TOTAL, 0) * 100, 1) AS PCT_SCANNED,
  QUERY_TEXT
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME >= DATEADD('day', -7, CURRENT_TIMESTAMP())
  AND EXECUTION_STATUS = 'SUCCESS'
  AND TOTAL_ELAPSED_TIME > 0
ORDER BY TOTAL_ELAPSED_TIME DESC
LIMIT 20;

-- Warehouse utilization
SELECT
  WAREHOUSE_NAME,
  DATE_TRUNC('hour', START_TIME) AS HOUR,
  COUNT(*) AS QUERY_COUNT,
  AVG(TOTAL_ELAPSED_TIME) / 1000 AS AVG_ELAPSED_SEC,
  SUM(CREDITS_USED_COMPUTE) AS CREDITS
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE START_TIME >= DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 1, 2;
```

**Cortex Analyst — Self-Service with Guardrails:**

Cortex Analyst lets business users ask natural-language questions against semantic models. Powerful, but deploy with eyes open:

| Strength | Limitation |
|---|---|
| Natural language → SQL | Non-deterministic: same question can produce different SQL on different runs |
| No SQL knowledge required | Semantic model requires significant upfront investment (YAML definitions, relationships, metrics) |
| Answers ad-hoc questions | Complex queries (multi-hop joins, window functions, correlated subqueries) may produce incorrect results |
| Reduces BI team bottleneck | No built-in validation — users may act on wrong answers without checking |
| Integrates with Snowsight | Per-query LLM cost at scale; no native usage quotas per user |

**Recommended guardrails:**
- Start with a narrow semantic model (1-2 fact tables, key dimensions) and expand based on validated usage
- Build a "verified answers" library — known questions with confirmed-correct SQL that Cortex can reference
- Monitor `QUERY_HISTORY` for Cortex-originated queries; alert on unexpected cost spikes
- Train users to validate LLM-generated results against known metrics before sharing externally
- Set warehouse resource monitors specifically on the warehouse Cortex Analyst uses

### Anti-Patterns

- One XLARGE warehouse for everything (no workload isolation)
- AUTO_SUSPEND = 0 (never suspends — burning credits 24/7)
- No resource monitors — surprise bills
- Materialized views on rapidly changing base tables (constant refresh cost)
- Running BI dashboards against RAW tables (no curation layer)
- Query acceleration on warehouses with predictable, uniform queries (no benefit)

### What Goes Wrong

- **No resource monitors** → analyst runs accidental cross-join on XL warehouse; $5,000 bill in 20 minutes; discovered at end-of-month invoice
- **Materialized views on volatile tables** → base table changes every minute; MV refresh cost exceeds the query savings; net negative ROI
- **Self-service without guardrails** → Cortex Analyst generates valid but expensive SQL; users don't understand cost; 100 ad-hoc LLM-routed queries per day at $0.50+ each
- **Multi-cluster auto-scale with no ceiling** → Black Friday traffic spins up 10 clusters; budget gone in hours; nobody set MAX_CLUSTER_COUNT
- **BI tool connected to RAW layer** → dashboards break on schema changes; no contract between producers and consumers; every source change is an incident

> **Cost note — Cortex Analyst:** Each query routes through an LLM. At scale (hundreds of daily ad-hoc queries), costs add up. Monitor `QUERY_HISTORY` for Cortex-originated queries and set usage alerts. Consider whether a curated dashboard answers 80% of questions cheaper.

### Implementation Checklist

- [ ] Create dedicated warehouses per workload type (ingest, transform, analytics, BI)
- [ ] Set up resource monitors with credit quotas and alerts
- [ ] Enable query acceleration on analytics warehouses
- [ ] Create materialized views for frequently accessed BI aggregations
- [ ] Monitor query performance weekly (partition efficiency, spilling)
- [ ] Right-size warehouses monthly based on QUERY_HISTORY analysis
- [ ] Implement Snowsight dashboards for executive reporting
- [ ] Define semantic views for Cortex Analyst self-service

---

## 10. Metadata Management

### DAMA Principle

Metadata Management enables access to high-quality, integrated metadata across three categories: technical (schemas, types, lineage), business (definitions, ownership, classification), and operational (job execution, volumes, access patterns). Without managed metadata, organizations cannot answer: where did this data come from, who owns it, what does it mean, who uses it?

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Technical metadata | INFORMATION_SCHEMA, ACCOUNT_USAGE views | `SELECT * FROM INFORMATION_SCHEMA.TABLES` |
| Business metadata | Object tags, COMMENT ON | `COMMENT ON TABLE ... IS '...'` |
| Operational metadata | QUERY_HISTORY, COPY_HISTORY, LOAD_HISTORY | ACCOUNT_USAGE views |
| Lineage | ACCESS_HISTORY (column-level lineage) | `SELECT * FROM ACCESS_HISTORY` |
| Data catalog | Horizon Catalog, Universal Search | Snowsight search |
| Impact analysis | OBJECT_DEPENDENCIES | `SELECT * FROM OBJECT_DEPENDENCIES` |
| Discovery | SHOW, DESCRIBE, GET_DDL | `SELECT GET_DDL('TABLE', ...)` |
| Metadata governance | Tag-based policies, tag lineage | Tags propagate through lineage |
| Business glossary | Tags + comments as glossary | Custom glossary tables |

### Implementation Best Practices

**Tagging Taxonomy:**

```sql
-- Classification taxonomy
CREATE TAG GOVERNANCE.TAGS.DATA_DOMAIN
  ALLOWED_VALUES 'FINANCE','HR','SALES','MARKETING','PRODUCT','OPERATIONS';

CREATE TAG GOVERNANCE.TAGS.SENSITIVITY
  ALLOWED_VALUES 'PUBLIC','INTERNAL','CONFIDENTIAL','RESTRICTED';

CREATE TAG GOVERNANCE.TAGS.PII_TYPE
  ALLOWED_VALUES 'NAME','EMAIL','PHONE','SSN','ADDRESS','DOB','FINANCIAL';

CREATE TAG GOVERNANCE.TAGS.DATA_OWNER
  COMMENT = 'Team or individual responsible for data quality';

CREATE TAG GOVERNANCE.TAGS.SLA_TIER
  ALLOWED_VALUES 'PLATINUM','GOLD','SILVER','BRONZE'
  COMMENT = 'Freshness SLA tier for monitoring';

CREATE TAG GOVERNANCE.TAGS.COST_CENTER
  COMMENT = 'Cost allocation tag for chargeback';

-- Apply comprehensively
ALTER TABLE ANALYTICS.SALES.FCT_ORDERS SET TAG
  GOVERNANCE.TAGS.DATA_DOMAIN = 'SALES',
  GOVERNANCE.TAGS.DATA_OWNER = 'revenue-analytics',
  GOVERNANCE.TAGS.SLA_TIER = 'PLATINUM',
  GOVERNANCE.TAGS.COST_CENTER = 'CC-4521';

COMMENT ON TABLE ANALYTICS.SALES.FCT_ORDERS IS
  'Fact table of all confirmed orders. Grain: one row per order line item. Updated every 5 minutes via dynamic table from staging.';
```

**Lineage Tracking:**

```sql
-- Column-level lineage: what feeds ANALYTICS.SALES.FCT_ORDERS?
SELECT
  DIRECTSOURCES.value:objectName::STRING AS SOURCE_OBJECT,
  DIRECTSOURCES.value:columnName::STRING AS SOURCE_COLUMN,
  OBJECTS.value:objectName::STRING AS TARGET_OBJECT,
  COLUMNS.value:columnName::STRING AS TARGET_COLUMN
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY,
  LATERAL FLATTEN(input => OBJECTS_MODIFIED) OBJECTS,
  LATERAL FLATTEN(input => OBJECTS.value:columns) COLUMNS,
  LATERAL FLATTEN(input => COLUMNS.value:directSources) DIRECTSOURCES
WHERE OBJECTS.value:objectName::STRING = 'ANALYTICS.SALES.FCT_ORDERS'
  AND QUERY_START_TIME >= DATEADD('day', -30, CURRENT_TIMESTAMP())
LIMIT 100;
```

**Object Dependencies (Impact Analysis):**

```sql
-- What depends on this table? (downstream impact)
SELECT
  REFERENCING_OBJECT_NAME,
  REFERENCING_OBJECT_DOMAIN,  -- VIEW, FUNCTION, TASK, etc.
  REFERENCING_OBJECT_ID
FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE REFERENCED_OBJECT_NAME = 'FCT_ORDERS'
  AND REFERENCED_SCHEMA_NAME = 'SALES'
  AND REFERENCED_DATABASE_NAME = 'ANALYTICS';

-- What does this view depend on? (upstream sources)
SELECT
  REFERENCED_OBJECT_NAME,
  REFERENCED_OBJECT_DOMAIN,
  REFERENCED_DATABASE_NAME || '.' || REFERENCED_SCHEMA_NAME || '.' || REFERENCED_OBJECT_NAME AS FQN
FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES
WHERE REFERENCING_OBJECT_NAME = 'VW_REVENUE_DASHBOARD'
  AND REFERENCING_DATABASE_NAME = 'ANALYTICS';
```

**Tag-Based Governance Report:**

```sql
-- Coverage report: what % of tables are tagged?
SELECT
  TABLE_CATALOG AS DATABASE_NAME,
  TABLE_SCHEMA AS SCHEMA_NAME,
  COUNT(*) AS TOTAL_TABLES,
  COUNT(tr.TAG_VALUE) AS TAGGED_TABLES,
  ROUND(COUNT(tr.TAG_VALUE) / COUNT(*) * 100, 1) AS PCT_TAGGED
FROM INFORMATION_SCHEMA.TABLES t
LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES tr
  ON t.TABLE_CATALOG = tr.OBJECT_DATABASE
  AND t.TABLE_SCHEMA = tr.OBJECT_SCHEMA
  AND t.TABLE_NAME = tr.OBJECT_NAME
  AND tr.TAG_NAME = 'DATA_DOMAIN'
  AND tr.DOMAIN = 'TABLE'
WHERE t.TABLE_TYPE = 'BASE TABLE'
GROUP BY 1, 2
ORDER BY PCT_TAGGED ASC;
```

### Anti-Patterns

- No COMMENT ON anything — metadata exists only in external wikis nobody reads
- Tags without ALLOWED_VALUES — inconsistent free-text values
- Not using ACCESS_HISTORY for lineage — paying for external lineage tools unnecessarily
- Tagging sporadically (some tables tagged, most not) — incomplete picture
- Ignoring OBJECT_DEPENDENCIES before refactoring — breaking downstream consumers
- No business glossary or term definitions anywhere

### What Goes Wrong

- **Sporadic tagging** → 30% of tables tagged, 70% not; governance reports show false completeness; auditors find gaps immediately
- **OBJECT_DEPENDENCIES ignored before refactoring** → rename a view; 15 downstream tasks break; no impact analysis was done; 3-hour incident
- **ACCESS_HISTORY lineage trusted blindly** → shows read/write but not semantic transformation logic; "this table feeds that table" ≠ "here's the business rule"
- **Metadata without ownership** → COMMENT ON exists but nobody maintains it; descriptions drift from reality; worse than no documentation (actively misleading)
- **No business glossary** → "revenue" means 3 different things across 4 teams; metadata exists but semantic alignment doesn't

### Implementation Checklist

- [ ] Define tagging taxonomy (domain, sensitivity, PII type, owner, SLA)
- [ ] Tag all production tables and views systematically
- [ ] Add COMMENT ON all tables/views with business definitions
- [ ] Set up lineage queries using ACCESS_HISTORY
- [ ] Create impact analysis queries using OBJECT_DEPENDENCIES
- [ ] Build metadata coverage report (% tagged by schema)
- [ ] Implement business glossary (tag + comment or dedicated reference table)
- [ ] Schedule automated classification scans (SYSTEM$CLASSIFY)

---

## 11. Data Quality

### DAMA Principle

Data Quality Management measures, assesses, improves, and ensures fitness of data for use across six dimensions: **accuracy** (correctly represents reality), **completeness** (all required data present), **consistency** (agrees across systems), **timeliness** (current and available when needed), **validity** (conforms to business rules), and **uniqueness** (no unintended duplicates).

Effective quality programs are proactive — embedding checks into pipelines, defining ownership, and establishing automated remediation workflows.

### Snowflake Feature Mapping

| DAMA Concept | Snowflake Feature | SQL/Config |
|---|---|---|
| Quality measurement | Data Metric Functions (DMFs) | `CREATE DATA METRIC FUNCTION` |
| System metrics | Built-in DMFs (NULL_COUNT, DUPLICATE_COUNT, FRESHNESS) | `SNOWFLAKE.CORE.NULL_COUNT` |
| Quality monitoring | SYSTEM$DATA_METRIC_SCAN | `SELECT * FROM TABLE(SYSTEM$DATA_METRIC_SCAN(...))` |
| Completeness | NULL_COUNT + custom completeness DMF | Custom DMF |
| Uniqueness | DUPLICATE_COUNT | System DMF |
| Freshness | FRESHNESS | System DMF |
| Validity | Custom DMFs with business rules | User-defined DMF |
| Profiling | APPROX_COUNT_DISTINCT, HLL, percentiles | Statistical functions |
| Alerting | Snowflake Alerts + notifications | `CREATE ALERT` |
| Quality gates | Tasks with conditional logic | `WHEN` clause on tasks |

### Implementation Best Practices

**Custom Data Metric Function:**

```sql
-- Completeness: % of rows with all required fields populated
CREATE DATA METRIC FUNCTION GOVERNANCE.DQ.COMPLETENESS_PCT(
  ARG_T TABLE(
    ARG_C1 STRING,
    ARG_C2 STRING,
    ARG_C3 STRING
  )
)
RETURNS NUMBER
AS
$$
  SELECT ROUND(
    COUNT_IF(ARG_C1 IS NOT NULL AND ARG_C2 IS NOT NULL AND ARG_C3 IS NOT NULL)
    / NULLIF(COUNT(*), 0) * 100, 2
  )
  FROM ARG_T
$$;

-- Validity: email format check
CREATE DATA METRIC FUNCTION GOVERNANCE.DQ.INVALID_EMAIL_COUNT(
  ARG_T TABLE(ARG_C1 STRING)
)
RETURNS NUMBER
AS
$$
  SELECT COUNT_IF(
    ARG_C1 IS NOT NULL
    AND NOT RLIKE(ARG_C1, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$')
  )
  FROM ARG_T
$$;
```

**Attaching DMFs to Tables:**

```sql
-- Attach system and custom DMFs
ALTER TABLE ANALYTICS.SALES.FCT_ORDERS
  SET DATA_METRIC_SCHEDULE = 'TRIGGER_ON_CHANGES';

ALTER TABLE ANALYTICS.SALES.FCT_ORDERS
  ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.NULL_COUNT ON (CUSTOMER_SK);

ALTER TABLE ANALYTICS.SALES.FCT_ORDERS
  ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.DUPLICATE_COUNT ON (ORDER_SK);

ALTER TABLE ANALYTICS.SALES.FCT_ORDERS
  ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.FRESHNESS ON (ORDER_DATE);

ALTER TABLE ANALYTICS.SALES.DIM_CUSTOMER
  ADD DATA METRIC FUNCTION GOVERNANCE.DQ.INVALID_EMAIL_COUNT ON (EMAIL);
```

**Monitoring with SYSTEM$DATA_METRIC_SCAN:**

```sql
-- Check latest metric results
SELECT *
FROM TABLE(SYSTEM$DATA_METRIC_SCAN(
  REF_ENTITY_NAME => 'ANALYTICS.SALES.FCT_ORDERS',
  REF_ENTITY_DOMAIN => 'TABLE'
))
ORDER BY MEASUREMENT_TIME DESC;
```

**Quality Alert:**

```sql
-- Alert on quality violation
CREATE ALERT GOVERNANCE.DQ.ALERT_HIGH_NULL_RATE
  WAREHOUSE = TRANSFORM_WH
  SCHEDULE = 'USING CRON 0 8 * * * America/New_York'  -- daily 8am
IF (EXISTS (
  SELECT 1
  FROM TABLE(SYSTEM$DATA_METRIC_SCAN(
    REF_ENTITY_NAME => 'ANALYTICS.SALES.FCT_ORDERS',
    REF_ENTITY_DOMAIN => 'TABLE'
  ))
  WHERE METRIC_NAME = 'NULL_COUNT'
    AND VALUE > 100  -- threshold
    AND MEASUREMENT_TIME >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
))
THEN
  CALL SYSTEM$SEND_EMAIL(
    'DQ_NOTIFICATIONS',
    'data-team@company.com',
    'Data Quality Alert: High NULL rate in FCT_ORDERS',
    'NULL_COUNT exceeded threshold (>100) in the last 24 hours. Please investigate.'
  );

ALTER ALERT GOVERNANCE.DQ.ALERT_HIGH_NULL_RATE RESUME;
```

**Quality Score Rollup:**

```sql
-- Quality score table (materialized via task)
CREATE TABLE GOVERNANCE.DQ.QUALITY_SCORES (
  TABLE_FQN STRING,
  METRIC_NAME STRING,
  METRIC_VALUE NUMBER,
  THRESHOLD NUMBER,
  STATUS STRING,  -- 'PASS', 'WARN', 'FAIL'
  MEASURED_AT TIMESTAMP_NTZ
);

-- Task to populate scores
CREATE TASK GOVERNANCE.DQ.TSK_QUALITY_SCORE_REFRESH
  WAREHOUSE = TRANSFORM_WH
  SCHEDULE = 'USING CRON 0 */4 * * * America/New_York'
AS
INSERT INTO GOVERNANCE.DQ.QUALITY_SCORES
SELECT
  REF_ENTITY_NAME AS TABLE_FQN,
  METRIC_NAME,
  VALUE AS METRIC_VALUE,
  CASE
    WHEN METRIC_NAME = 'NULL_COUNT' THEN 50
    WHEN METRIC_NAME = 'DUPLICATE_COUNT' THEN 0
    WHEN METRIC_NAME = 'FRESHNESS' THEN 3600  -- 1 hour in seconds
    ELSE 0
  END AS THRESHOLD,
  CASE
    WHEN METRIC_NAME = 'DUPLICATE_COUNT' AND VALUE > 0 THEN 'FAIL'
    WHEN METRIC_NAME = 'NULL_COUNT' AND VALUE > 50 THEN 'WARN'
    WHEN METRIC_NAME = 'NULL_COUNT' AND VALUE > 200 THEN 'FAIL'
    WHEN METRIC_NAME = 'FRESHNESS' AND VALUE > 3600 THEN 'FAIL'
    ELSE 'PASS'
  END AS STATUS,
  CURRENT_TIMESTAMP()
FROM TABLE(SYSTEM$DATA_METRIC_SCAN(
  REF_ENTITY_NAME => 'ANALYTICS.SALES.FCT_ORDERS',
  REF_ENTITY_DOMAIN => 'TABLE'
))
WHERE MEASUREMENT_TIME >= DATEADD('hour', -4, CURRENT_TIMESTAMP());

ALTER TASK GOVERNANCE.DQ.TSK_QUALITY_SCORE_REFRESH RESUME;
```

**Data Profiling:**

```sql
-- Profile a table
SELECT
  COUNT(*) AS ROW_COUNT,
  COUNT(DISTINCT CUSTOMER_SK) AS UNIQUE_CUSTOMERS,
  MIN(ORDER_DATE) AS MIN_DATE,
  MAX(ORDER_DATE) AS MAX_DATE,
  DATEDIFF('day', MAX(ORDER_DATE), CURRENT_DATE()) AS DAYS_SINCE_LAST_ORDER,
  AVG(TOTAL_AMOUNT) AS AVG_AMOUNT,
  MEDIAN(TOTAL_AMOUNT) AS MEDIAN_AMOUNT,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY TOTAL_AMOUNT) AS P95_AMOUNT,
  COUNT_IF(TOTAL_AMOUNT < 0) AS NEGATIVE_AMOUNTS,
  COUNT_IF(CUSTOMER_SK IS NULL) AS NULL_CUSTOMER_COUNT
FROM ANALYTICS.SALES.FCT_ORDERS;
```

**Remediation Workflow Pattern:**

The missing link between detection and resolution. Without this, alerts are noise.

```
DETECT → CLASSIFY → ROUTE → FIX → VERIFY → CLOSE

1. DETECT: DMF or alert fires
2. CLASSIFY: Severity (P1 = business-critical, P2 = degraded, P3 = cosmetic)
3. ROUTE: To data owner based on TAG (GOVERNANCE.TAGS.DATA_OWNER)
4. FIX: Owner investigates root cause, applies correction
5. VERIFY: Re-run DMF to confirm fix; check downstream consumers
6. CLOSE: Document root cause in quality log; update prevention rules if systemic
```

**Quality SLA Template:**

```sql
-- Quality SLA reference table
CREATE TABLE GOVERNANCE.DQ.QUALITY_SLAS (
  TABLE_FQN STRING NOT NULL,
  METRIC_NAME STRING NOT NULL,
  WARN_THRESHOLD NUMBER,
  FAIL_THRESHOLD NUMBER,
  OWNER_TEAM STRING NOT NULL,           -- who gets paged
  ESCALATION_CONTACT STRING,            -- if unresolved in SLA window
  RESPONSE_TIME_MINUTES NUMBER NOT NULL, -- P1: 30, P2: 240, P3: 1440
  BUSINESS_IMPACT STRING,               -- what breaks downstream
  PRIMARY KEY (TABLE_FQN, METRIC_NAME)
);

-- Example SLA entries
INSERT INTO GOVERNANCE.DQ.QUALITY_SLAS VALUES
  ('ANALYTICS.SALES.FCT_ORDERS', 'FRESHNESS', 1800, 3600, 'data-eng', 'vp-data', 30, 'Executive dashboard shows stale revenue'),
  ('ANALYTICS.SALES.FCT_ORDERS', 'DUPLICATE_COUNT', 1, 10, 'data-eng', 'data-steward', 240, 'Revenue double-counted in reports'),
  ('ANALYTICS.MDM.MST_CUSTOMER', 'NULL_COUNT', 5, 50, 'mdm-team', 'data-steward', 240, 'Customer comms sent to wrong addresses');
```

**Prevention Through Producer Contracts:**

```sql
-- Schema contract: producer guarantees column types and non-null constraints
-- If upstream changes break this contract, the Dynamic Table fails loudly
CREATE DYNAMIC TABLE ANALYTICS.SALES.DT_VALIDATED_ORDERS
  TARGET_LAG = '1 hour'
  WAREHOUSE = TRANSFORM_WH
AS
SELECT
  ORDER_ID::NUMBER AS ORDER_ID,           -- explicit cast = contract
  ORDER_DATE::DATE AS ORDER_DATE,         -- fails if NULL or wrong type
  CUSTOMER_ID::NUMBER AS CUSTOMER_ID,
  TOTAL_AMOUNT::NUMBER(12,2) AS TOTAL_AMOUNT
FROM STAGING.CLEANSED.ORDERS
WHERE ORDER_ID IS NOT NULL                 -- reject nulls at boundary
  AND ORDER_DATE IS NOT NULL
  AND TOTAL_AMOUNT >= 0;                   -- business rule: no negative orders
```

> **Root cause discipline:** When the same quality issue recurs 3+ times, stop treating it as a monitoring problem. Contact the upstream data producer. Establish a producer SLA. If the producer is an external system you don't control, add defensive validation at the ingestion boundary — not downstream.

### Anti-Patterns

- Quality checks only in production (should run in staging before promotion)
- DMFs on every column of every table (expensive — prioritize critical tables)
- Alerts without owners or response procedures (alert fatigue)
- Manual quality reports instead of automated DMFs (stale, forgotten)
- No quality gate before data reaches analytics layer (garbage in, garbage out)
- Freshness checks without accounting for business hours / expected schedules

### What Goes Wrong

- **DMFs without remediation workflows** → alert fires; Slack notification sent; nobody knows who owns the fix; alert is snoozed; becomes permanent background noise
- **Quality gates without business alignment** → transform task halts on 0.1% null rate; downstream dashboard shows stale data for 6 hours; business impact of stale data exceeds impact of 0.1% nulls
- **Alert fatigue** → every table has freshness + null + duplicate checks; 200 alerts/day; team ignores all of them; real issues buried in noise
- **No root cause process** → same quality issue recurs monthly; each time treated as new incident; upstream producer never contacted; never fixed at source
- **Quality monitoring without cost-of-poor-quality measurement** → can't justify investment in prevention because nobody quantified the downstream damage

### Implementation Checklist

- [ ] Identify critical tables requiring quality monitoring
- [ ] Create custom DMFs for business-specific validation rules
- [ ] Attach system DMFs (NULL_COUNT, DUPLICATE_COUNT, FRESHNESS) to critical tables
- [ ] Set DATA_METRIC_SCHEDULE appropriate per table (TRIGGER_ON_CHANGES or CRON)
- [ ] Create alerts for quality violations with email notifications
- [ ] Build quality score rollup table for executive dashboards
- [ ] Define quality gates in CDC/transform tasks (skip processing if quality fails)
- [ ] Schedule monthly data profiling reviews for drift detection
- [ ] Document quality thresholds and escalation procedures per data domain

---

## Implementation Maturity Model

Not everything needs to happen at once. Adopt incrementally based on organizational readiness:

### Level 1 — Foundation (Start Here)

| Area | What to Implement |
|---|---|
| Governance | Basic role hierarchy (functional + access roles), object tagging taxonomy |
| Architecture | RAW → STAGING → ANALYTICS database structure, naming conventions |
| Security | MFA enforcement, key-pair auth for service accounts, account-level network policy |
| Storage | Time Travel retention set per database tier, transient tables for scratch |
| Quality | Identify top 5 critical tables, attach system DMFs (freshness, nulls, duplicates) |

**You're ready for Level 2 when:** All production objects are tagged with at least domain + owner, RBAC is functioning, and you have monitoring (not just configuration).

### Level 2 — Controlled

| Area | What to Implement |
|---|---|
| Governance | Tag-based masking policies, ACCESS_HISTORY monitoring dashboards, quarterly access reviews |
| Security | Row access policies, CLASSIFY on all databases, exfiltration detection queries scheduled |
| Integration | Snowpipe for continuous ingestion, Streams+Tasks for CDC, error handling on all pipelines |
| Quality | Quality SLAs defined per critical table, alert routing to owners, remediation workflow documented |
| Metadata | COMMENT ON all production objects, lineage queries via ACCESS_HISTORY |

**You're ready for Level 3 when:** You can answer "who accessed PII in the last 30 days?" and "what's the quality of our top 10 tables?" in under 5 minutes.

### Level 3 — Measured

| Area | What to Implement |
|---|---|
| Quality | Quality gates block bad data from reaching analytics, root cause process for recurring issues |
| Architecture | Dynamic Tables for automated transformation, Iceberg for open-format interop |
| Security | Privilege escalation auditing, external surface audit (stages, functions, shares), supply chain controls |
| BI | Cortex Analyst with narrow semantic models, verified answers library, cost monitoring |
| MDM | Entity resolution pipeline with blocking + confidence scoring, golden record via secure views |

**You're ready for Level 4 when:** Quality issues are caught before they reach consumers, cost is predictable, and self-service users get correct answers without engineering intervention.

### Level 4 — Optimized

| Area | What to Implement |
|---|---|
| Governance | Governance-as-code (tags, policies, roles in version control via Terraform/Schemachange) |
| Quality | Producer SLAs enforced, cost-of-poor-quality measured, prevention > detection |
| Security | Zero-trust model, automated anomaly detection, real-time exfiltration alerting |
| Architecture | Multi-account strategy, cross-cloud replication, automated DR testing |
| BI | Semantic models covering all key domains, usage analytics on self-service, cost allocation per team |

---

## Summary: DAMA → Snowflake Quick Reference

| # | DAMA Area | Key Snowflake Features |
|---|---|---|
| 1 | Data Governance | Roles, Tags, Masking Policies, ACCESS_HISTORY, Horizon |
| 2 | Data Architecture | Multi-DB design, Shares, Replication, Iceberg, Failover Groups |
| 3 | Data Modeling | Star/Vault DDL, Dynamic Tables, VARIANT, Streams, Clustering |
| 4 | Data Storage | Time Travel, UNDROP, CLONE, Transient tables, Storage metrics |
| 5 | Data Security | Network policies, MFA, Masking, Row Access, CLASSIFY, Audit |
| 6 | Data Integration | Snowpipe, Streams+Tasks, Dynamic Tables, COPY INTO, External Functions |
| 7 | Documents & Content | Stages, Directory tables, PARSE_DOCUMENT, Cortex, Pre-signed URLs |
| 8 | Reference & Master Data | Secure views, Shares, MERGE/SCD, JAROWINKLER, Cross-ref tables |
| 9 | DW & BI | Virtual WH, Multi-cluster, Resource monitors, MV, QAS, Cortex Analyst |
| 10 | Metadata | INFORMATION_SCHEMA, Tags, COMMENT ON, ACCESS_HISTORY lineage, OBJECT_DEPENDENCIES |
| 11 | Data Quality | DMFs, SYSTEM$DATA_METRIC_SCAN, Alerts, Quality gates, Profiling |

---

*Technical implementation companion for Snowflake practitioners. Based on DAMA DMBOK2 (Data Management Body of Knowledge, 2nd Edition). For organizational governance foundations — operating models, stewardship, change management — consult DMBOK2 directly.*
