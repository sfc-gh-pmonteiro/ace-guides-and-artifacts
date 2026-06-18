# Guía de Referencia: Control de Acceso y Seguridad en Snowflake

> **Público objetivo:** Equipos con experiencia en SQL (Oracle, SQL Server, PostgreSQL) que están comenzando con Snowflake.

> **Objetivo:** Proporcionar una referencia práctica y completa para configurar seguridad y control de acceso en cualquier cuenta Snowflake.

---

## Contenido

1. [Seguridad en Capas](#1-seguridad-en-capas)
2. [Conceptos Fundamentales de Control de Acceso](#2-conceptos-fundamentales-de-control-de-acceso)
3. [Jerarquía de Objetos](#3-jerarquía-de-objetos)
4. [Roles del Sistema (System-Defined Roles)](#4-roles-del-sistema-system-defined-roles)
5. [Functional Roles vs Access Roles](#5-functional-roles-vs-access-roles)
6. [Patrones de Autorización (Design Patterns)](#6-patrones-de-autorización-design-patterns)
7. [Grants: Current y Future](#7-grants-current-y-future)
8. [Ownership (Propiedad)](#8-ownership-propiedad)
9. [Herencia de Roles y Cambio de Contexto](#9-herencia-de-roles-y-cambio-de-contexto)
10. [Ambientes con SSO y SCIM: División de Responsabilidades](#10-ambientes-con-sso-y-scim-división-de-responsabilidades)
11. [Ejemplos Prácticos Completos](#11-ejemplos-prácticos-completos)
12. [Checklist de Configuración de Seguridad](#12-checklist-de-configuración-de-seguridad)
13. [Apéndice: Comandos SQL de Referencia Rápida](#13-apéndice-comandos-sql-de-referencia-rápida)

---

## 1. Seguridad en Capas

Snowflake implementa seguridad en **cuatro capas** complementarias. Cada capa actúa de forma independiente, creando defensa en profundidad:

```
┌──────────────────┐    ┌──────────────────────────┐    ┌──────────────────────────┐    ┌─────────────────────────────────┐
│ 1. Network Access│───►│ 2. Authentication (AuthN)│───►│ 3. Authorization (AuthZ) │───►│ 4. Continuous Data Protection   │
│   (IP/Red)       │    │   (Identidad)            │    │ (RBAC - Foco de la guía) │    │   (Cifrado/Time Travel)         │
└──────────────────┘    └──────────────────────────┘    └──────────────────────────┘    └─────────────────────────────────┘
```

| Capa | Qué hace | Mecanismos |
|------|----------|------------|
| **1. Network Access** | Restringe qué IPs o redes pueden acceder a la cuenta | Network Policies, AWS PrivateLink, Azure Private Link, GCP Private Service Connect |
| **2. Authentication (AuthN)** | Verifica la identidad del usuario o proceso | Username/Password, SSO (SAML 2.0), Key Pair, OAuth, MFA |
| **3. Authorization (AuthZ)** | Define qué puede hacer cada identidad | **RBAC** — Roles, Privileges, Grants |
| **4. Continuous Data Protection** | Protege los datos en reposo y en tránsito | Cifrado AES-256, Re-keying anual, Time Travel (hasta 90 días), Fail-safe, Data Masking, Row Access Policies, Tokenización |

**Foco de esta guía:** Capa 3 — Authorization vía RBAC.

> **Nota para quienes vienen de otras bases de datos:** En Snowflake **no** se pueden conceder privilegios directamente a usuarios. Toda autorización pasa por Roles. Los usuarios solo "asumen" un Role para acceder a objetos.

---

## 2. Conceptos Fundamentales de Control de Acceso

### 2.1 Securable Object (Objeto Protegido)

Cualquier entidad a la cual se puede conceder acceso. Si no existe un GRANT explícito, el acceso es **denegado por defecto**.

Ejemplos: Database, Schema, Table, View, Stage, Warehouse, Task, Stream, Stored Procedure, Function.

### 2.2 Privilege (Privilegio)

Un permiso específico sobre un objeto. Ejemplos:

| Privilegio | Se aplica a | Permite |
|------------|-------------|---------|
| `USAGE` | Database, Schema, Warehouse | Acceder/usar el objeto |
| `SELECT` | Table, View | Leer datos |
| `INSERT` | Table | Insertar datos |
| `CREATE TABLE` | Schema | Crear tablas en el schema |
| `OPERATE` | Warehouse | Iniciar/detener/redimensionar |
| `OWNERSHIP` | Cualquier objeto | Control total (ver sección 8) |

### 2.3 Role

Una entidad a la cual se conceden privilegios. Existen dos tipos:

- **Account Role** — existe a nivel de la cuenta, puede asignarse a usuarios
- **Database Role** — existe dentro de un database, ideal para controlar acceso a objetos de ese database

### 2.4 User (Usuario)

Representa una identidad que se conecta a Snowflake. Puede ser:

- **Humano** — persona real que inicia sesión
- **Service Account** — cuenta de servicio usada por aplicaciones, pipelines de datos, etc.

### 2.5 Métodos de Control de Acceso

| Método | Descripción | Cuándo usar |
|--------|-------------|-------------|
| **RBAC** (Role-Based Access Control) | Privilegios se asignan a Roles, Roles a Users | **Estándar recomendado** para toda configuración |
| **DAC** (Discretionary Access Control) | El dueño del objeto puede conceder acceso a otros | Funciona, pero difícil de mantener a escala |
| **Managed Access Schema** | Quita al dueño del objeto la capacidad de dar grants; solo el dueño del schema puede conceder | Ambientes regulados donde se requiere control centralizado |

```sql
-- Crear un schema con Managed Access
CREATE SCHEMA ventas.staging WITH MANAGED ACCESS;
```

---

## 3. Jerarquía de Objetos

Los objetos en Snowflake siguen una jerarquía de contención. Un objeto **no puede existir** fuera de su contenedor:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Organization (opcional)                                                │
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

### Objetos por Nivel

| Nivel | Objetos | Observación |
|-------|---------|-------------|
| **Organization** | Accounts | Gestiona billing, replicación, data sharing entre cuentas |
| **Account** | Users, Roles, Warehouses, Tasks, Resource Monitors, Integrations, Failover Groups | Objetos "standalone" que no pertenecen a un database |
| **Database** | Schemas, Database Roles | Puede compartirse y replicarse |
| **Schema** | Tables, Views, Stages, Pipes, Streams, UDFs, Stored Procedures, Sequences, File Formats | Frontera de seguridad recomendada |

> **Consejo práctico:** Use Schemas como **frontera de seguridad**. Es mucho más fácil gestionar acceso por schema que por objeto individual.

---

## 4. Roles del Sistema (System-Defined Roles)

Snowflake crea automáticamente estos roles con privilegios específicos. Forman una jerarquía fija:

```
         ORGADMIN
            │
       ACCOUNTADMIN
        ┌────┴────┐
  SECURITYADMIN   SYSADMIN
        │              │
   USERADMIN    Custom Roles (sus roles)
                       │
                    PUBLIC
```

### Detalle de Cada Role

| Role | Responsabilidad | Restricciones |
|------|-----------------|---------------|
| **ORGADMIN** | Gestionar cuentas en la organización, billing, replicación | Concedido a solo 1 usuario por organización |
| **ACCOUNTADMIN** | Configuración de la cuenta, resource monitors, shares, sesiones | **Nunca usar en el día a día** (ver buenas prácticas abajo) |
| **SECURITYADMIN** | Tiene `MANAGE GRANTS` — puede modificar cualquier grant/role en la cuenta. Hereda USERADMIN | Usar cuando se necesite gestionar grants globalmente |
| **USERADMIN** | Creación y eliminación de Users y Roles en el día a día | Solo gestiona users/roles que él creó |
| **SYSADMIN** | Crear databases, schemas, tablas, warehouses. Usar roles debajo de ella | Role operacional principal para objetos |
| **PUBLIC** | Concedido automáticamente a **todos** los users y roles | Cualquier grant a PUBLIC es accesible por todos |

### Buenas Prácticas para ACCOUNTADMIN

| Regla | Justificación |
|-------|---------------|
| Asignar a **mínimo 2** y **máximo 4** usuarios | Evitar punto único de falla, pero limitar superficie de ataque |
| **MFA obligatorio** para todos con ACCOUNTADMIN | Proteger contra compromiso de credenciales |
| **Nunca** definir como `DEFAULT_ROLE` de ningún usuario | Evitar uso accidental en el día a día |
| **Nunca** crear objetos con ACCOUNTADMIN | Los objetos quedan "atrapados" en este role; use SYSADMIN |
| Mantener username/password como respaldo al SSO | Si SSO falla y MFA bloquea, el reset puede tomar hasta 2 días hábiles |

```sql
-- Verificar quién tiene ACCOUNTADMIN
SHOW GRANTS OF ROLE ACCOUNTADMIN;

-- Verificar si algún usuario tiene ACCOUNTADMIN como default
SELECT user_name, default_role
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE default_role = 'ACCOUNTADMIN'
  AND deleted_on IS NULL;
```

---

## 5. Functional Roles vs Access Roles

Este es el **patrón arquitectónico más importante** para una implementación escalable de RBAC en Snowflake.

### 5.1 Definiciones

| Concepto | Qué es | Reglas |
|----------|--------|--------|
| **Access Role** | Role que contiene solo privilegios sobre objetos | **Nunca** se concede directamente a un usuario |
| **Functional Role** | Role que se concede a usuarios y contiene Access Roles | Es lo que el usuario "asume" para trabajar |

### 5.2 Regla de Oro

```
Usuario ←(grant)← Functional Role ←(grant)← Access Role ←(grant)← Privilegios sobre objetos
```

```
                                    ┌─── Access Role: Lectura ───► Schema X (SELECT)
                                    │
Usuario ───► Functional Role ───────┤
                                    │
                                    └─── Access Role: Escritura ──► Schema Y (INSERT/UPDATE)
```

### 5.3 Tipos de Access Roles

| Tipo | Alcance | Cuándo usar |
|------|---------|-------------|
| **Database Role** | Objetos dentro de un database | **Preferido** — acceso a tablas, views, schemas |
| **Account Role** | Objetos de cuenta (warehouses, integrations) | Cuando el objeto no pertenece a un database |

### 5.4 Tipos de Functional Roles

| Tipo | Descripción | Cuántos por usuario |
|------|-------------|---------------------|
| **Dept-Job** (Departamento-Función) | Agrupa personas por responsabilidad funcional (ej: Marketing-Analista) | Típicamente 1 |
| **Data Product** | Da acceso a un conjunto de datos preparado para consumo | 1 a muchos |
| **User Aggregate** | Role individual que agrega todos los roles de un usuario | Exactamente 1 |

### 5.5 Convenciones de Nomenclatura Recomendadas

| Tipo de Role | Prefijo | Ejemplo |
|--------------|---------|---------|
| Functional Role | `FR_` | `FR_MARKETING_ANALYST` |
| Access Role (Account) | `AR_` | `AR_WH_ANALYTICS_USAGE` |
| Database Role (lectura de schema) | `SC_<schema>_RO` | `SC_VENTAS_RO` |
| Database Role (escritura en schema) | `SC_<schema>_RW` | `SC_VENTAS_RW` |
| Database Role (nivel de database) | `DB_<database>_RO` | `DB_ANALYTICS_RO` |

---

## 6. Patrones de Autorización (Design Patterns)

### 6.1 Patrón NO Recomendado: Objeto por Objeto (DAC)

```sql
-- EVITE ESTO en producción con muchos objetos
GRANT SELECT ON TABLE ventas.public.pedidos TO ROLE analista;
GRANT SELECT ON TABLE ventas.public.clientes TO ROLE analista;
GRANT SELECT ON TABLE ventas.public.productos TO ROLE analista;
-- ... repita para cientos de tablas y decenas de roles
```

**Problemas:**
- Cada objeto tiene un dueño individual — difícil de rastrear
- Las permutaciones de objetos x roles pueden superar 10,000
- Imposible auditar si los privilegios son correctos
- Los objetos nuevos no reciben grants automáticamente

### 6.2 Patrón Recomendado: Privilegios por Schema

```sql
-- UN role de lectura por schema + Future Grants = gestión simple
GRANT USAGE ON DATABASE ventas TO ROLE sc_staging_ro;
GRANT USAGE ON SCHEMA ventas.staging TO ROLE sc_staging_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA ventas.staging TO ROLE sc_staging_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ventas.staging TO ROLE sc_staging_ro;
```

**Ventajas:**
- Un grant por schema por role por tipo de acceso (RO/RW)
- Fácil de auditar
- Los Future Grants garantizan acceso automático a objetos nuevos
- Ideal: un role `_RO` y un `_RW` por schema

### 6.3 Managed Access Schema

Cuando necesita que **solo el dueño del schema** (y no el dueño de cada tabla) pueda conceder acceso:

```sql
CREATE SCHEMA ventas.financiero WITH MANAGED ACCESS;

-- Ahora, aunque el role X cree una tabla en este schema,
-- solo el dueño del schema (o SECURITYADMIN) puede dar grants sobre ella.
```

**Usar cuando:** Ambientes regulados, datos sensibles (PII, financiero, salud).

---

## 7. Grants: Current y Future

### 7.1 Current Grants

Se aplican **inmediatamente** a objetos que **ya existen**:

```sql
-- Conceder SELECT en TODAS las tablas existentes en el schema
GRANT SELECT ON ALL TABLES IN SCHEMA ventas.staging TO ROLE sc_staging_ro;

-- Conceder USAGE en TODOS los warehouses existentes
GRANT USAGE ON ALL WAREHOUSES IN ACCOUNT TO ROLE fr_analista;
```

### 7.2 Future Grants

Garantizan que objetos **creados en el futuro** reciban automáticamente los mismos privilegios:

```sql
-- Tablas creadas en el futuro en este schema heredarán SELECT
GRANT SELECT ON FUTURE TABLES IN SCHEMA ventas.staging TO ROLE sc_staging_ro;

-- Views creadas en el futuro
GRANT SELECT ON FUTURE VIEWS IN SCHEMA ventas.staging TO ROLE sc_staging_ro;

-- Future Grants a nivel del database (se aplica a todos los schemas)
GRANT USAGE ON FUTURE SCHEMAS IN DATABASE ventas TO ROLE db_ventas_ro;
```

### 7.3 Combinación Esencial

Siempre ejecute **ambos** al configurar un nuevo role:

```sql
-- 1. Current: objetos existentes
GRANT SELECT ON ALL TABLES IN SCHEMA ventas.staging TO ROLE sc_staging_ro;

-- 2. Future: objetos que se crearán
GRANT SELECT ON FUTURE TABLES IN SCHEMA ventas.staging TO ROLE sc_staging_ro;
```

> **Error común:** Configurar solo Future Grants y olvidar que las tablas existentes no recibieron acceso.

---

## 8. Ownership (Propiedad)

### 8.1 Qué es OWNERSHIP

`OWNERSHIP` es un privilegio especial que se asigna automáticamente al role que **creó** el objeto (a menos que exista un Future Grant de OWNERSHIP).

### 8.2 Qué concede OWNERSHIP

| Puede | No puede |
|-------|----------|
| Control total sobre el objeto | — |
| Conceder/revocar acceso al objeto | — |
| Renombrar el objeto | — |
| Eliminar (DROP) el objeto | — |

### 8.3 Lo que OWNERSHIP de un Role NO concede

> **Atención:** Tener OWNERSHIP de un Role **no** da acceso a los objetos concedidos a ese role. Ownership de role ≠ privilegio de usar lo que el role accede.

### 8.4 Transferencia de Ownership

```sql
-- Transferir ownership de una tabla a otro role
GRANT OWNERSHIP ON TABLE ventas.staging.pedidos
  TO ROLE sysadmin
  REVOKE CURRENT GRANTS;

-- Transferir ownership de todas las tablas en un schema
GRANT OWNERSHIP ON ALL TABLES IN SCHEMA ventas.staging
  TO ROLE sysadmin
  REVOKE CURRENT GRANTS;

-- Future Ownership: tablas nuevas ya nacen con el dueño correcto
GRANT OWNERSHIP ON FUTURE TABLES IN SCHEMA ventas.staging
  TO ROLE sysadmin;
```

### 8.5 Jerarquía de Ownership Recomendada

| Objeto | Owner recomendado |
|--------|-------------------|
| Databases | SYSADMIN |
| Schemas | SYSADMIN |
| Tablas, Views | SYSADMIN (vía Future Ownership) |
| Warehouses | SYSADMIN |
| Users y Roles | USERADMIN |

> **Regla:** Los usuarios **nunca** deben ser dueños de objetos. RBAC está centrado en Roles.

---

## 9. Herencia de Roles y Cambio de Contexto

### 9.1 Cómo Funciona la Herencia

Cuando un role se concede a otro, el role superior **hereda todos los privilegios** del role inferior:

```
  FR_DATA_SCIENTIST     ← tiene acceso a TODO lo de abajo
         │
    FR_ANALYST           ← tiene acceso a BI_REPORTS + PUBLIC
         │
   FR_BI_REPORTS         ← tiene acceso solo a lo concedido + PUBLIC
         │
      PUBLIC             ← base (todos heredan)
```

En este ejemplo:
- `FR_DATA_SCIENTIST` tiene acceso a **todo** lo que `FR_ANALYST`, `FR_BI_REPORTS` y `PUBLIC` pueden acceder
- `FR_ANALYST` tiene acceso a todo de `FR_BI_REPORTS` y `PUBLIC`
- `FR_BI_REPORTS` tiene acceso solo a lo concedido a ella + `PUBLIC`

### 9.2 Cambio de Contexto (USE ROLE)

El usuario elige qué role usar en cualquier momento:

```sql
-- Definir contexto completo
USE ROLE fr_analyst;
USE WAREHOUSE wh_analytics_small;
USE DATABASE ventas;
USE SCHEMA staging;

-- Ahora las consultas usan los privilegios de FR_ANALYST
SELECT * FROM pedidos LIMIT 10;
```

Cuando el usuario cambia de role, los privilegios cambian **inmediatamente**:

```sql
-- Con FR_DATA_SCIENTIST: acceso amplio
USE ROLE fr_data_scientist;
SELECT * FROM ventas.working.modelo_churn; -- OK

-- Cambio a FR_BI_REPORTS: acceso restringido
USE ROLE fr_bi_reports;
SELECT * FROM ventas.working.modelo_churn; -- ERROR: Insufficient privileges
```

### 9.3 Secondary Roles

Por defecto, solo el role activo (Primary Role) define los privilegios. Pero se pueden activar **Secondary Roles** para combinar accesos:

```sql
-- Activar todos los roles del usuario simultáneamente
USE SECONDARY ROLES ALL;

-- Ahora el usuario tiene acceso combinado de TODOS sus roles
-- Útil para Data Product Roles (el usuario puede tener varios)
```

> **Cuándo usar Secondary Roles:**
> - Data Product Roles — el usuario necesita cruzar datos de múltiples productos
> - Escenarios de masking policy que dependen de roles específicos

---

## 10. Ambientes con SSO y SCIM: División de Responsabilidades

Cuando el cliente utiliza **SSO** (Single Sign-On) para autenticación y **SCIM** (System for Cross-domain Identity Management) para aprovisionamiento automático de usuarios y grupos, la gestión de acceso se divide entre **dos equipos distintos**:

- **Equipo de Identidad (IdP)** — gestiona QUIÉN existe y a qué grupos pertenece
- **Administrador de Gobernanza Snowflake** — gestiona QUÉ puede acceder cada grupo

> **Este es el punto que más genera confusión:** Crear el usuario y el grupo en el IdP **no** es suficiente para dar acceso a datos. El aprovisionamiento SCIM crea la "cáscara" (users y functional roles) en Snowflake, pero la conexión con los datos (access roles, grants) debe hacerse manualmente por el admin Snowflake.

### 10.1 Visión General: Flujo IdP → Snowflake

```
┌─────────────────────────────────────┐         ┌──────────────────────────────────────────┐
│  IDENTITY PROVIDER (Okta/Azure AD)  │         │           SNOWFLAKE                      │
│                                     │  SCIM   │                                          │
│  • Usuarios (crear/desactivar)      │────────►│  • Users (creados automáticamente)       │
│  • Grupos (ej: "GRP_BI_ANALYST")    │────────►│  • Roles (creados automáticamente)       │
│  • Membership (quién está en grupo) │────────►│  • Grants de Role → User (automático)    │
│                                     │         │                                          │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │         │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─     │
│  El IdP NO hace:                    │         │  El Admin Snowflake hace manualmente:    │
│  • Crear Access Roles               │         │  • Crear Database Roles (Access Roles)   │
│  • Dar grants en tablas/schemas     │         │  • GRANT Access Role → Functional Role   │
│  • Configurar warehouses            │         │  • GRANT USAGE en warehouses             │
│  • Definir Future Grants            │         │  • Configurar Future Grants              │
└─────────────────────────────────────┘         └──────────────────────────────────────────┘
```

### 10.2 Lo que SCIM Aprovisiona Automáticamente

Cuando SCIM está activo, las siguientes operaciones son **automáticas** (gestionadas por el IdP):

| Qué se aprovisiona | Cómo aparece en Snowflake | Ejemplo |
|--------------------|---------------------------|---------|
| Usuario nuevo en el IdP | `CREATE USER` automático | Usuario `maria.silva` aparece en Snowflake |
| Grupo en el IdP | Account Role creado | Grupo `GRP_BI_ANALYST` → Role `GRP_BI_ANALYST` |
| Usuario agregado al grupo | `GRANT ROLE ... TO USER` automático | Maria recibe el role `GRP_BI_ANALYST` |
| Usuario removido del grupo | `REVOKE ROLE ... FROM USER` automático | Maria pierde el role |
| Usuario desactivado en IdP | `ALTER USER ... SET DISABLED = TRUE` | Acceso bloqueado inmediatamente |

**Resultado:** Los roles creados por SCIM funcionan como **Functional Roles** — son lo que el usuario "asume" para trabajar. Sin embargo, estos roles llegan a Snowflake **vacíos**: sin ningún privilegio sobre datos.

### 10.3 Lo que SCIM NO Hace (Responsabilidad del Admin Snowflake)

SCIM **nunca** ejecuta estas operaciones:

| Acción | Por qué no la hace SCIM | Quién la hace |
|--------|-------------------------|---------------|
| Crear Database Roles (Access Roles) | El IdP no conoce la estructura de datos de Snowflake | Admin de Gobernanza Snowflake |
| `GRANT SELECT ON ALL TABLES IN SCHEMA ...` | El IdP no gestiona privilegios granulares | Admin de Gobernanza Snowflake |
| `GRANT DATABASE ROLE ... TO ROLE ...` | Conectar datos a grupos es decisión de negocio/gobernanza | Admin de Gobernanza Snowflake |
| `GRANT USAGE ON WAREHOUSE ...` | Los warehouses son recursos de cómputo, no de identidad | Admin de Gobernanza Snowflake |
| Configurar Future Grants | Requiere conocimiento de la estructura de schemas | Admin de Gobernanza Snowflake |
| Crear Managed Access Schemas | Decisión de arquitectura de seguridad | Admin de Gobernanza Snowflake |

### 10.4 Flujo Completo: Del IdP al Acceso a Datos (Paso a Paso)

#### Escenario: Nuevo analista de BI necesita acceder a datos de ventas

**Paso 1 — En el IdP (realizado por el equipo de Identidad):**

El administrador de Okta/Azure AD agrega al usuario `carlos.souza` al grupo `GRP_BI_ANALYST`.

Resultado en Snowflake (automático vía SCIM):
```sql
-- Estas operaciones ocurren AUTOMÁTICAMENTE (no se ejecuta nada):
-- CREATE USER carlos.souza ...;
-- CREATE ROLE GRP_BI_ANALYST;  (si no existía)
-- GRANT ROLE GRP_BI_ANALYST TO USER carlos.souza;
```

**Paso 2 — En Snowflake (realizado por el Admin de Gobernanza):**

El admin debe asegurar que el Functional Role `GRP_BI_ANALYST` tenga acceso a los datos correctos. Esto se hace **una sola vez** (no es necesario repetir para cada nuevo miembro del grupo):

```sql
-- 2a. Crear los Access Roles (Database Roles) si aún no existen
USE ROLE SYSADMIN;
CREATE DATABASE ROLE IF NOT EXISTS ventas.sc_presentation_ro;

-- 2b. Conceder privilegios a los Access Roles
GRANT USAGE ON SCHEMA ventas.presentation TO DATABASE ROLE ventas.sc_presentation_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA ventas.presentation TO DATABASE ROLE ventas.sc_presentation_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ventas.presentation TO DATABASE ROLE ventas.sc_presentation_ro;

-- 2c. Conectar el Access Role al Functional Role (aprovisionado por SCIM)
USE ROLE SECURITYADMIN;
GRANT DATABASE ROLE ventas.sc_presentation_ro TO ROLE grp_bi_analyst;

-- 2d. Conceder warehouse
GRANT USAGE ON WAREHOUSE wh_analytics TO ROLE grp_bi_analyst;

-- 2e. Conceder USAGE en el database
GRANT USAGE ON DATABASE ventas TO ROLE grp_bi_analyst;
```

**Paso 3 — Verificación:**

```sql
-- El admin puede verificar que el usuario ahora tiene acceso:
SHOW GRANTS TO USER carlos.souza;
-- Debe mostrar: GRP_BI_ANALYST

SHOW GRANTS TO ROLE grp_bi_analyst;
-- Debe mostrar: USAGE en database, warehouse, y el database role

SHOW GRANTS TO DATABASE ROLE ventas.sc_presentation_ro;
-- Debe mostrar: SELECT en tablas del schema presentation
```

#### Resumen Visual del Flujo

```
┌──────────────────────┐    ┌─────────────────────────────────┐    ┌────────────────────────────┐
│ IdP (Automático)     │    │ Admin Snowflake (Manual)        │    │ Resultado Final            │
│                      │    │                                 │    │                            │
│ 1. Crea usuario      │    │ 3. Crea Access Roles            │    │ Usuario inicia sesión SSO  │
│ 2. Agrega al grupo   │───►│ 4. GRANT Access Role → FR       │───►│ Usa el role del grupo      │
│                      │    │ 5. GRANT Warehouse → FR         │    │ Accede a las tablas        │
│                      │    │                                 │    │                            │
└──────────────────────┘    └─────────────────────────────────┘    └────────────────────────────┘
        QUIÉN                          QUÉ                              RESULTADO
```

### 10.5 Errores Comunes y Cómo Evitarlos

| Síntoma | Causa Raíz | Solución |
|---------|------------|----------|
| "Creé el grupo en Okta pero el usuario no ve ninguna tabla" | El Functional Role (grupo SCIM) no tiene Access Roles asignados | `GRANT DATABASE ROLE ... TO ROLE <grupo_scim>` |
| "El usuario puede iniciar sesión pero no aparece ningún warehouse" | Falta `GRANT USAGE ON WAREHOUSE` para el role SCIM | `GRANT USAGE ON WAREHOUSE wh TO ROLE <grupo_scim>` |
| "El usuario no puede hacer USE DATABASE" | Falta `GRANT USAGE ON DATABASE` | `GRANT USAGE ON DATABASE db TO ROLE <grupo_scim>` |
| "Usuarios nuevos en el grupo ven tablas, pero no ven tablas creadas recientemente" | Faltan Future Grants en el schema | `GRANT SELECT ON FUTURE TABLES IN SCHEMA ... TO DATABASE ROLE ...` |
| "Removí al usuario del grupo en Okta pero aún accede a datos" | SCIM puede tener retraso de sincronización (minutos) | Verificar estado del SCIM provisioning en el IdP; si es urgente, deshabilitar manualmente: `ALTER USER ... SET DISABLED = TRUE` |
| "El role aprovisionado por SCIM no aparece en la jerarquía de SYSADMIN" | Los roles SCIM no se conceden automáticamente a SYSADMIN | `GRANT ROLE <grupo_scim> TO ROLE SYSADMIN` (buena práctica) |
| "El admin de gobernanza no puede dar grants en el role SCIM" | El admin necesita usar SECURITYADMIN (que tiene MANAGE GRANTS) | `USE ROLE SECURITYADMIN;` antes de ejecutar los GRANTs |

### 10.6 Buenas Prácticas para Ambientes con SCIM

1. **Convención de nomenclatura en el IdP:** Use prefijos claros para grupos que serán aprovisionados (ej: `GRP_`, `SF_`, `SNOW_`). Esto facilita identificar en Snowflake qué roles vinieron del SCIM.

2. **Conecte los roles SCIM a la jerarquía:** Siempre ejecute `GRANT ROLE <grupo_scim> TO ROLE SYSADMIN` para evitar roles huérfanos.

3. **Documente la matriz de responsabilidades:**

| Acción | Responsable | Herramienta |
|--------|-------------|-------------|
| Crear/desactivar usuario | Equipo de Identidad | Okta/Azure AD |
| Crear/modificar grupo | Equipo de Identidad | Okta/Azure AD |
| Agregar usuario al grupo | Gestor del equipo | Okta/Azure AD |
| Crear Access Roles | Admin Gobernanza Snow | SQL en Snowflake |
| Conectar Access Role a Functional Role | Admin Gobernanza Snow | SQL en Snowflake |
| Conceder warehouses | Admin Gobernanza Snow | SQL en Snowflake |
| Auditar accesos | Admin Gobernanza Snow | SQL en Snowflake |

4. **Configure SCIM una vez, gestione grants continuamente:** El trabajo inicial de configurar la integración SCIM es puntual. Sin embargo, a medida que se crean nuevos schemas, databases o productos de datos, el admin Snowflake necesita crear continuamente Access Roles y conectarlos a los Functional Roles existentes.

5. **Use el `DEFAULT_ROLE` correcto:** Configure en el IdP o en Snowflake para que el `DEFAULT_ROLE` del usuario sea el Functional Role principal:

```sql
-- Verificar default roles de usuarios aprovisionados por SCIM
SELECT user_name, default_role, has_password, ext_authn_duo
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND created_on >= DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY created_on DESC;
```

### 10.7 SQL de Referencia para Ambientes con SCIM

```sql
-- Ver todos los roles creados por SCIM (patrón de nombre con prefijo)
SHOW ROLES LIKE 'GRP_%';

-- Ver usuarios aprovisionados recientemente (sin password = probablemente SSO/SCIM)
SELECT user_name, login_name, display_name, default_role, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE has_password = 'false'
  AND deleted_on IS NULL
ORDER BY created_on DESC;

-- Ver qué roles SCIM ya tienen Access Roles asignados
SHOW GRANTS TO ROLE grp_bi_analyst;

-- Ver qué roles SCIM NO tienen ningún Database Role (posible problema)
-- Compare los roles con prefijo SCIM contra los que tienen grants de database role
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

-- Asignar Access Roles en lote a un Functional Role SCIM
USE ROLE SECURITYADMIN;
GRANT DATABASE ROLE ventas.sc_raw_ro TO ROLE grp_data_engineer;
GRANT DATABASE ROLE ventas.sc_staging_rw TO ROLE grp_data_engineer;
GRANT DATABASE ROLE ventas.sc_presentation_ro TO ROLE grp_data_engineer;
GRANT USAGE ON WAREHOUSE wh_ingesta TO ROLE grp_data_engineer;
GRANT USAGE ON DATABASE ventas TO ROLE grp_data_engineer;

-- Auditoría: quién accedió a qué en las últimas 24h vía roles SCIM
SELECT user_name, role_name, query_type, LEFT(query_text, 100) AS query_preview, start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE role_name LIKE 'GRP_%'
  AND start_time >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
ORDER BY start_time DESC
LIMIT 100;
```

---

## 11. Ejemplos Prácticos Completos

### 11.1 Escenario: Configurar Acceso para una Empresa

Estructura de la empresa ficticia:
- Database: `ANALYTICS`
- Schemas: `RAW`, `STAGING`, `PRESENTATION`
- Equipos: Ingeniería de Datos, BI, Ciencia de Datos

#### Paso 1: Crear Database Roles (Access Roles)

```sql
USE ROLE SYSADMIN;

-- Database roles para schema RAW
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_raw_ro;
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_raw_rw;

-- Database roles para schema STAGING
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_staging_ro;
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_staging_rw;

-- Database roles para schema PRESENTATION
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_presentation_ro;
CREATE DATABASE ROLE IF NOT EXISTS analytics.sc_presentation_rw;
```

#### Paso 2: Conceder Privilegios a los Access Roles

```sql
-- === Schema RAW - Lectura ===
GRANT USAGE ON SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON ALL VIEWS IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;

-- === Schema RAW - Escritura ===
GRANT USAGE ON SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;
GRANT CREATE TABLE ON SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;

-- === Schema STAGING - Lectura ===
GRANT USAGE ON SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_ro;

-- === Schema STAGING - Escritura ===
GRANT USAGE ON SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;
GRANT CREATE TABLE, CREATE VIEW ON SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;

-- === Schema PRESENTATION - Lectura ===
GRANT USAGE ON SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON ALL VIEWS IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;

-- === Schema PRESENTATION - Escritura ===
GRANT USAGE ON SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
GRANT CREATE TABLE, CREATE VIEW ON SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
```

#### Paso 3: Crear Functional Roles y Asignar Access Roles

```sql
USE ROLE USERADMIN;

-- Functional Roles por equipo
CREATE ROLE IF NOT EXISTS fr_ingenieria_datos;
CREATE ROLE IF NOT EXISTS fr_bi_analyst;
CREATE ROLE IF NOT EXISTS fr_data_scientist;

-- Garantizar que SYSADMIN hereda todos los custom roles
GRANT ROLE fr_ingenieria_datos TO ROLE SYSADMIN;
GRANT ROLE fr_bi_analyst TO ROLE SYSADMIN;
GRANT ROLE fr_data_scientist TO ROLE SYSADMIN;

USE ROLE SECURITYADMIN;

-- Ingeniería de Datos: escritura en RAW y STAGING, lectura en PRESENTATION
GRANT DATABASE ROLE analytics.sc_raw_rw TO ROLE fr_ingenieria_datos;
GRANT DATABASE ROLE analytics.sc_staging_rw TO ROLE fr_ingenieria_datos;
GRANT DATABASE ROLE analytics.sc_presentation_ro TO ROLE fr_ingenieria_datos;

-- BI Analyst: lectura en STAGING y PRESENTATION
GRANT DATABASE ROLE analytics.sc_staging_ro TO ROLE fr_bi_analyst;
GRANT DATABASE ROLE analytics.sc_presentation_ro TO ROLE fr_bi_analyst;

-- Data Scientist: lectura en todo, escritura en STAGING
GRANT DATABASE ROLE analytics.sc_raw_ro TO ROLE fr_data_scientist;
GRANT DATABASE ROLE analytics.sc_staging_rw TO ROLE fr_data_scientist;
GRANT DATABASE ROLE analytics.sc_presentation_ro TO ROLE fr_data_scientist;

-- USAGE en el database para todos los functional roles
GRANT USAGE ON DATABASE analytics TO ROLE fr_ingenieria_datos;
GRANT USAGE ON DATABASE analytics TO ROLE fr_bi_analyst;
GRANT USAGE ON DATABASE analytics TO ROLE fr_data_scientist;
```

#### Paso 4: Crear Warehouses y Conceder Acceso

```sql
USE ROLE SYSADMIN;

-- Warehouses por perfil de uso
CREATE WAREHOUSE IF NOT EXISTS wh_ingesta
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

-- Conceder USAGE en los warehouses
GRANT USAGE ON WAREHOUSE wh_ingesta TO ROLE fr_ingenieria_datos;
GRANT USAGE ON WAREHOUSE wh_analytics TO ROLE fr_bi_analyst;
GRANT USAGE ON WAREHOUSE wh_analytics TO ROLE fr_data_scientist;
GRANT USAGE ON WAREHOUSE wh_datascience TO ROLE fr_data_scientist;
```

#### Paso 5: Crear Usuarios y Asignar Roles

```sql
USE ROLE USERADMIN;

-- Crear usuario humano
CREATE USER IF NOT EXISTS maria_silva
  PASSWORD = 'CambiarEnPrimerLogin!'
  DEFAULT_ROLE = 'FR_BI_ANALYST'
  DEFAULT_WAREHOUSE = 'WH_ANALYTICS'
  MUST_CHANGE_PASSWORD = TRUE;

-- Asignar functional role al usuario
GRANT ROLE fr_bi_analyst TO USER maria_silva;

-- Crear service account (sin password — usa key pair)
CREATE USER IF NOT EXISTS svc_pipeline_ingesta
  DEFAULT_ROLE = 'FR_INGENIERIA_DATOS'
  DEFAULT_WAREHOUSE = 'WH_INGESTA'
  TYPE = SERVICE;

GRANT ROLE fr_ingenieria_datos TO USER svc_pipeline_ingesta;
```

### 11.2 Verificación de Acceso

```sql
-- Ver todos los roles de un usuario
SHOW GRANTS TO USER maria_silva;

-- Ver todos los privilegios de un role
SHOW GRANTS TO ROLE fr_bi_analyst;

-- Ver quién tiene acceso a una tabla específica
SHOW GRANTS ON TABLE analytics.presentation.ventas_mensuales;

-- Ver la jerarquía completa de roles
SHOW GRANTS OF ROLE fr_data_scientist;
```

---

## 12. Checklist de Configuración de Seguridad

Use esta lista al configurar una cuenta nueva o auditar una cuenta existente:

### Capa 1: Network Access

- [ ] Network Policy creada para restringir IPs de acceso
- [ ] Considerar Private Link si la empresa requiere tráfico privado

```sql
-- Ejemplo: crear network policy
CREATE NETWORK POLICY IF NOT EXISTS corp_network_policy
  ALLOWED_IP_LIST = ('10.0.0.0/8', '172.16.0.0/12', '200.100.50.0/24')
  COMMENT = 'Permite solo red corporativa';

-- Aplicar a la cuenta
ALTER ACCOUNT SET NETWORK_POLICY = 'CORP_NETWORK_POLICY';
```

### Capa 2: Authentication

- [ ] MFA habilitado para **todos** los usuarios con ACCOUNTADMIN
- [ ] MFA habilitado para SECURITYADMIN
- [ ] SSO configurado (SAML 2.0) como método primario
- [ ] Key Pair configurado para service accounts (sin password)
- [ ] Password policy definida

```sql
-- Verificar MFA de los admins
SELECT user_name, has_mfa_enrolled, default_role
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND default_role IN ('ACCOUNTADMIN', 'SECURITYADMIN');
```

### Capa 3: Authorization (RBAC)

- [ ] ACCOUNTADMIN asignado a 2-4 usuarios solamente
- [ ] Ningún usuario tiene ACCOUNTADMIN como DEFAULT_ROLE
- [ ] Ningún objeto creado con ACCOUNTADMIN (deben crearse con SYSADMIN)
- [ ] Jerarquía de roles definida: System Roles > Functional Roles > Access Roles
- [ ] Database Roles usados para acceso a objetos de database
- [ ] Future Grants configurados en todos los schemas
- [ ] Managed Access Schema usado para datos sensibles
- [ ] Todos los custom roles heredan a SYSADMIN (no quedan "huérfanos")
- [ ] Role PUBLIC no tiene grants innecesarios

```sql
-- Verificar roles huérfanos (no conectados a la jerarquía)
-- Roles que no fueron concedidos a ningún otro role
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

### Capa 4: Continuous Data Protection

- [ ] Time Travel configurado (por defecto: 1 día; producción: hasta 90 días)
- [ ] Data Masking Policies aplicadas a columnas PII
- [ ] Row Access Policies para datos multi-tenant

### Auditoría y Monitoreo

- [ ] Resource Monitors configurados para controlar costos
- [ ] ACCOUNT_USAGE monitoreado periódicamente
- [ ] Alertas para grants de ACCOUNTADMIN
- [ ] Alertas para logins desde IPs desconocidos

---

## 13. Apéndice: Comandos SQL de Referencia Rápida

### Gestión de Roles

| Acción | SQL |
|--------|-----|
| Crear role | `CREATE ROLE fr_nombre;` |
| Crear database role | `CREATE DATABASE ROLE db.nombre_role;` |
| Conceder role a usuario | `GRANT ROLE fr_nombre TO USER juan;` |
| Conceder role a otro role | `GRANT ROLE fr_junior TO ROLE fr_senior;` |
| Conceder database role a role | `GRANT DATABASE ROLE db.role TO ROLE fr_nombre;` |
| Remover role de usuario | `REVOKE ROLE fr_nombre FROM USER juan;` |
| Listar roles | `SHOW ROLES;` |
| Listar database roles | `SHOW DATABASE ROLES IN DATABASE db;` |

### Gestión de Grants

| Acción | SQL |
|--------|-----|
| Grant de lectura en schema | `GRANT USAGE ON SCHEMA db.sch TO ROLE r; GRANT SELECT ON ALL TABLES IN SCHEMA db.sch TO ROLE r;` |
| Future grant de tablas | `GRANT SELECT ON FUTURE TABLES IN SCHEMA db.sch TO ROLE r;` |
| Grant de warehouse | `GRANT USAGE ON WAREHOUSE wh TO ROLE r;` |
| Revocar grant | `REVOKE SELECT ON TABLE db.sch.t FROM ROLE r;` |
| Ver grants de un objeto | `SHOW GRANTS ON TABLE db.sch.t;` |
| Ver grants de un role | `SHOW GRANTS TO ROLE r;` |
| Ver grants de un usuario | `SHOW GRANTS TO USER u;` |

### Gestión de Usuarios

| Acción | SQL |
|--------|-----|
| Crear usuario | `CREATE USER nombre PASSWORD='x' DEFAULT_ROLE='r' MUST_CHANGE_PASSWORD=TRUE;` |
| Crear service account | `CREATE USER svc_nombre DEFAULT_ROLE='r' TYPE=SERVICE;` |
| Cambiar default role | `ALTER USER nombre SET DEFAULT_ROLE = 'nuevo_role';` |
| Deshabilitar usuario | `ALTER USER nombre SET DISABLED = TRUE;` |
| Eliminar usuario | `DROP USER nombre;` |

### Gestión de Ownership

| Acción | SQL |
|--------|-----|
| Transferir ownership | `GRANT OWNERSHIP ON TABLE db.sch.t TO ROLE r REVOKE CURRENT GRANTS;` |
| Ownership futuro | `GRANT OWNERSHIP ON FUTURE TABLES IN SCHEMA db.sch TO ROLE r;` |

### Cambio de Contexto

| Acción | SQL |
|--------|-----|
| Cambiar role | `USE ROLE fr_nombre;` |
| Activar secondary roles | `USE SECONDARY ROLES ALL;` |
| Definir warehouse | `USE WAREHOUSE wh_nombre;` |
| Definir database | `USE DATABASE db_nombre;` |
| Definir schema | `USE SCHEMA schema_nombre;` |

---

## Glosario

| Término | Definición |
|---------|-----------|
| **RBAC** | Role-Based Access Control — modelo donde privilegios se asignan a Roles |
| **DAC** | Discretionary Access Control — el dueño del objeto controla el acceso |
| **MAS** | Managed Access Schema — schema donde solo el dueño del schema gestiona grants |
| **Grant** | Comando que concede un privilegio a un role |
| **Revoke** | Comando que remueve un privilegio de un role |
| **Ownership** | Privilegio especial de control total sobre un objeto |
| **Future Grant** | Grant que se aplica automáticamente a objetos creados en el futuro |
| **Functional Role** | Role asignado a usuarios, agrupa Access Roles |
| **Access Role** | Role que contiene solo privilegios sobre objetos |
| **Database Role** | Role que existe dentro de un database específico |
| **Secondary Roles** | Mecanismo para activar múltiples roles simultáneamente |
| **Network Policy** | Regla que restringe qué IPs pueden acceder a la cuenta |
| **MFA** | Multi-Factor Authentication — capa extra de verificación de identidad |
| **Service Account** | Usuario no humano usado por aplicaciones y pipelines |

---

> **Documento generado como referencia para configuración de seguridad Snowflake.**
> Basado en el material "RBAC Access Management Overview" del Snowflake Activation Team.
