# Reference Guide: Access Control and Security in Snowflake

> **Target audience:** Teams experienced in SQL (Oracle, SQL Server, PostgreSQL) who are getting started with Snowflake.

> **Objective:** Provide a practical and complete reference for configuring security and access control in any Snowflake account.

---

## Table of Contents

1. [Layered Security](#1-layered-security)
2. [Core Access Control Concepts](#2-core-access-control-concepts)
3. [Object Hierarchy](#3-object-hierarchy)
4. [System-Defined Roles](#4-system-defined-roles)
5. [Functional Roles vs Access Roles](#5-functional-roles-vs-access-roles)
6. [Authorization Design Patterns](#6-authorization-design-patterns)
7. [Grants: Current and Future](#7-grants-current-and-future)
8. [Ownership](#8-ownership)
9. [Role Inheritance and Context Switching](#9-role-inheritance-and-context-switching)
10. [SSO and SCIM Environments: Division of Responsibilities](#10-sso-and-scim-environments-division-of-responsibilities)
11. [Complete Practical Examples](#11-complete-practical-examples)
12. [Security Configuration Checklist](#12-security-configuration-checklist)
13. [Appendix: Quick Reference SQL Commands](#13-appendix-quick-reference-sql-commands)

---

## 1. Layered Security

Snowflake implements security across **four complementary layers**. Each layer operates independently, creating defense in depth:

```
┌──────────────────┐    ┌──────────────────────────┐    ┌──────────────────────────┐    ┌─────────────────────────────────┐
│ 1. Network Access│───►│ 2. Authentication (AuthN)│───►│ 3. Authorization (AuthZ) │───►│ 4. Continuous Data Protection   │
│   (IP/Network)   │    │   (Identity)             │    │ (RBAC - Focus of guide)  │    │   (Encryption/Time Travel)      │
└──────────────────┘    └──────────────────────────┘    └──────────────────────────┘    └─────────────────────────────────┘
```

| Layer | What it does | Mechanisms |
|-------|--------------|------------|
| **1. Network Access** | Restricts which IPs or networks can access the account | Network Policies, AWS PrivateLink, Azure Private Link, GCP Private Service Connect |
| **2. Authentication (AuthN)** | Verifies the identity of the user or process | Username/Password, SSO (SAML 2.0), Key Pair, OAuth, MFA |
| **3. Authorization (AuthZ)** | Defines what each identity can do | **RBAC** — Roles, Privileges, Grants |
| **4. Continuous Data Protection** | Protects data at rest and in transit | AES-256 Encryption, Annual Re-keying, Time Travel (up to 90 days), Fail-safe, Data Masking, Row Access Policies, Tokenization |

**Focus of this guide:** Layer 3 — Authorization via RBAC.

> **Note for those coming from other databases:** In Snowflake, you **cannot** grant privileges directly to users. All authorization goes through Roles. Users simply "assume" a Role to access objects.

---

## 2. Core Access Control Concepts

### 2.1 Securable Object

Any entity to which access can be granted. If there is no explicit GRANT, access is **denied by default**.

Examples: Database, Schema, Table, View, Stage, Warehouse, Task, Stream, Stored Procedure, Function.

### 2.2 Privilege

A specific permission on an object. Examples:

| Privilege | Applies to | Allows |
|-----------|------------|--------|
| `USAGE` | Database, Schema, Warehouse | Access/use the object |
| `SELECT` | Table, View | Read data |
| `INSERT` | Table | Insert data |
| `CREATE TABLE` | Schema | Create tables in the schema |
| `OPERATE` | Warehouse | Start/stop/resize |
| `OWNERSHIP` | Any object | Full control (see section 8) |

### 2.3 Role

An entity to which privileges are granted. There are two types:

- **Account Role** — exists at the account level, can be assigned to users
- **Database Role** — exists within a database, ideal for controlling access to objects in that database

### 2.4 User

Represents an identity that connects to Snowflake. Can be:

- **Human** — a real person who logs in
- **Service Account** — a service account used by applications, data pipelines, etc.

### 2.5 Access Control Methods

| Method | Description | When to use |
|--------|-------------|-------------|
| **RBAC** (Role-Based Access Control) | Privileges are assigned to Roles, Roles to Users | **Recommended standard** for all configurations |
| **DAC** (Discretionary Access Control) | The object owner can grant access to others | Works, but difficult to maintain at scale |
| **Managed Access Schema** | Removes the object owner's ability to grant; only the schema owner can grant | Regulated environments where centralized control is required |

```sql
-- Create a schema with Managed Access
CREATE SCHEMA sales.staging WITH MANAGED ACCESS;
```

---

## 3. Object Hierarchy

Objects in Snowflake follow a containment hierarchy. An object **cannot exist** outside its container:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Organization (optional)                                                │
│  └── Account(s)                                                         │
│       ├── Users, Roles, Warehouses, Tasks, Resource Monitors,           │
│       │   Integrations, Failover Groups                                 │
│       └── Database(s)                                                   │
│            ├── Database Roles                                           │
│            └── Schema(s)                                                │
│                 └── Tables, Views, Stages, Pipes, Streams,              │
│                     UDFs, Stored Procedures, File Formats               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Objects by Level

| Level | Objects | Notes |
|-------|---------|-------|
| **Organization** | Accounts | Manages billing, replication, data sharing between accounts |
| **Account** | Users, Roles, Warehouses, Tasks, Resource Monitors, Integrations, Failover Groups | Standalone objects that do not belong to a database |
| **Database** | Schemas, Database Roles | Can be shared and replicated |
| **Schema** | Tables, Views, Stages, Pipes, Streams, UDFs, Stored Procedures, Sequences, File Formats | Recommended security boundary |

> **Practical tip:** Use Schemas as **security boundaries**. It is much easier to manage access by schema than by individual object.

---

## 4. System-Defined Roles

Snowflake automatically creates these roles with specific privileges. They form a fixed hierarchy:

```
         ORGADMIN
            │
       ACCOUNTADMIN
        ┌────┴────┐
  SECURITYADMIN   SYSADMIN
        │              │
   USERADMIN    Custom Roles (your roles)
                       │
                    PUBLIC
```

### Role Details

| Role | Responsibility | Restrictions |
|------|---------------|-------------|
| **ORGADMIN** | Manage accounts in the organization, billing, replication | Granted to only 1 user per organization |
| **ACCOUNTADMIN** | Account configuration, resource monitors, shares, sessions | **Never use day-to-day** (see best practices below) |
| **SECURITYADMIN** | Has `MANAGE GRANTS` — can modify any grant/role in the account. Inherits USERADMIN | Use when you need to manage grants globally |
| **USERADMIN** | Day-to-day creation and removal of Users and Roles | Only manages users/roles it created |
| **SYSADMIN** | Create databases, schemas, tables, warehouses. Use roles below it | Main operational role for objects |
| **PUBLIC** | Automatically granted to **all** users and roles | Any grant to PUBLIC is accessible by everyone |

### Best Practices for ACCOUNTADMIN

| Rule | Justification |
|------|---------------|
| Assign to **at least 2** and **at most 4** users | Avoid single point of failure while limiting attack surface |
| **MFA required** for everyone with ACCOUNTADMIN | Protect against credential compromise |
| **Never** set as any user's `DEFAULT_ROLE` | Avoid accidental day-to-day use |
| **Never** create objects with ACCOUNTADMIN | Objects get "locked" in this role; use SYSADMIN |
| Keep username/password as SSO backup | If SSO fails and MFA blocks access, reset can take up to 2 business days |

```sql
-- Check who has ACCOUNTADMIN
SHOW GRANTS OF ROLE ACCOUNTADMIN;

-- Check if any user has ACCOUNTADMIN as default
SELECT user_name, default_role
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE default_role = 'ACCOUNTADMIN'
  AND deleted_on IS NULL;
```

---

## 5. Functional Roles vs Access Roles

This is the **most important architectural pattern** for a scalable RBAC implementation in Snowflake.

### 5.1 Definitions

| Concept | What it is | Rules |
|---------|------------|-------|
| **Access Role** | Role that contains only privileges on objects | **Never** granted directly to a user |
| **Functional Role** | Role granted to users, contains Access Roles | What the user "assumes" to work |

### 5.2 Golden Rule

```
User ←(grant)← Functional Role ←(grant)← Access Role ←(grant)← Privileges on objects
```

```
                                    ┌─── Access Role: Read ──────► Schema X (SELECT)
                                    │
User ───► Functional Role ──────────┤
                                    │
                                    └─── Access Role: Write ─────► Schema Y (INSERT/UPDATE)
```

### 5.3 Types of Access Roles

| Type | Scope | When to use |
|------|-------|-------------|
| **Database Role** | Objects within a database | **Preferred** — access to tables, views, schemas |
| **Account Role** | Account-level objects (warehouses, integrations) | When the object does not belong to a database |

### 5.4 Types of Functional Roles

| Type | Description | How many per user |
|------|-------------|-------------------|
| **Dept-Job** (Department-Function) | Groups people by functional responsibility (e.g., Marketing-Analyst) | Typically 1 |
| **Data Product** | Grants access to a curated dataset for consumption | 1 to many |
| **User Aggregate** | Individual role that aggregates all roles for a user | Exactly 1 |

### 5.5 Recommended Naming Conventions

| Role Type | Prefix | Example |
|-----------|--------|---------|
| Functional Role | `FR_` | `FR_MARKETING_ANALYST` |
| Access Role (Account) | `AR_` | `AR_WH_ANALYTICS_USAGE` |
| Database Role (schema read) | `SC_<schema>_RO` | `SC_SALES_RO` |
| Database Role (schema write) | `SC_<schema>_RW` | `SC_SALES_RW` |
| Database Role (database level) | `DB_<database>_RO` | `DB_ANALYTICS_RO` |

---

## 6. Authorization Design Patterns

### 6.1 NOT Recommended Pattern: Object-by-Object (DAC)

```sql
-- AVOID THIS in production with many objects
GRANT SELECT ON TABLE sales.public.orders TO ROLE analyst;
GRANT SELECT ON TABLE sales.public.customers TO ROLE analyst;
GRANT SELECT ON TABLE sales.public.products TO ROLE analyst;
-- ... repeat for hundreds of tables and dozens of roles
```

**Problems:**
- Each object has an individual owner — difficult to track
- Permutations of objects x roles can exceed 10,000
- Impossible to audit whether privileges are correct
- New objects do not receive grants automatically

### 6.2 Recommended Pattern: Privileges by Schema

```sql
-- ONE read role per schema + Future Grants = simple management
GRANT USAGE ON DATABASE sales TO ROLE sc_staging_ro;
GRANT USAGE ON SCHEMA sales.staging TO ROLE sc_staging_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA sales.staging TO ROLE sc_staging_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.staging TO ROLE sc_staging_ro;
```

**Advantages:**
- One grant per schema per role per access type (RO/RW)
- Easy to audit
- Future Grants ensure automatic access to new objects
- Ideal: one `_RO` and one `_RW` role per schema

### 6.3 Managed Access Schema

When you need **only the schema owner** (and not each table's owner) to be able to grant access:

```sql
CREATE SCHEMA sales.finance WITH MANAGED ACCESS;

-- Now, even if role X creates a table in this schema,
-- only the schema owner (or SECURITYADMIN) can grant access to it.
```

**Use when:** Regulated environments, sensitive data (PII, financial, healthcare).

---

## 7. Grants: Current and Future

### 7.1 Current Grants

Apply **immediately** to objects that **already exist**:

```sql
-- Grant SELECT on ALL existing tables in the schema
GRANT SELECT ON ALL TABLES IN SCHEMA sales.staging TO ROLE sc_staging_ro;

-- Grant USAGE on ALL existing warehouses
GRANT USAGE ON ALL WAREHOUSES IN ACCOUNT TO ROLE fr_analyst;
```

### 7.2 Future Grants

Ensure that objects **created in the future** automatically receive the same privileges:

```sql
-- Tables created in the future in this schema will inherit SELECT
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.staging TO ROLE sc_staging_ro;

-- Views created in the future
GRANT SELECT ON FUTURE VIEWS IN SCHEMA sales.staging TO ROLE sc_staging_ro;

-- Future Grants at the database level (applies to all schemas)
GRANT USAGE ON FUTURE SCHEMAS IN DATABASE sales TO ROLE db_sales_ro;
```

### 7.3 Essential Combination

Always execute **both** when configuring a new role:

```sql
-- 1. Current: existing objects
GRANT SELECT ON ALL TABLES IN SCHEMA sales.staging TO ROLE sc_staging_ro;

-- 2. Future: objects to be created
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.staging TO ROLE sc_staging_ro;
```

> **Common pitfall:** Configuring only Future Grants and forgetting that existing tables did not receive access.

---

## 8. Ownership

### 8.1 What is OWNERSHIP

`OWNERSHIP` is a special privilege that is automatically assigned to the role that **created** the object (unless a Future Grant for OWNERSHIP exists).

### 8.2 What OWNERSHIP Grants

| Can | Cannot |
|-----|--------|
| Full control over the object | — |
| Grant/revoke access to the object | — |
| Rename the object | — |
| Drop the object | — |

### 8.3 What OWNERSHIP of a Role Does NOT Grant

> **Caution:** Having OWNERSHIP of a Role **does not** give access to the objects granted to that role. Ownership of a role ≠ privilege to use what the role accesses.

### 8.4 Transferring Ownership

```sql
-- Transfer ownership of a table to another role
GRANT OWNERSHIP ON TABLE sales.staging.orders
  TO ROLE sysadmin
  REVOKE CURRENT GRANTS;

-- Transfer ownership of all tables in a schema
GRANT OWNERSHIP ON ALL TABLES IN SCHEMA sales.staging
  TO ROLE sysadmin
  REVOKE CURRENT GRANTS;

-- Future Ownership: new tables are created with the correct owner
GRANT OWNERSHIP ON FUTURE TABLES IN SCHEMA sales.staging
  TO ROLE sysadmin;
```

### 8.5 Recommended Ownership Hierarchy

| Object | Recommended Owner |
|--------|-------------------|
| Databases | SYSADMIN |
| Schemas | SYSADMIN |
| Tables, Views | SYSADMIN (via Future Ownership) |
| Warehouses | SYSADMIN |
| Users and Roles | USERADMIN |

> **Rule:** Users should **never** own objects. RBAC is role-centric.

---

## 9. Role Inheritance and Context Switching

### 9.1 How Inheritance Works

When a role is granted to another, the higher role **inherits all privileges** of the lower role:

```
  FR_DATA_SCIENTIST     ← has access to EVERYTHING below
         │
    FR_ANALYST           ← has access to BI_REPORTS + PUBLIC
         │
   FR_BI_REPORTS         ← has access only to what was granted + PUBLIC
         │
      PUBLIC             ← base (all inherit)
```

In this example:
- `FR_DATA_SCIENTIST` has access to **everything** that `FR_ANALYST`, `FR_BI_REPORTS`, and `PUBLIC` can access
- `FR_ANALYST` has access to everything from `FR_BI_REPORTS` and `PUBLIC`
- `FR_BI_REPORTS` has access only to what was granted to it + `PUBLIC`

### 9.2 Context Switching (USE ROLE)

The user chooses which role to use at any time:

```sql
-- Set full context
USE ROLE fr_analyst;
USE WAREHOUSE wh_analytics_small;
USE DATABASE sales;
USE SCHEMA staging;

-- Now queries use FR_ANALYST's privileges
SELECT * FROM orders LIMIT 10;
```

When the user switches roles, privileges change **immediately**:

```sql
-- With FR_DATA_SCIENTIST: broad access
USE ROLE fr_data_scientist;
SELECT * FROM sales.working.churn_model; -- OK

-- Switch to FR_BI_REPORTS: restricted access
USE ROLE fr_bi_reports;
SELECT * FROM sales.working.churn_model; -- ERROR: Insufficient privileges
```

### 9.3 Secondary Roles

By default, only the active role (Primary Role) defines privileges. But you can enable **Secondary Roles** to combine access:

```sql
-- Activate all of the user's roles simultaneously
USE SECONDARY ROLES ALL;

-- Now the user has combined access from ALL their roles
-- Useful for Data Product Roles (the user may have several)
```

> **When to use Secondary Roles:**
> - Data Product Roles — the user needs to cross-reference data from multiple products
> - Masking policy scenarios that depend on specific roles

---

## 10. SSO and SCIM Environments: Division of Responsibilities

When the customer uses **SSO** (Single Sign-On) for authentication and **SCIM** (System for Cross-domain Identity Management) for automatic provisioning of users and groups, access management is divided between **two distinct teams**:

- **Identity Team (IdP)** — manages WHO exists and which groups they belong to
- **Snowflake Governance Administrator** — manages WHAT each group can access

> **This is the point that causes the most confusion:** Creating the user and group in the IdP **is not** sufficient to grant data access. SCIM provisioning creates the "shell" (users and functional roles) in Snowflake, but the connection to data (access roles, grants) must be done manually by the Snowflake admin.

### 10.1 Overview: IdP → Snowflake Flow

```
┌─────────────────────────────────────┐         ┌──────────────────────────────────────────┐
│  IDENTITY PROVIDER (Okta/Azure AD)  │         │           SNOWFLAKE                      │
│                                     │  SCIM   │                                          │
│  • Users (create/deactivate)        │────────►│  • Users (created automatically)         │
│  • Groups (e.g., "GRP_BI_ANALYST")  │────────►│  • Roles (created automatically)         │
│  • Membership (who is in the group) │────────►│  • Role → User Grants (automatic)        │
│                                     │         │                                          │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │         │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─     │
│  The IdP does NOT:                  │         │  The Snowflake Admin does manually:      │
│  • Create Access Roles              │         │  • Create Database Roles (Access Roles)  │
│  • Grant on tables/schemas          │         │  • GRANT Access Role → Functional Role   │
│  • Configure warehouses             │         │  • GRANT USAGE on warehouses             │
│  • Set up Future Grants             │         │  • Configure Future Grants               │
└─────────────────────────────────────┘         └──────────────────────────────────────────┘
```

### 10.2 What SCIM Provisions Automatically

When SCIM is active, the following operations are **automatic** (managed by the IdP):

| What is provisioned | How it appears in Snowflake | Example |
|---------------------|----------------------------|---------|
| New user in the IdP | Automatic `CREATE USER` | User `maria.silva` appears in Snowflake |
| Group in the IdP | Account Role created | Group `GRP_BI_ANALYST` → Role `GRP_BI_ANALYST` |
| User added to group | Automatic `GRANT ROLE ... TO USER` | Maria receives the role `GRP_BI_ANALYST` |
| User removed from group | Automatic `REVOKE ROLE ... FROM USER` | Maria loses the role |
| User deactivated in IdP | `ALTER USER ... SET DISABLED = TRUE` | Access blocked immediately |

**Result:** Roles created by SCIM function as **Functional Roles** — they are what the user "assumes" to work. However, these roles arrive in Snowflake **empty**: with no privileges on data.

### 10.3 What SCIM Does NOT Do (Snowflake Admin Responsibility)

SCIM **never** performs these operations:

| Action | Why SCIM doesn't do it | Who does |
|--------|------------------------|----------|
| Create Database Roles (Access Roles) | The IdP doesn't know Snowflake's data structure | Snowflake Governance Admin |
| `GRANT SELECT ON ALL TABLES IN SCHEMA ...` | The IdP doesn't manage granular privileges | Snowflake Governance Admin |
| `GRANT DATABASE ROLE ... TO ROLE ...` | Connecting data to groups is a business/governance decision | Snowflake Governance Admin |
| `GRANT USAGE ON WAREHOUSE ...` | Warehouses are compute resources, not identity resources | Snowflake Governance Admin |
| Configure Future Grants | Requires knowledge of the schema structure | Snowflake Governance Admin |
| Create Managed Access Schemas | Security architecture decision | Snowflake Governance Admin |

### 10.4 Complete Flow: From IdP to Data Access (Step by Step)

#### Scenario: New BI analyst needs access to sales data

**Step 1 — In the IdP (done by the Identity team):**

The Okta/Azure AD administrator adds user `carlos.souza` to group `GRP_BI_ANALYST`.

Result in Snowflake (automatic via SCIM):
```sql
-- These operations happen AUTOMATICALLY (you don't execute anything):
-- CREATE USER carlos.souza ...;
-- CREATE ROLE GRP_BI_ANALYST;  (if it didn't already exist)
-- GRANT ROLE GRP_BI_ANALYST TO USER carlos.souza;
```

**Step 2 — In Snowflake (done by the Governance Admin):**

The admin needs to ensure that the Functional Role `GRP_BI_ANALYST` has access to the correct data. This is done **once** (no need to repeat for each new group member):

```sql
-- 2a. Create the Access Roles (Database Roles) if they don't exist yet
USE ROLE SYSADMIN;
CREATE DATABASE ROLE IF NOT EXISTS sales.sc_presentation_ro;

-- 2b. Grant privileges to the Access Roles
GRANT USAGE ON SCHEMA sales.presentation TO DATABASE ROLE sales.sc_presentation_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA sales.presentation TO DATABASE ROLE sales.sc_presentation_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA sales.presentation TO DATABASE ROLE sales.sc_presentation_ro;

-- 2c. Connect the Access Role to the Functional Role (provisioned by SCIM)
USE ROLE SECURITYADMIN;
GRANT DATABASE ROLE sales.sc_presentation_ro TO ROLE grp_bi_analyst;

-- 2d. Grant warehouse
GRANT USAGE ON WAREHOUSE wh_analytics TO ROLE grp_bi_analyst;

-- 2e. Grant USAGE on the database
GRANT USAGE ON DATABASE sales TO ROLE grp_bi_analyst;
```

**Step 3 — Verification:**

```sql
-- The admin can verify that the user now has access:
SHOW GRANTS TO USER carlos.souza;
-- Should show: GRP_BI_ANALYST

SHOW GRANTS TO ROLE grp_bi_analyst;
-- Should show: USAGE on database, warehouse, and the database role

SHOW GRANTS TO DATABASE ROLE sales.sc_presentation_ro;
-- Should show: SELECT on tables in the presentation schema
```

#### Visual Summary of the Flow

```
┌──────────────────────┐    ┌─────────────────────────────────┐    ┌────────────────────────────┐
│ IdP (Automatic)      │    │ Snowflake Admin (Manual)        │    │ Final Result               │
│                      │    │                                 │    │                            │
│ 1. Creates user      │    │ 3. Creates Access Roles         │    │ User logs in via SSO       │
│ 2. Adds to group     │───►│ 4. GRANT Access Role → FR       │───►│ Uses the group role        │
│                      │    │ 5. GRANT Warehouse → FR         │    │ Accesses the tables        │
│                      │    │                                 │    │                            │
└──────────────────────┘    └─────────────────────────────────┘    └────────────────────────────┘
        WHO                           WHAT                              RESULT
```

### 10.5 Common Mistakes and How to Avoid Them

| Symptom | Root Cause | Solution |
|---------|------------|----------|
| "I created the group in Okta but the user can't see any tables" | The Functional Role (SCIM group) has no Access Roles assigned | `GRANT DATABASE ROLE ... TO ROLE <scim_group>` |
| "The user can log in but no warehouse appears" | Missing `GRANT USAGE ON WAREHOUSE` for the SCIM role | `GRANT USAGE ON WAREHOUSE wh TO ROLE <scim_group>` |
| "The user can't do USE DATABASE" | Missing `GRANT USAGE ON DATABASE` | `GRANT USAGE ON DATABASE db TO ROLE <scim_group>` |
| "New users in the group see tables, but don't see recently created tables" | Missing Future Grants on the schema | `GRANT SELECT ON FUTURE TABLES IN SCHEMA ... TO DATABASE ROLE ...` |
| "I removed the user from the group in Okta but they still access data" | SCIM may have sync delay (minutes) | Check SCIM provisioning status in the IdP; if urgent, disable manually: `ALTER USER ... SET DISABLED = TRUE` |
| "The SCIM-provisioned role doesn't appear in SYSADMIN's hierarchy" | SCIM roles are not automatically granted to SYSADMIN | `GRANT ROLE <scim_group> TO ROLE SYSADMIN` (best practice) |
| "The governance admin can't grant on the SCIM role" | The admin needs to use SECURITYADMIN (which has MANAGE GRANTS) | `USE ROLE SECURITYADMIN;` before executing GRANTs |

### 10.6 Best Practices for SCIM Environments

1. **Naming convention in the IdP:** Use clear prefixes for groups that will be provisioned (e.g., `GRP_`, `SF_`, `SNOW_`). This makes it easy to identify in Snowflake which roles came from SCIM.

2. **Connect SCIM roles to the hierarchy:** Always execute `GRANT ROLE <scim_group> TO ROLE SYSADMIN` to avoid orphan roles.

3. **Document the responsibility matrix:**

| Action | Responsible | Tool |
|--------|-------------|------|
| Create/deactivate user | Identity Team | Okta/Azure AD |
| Create/modify group | Identity Team | Okta/Azure AD |
| Add user to group | Team Manager | Okta/Azure AD |
| Create Access Roles | Snow Governance Admin | SQL in Snowflake |
| Connect Access Role to Functional Role | Snow Governance Admin | SQL in Snowflake |
| Grant warehouses | Snow Governance Admin | SQL in Snowflake |
| Audit access | Snow Governance Admin | SQL in Snowflake |

4. **Configure SCIM once, manage grants continuously:** The initial work of setting up SCIM integration is a one-time effort. However, as new schemas, databases, or data products are created, the Snowflake admin must continuously create Access Roles and connect them to existing Functional Roles.

5. **Use the correct `DEFAULT_ROLE`:** Configure in the IdP or in Snowflake so that the user's `DEFAULT_ROLE` is their primary Functional Role:

```sql
-- Check default roles of SCIM-provisioned users
SELECT user_name, default_role, has_password, ext_authn_duo
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND created_on >= DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY created_on DESC;
```

### 10.7 SQL Reference for SCIM Environments

```sql
-- View all roles created by SCIM (name pattern with prefix)
SHOW ROLES LIKE 'GRP_%';

-- View recently provisioned users (no password = likely SSO/SCIM)
SELECT user_name, login_name, display_name, default_role, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE has_password = 'false'
  AND deleted_on IS NULL
ORDER BY created_on DESC;

-- View which SCIM roles already have Access Roles assigned
SHOW GRANTS TO ROLE grp_bi_analyst;

-- View which SCIM roles do NOT have any Database Role (possible issue)
-- Compare SCIM-prefixed roles against those with database role grants
SELECT r.name AS role_scim
FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES r
WHERE r.name LIKE 'GRP_%'
  AND r.deleted_on IS NULL
  AND r.name NOT IN (
    SELECT grantee_name
    FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
    WHERE granted_on = 'DATABASE_ROLE'
      AND deleted_on IS NULL
  );

-- Batch assign Access Roles to a SCIM Functional Role
USE ROLE SECURITYADMIN;
GRANT DATABASE ROLE sales.sc_raw_ro TO ROLE grp_data_engineer;
GRANT DATABASE ROLE sales.sc_staging_rw TO ROLE grp_data_engineer;
GRANT DATABASE ROLE sales.sc_presentation_ro TO ROLE grp_data_engineer;
GRANT USAGE ON WAREHOUSE wh_ingestion TO ROLE grp_data_engineer;
GRANT USAGE ON DATABASE sales TO ROLE grp_data_engineer;

-- Audit: who accessed what in the last 24h via SCIM roles
SELECT user_name, role_name, query_type, LEFT(query_text, 100) AS query_preview, start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE role_name LIKE 'GRP_%'
  AND start_time >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
ORDER BY start_time DESC
LIMIT 100;
```

---

## 11. Complete Practical Examples

### 11.1 Scenario: Configure Access for a Company

Fictional company structure:
- Database: `ANALYTICS`
- Schemas: `RAW`, `STAGING`, `PRESENTATION`
- Teams: Data Engineering, BI, Data Science

#### Step 1: Create Database Roles (Access Roles)

```sql
USE ROLE SYSADMIN;

-- Database roles for RAW schema
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_raw_ro;
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_raw_rw;

-- Database roles for STAGING schema
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_staging_ro;
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_staging_rw;

-- Database roles for PRESENTATION schema
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_presentation_ro;
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_presentation_rw;
```

#### Step 2: Grant Privileges to Access Roles

```sql
-- === RAW Schema - Read ===
GRANT USAGE ON SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON ALL VIEWS IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;

-- === RAW Schema - Write ===
GRANT USAGE ON SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;
GRANT CREATE TABLE ON SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;

-- === STAGING Schema - Read ===
GRANT USAGE ON SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_ro;

-- === STAGING Schema - Write ===
GRANT USAGE ON SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;
GRANT CREATE TABLE, CREATE VIEW ON SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;

-- === PRESENTATION Schema - Read ===
GRANT USAGE ON SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON ALL VIEWS IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;

-- === PRESENTATION Schema - Write ===
GRANT USAGE ON SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
GRANT CREATE TABLE, CREATE VIEW ON SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
```

#### Step 3: Create Functional Roles and Assign Access Roles

```sql
USE ROLE USERADMIN;

-- Functional Roles by team
CREATE ROLE IF NOT EXISTS fr_data_engineering;
CREATE ROLE IF NOT EXISTS fr_bi_analyst;
CREATE ROLE IF NOT EXISTS fr_data_scientist;

-- Ensure SYSADMIN inherits all custom roles
GRANT ROLE fr_data_engineering TO ROLE SYSADMIN;
GRANT ROLE fr_bi_analyst TO ROLE SYSADMIN;
GRANT ROLE fr_data_scientist TO ROLE SYSADMIN;

USE ROLE SECURITYADMIN;

-- Data Engineering: write to RAW and STAGING, read from PRESENTATION
GRANT DATABASE ROLE analytics.sc_raw_rw TO ROLE fr_data_engineering;
GRANT DATABASE ROLE analytics.sc_staging_rw TO ROLE fr_data_engineering;
GRANT DATABASE ROLE analytics.sc_presentation_ro TO ROLE fr_data_engineering;

-- BI Analyst: read from STAGING and PRESENTATION
GRANT DATABASE ROLE analytics.sc_staging_ro TO ROLE fr_bi_analyst;
GRANT DATABASE ROLE analytics.sc_presentation_ro TO ROLE fr_bi_analyst;

-- Data Scientist: read from everything, write to STAGING
GRANT DATABASE ROLE analytics.sc_raw_ro TO ROLE fr_data_scientist;
GRANT DATABASE ROLE analytics.sc_staging_rw TO ROLE fr_data_scientist;
GRANT DATABASE ROLE analytics.sc_presentation_ro TO ROLE fr_data_scientist;

-- USAGE on the database for all functional roles
GRANT USAGE ON DATABASE analytics TO ROLE fr_data_engineering;
GRANT USAGE ON DATABASE analytics TO ROLE fr_bi_analyst;
GRANT USAGE ON DATABASE analytics TO ROLE fr_data_scientist;
```

#### Step 4: Create Warehouses and Grant Access

```sql
USE ROLE SYSADMIN;

-- Warehouses by usage profile
CREATE WAREHOUSE IF NOT EXISTS wh_ingestion
  WAREHOUSE_SIZE = 'MEDIUM'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE;

CREATE WAREHOUSE IF NOT EXISTS wh_analytics
  WAREHOUSE_SIZE = 'SMALL'
  AUTO_SUSPEND = 120
  AUTO_RESUME = TRUE;

CREATE WAREHOUSE IF NOT EXISTS wh_datascience
  WAREHOUSE_SIZE = 'LARGE'
  AUTO_SUSPEND = 300
  AUTO_RESUME = TRUE;

-- Grant USAGE on warehouses
GRANT USAGE ON WAREHOUSE wh_ingestion TO ROLE fr_data_engineering;
GRANT USAGE ON WAREHOUSE wh_analytics TO ROLE fr_bi_analyst;
GRANT USAGE ON WAREHOUSE wh_analytics TO ROLE fr_data_scientist;
GRANT USAGE ON WAREHOUSE wh_datascience TO ROLE fr_data_scientist;
```

#### Step 5: Create Users and Assign Roles

```sql
USE ROLE USERADMIN;

-- Create human user
CREATE USER IF NOT EXISTS maria_silva
  PASSWORD = 'ChangeOnFirstLogin!'
  DEFAULT_ROLE = 'FR_BI_ANALYST'
  DEFAULT_WAREHOUSE = 'WH_ANALYTICS'
  MUST_CHANGE_PASSWORD = TRUE;

-- Assign functional role to user
GRANT ROLE fr_bi_analyst TO USER maria_silva;

-- Create service account (no password — uses key pair)
CREATE USER IF NOT EXISTS svc_pipeline_ingestion
  DEFAULT_ROLE = 'FR_DATA_ENGINEERING'
  DEFAULT_WAREHOUSE = 'WH_INGESTION'
  TYPE = SERVICE;

GRANT ROLE fr_data_engineering TO USER svc_pipeline_ingestion;
```

### 11.2 Access Verification

```sql
-- View all roles for a user
SHOW GRANTS TO USER maria_silva;

-- View all privileges for a role
SHOW GRANTS TO ROLE fr_bi_analyst;

-- View who has access to a specific table
SHOW GRANTS ON TABLE analytics.presentation.monthly_sales;

-- View the complete role hierarchy
SHOW GRANTS OF ROLE fr_data_scientist;
```

---

## 12. Security Configuration Checklist

Use this list when configuring a new account or auditing an existing one:

### Layer 1: Network Access

- [ ] Network Policy created to restrict access IPs
- [ ] Consider Private Link if the organization requires private traffic

```sql
-- Example: create network policy
CREATE NETWORK POLICY IF NOT EXISTS corp_network_policy
  ALLOWED_IP_LIST = ('10.0.0.0/8', '172.16.0.0/12', '200.100.50.0/24')
  COMMENT = 'Allows only corporate network';

-- Apply to the account
ALTER ACCOUNT SET NETWORK_POLICY = 'CORP_NETWORK_POLICY';
```

### Layer 2: Authentication

- [ ] MFA enabled for **all** users with ACCOUNTADMIN
- [ ] MFA enabled for SECURITYADMIN
- [ ] SSO configured (SAML 2.0) as primary method
- [ ] Key Pair configured for service accounts (no password)
- [ ] Password policy defined

```sql
-- Check MFA status for admins
SELECT user_name, has_mfa_enrolled, default_role
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND default_role IN ('ACCOUNTADMIN', 'SECURITYADMIN');
```

### Layer 3: Authorization (RBAC)

- [ ] ACCOUNTADMIN assigned to 2-4 users only
- [ ] No user has ACCOUNTADMIN as DEFAULT_ROLE
- [ ] No objects created with ACCOUNTADMIN (should be created with SYSADMIN)
- [ ] Role hierarchy defined: System Roles > Functional Roles > Access Roles
- [ ] Database Roles used for database object access
- [ ] Future Grants configured on all schemas
- [ ] Managed Access Schema used for sensitive data
- [ ] All custom roles inherit to SYSADMIN (no "orphan" roles)
- [ ] PUBLIC role has no unnecessary grants

```sql
-- Check for orphan roles (not connected to the hierarchy)
-- Roles not granted to any other role
SELECT r.name AS role_name
FROM SNOWFLAKE.ACCOUNT_USAGE.ROLES r
WHERE r.deleted_on IS NULL
  AND r.name NOT IN ('ACCOUNTADMIN','SECURITYADMIN','USERADMIN','SYSADMIN','ORGADMIN','PUBLIC')
  AND r.name NOT IN (
    SELECT DISTINCT grantee_name
    FROM SNOWFLAKE.ACCOUNT_USAGE.GRANTS_TO_ROLES
    WHERE granted_on = 'ROLE'
      AND deleted_on IS NULL
  );
```

### Layer 4: Continuous Data Protection

- [ ] Time Travel configured (default: 1 day; production: up to 90 days)
- [ ] Data Masking Policies applied to PII columns
- [ ] Row Access Policies for multi-tenant data

### Auditing and Monitoring

- [ ] Resource Monitors configured to control costs
- [ ] ACCOUNT_USAGE monitored periodically
- [ ] Alerts for ACCOUNTADMIN grants
- [ ] Alerts for logins from unknown IPs

---

## 13. Appendix: Quick Reference SQL Commands

### Role Management

| Action | SQL |
|--------|-----|
| Create role | `CREATE ROLE fr_name;` |
| Create database role | `CREATE DATABASE ROLE db.role_name;` |
| Grant role to user | `GRANT ROLE fr_name TO USER john;` |
| Grant role to another role | `GRANT ROLE fr_junior TO ROLE fr_senior;` |
| Grant database role to role | `GRANT DATABASE ROLE db.role TO ROLE fr_name;` |
| Remove role from user | `REVOKE ROLE fr_name FROM USER john;` |
| List roles | `SHOW ROLES;` |
| List database roles | `SHOW DATABASE ROLES IN DATABASE db;` |

### Grant Management

| Action | SQL |
|--------|-----|
| Grant read on schema | `GRANT USAGE ON SCHEMA db.sch TO ROLE r; GRANT SELECT ON ALL TABLES IN SCHEMA db.sch TO ROLE r;` |
| Future grant on tables | `GRANT SELECT ON FUTURE TABLES IN SCHEMA db.sch TO ROLE r;` |
| Grant warehouse | `GRANT USAGE ON WAREHOUSE wh TO ROLE r;` |
| Revoke grant | `REVOKE SELECT ON TABLE db.sch.t FROM ROLE r;` |
| View grants on an object | `SHOW GRANTS ON TABLE db.sch.t;` |
| View grants for a role | `SHOW GRANTS TO ROLE r;` |
| View grants for a user | `SHOW GRANTS TO USER u;` |

### User Management

| Action | SQL |
|--------|-----|
| Create user | `CREATE USER name PASSWORD='x' DEFAULT_ROLE='r' MUST_CHANGE_PASSWORD=TRUE;` |
| Create service account | `CREATE USER svc_name DEFAULT_ROLE='r' TYPE=SERVICE;` |
| Change default role | `ALTER USER name SET DEFAULT_ROLE = 'new_role';` |
| Disable user | `ALTER USER name SET DISABLED = TRUE;` |
| Drop user | `DROP USER name;` |

### Ownership Management

| Action | SQL |
|--------|-----|
| Transfer ownership | `GRANT OWNERSHIP ON TABLE db.sch.t TO ROLE r REVOKE CURRENT GRANTS;` |
| Future ownership | `GRANT OWNERSHIP ON FUTURE TABLES IN SCHEMA db.sch TO ROLE r;` |

### Context Switching

| Action | SQL |
|--------|-----|
| Switch role | `USE ROLE fr_name;` |
| Activate secondary roles | `USE SECONDARY ROLES ALL;` |
| Set warehouse | `USE WAREHOUSE wh_name;` |
| Set database | `USE DATABASE db_name;` |
| Set schema | `USE SCHEMA schema_name;` |

---

## Glossary

| Term | Definition |
|------|-----------|
| **RBAC** | Role-Based Access Control — model where privileges are assigned to Roles |
| **DAC** | Discretionary Access Control — the object owner controls access |
| **MAS** | Managed Access Schema — schema where only the schema owner manages grants |
| **Grant** | Command that grants a privilege to a role |
| **Revoke** | Command that removes a privilege from a role |
| **Ownership** | Special privilege of full control over an object |
| **Future Grant** | Grant that automatically applies to objects created in the future |
| **Functional Role** | Role assigned to users, aggregates Access Roles |
| **Access Role** | Role that contains only privileges on objects |
| **Database Role** | Role that exists within a specific database |
| **Secondary Roles** | Mechanism to activate multiple roles simultaneously |
| **Network Policy** | Rule that restricts which IPs can access the account |
| **MFA** | Multi-Factor Authentication — extra layer of identity verification |
| **Service Account** | Non-human user used by applications and pipelines |

---

> **Document generated as a reference for Snowflake security configuration.**
> Based on the "RBAC Access Management Overview" material from the Snowflake Activation Team.
