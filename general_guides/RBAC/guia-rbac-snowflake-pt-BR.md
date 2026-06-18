# Guia de Referência: Controle de Acesso e Segurança no Snowflake

> **Público-alvo:** Times com experiência em SQL (Oracle, SQL Server, PostgreSQL) que estão iniciando no Snowflake.

> **Objetivo:** Fornecer uma referência prática e completa para configurar segurança e controle de acesso em qualquer conta Snowflake.

---

## Sumário

1. [Segurança em Camadas](#1-segurança-em-camadas)
2. [Conceitos Fundamentais de Controle de Acesso](#2-conceitos-fundamentais-de-controle-de-acesso)
3. [Hierarquia de Objetos](#3-hierarquia-de-objetos)
4. [Roles do Sistema (System-Defined Roles)](#4-roles-do-sistema-system-defined-roles)
5. [Functional Roles vs Access Roles](#5-functional-roles-vs-access-roles)
6. [Padrões de Autorização (Design Patterns)](#6-padrões-de-autorização-design-patterns)
7. [Grants: Current e Future](#7-grants-current-e-future)
8. [Ownership (Propriedade)](#8-ownership-propriedade)
9. [Herança de Roles e Troca de Contexto](#9-herança-de-roles-e-troca-de-contexto)
10. [Ambientes com SSO e SCIM: Divisão de Responsabilidades](#10-ambientes-com-sso-e-scim-divisão-de-responsabilidades)
11. [Exemplos Práticos Completos](#11-exemplos-práticos-completos)
12. [Checklist de Configuração de Segurança](#12-checklist-de-configuração-de-segurança)
13. [Apêndice: Comandos SQL de Referência Rápida](#13-apêndice-comandos-sql-de-referência-rápida)

---

## 1. Segurança em Camadas

O Snowflake implementa segurança em **quatro camadas** complementares. Cada camada atua de forma independente, criando defesa em profundidade:

```
┌──────────────────┐    ┌──────────────────────────┐    ┌──────────────────────────┐    ┌─────────────────────────────────┐
│ 1. Network Access│───►│ 2. Authentication (AuthN)│───►│ 3. Authorization (AuthZ) │───►│ 4. Continuous Data Protection   │
│   (IP/Rede)      │    │   (Identidade)           │    │ (RBAC - Foco deste guia) │    │   (Criptografia/Time Travel)    │
└──────────────────┘    └──────────────────────────┘    └──────────────────────────┘    └─────────────────────────────────┘
```

| Camada | O que faz | Mecanismos |
|--------|-----------|------------|
| **1. Network Access** | Restringe quais IPs ou redes podem acessar a conta | Network Policies, AWS PrivateLink, Azure Private Link, GCP Private Service Connect |
| **2. Authentication (AuthN)** | Verifica a identidade do usuário ou processo | Username/Password, SSO (SAML 2.0), Key Pair, OAuth, MFA |
| **3. Authorization (AuthZ)** | Define o que cada identidade pode fazer | **RBAC** — Roles, Privileges, Grants |
| **4. Continuous Data Protection** | Protege os dados em repouso e em trânsito | Criptografia AES-256, Re-keying anual, Time Travel (até 90 dias), Fail-safe, Data Masking, Row Access Policies, Tokenização |

**Foco deste guia:** Camada 3 — Authorization via RBAC.

> **Nota para quem vem de outros bancos:** No Snowflake, você **não** pode conceder privilégios diretamente a usuários. Toda autorização passa por Roles. Usuários apenas "vestem" uma Role para acessar objetos.

---

## 2. Conceitos Fundamentais de Controle de Acesso

### 2.1 Securable Object (Objeto Protegido)

Qualquer entidade à qual se pode conceder acesso. Se não houver um GRANT explícito, o acesso é **negado por padrão**.

Exemplos: Database, Schema, Table, View, Stage, Warehouse, Task, Stream, Stored Procedure, Function.

### 2.2 Privilege (Privilégio)

Uma permissão específica sobre um objeto. Exemplos:

| Privilégio | Aplica-se a | Permite |
|------------|-------------|---------|
| `USAGE` | Database, Schema, Warehouse | Acessar/usar o objeto |
| `SELECT` | Table, View | Ler dados |
| `INSERT` | Table | Inserir dados |
| `CREATE TABLE` | Schema | Criar tabelas no schema |
| `OPERATE` | Warehouse | Iniciar/parar/redimensionar |
| `OWNERSHIP` | Qualquer objeto | Controle total (ver seção 8) |

### 2.3 Role

Uma entidade à qual privilégios são concedidos. Existem dois tipos:

- **Account Role** — existe no nível da conta, pode ser atribuída a usuários
- **Database Role** — existe dentro de um database, ideal para controlar acesso a objetos daquele database

### 2.4 User (Usuário)

Representa uma identidade que se conecta ao Snowflake. Pode ser:

- **Humano** — pessoa real que faz login
- **Service Account** — conta de serviço usada por aplicações, pipelines de dados, etc.

### 2.5 Métodos de Controle de Acesso

| Método | Descrição | Quando usar |
|--------|-----------|-------------|
| **RBAC** (Role-Based Access Control) | Privilégios são atribuídos a Roles, Roles a Users | **Padrão recomendado** para toda configuração |
| **DAC** (Discretionary Access Control) | O dono do objeto pode conceder acesso a outros | Funciona, mas difícil de manter em escala |
| **Managed Access Schema** | Remove do dono do objeto a capacidade de dar grants; apenas o dono do schema pode conceder | Ambientes regulados onde controle centralizado é necessário |

```sql
-- Criar um schema com Managed Access
CREATE SCHEMA vendas.staging WITH MANAGED ACCESS;
```

---

## 3. Hierarquia de Objetos

Os objetos no Snowflake seguem uma hierarquia de contenção. Um objeto **não pode existir** fora do seu contêiner:

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

### Objetos por Nível

| Nível | Objetos | Observação |
|-------|---------|------------|
| **Organization** | Accounts | Gerencia billing, replicação, data sharing entre contas |
| **Account** | Users, Roles, Warehouses, Tasks, Resource Monitors, Integrations, Failover Groups | Objetos "standalone" que não pertencem a um database |
| **Database** | Schemas, Database Roles | Pode ser compartilhado e replicado |
| **Schema** | Tables, Views, Stages, Pipes, Streams, UDFs, Stored Procedures, Sequences, File Formats | Boundary de segurança recomendado |

> **Dica prática:** Use Schemas como **fronteira de segurança**. É muito mais fácil gerenciar acesso por schema do que por objeto individual.

---

## 4. Roles do Sistema (System-Defined Roles)

O Snowflake cria automaticamente estas roles com privilégios específicos. Elas formam uma hierarquia fixa:

```
         ORGADMIN
            │
       ACCOUNTADMIN
        ┌────┴────┐
  SECURITYADMIN   SYSADMIN
        │              │
   USERADMIN    Custom Roles (suas roles)
                       │
                    PUBLIC
```

### Detalhamento de Cada Role

| Role | Responsabilidade | Restrições |
|------|-----------------|------------|
| **ORGADMIN** | Gerenciar contas na organização, billing, replicação | Concedida a apenas 1 usuário por organização |
| **ACCOUNTADMIN** | Configuração da conta, resource monitors, shares, sessões | **Nunca use no dia-a-dia** (ver boas práticas abaixo) |
| **SECURITYADMIN** | Tem `MANAGE GRANTS` — pode alterar qualquer grant/role na conta. Herda USERADMIN | Use quando precisar gerenciar grants globalmente |
| **USERADMIN** | Criação e remoção de Users e Roles no dia-a-dia | Só gerencia users/roles que ele criou |
| **SYSADMIN** | Criar databases, schemas, tabelas, warehouses. Usar roles abaixo dela | Role operacional principal para objetos |
| **PUBLIC** | Concedida automaticamente a **todos** os users e roles | Qualquer grant ao PUBLIC é acessível por todos |

### Boas Práticas para ACCOUNTADMIN

| Regra | Justificativa |
|-------|---------------|
| Atribuir a **no mínimo 2** e **no máximo 4** usuários | Evitar ponto único de falha, mas limitar superfície de ataque |
| **MFA obrigatório** para todos com ACCOUNTADMIN | Proteger contra comprometimento de credenciais |
| **Nunca** definir como `DEFAULT_ROLE` de nenhum usuário | Evitar uso acidental no dia-a-dia |
| **Nunca** criar objetos com ACCOUNTADMIN | Objetos ficam "presos" nessa role; use SYSADMIN |
| Manter username/password como backup ao SSO | Se SSO falhar e MFA bloquear, o reset pode levar até 2 dias úteis |

```sql
-- Verificar quem tem ACCOUNTADMIN
SHOW GRANTS OF ROLE ACCOUNTADMIN;

-- Verificar se algum usuário tem ACCOUNTADMIN como default
SELECT user_name, default_role
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE default_role = 'ACCOUNTADMIN'
  AND deleted_on IS NULL;
```

---

## 5. Functional Roles vs Access Roles

Este é o **padrão arquitetural mais importante** para uma implementação escalável de RBAC no Snowflake.

### 5.1 Definições

| Conceito | O que é | Regras |
|----------|---------|--------|
| **Access Role** | Role que contém apenas privilégios sobre objetos | **Nunca** é concedida diretamente a um usuário |
| **Functional Role** | Role que é concedida a usuários e contém Access Roles | É o que o usuário "veste" para trabalhar |

### 5.2 Regra de Ouro

```
Usuário ←(grant)← Functional Role ←(grant)← Access Role ←(grant)← Privileges sobre objetos
```

```
                                    ┌─── Access Role: Leitura ───► Schema X (SELECT)
                                    │
Usuário ───► Functional Role ───────┤
                                    │
                                    └─── Access Role: Escrita ────► Schema Y (INSERT/UPDATE)
```

### 5.3 Tipos de Access Roles

| Tipo | Escopo | Quando usar |
|------|--------|-------------|
| **Database Role** | Objetos dentro de um database | **Preferido** — acesso a tabelas, views, schemas |
| **Account Role** | Objetos de conta (warehouses, integrations) | Quando o objeto não pertence a um database |

### 5.4 Tipos de Functional Roles

| Tipo | Descrição | Quantas por usuário |
|------|-----------|---------------------|
| **Dept-Job** (Departamento-Função) | Agrupa pessoas pela responsabilidade funcional (ex: Marketing-Analista) | Tipicamente 1 |
| **Data Product** | Dá acesso a um conjunto de dados preparado para consumo | 1 a muitas |
| **User Aggregate** | Role individual que agrega todas as roles de um usuário | Exatamente 1 |

### 5.5 Convenções de Nomenclatura Recomendadas

| Tipo de Role | Prefixo | Exemplo |
|--------------|---------|---------|
| Functional Role | `FR_` | `FR_MARKETING_ANALYST` |
| Access Role (Account) | `AR_` | `AR_WH_ANALYTICS_USAGE` |
| Database Role (leitura de schema) | `SC_<schema>_RO` | `SC_VENDAS_RO` |
| Database Role (escrita em schema) | `SC_<schema>_RW` | `SC_VENDAS_RW` |
| Database Role (nível de database) | `DB_<database>_RO` | `DB_ANALYTICS_RO` |

---

## 6. Padrões de Autorização (Design Patterns)

### 6.1 Padrão NÃO Recomendado: Objeto-a-Objeto (DAC)

```sql
-- EVITE ISSO em produção com muitos objetos
GRANT SELECT ON TABLE vendas.public.pedidos TO ROLE analista;
GRANT SELECT ON TABLE vendas.public.clientes TO ROLE analista;
GRANT SELECT ON TABLE vendas.public.produtos TO ROLE analista;
-- ... repita para centenas de tabelas e dezenas de roles
```

**Problemas:**
- Cada objeto tem um dono individual — difícil rastrear
- Permutações de objetos x roles podem passar de 10.000
- Impossível auditar se os privilégios estão corretos
- Novos objetos não recebem grants automaticamente

### 6.2 Padrão Recomendado: Privilégios por Schema

```sql
-- UMA role de leitura por schema + Future Grants = gestão simples
GRANT USAGE ON DATABASE vendas TO ROLE sc_staging_ro;
GRANT USAGE ON SCHEMA vendas.staging TO ROLE sc_staging_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA vendas.staging TO ROLE sc_staging_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA vendas.staging TO ROLE sc_staging_ro;
```

**Vantagens:**
- Um grant por schema por role por tipo de acesso (RO/RW)
- Fácil de auditar
- Future Grants garantem acesso automático a novos objetos
- Ideal: uma role `_RO` e uma `_RW` por schema

### 6.3 Managed Access Schema

Quando você precisa que **apenas o dono do schema** (e não o dono de cada tabela) possa conceder acesso:

```sql
CREATE SCHEMA vendas.financeiro WITH MANAGED ACCESS;

-- Agora, mesmo que role X crie uma tabela neste schema,
-- apenas o dono do schema (ou SECURITYADMIN) pode dar grants sobre ela.
```

**Use quando:** Ambientes regulados, dados sensíveis (PII, financeiro, saúde).

---

## 7. Grants: Current e Future

### 7.1 Current Grants

Aplicam-se **imediatamente** a objetos que **já existem**:

```sql
-- Conceder SELECT em TODAS as tabelas existentes no schema
GRANT SELECT ON ALL TABLES IN SCHEMA vendas.staging TO ROLE sc_staging_ro;

-- Conceder USAGE em TODOS os warehouses existentes
GRANT USAGE ON ALL WAREHOUSES IN ACCOUNT TO ROLE fr_analista;
```

### 7.2 Future Grants

Garantem que objetos **criados no futuro** recebam automaticamente os mesmos privilégios:

```sql
-- Tabelas criadas no futuro neste schema herdarão SELECT
GRANT SELECT ON FUTURE TABLES IN SCHEMA vendas.staging TO ROLE sc_staging_ro;

-- Views criadas no futuro
GRANT SELECT ON FUTURE VIEWS IN SCHEMA vendas.staging TO ROLE sc_staging_ro;

-- Future Grants no nível do database (aplica-se a todos os schemas)
GRANT USAGE ON FUTURE SCHEMAS IN DATABASE vendas TO ROLE db_vendas_ro;
```

### 7.3 Combinação Essencial

Sempre execute **ambos** ao configurar uma nova role:

```sql
-- 1. Current: objetos existentes
GRANT SELECT ON ALL TABLES IN SCHEMA vendas.staging TO ROLE sc_staging_ro;

-- 2. Future: objetos que serão criados
GRANT SELECT ON FUTURE TABLES IN SCHEMA vendas.staging TO ROLE sc_staging_ro;
```

> **Armadilha comum:** Configurar apenas Future Grants e esquecer que as tabelas existentes não receberam acesso.

---

## 8. Ownership (Propriedade)

### 8.1 O que é OWNERSHIP

`OWNERSHIP` é um privilégio especial que é automaticamente atribuído à role que **criou** o objeto (a menos que um Future Grant de OWNERSHIP exista).

### 8.2 O que OWNERSHIP concede

| Pode | Não pode |
|------|----------|
| Controle total sobre o objeto | — |
| Conceder/revogar acesso ao objeto | — |
| Renomear o objeto | — |
| Dropar o objeto | — |

### 8.3 O que OWNERSHIP de uma Role NÃO concede

> **Atenção:** Ter OWNERSHIP de uma Role **não** dá acesso aos objetos concedidos a essa role. Ownership de role ≠ privilégio de usar o que a role acessa.

### 8.4 Transferência de Ownership

```sql
-- Transferir ownership de uma tabela para outra role
GRANT OWNERSHIP ON TABLE vendas.staging.pedidos
  TO ROLE sysadmin
  REVOKE CURRENT GRANTS;

-- Transferir ownership de todas as tabelas em um schema
GRANT OWNERSHIP ON ALL TABLES IN SCHEMA vendas.staging
  TO ROLE sysadmin
  REVOKE CURRENT GRANTS;

-- Future Ownership: novas tabelas já nascem com o dono correto
GRANT OWNERSHIP ON FUTURE TABLES IN SCHEMA vendas.staging
  TO ROLE sysadmin;
```

### 8.5 Hierarquia de Ownership Recomendada

| Objeto | Owner recomendado |
|--------|-------------------|
| Databases | SYSADMIN |
| Schemas | SYSADMIN |
| Tabelas, Views | SYSADMIN (via Future Ownership) |
| Warehouses | SYSADMIN |
| Users e Roles | USERADMIN |

> **Regra:** Usuários **nunca** devem ser donos de objetos. RBAC é centrado em Roles.

---

## 9. Herança de Roles e Troca de Contexto

### 9.1 Como Funciona a Herança

Quando uma role é concedida a outra, a role superior **herda todos os privilégios** da role inferior:

```
  FR_DATA_SCIENTIST     ← tem acesso a TUDO abaixo
         │
    FR_ANALYST           ← tem acesso a BI_REPORTS + PUBLIC
         │
   FR_BI_REPORTS         ← tem acesso apenas ao que foi concedido + PUBLIC
         │
      PUBLIC             ← base (todos herdam)
```

Neste exemplo:
- `FR_DATA_SCIENTIST` tem acesso a **tudo** que `FR_ANALYST`, `FR_BI_REPORTS` e `PUBLIC` podem acessar
- `FR_ANALYST` tem acesso a tudo de `FR_BI_REPORTS` e `PUBLIC`
- `FR_BI_REPORTS` tem acesso apenas ao que foi concedido a ela + `PUBLIC`

### 9.2 Troca de Contexto (USE ROLE)

O usuário escolhe qual role usar a qualquer momento:

```sql
-- Definir contexto completo
USE ROLE fr_analyst;
USE WAREHOUSE wh_analytics_small;
USE DATABASE vendas;
USE SCHEMA staging;

-- Agora consultas usam os privilégios de FR_ANALYST
SELECT * FROM pedidos LIMIT 10;
```

Quando o usuário troca de role, os privilégios mudam **imediatamente**:

```sql
-- Com FR_DATA_SCIENTIST: acesso amplo
USE ROLE fr_data_scientist;
SELECT * FROM vendas.working.modelo_churn; -- OK

-- Troca para FR_BI_REPORTS: acesso restrito
USE ROLE fr_bi_reports;
SELECT * FROM vendas.working.modelo_churn; -- ERRO: Insufficient privileges
```

### 9.3 Secondary Roles

Por padrão, apenas a role ativa (Primary Role) define os privilégios. Mas você pode ativar **Secondary Roles** para combinar acessos:

```sql
-- Ativar todas as roles do usuário simultaneamente
USE SECONDARY ROLES ALL;

-- Agora o usuário tem acesso combinado de TODAS as suas roles
-- Útil para Data Product Roles (o usuário pode ter várias)
```

> **Quando usar Secondary Roles:**
> - Data Product Roles — o usuário precisa cruzar dados de múltiplos produtos
> - Cenários de masking policy que dependem de roles específicas

---

## 10. Ambientes com SSO e SCIM: Divisão de Responsabilidades

Quando o cliente utiliza **SSO** (Single Sign-On) para autenticação e **SCIM** (System for Cross-domain Identity Management) para provisionamento automático de usuários e grupos, a gestão de acesso se divide entre **duas equipes distintas**:

- **Time de Identidade (IdP)** — gerencia QUEM existe e a quais grupos pertence
- **Administrador de Governança Snowflake** — gerencia O QUE cada grupo pode acessar

> **Este é o ponto que mais gera confusão:** Criar o usuário e o grupo no IdP **não** é suficiente para dar acesso a dados. O provisionamento SCIM cria a "casca" (users e functional roles) no Snowflake, mas a conexão com os dados (access roles, grants) precisa ser feita manualmente pelo admin Snowflake.

### 10.1 Visão Geral: Fluxo IdP → Snowflake

```
┌─────────────────────────────────────┐         ┌──────────────────────────────────────────┐
│  IDENTITY PROVIDER (Okta/Azure AD)  │         │           SNOWFLAKE                      │
│                                     │  SCIM   │                                          │
│  • Usuários (criar/desativar)       │────────►│  • Users (criados automaticamente)       │
│  • Grupos (ex: "GRP_BI_ANALYST")    │────────►│  • Roles (criadas automaticamente)       │
│  • Membership (quem está no grupo)  │────────►│  • Grants de Role → User (automático)    │
│                                     │         │                                          │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │         │  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─     │
│  O IdP NÃO faz:                     │         │  O Admin Snowflake FAZ manualmente:      │
│  • Criar Access Roles               │         │  • Criar Database Roles (Access Roles)   │
│  • Dar grants em tabelas/schemas    │         │  • GRANT Access Role → Functional Role   │
│  • Configurar warehouses            │         │  • GRANT USAGE em warehouses             │
│  • Definir Future Grants            │         │  • Configurar Future Grants              │
└─────────────────────────────────────┘         └──────────────────────────────────────────┘
```

### 10.2 O que o SCIM Provisiona Automaticamente

Quando SCIM está ativo, as seguintes operações são **automáticas** (gerenciadas pelo IdP):

| O que é provisionado | Como aparece no Snowflake | Exemplo |
|----------------------|---------------------------|---------|
| Usuário novo no IdP | `CREATE USER` automático | Usuário `maria.silva` aparece no Snowflake |
| Grupo no IdP | Role de conta (Account Role) criada | Grupo `GRP_BI_ANALYST` → Role `GRP_BI_ANALYST` |
| Usuário adicionado ao grupo | `GRANT ROLE ... TO USER` automático | Maria recebe a role `GRP_BI_ANALYST` |
| Usuário removido do grupo | `REVOKE ROLE ... FROM USER` automático | Maria perde a role |
| Usuário desativado no IdP | `ALTER USER ... SET DISABLED = TRUE` | Acesso bloqueado imediatamente |

**Resultado:** As roles criadas pelo SCIM funcionam como **Functional Roles** — são o que o usuário "veste" para trabalhar. Porém, essas roles chegam ao Snowflake **vazias**: sem nenhum privilégio sobre dados.

### 10.3 O que o SCIM NÃO Faz (Responsabilidade do Admin Snowflake)

O SCIM **nunca** executa estas operações:

| Ação | Por que não é feita pelo SCIM | Quem faz |
|------|-------------------------------|----------|
| Criar Database Roles (Access Roles) | O IdP não conhece a estrutura de dados do Snowflake | Admin de Governança Snowflake |
| `GRANT SELECT ON ALL TABLES IN SCHEMA ...` | O IdP não gerencia privilégios granulares | Admin de Governança Snowflake |
| `GRANT DATABASE ROLE ... TO ROLE ...` | Conectar dados a grupos é decisão de negócio/governança | Admin de Governança Snowflake |
| `GRANT USAGE ON WAREHOUSE ...` | Warehouses são recursos de compute, não de identidade | Admin de Governança Snowflake |
| Configurar Future Grants | Requer conhecimento da estrutura de schemas | Admin de Governança Snowflake |
| Criar Managed Access Schemas | Decisão de arquitetura de segurança | Admin de Governança Snowflake |

### 10.4 Fluxo Completo: Do IdP ao Acesso a Dados (Passo a Passo)

#### Cenário: Novo analista de BI precisa acessar dados de vendas

**Passo 1 — No IdP (feito pelo time de Identidade):**

O administrador do Okta/Azure AD adiciona o usuário `carlos.souza` ao grupo `GRP_BI_ANALYST`.

Resultado no Snowflake (automático via SCIM):
```sql
-- Estas operações acontecem AUTOMATICAMENTE (você não executa nada):
-- CREATE USER carlos.souza ...;
-- CREATE ROLE GRP_BI_ANALYST;  (se ainda não existia)
-- GRANT ROLE GRP_BI_ANALYST TO USER carlos.souza;
```

**Passo 2 — No Snowflake (feito pelo Admin de Governança):**

O admin precisa garantir que a Functional Role `GRP_BI_ANALYST` tenha acesso aos dados corretos. Isso é feito **uma única vez** (não precisa repetir para cada novo membro do grupo):

```sql
-- 2a. Criar as Access Roles (Database Roles) se ainda não existem
USE ROLE SYSADMIN;
CREATE DATABASE ROLE IF NOT EXISTS vendas.sc_presentation_ro;

-- 2b. Conceder privilégios às Access Roles
GRANT USAGE ON SCHEMA vendas.presentation TO DATABASE ROLE vendas.sc_presentation_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA vendas.presentation TO DATABASE ROLE vendas.sc_presentation_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA vendas.presentation TO DATABASE ROLE vendas.sc_presentation_ro;

-- 2c. Conectar a Access Role à Functional Role (provisionada pelo SCIM)
USE ROLE SECURITYADMIN;
GRANT DATABASE ROLE vendas.sc_presentation_ro TO ROLE grp_bi_analyst;

-- 2d. Conceder warehouse
GRANT USAGE ON WAREHOUSE wh_analytics TO ROLE grp_bi_analyst;

-- 2e. Conceder USAGE no database
GRANT USAGE ON DATABASE vendas TO ROLE grp_bi_analyst;
```

**Passo 3 — Verificação:**

```sql
-- O admin pode verificar que o usuário agora tem acesso:
SHOW GRANTS TO USER carlos.souza;
-- Deve mostrar: GRP_BI_ANALYST

SHOW GRANTS TO ROLE grp_bi_analyst;
-- Deve mostrar: USAGE em database, warehouse, e a database role

SHOW GRANTS TO DATABASE ROLE vendas.sc_presentation_ro;
-- Deve mostrar: SELECT em tabelas do schema presentation
```

#### Resumo Visual do Fluxo

```
┌──────────────────────┐    ┌─────────────────────────────────┐    ┌────────────────────────────┐
│ IdP (Automático)     │    │ Admin Snowflake (Manual)        │    │ Resultado Final            │
│                      │    │                                 │    │                            │
│ 1. Cria usuário      │    │ 3. Cria Access Roles            │    │ Usuário faz login via SSO  │
│ 2. Adiciona ao grupo │───►│ 4. GRANT Access Role → FR       │───►│ Usa a role do grupo        │
│                      │    │ 5. GRANT Warehouse → FR         │    │ Acessa as tabelas          │
│                      │    │                                 │    │                            │
└──────────────────────┘    └─────────────────────────────────┘    └────────────────────────────┘
        QUEM                          O QUE                              RESULTADO
```

### 10.5 Erros Comuns e Como Evitar

| Sintoma | Causa Raiz | Solução |
|---------|------------|---------|
| "Criei o grupo no Okta mas o usuário não vê nenhuma tabela" | A Functional Role (grupo SCIM) não tem Access Roles atribuídas | `GRANT DATABASE ROLE ... TO ROLE <grupo_scim>` |
| "O usuário consegue logar mas não aparece nenhum warehouse" | Falta `GRANT USAGE ON WAREHOUSE` para a role SCIM | `GRANT USAGE ON WAREHOUSE wh TO ROLE <grupo_scim>` |
| "O usuário não consegue fazer USE DATABASE" | Falta `GRANT USAGE ON DATABASE` | `GRANT USAGE ON DATABASE db TO ROLE <grupo_scim>` |
| "Novos usuários no grupo veem tabelas, mas não veem tabelas criadas recentemente" | Faltam Future Grants no schema | `GRANT SELECT ON FUTURE TABLES IN SCHEMA ... TO DATABASE ROLE ...` |
| "Removi o usuário do grupo no Okta mas ele ainda acessa dados" | SCIM pode ter delay de sincronização (minutos) | Verifique o status do SCIM provisioning no IdP; se urgente, desabilite manualmente: `ALTER USER ... SET DISABLED = TRUE` |
| "A role provisionada pelo SCIM não aparece na hierarquia de SYSADMIN" | Roles SCIM não são automaticamente concedidas a SYSADMIN | `GRANT ROLE <grupo_scim> TO ROLE SYSADMIN` (boa prática) |
| "O admin de governança não consegue dar grants na role SCIM" | O admin precisa usar SECURITYADMIN (que tem MANAGE GRANTS) | `USE ROLE SECURITYADMIN;` antes de executar os GRANTs |

### 10.6 Boas Práticas para Ambientes com SCIM

1. **Convenção de nomenclatura no IdP:** Use prefixos claros para grupos que serão provisionados (ex: `GRP_`, `SF_`, `SNOW_`). Isso facilita identificar no Snowflake quais roles vieram do SCIM.

2. **Conecte as roles SCIM à hierarquia:** Sempre execute `GRANT ROLE <grupo_scim> TO ROLE SYSADMIN` para evitar roles órfãs.

3. **Documente a matriz de responsabilidades:**

| Ação | Responsável | Ferramenta |
|------|-------------|------------|
| Criar/desativar usuário | Time de Identidade | Okta/Azure AD |
| Criar/modificar grupo | Time de Identidade | Okta/Azure AD |
| Adicionar usuário ao grupo | Gestor do time | Okta/Azure AD |
| Criar Access Roles | Admin Governança Snow | SQL no Snowflake |
| Conectar Access Role a Functional Role | Admin Governança Snow | SQL no Snowflake |
| Conceder warehouses | Admin Governança Snow | SQL no Snowflake |
| Auditar acessos | Admin Governança Snow | SQL no Snowflake |

4. **Configure o SCIM uma vez, gerencie grants continuamente:** O trabalho inicial de configurar a integração SCIM é pontual. Porém, à medida que novos schemas, databases ou produtos de dados são criados, o admin Snowflake precisa continuamente criar Access Roles e conectá-las às Functional Roles existentes.

5. **Use o `DEFAULT_ROLE` correto:** Configure no IdP ou no Snowflake para que o `DEFAULT_ROLE` do usuário seja a Functional Role principal:

```sql
-- Verificar default roles dos usuários provisionados por SCIM
SELECT user_name, default_role, has_password, ext_authn_duo
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND created_on >= DATEADD('day', -30, CURRENT_TIMESTAMP())
ORDER BY created_on DESC;
```

### 10.7 SQL de Referência para Ambientes com SCIM

```sql
-- Ver todas as roles que foram criadas pelo SCIM (padrão de nome com prefixo)
SHOW ROLES LIKE 'GRP_%';

-- Ver usuários provisionados recentemente (sem password = provavelmente SSO/SCIM)
SELECT user_name, login_name, display_name, default_role, created_on
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE has_password = 'false'
  AND deleted_on IS NULL
ORDER BY created_on DESC;

-- Ver quais roles SCIM já possuem Access Roles atribuídas
SHOW GRANTS TO ROLE grp_bi_analyst;

-- Ver quais roles SCIM NÃO possuem nenhuma Database Role (possível problema)
-- Compare as roles com prefixo SCIM contra as que têm grants de database role
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

-- Atribuir Access Roles em lote a uma Functional Role SCIM
USE ROLE SECURITYADMIN;
GRANT DATABASE ROLE vendas.sc_raw_ro TO ROLE grp_data_engineer;
GRANT DATABASE ROLE vendas.sc_staging_rw TO ROLE grp_data_engineer;
GRANT DATABASE ROLE vendas.sc_presentation_ro TO ROLE grp_data_engineer;
GRANT USAGE ON WAREHOUSE wh_ingestao TO ROLE grp_data_engineer;
GRANT USAGE ON DATABASE vendas TO ROLE grp_data_engineer;

-- Auditoria: quem acessou o quê nas últimas 24h via roles SCIM
SELECT user_name, role_name, query_type, LEFT(query_text, 100) AS query_preview, start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE role_name LIKE 'GRP_%'
  AND start_time >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
ORDER BY start_time DESC
LIMIT 100;
```

---

## 11. Exemplos Práticos Completos

### 11.1 Cenário: Configurar Acesso para uma Empresa

Estrutura da empresa fictícia:
- Database: `ANALYTICS`
- Schemas: `RAW`, `STAGING`, `PRESENTATION`
- Times: Engenharia de Dados, BI, Ciência de Dados

#### Passo 1: Criar Database Roles (Access Roles)

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

#### Passo 2: Conceder Privilégios às Access Roles

```sql
-- === Schema RAW - Leitura ===
GRANT USAGE ON SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON ALL VIEWS IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_ro;

-- === Schema RAW - Escrita ===
GRANT USAGE ON SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;
GRANT CREATE TABLE ON SCHEMA analytics.raw TO DATABASE ROLE analytics.sc_raw_rw;

-- === Schema STAGING - Leitura ===
GRANT USAGE ON SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_ro;

-- === Schema STAGING - Escrita ===
GRANT USAGE ON SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;
GRANT CREATE TABLE, CREATE VIEW ON SCHEMA analytics.staging TO DATABASE ROLE analytics.sc_staging_rw;

-- === Schema PRESENTATION - Leitura ===
GRANT USAGE ON SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON FUTURE TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON ALL VIEWS IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_ro;

-- === Schema PRESENTATION - Escrita ===
GRANT USAGE ON SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
GRANT SELECT, INSERT, UPDATE, DELETE ON FUTURE TABLES IN SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
GRANT CREATE TABLE, CREATE VIEW ON SCHEMA analytics.presentation TO DATABASE ROLE analytics.sc_presentation_rw;
```

#### Passo 3: Criar Functional Roles e Atribuir Access Roles

```sql
USE ROLE USERADMIN;

-- Functional Roles por time
CREATE ROLE IF NOT EXISTS fr_engenharia_dados;
CREATE ROLE IF NOT EXISTS fr_bi_analyst;
CREATE ROLE IF NOT EXISTS fr_data_scientist;

-- Garantir que SYSADMIN herda todas as custom roles
GRANT ROLE fr_engenharia_dados TO ROLE SYSADMIN;
GRANT ROLE fr_bi_analyst TO ROLE SYSADMIN;
GRANT ROLE fr_data_scientist TO ROLE SYSADMIN;

USE ROLE SECURITYADMIN;

-- Engenharia de Dados: escrita em RAW e STAGING, leitura em PRESENTATION
GRANT DATABASE ROLE analytics.sc_raw_rw TO ROLE fr_engenharia_dados;
GRANT DATABASE ROLE analytics.sc_staging_rw TO ROLE fr_engenharia_dados;
GRANT DATABASE ROLE analytics.sc_presentation_ro TO ROLE fr_engenharia_dados;

-- BI Analyst: leitura em STAGING e PRESENTATION
GRANT DATABASE ROLE analytics.sc_staging_ro TO ROLE fr_bi_analyst;
GRANT DATABASE ROLE analytics.sc_presentation_ro TO ROLE fr_bi_analyst;

-- Data Scientist: leitura em tudo, escrita em STAGING
GRANT DATABASE ROLE analytics.sc_raw_ro TO ROLE fr_data_scientist;
GRANT DATABASE ROLE analytics.sc_staging_rw TO ROLE fr_data_scientist;
GRANT DATABASE ROLE analytics.sc_presentation_ro TO ROLE fr_data_scientist;

-- USAGE no database para todas as functional roles
GRANT USAGE ON DATABASE analytics TO ROLE fr_engenharia_dados;
GRANT USAGE ON DATABASE analytics TO ROLE fr_bi_analyst;
GRANT USAGE ON DATABASE analytics TO ROLE fr_data_scientist;
```

#### Passo 4: Criar Warehouses e Conceder Acesso

```sql
USE ROLE SYSADMIN;

-- Warehouses por perfil de uso
CREATE WAREHOUSE IF NOT EXISTS wh_ingestao
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

-- Conceder USAGE nos warehouses
GRANT USAGE ON WAREHOUSE wh_ingestao TO ROLE fr_engenharia_dados;
GRANT USAGE ON WAREHOUSE wh_analytics TO ROLE fr_bi_analyst;
GRANT USAGE ON WAREHOUSE wh_analytics TO ROLE fr_data_scientist;
GRANT USAGE ON WAREHOUSE wh_datascience TO ROLE fr_data_scientist;
```

#### Passo 5: Criar Usuários e Atribuir Roles

```sql
USE ROLE USERADMIN;

-- Criar usuário humano
CREATE USER IF NOT EXISTS maria_silva
  PASSWORD = 'TrocarNoPrimeiroLogin!'
  DEFAULT_ROLE = 'FR_BI_ANALYST'
  DEFAULT_WAREHOUSE = 'WH_ANALYTICS'
  MUST_CHANGE_PASSWORD = TRUE;

-- Atribuir functional role ao usuário
GRANT ROLE fr_bi_analyst TO USER maria_silva;

-- Criar service account (sem password — usa key pair)
CREATE USER IF NOT EXISTS svc_pipeline_ingestao
  DEFAULT_ROLE = 'FR_ENGENHARIA_DADOS'
  DEFAULT_WAREHOUSE = 'WH_INGESTAO'
  TYPE = SERVICE;

GRANT ROLE fr_engenharia_dados TO USER svc_pipeline_ingestao;
```

### 11.2 Verificação de Acesso

```sql
-- Ver todas as roles de um usuário
SHOW GRANTS TO USER maria_silva;

-- Ver todos os privilégios de uma role
SHOW GRANTS TO ROLE fr_bi_analyst;

-- Ver quem tem acesso a uma tabela específica
SHOW GRANTS ON TABLE analytics.presentation.vendas_mensal;

-- Ver a hierarquia completa de roles
SHOW GRANTS OF ROLE fr_data_scientist;
```

---

## 12. Checklist de Configuração de Segurança

Use esta lista ao configurar uma conta nova ou auditar uma conta existente:

### Camada 1: Network Access

- [ ] Network Policy criada para restringir IPs de acesso
- [ ] Considerar Private Link se a empresa exige tráfego privado

```sql
-- criar breaking glass
CREATE NETWORK POLICY IF NOT EXISTS breaking_glass
  ALLOWED_IP_LIST = ('0.0.0.0/255')
  COMMENT = 'Permite apenas rede corporativa';

ALTER USER <USUARIO_FAZENDO_SETUP> SET NETWORK_POLICY = 'breaking_glass';

-- Exemplo: criar network policy
CREATE NETWORK POLICY IF NOT EXISTS corp_network_policy
  ALLOWED_IP_LIST = ('10.0.0.0/8', '172.16.0.0/12', '200.100.50.0/24')
  COMMENT = 'Permite apenas rede corporativa';

-- Aplicar à conta
ALTER ACCOUNT SET NETWORK_POLICY = 'CORP_NETWORK_POLICY';
```

### Camada 2: Authentication

- [ ] MFA habilitado para **todos** os usuários com ACCOUNTADMIN
- [ ] MFA habilitado para SECURITYADMIN
- [ ] SSO configurado (SAML 2.0) como método primário
- [ ] Key Pair configurado para service accounts (sem password)
- [ ] Password policy definida

```sql
-- Verificar MFA dos admins
SELECT user_name, has_mfa_enrolled, default_role
FROM SNOWFLAKE.ACCOUNT_USAGE.USERS
WHERE deleted_on IS NULL
  AND default_role IN ('ACCOUNTADMIN', 'SECURITYADMIN');
```

### Camada 3: Authorization (RBAC)

- [ ] ACCOUNTADMIN atribuído a 2-4 usuários apenas
- [ ] Nenhum usuário tem ACCOUNTADMIN como DEFAULT_ROLE
- [ ] Nenhum objeto criado com ACCOUNTADMIN (devem ser criados com SYSADMIN)
- [ ] Hierarquia de roles definida: System Roles > Functional Roles > Access Roles
- [ ] Database Roles usados para acesso a objetos de database
- [ ] Future Grants configurados em todos os schemas
- [ ] Managed Access Schema usado para dados sensíveis
- [ ] Todas as custom roles herdam para SYSADMIN (não ficam "órfãs")
- [ ] Role PUBLIC não tem grants desnecessários

```sql
-- Verificar roles órfãs (não conectadas à hierarquia)
-- Roles que não foram concedidas a nenhuma outra role
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

### Camada 4: Continuous Data Protection

- [ ] Time Travel configurado (padrão: 1 dia; produção: até 90 dias)
- [ ] Data Masking Policies aplicadas a colunas PII
- [ ] Row Access Policies para dados multi-tenant

### Auditoria e Monitoramento

- [ ] Resource Monitors configurados para controlar custos
- [ ] ACCOUNT_USAGE monitorado periodicamente
- [ ] Alertas para grants de ACCOUNTADMIN
- [ ] Alertas para logins de IPs desconhecidos

---

## 13. Apêndice: Comandos SQL de Referência Rápida

### Gerenciamento de Roles

| Ação | SQL |
|------|-----|
| Criar role | `CREATE ROLE fr_nome;` |
| Criar database role | `CREATE DATABASE ROLE db.nome_role;` |
| Conceder role a usuário | `GRANT ROLE fr_nome TO USER joao;` |
| Conceder role a outra role | `GRANT ROLE fr_junior TO ROLE fr_senior;` |
| Conceder database role a role | `GRANT DATABASE ROLE db.role TO ROLE fr_nome;` |
| Remover role de usuário | `REVOKE ROLE fr_nome FROM USER joao;` |
| Listar roles | `SHOW ROLES;` |
| Listar database roles | `SHOW DATABASE ROLES IN DATABASE db;` |

### Gerenciamento de Grants

| Ação | SQL |
|------|-----|
| Grant de leitura em schema | `GRANT USAGE ON SCHEMA db.sch TO ROLE r; GRANT SELECT ON ALL TABLES IN SCHEMA db.sch TO ROLE r;` |
| Future grant de tabelas | `GRANT SELECT ON FUTURE TABLES IN SCHEMA db.sch TO ROLE r;` |
| Grant de warehouse | `GRANT USAGE ON WAREHOUSE wh TO ROLE r;` |
| Revogar grant | `REVOKE SELECT ON TABLE db.sch.t FROM ROLE r;` |
| Ver grants de um objeto | `SHOW GRANTS ON TABLE db.sch.t;` |
| Ver grants de uma role | `SHOW GRANTS TO ROLE r;` |
| Ver grants de um usuário | `SHOW GRANTS TO USER u;` |

### Gerenciamento de Usuários

| Ação | SQL |
|------|-----|
| Criar usuário | `CREATE USER nome PASSWORD='x' DEFAULT_ROLE='r' MUST_CHANGE_PASSWORD=TRUE;` |
| Criar service account | `CREATE USER svc_nome DEFAULT_ROLE='r' TYPE=SERVICE;` |
| Alterar default role | `ALTER USER nome SET DEFAULT_ROLE = 'nova_role';` |
| Desabilitar usuário | `ALTER USER nome SET DISABLED = TRUE;` |
| Dropar usuário | `DROP USER nome;` |

### Gerenciamento de Ownership

| Ação | SQL |
|------|-----|
| Transferir ownership | `GRANT OWNERSHIP ON TABLE db.sch.t TO ROLE r REVOKE CURRENT GRANTS;` |
| Ownership futuro | `GRANT OWNERSHIP ON FUTURE TABLES IN SCHEMA db.sch TO ROLE r;` |

### Troca de Contexto

| Ação | SQL |
|------|-----|
| Trocar role | `USE ROLE fr_nome;` |
| Ativar secondary roles | `USE SECONDARY ROLES ALL;` |
| Definir warehouse | `USE WAREHOUSE wh_nome;` |
| Definir database | `USE DATABASE db_nome;` |
| Definir schema | `USE SCHEMA schema_nome;` |

---

## Glossário

| Termo | Definição |
|-------|-----------|
| **RBAC** | Role-Based Access Control — modelo onde privilégios são atribuídos a Roles |
| **DAC** | Discretionary Access Control — o dono do objeto controla o acesso |
| **MAS** | Managed Access Schema — schema onde apenas o dono do schema gerencia grants |
| **Grant** | Comando que concede um privilégio a uma role |
| **Revoke** | Comando que remove um privilégio de uma role |
| **Ownership** | Privilégio especial de controle total sobre um objeto |
| **Future Grant** | Grant que se aplica automaticamente a objetos criados no futuro |
| **Functional Role** | Role atribuída a usuários, agrupa Access Roles |
| **Access Role** | Role que contém apenas privilégios sobre objetos |
| **Database Role** | Role que existe dentro de um database específico |
| **Secondary Roles** | Mecanismo para ativar múltiplas roles simultaneamente |
| **Network Policy** | Regra que restringe quais IPs podem acessar a conta |
| **MFA** | Multi-Factor Authentication — camada extra de verificação de identidade |
| **Service Account** | Usuário não-humano usado por aplicações e pipelines |

---

> **Documento gerado como referência para configuração de segurança Snowflake.**
> Baseado no material "RBAC Access Management Overview" do Snowflake Activation Team.
