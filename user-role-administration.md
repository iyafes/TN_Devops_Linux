# PostgreSQL User & Role Administration

## Environment Reference

| Node | IP | Role |
|------|----|------|
| pnode1 | 172.16.93.136 | Replica |
| pnode4 | 172.16.93.142 | Primary (Leader) |
| pnode5 | 172.16.93.144 | Replica |
| haproxy | 172.16.93.139 | Load Balancer |

**Connect to primary (for all write operations):**
```bash
psql -h 172.16.93.139 -p 5000 -U postgres
```

> All commands in this guide run from a db node (pnode1, pnode4, or pnode5) where `psql` is available. HAProxy node does not have PostgreSQL client installed by default.

---

## 1. Core Concept — Role vs User

PostgreSQL does not have a separate concept of "user" and "role" — everything is a **role**. The only difference is whether the role has the `LOGIN` attribute:

```sql
CREATE USER myuser ...   -- equivalent to: CREATE ROLE myuser WITH LOGIN ...
CREATE ROLE myrole ...   -- no LOGIN by default, cannot connect to the database
```

Roles serve two purposes:
- **Login identity** — applications and humans connect as a login role
- **Permission group** — bundle a set of privileges that can be granted to other roles

This means you can build a hierarchy: create roles that hold permissions, then grant those roles to login users. When a user logs in, they inherit all permissions from their granted roles.

---

## 2. Role Attributes

When creating a role, you can set the following attributes:

| Attribute | Description |
|-----------|-------------|
| `LOGIN` | Allows the role to connect to the database |
| `SUPERUSER` | Bypasses all permission checks — use with extreme caution |
| `CREATEDB` | Allows creating new databases |
| `CREATEROLE` | Allows creating and managing other roles |
| `REPLICATION` | Allows initiating streaming replication |
| `CONNECTION LIMIT n` | Maximum simultaneous connections (default: no limit) |
| `PASSWORD 'x'` | Sets the login password |
| `VALID UNTIL 'timestamp'` | Password expiry date |

---

## 3. Connecting to the Cluster

In a Patroni-managed cluster, always connect through HAProxy:

- **Port 5000** → routes to the current Primary (use for all writes and DDL)
- **Port 5001** → routes to Replicas (use for read-only queries)

```bash
# Connect to primary
psql -h 172.16.93.139 -p 5000 -U postgres

# Connect to a replica (read-only)
psql -h 172.16.93.139 -p 5001 -U postgres
```

Any role created on the Primary automatically replicates to all Replicas via streaming replication — you do not need to create roles on each node separately.

---

## 4. Hands-On Practice

### 4.1 — Set Up a Practice Database

Connect to primary and create an isolated practice environment:

```sql
CREATE DATABASE practicedb;
\c practicedb

CREATE SCHEMA app_schema;

CREATE TABLE app_schema.employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    salary NUMERIC(10,2)
);

CREATE TABLE app_schema.departments (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    budget NUMERIC(12,2)
);

INSERT INTO app_schema.employees (name, department, salary) VALUES
('Alice', 'Engineering', 95000),
('Bob', 'Marketing', 75000),
('Charlie', 'Engineering', 105000),
('Diana', 'HR', 65000);

INSERT INTO app_schema.departments (name, budget) VALUES
('Engineering', 500000),
('Marketing', 200000),
('HR', 150000);
```

---

### 4.2 — Create Roles

**Create a permission group role (no login):**

```sql
CREATE ROLE readonly_role;
CREATE ROLE readwrite_role;
```

These roles cannot log in — they exist purely to hold permissions.

**Create login users:**

```sql
CREATE ROLE dev_user WITH LOGIN PASSWORD 'dev123';
CREATE ROLE app_user WITH LOGIN PASSWORD 'app123';
CREATE ROLE senior_dev WITH LOGIN PASSWORD 'senior123';
```

**Verify roles were created:**

```sql
SELECT rolname, rolcanlogin, rolsuper, rolcreatedb, rolcreaterole
FROM pg_roles
WHERE rolname IN ('readonly_role', 'readwrite_role', 'dev_user', 'app_user', 'senior_dev')
ORDER BY rolname;
```

Expected output:
```
     rolname      | rolcanlogin | rolsuper | rolcreatedb | rolcreaterole
------------------+-------------+----------+-------------+---------------
 app_user         | t           | f        | f           | f
 dev_user         | t           | f        | f           | f
 readonly_role    | f           | f        | f           | f
 readwrite_role   | f           | f        | f           | f
 senior_dev       | t           | f        | f           | f
```

---

### 4.3 — Grant Read-Only Permissions

PostgreSQL permissions work in layers. Granting access at one layer does not automatically grant access at layers below it — each layer must be granted explicitly:

```
Database  →  Schema  →  Table  →  Column
```

**Grant read-only access to `readonly_role`:**

```sql
-- Layer 1: allow connecting to the database
GRANT CONNECT ON DATABASE practicedb TO readonly_role;

-- Layer 2: allow seeing objects inside the schema
GRANT USAGE ON SCHEMA app_schema TO readonly_role;

-- Layer 3: allow reading all existing tables
GRANT SELECT ON ALL TABLES IN SCHEMA app_schema TO readonly_role;

-- Future tables: automatically grant SELECT when new tables are created
ALTER DEFAULT PRIVILEGES IN SCHEMA app_schema
    GRANT SELECT ON TABLES TO readonly_role;
```

**Why `ALTER DEFAULT PRIVILEGES`?**
`GRANT SELECT ON ALL TABLES` only applies to tables that exist right now. If a new table is created later, `readonly_role` would not have access to it. `ALTER DEFAULT PRIVILEGES` fixes this by automatically granting `SELECT` on any future tables created in that schema.

**Assign `readonly_role` to `dev_user`:**

```sql
GRANT readonly_role TO dev_user;
```

**Test — open a second terminal and connect as `dev_user`:**

```bash
psql -h 172.16.93.139 -p 5000 -U dev_user -d practicedb
```

```sql
-- This works
SELECT * FROM app_schema.employees;

-- This fails — no write permission
INSERT INTO app_schema.employees (name, department, salary)
VALUES ('Eve', 'Finance', 80000);
-- ERROR:  permission denied for table employees
```

---

### 4.4 — Grant Read-Write Permissions

```sql
-- Connect back as postgres
\c practicedb postgres

GRANT CONNECT ON DATABASE practicedb TO readwrite_role;
GRANT USAGE ON SCHEMA app_schema TO readwrite_role;

-- Grant DML on all existing tables
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app_schema TO readwrite_role;

-- Grant access to sequences (needed for SERIAL/INSERT to work)
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app_schema TO readwrite_role;

-- Future tables and sequences
ALTER DEFAULT PRIVILEGES IN SCHEMA app_schema
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite_role;

ALTER DEFAULT PRIVILEGES IN SCHEMA app_schema
    GRANT USAGE ON SEQUENCES TO readwrite_role;
```

**Why sequence permission?**
`SERIAL` columns use a sequence to generate the next ID. Without `USAGE` on the sequence, `INSERT` will fail even if the user has `INSERT` on the table.

**Assign `readwrite_role` to `app_user`:**

```sql
GRANT readwrite_role TO app_user;
```

**Test:**

```bash
psql -h 172.16.93.139 -p 5000 -U app_user -d practicedb
```

```sql
-- INSERT works
INSERT INTO app_schema.employees (name, department, salary)
VALUES ('Eve', 'Finance', 80000);

-- DELETE works
DELETE FROM app_schema.employees WHERE name = 'Eve';

-- DROP TABLE fails — DDL requires ownership
DROP TABLE app_schema.employees;
-- ERROR:  must be owner of table employees
```

---

### 4.5 — Role Hierarchy

Roles can be granted to other roles, building a hierarchy. Permissions flow downward through inheritance.

```sql
-- Grant readwrite_role to senior_dev
GRANT readwrite_role TO senior_dev;

-- Additionally grant CREATE on the schema (can create new tables)
GRANT CREATE ON SCHEMA app_schema TO senior_dev;
```

`senior_dev` now has:
- Everything from `readwrite_role` (SELECT, INSERT, UPDATE, DELETE)
- Ability to CREATE new tables in `app_schema`

But still cannot:
- Create databases
- Create roles
- Perform superuser operations

**View the full role hierarchy:**

```sql
SELECT
    r.rolname AS role,
    m.rolname AS member_of
FROM pg_roles r
JOIN pg_auth_members am ON am.member = r.oid
JOIN pg_roles m ON m.oid = am.roleid
ORDER BY r.rolname;
```

---

### 4.6 — Inspect Permissions

**View all roles:**
```sql
\du
```

**View database-level permissions:**
```sql
SELECT datname, datacl FROM pg_database WHERE datname = 'practicedb';
```

**View schema-level permissions:**
```sql
SELECT nspname, nspacl FROM pg_namespace WHERE nspname = 'app_schema';
```

**View table-level permissions:**
```sql
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'app_schema'
ORDER BY grantee, table_name, privilege_type;
```

**View all effective permissions for a specific user (including inherited from roles):**
```sql
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'dev_user'
   OR grantee IN (
       SELECT m.rolname
       FROM pg_auth_members am
       JOIN pg_roles r ON r.oid = am.member
       JOIN pg_roles m ON m.oid = am.roleid
       WHERE r.rolname = 'dev_user'
   )
ORDER BY table_name, privilege_type;
```

---

### 4.7 — Revoking Permissions

**Revoke a specific privilege:**
```sql
REVOKE INSERT ON ALL TABLES IN SCHEMA app_schema FROM readwrite_role;
```

**Revoke a role from a user:**
```sql
REVOKE readwrite_role FROM app_user;
```

**Drop a role:**
```sql
-- If the role owns any objects, transfer ownership first
REASSIGN OWNED BY app_user TO postgres;

-- Remove all privileges the role has granted
DROP OWNED BY app_user;

-- Then drop the role
DROP ROLE app_user;
```

**Why `REASSIGN OWNED` before dropping?**
If a role owns tables, sequences, or other objects, PostgreSQL will refuse to drop it. `REASSIGN OWNED` transfers all owned objects to another role. `DROP OWNED` removes any remaining privileges granted by or to that role.

---

### 4.8 — Special Role Attributes

```sql
-- Superuser (bypasses all permission checks — use sparingly)
CREATE ROLE dba_user WITH LOGIN PASSWORD 'dba123' SUPERUSER;

-- Can create databases
CREATE ROLE db_creator WITH LOGIN PASSWORD 'dc123' CREATEDB;

-- Can create and manage other roles
CREATE ROLE role_admin WITH LOGIN PASSWORD 'ra123' CREATEROLE;

-- Password expires on a specific date
CREATE ROLE temp_user WITH LOGIN PASSWORD 'temp123'
    VALID UNTIL '2026-12-31';

-- Limit simultaneous connections
CREATE ROLE limited_user WITH LOGIN PASSWORD 'lim123'
    CONNECTION LIMIT 5;
```

---

### 4.9 — pg_hba.conf and Connection Access Control

`pg_hba.conf` is PostgreSQL's connection-level firewall. It controls which users can connect, from which hosts, to which databases, and how they authenticate — before any SQL permission checks happen.

**File format:**
```
TYPE   DATABASE   USER      ADDRESS         METHOD
host   all        all       0.0.0.0/0       md5
host   replication replicator 172.16.93.0/24 md5
local  all        postgres                  peer
```

| Field | Description |
|-------|-------------|
| `TYPE` | `local` = Unix socket, `host` = TCP/IP |
| `DATABASE` | Database name, or `all`, or `replication` |
| `USER` | Role name, or `all` |
| `ADDRESS` | IP/CIDR range (only for `host` type) |
| `METHOD` | `md5` = password, `peer` = OS user match, `trust` = no auth |

**In a Patroni cluster, never edit `pg_hba.conf` directly.** Patroni overwrites it on restart. Always use `patronictl edit-config`:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config
```

Under the `pg_hba:` section:
```yaml
pg_hba:
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 172.16.93.0/24 md5
  - host all all 0.0.0.0/0 md5
  # Add a rule to restrict a specific user to a specific subnet
  - host practicedb dev_user 172.16.93.0/24 md5
```

Rules are evaluated **top to bottom** — the first matching rule wins. More specific rules must come before more general ones.

After saving, apply without a restart:
```bash
patronictl -c /etc/patroni/patroni.yml reload postgres-cluster
```

---

### 4.10 — Row Level Security (RLS)

Row Level Security lets you restrict which rows a user can see or modify within a table, based on a policy. Different users see different subsets of the same table.

**Enable RLS on a table:**
```sql
\c practicedb postgres

ALTER TABLE app_schema.employees ENABLE ROW LEVEL SECURITY;
```

**Create a policy — users only see rows matching their department setting:**
```sql
CREATE POLICY dept_isolation ON app_schema.employees
    USING (department = current_setting('app.current_dept', true));
```

**Create a test user:**
```sql
CREATE ROLE eng_user WITH LOGIN PASSWORD 'eng123';
GRANT readonly_role TO eng_user;
```

**Test RLS:**
```bash
psql -h 172.16.93.139 -p 5000 -U eng_user -d practicedb
```

```sql
-- Set the department context
SET app.current_dept = 'Engineering';

-- Only Engineering rows are returned
SELECT * FROM app_schema.employees;

-- Switch context
SET app.current_dept = 'Marketing';

-- Only Marketing rows are returned
SELECT * FROM app_schema.employees;
```

**Important behavior:**
- Table owners and superusers bypass RLS by default
- To force RLS even for the table owner:
```sql
ALTER TABLE app_schema.employees FORCE ROW LEVEL SECURITY;
```

**Remove RLS:**
```sql
DROP POLICY dept_isolation ON app_schema.employees;
ALTER TABLE app_schema.employees DISABLE ROW LEVEL SECURITY;
```

---

## 5. Common Mistakes and How to Avoid Them

**Mistake 1 — Granting table access without schema USAGE**

```sql
-- This alone is not enough
GRANT SELECT ON ALL TABLES IN SCHEMA app_schema TO some_role;

-- Without this, the user gets "schema does not exist" or "permission denied for schema"
GRANT USAGE ON SCHEMA app_schema TO some_role;
```

**Mistake 2 — Forgetting ALTER DEFAULT PRIVILEGES**

```sql
-- User has access to current tables but not future ones
GRANT SELECT ON ALL TABLES IN SCHEMA app_schema TO readonly_role;

-- Fix: add this so future tables are also covered
ALTER DEFAULT PRIVILEGES IN SCHEMA app_schema
    GRANT SELECT ON TABLES TO readonly_role;
```

**Mistake 3 — Dropping a role that owns objects**

```sql
-- This will fail if the role owns any objects
DROP ROLE some_role;
-- ERROR: role "some_role" cannot be dropped because some objects depend on it

-- Fix: reassign ownership first
REASSIGN OWNED BY some_role TO postgres;
DROP OWNED BY some_role;
DROP ROLE some_role;
```

**Mistake 4 — Editing pg_hba.conf directly in a Patroni cluster**

Direct edits to `/data/patroni/pg_hba.conf` are overwritten by Patroni on the next restart. Always use `patronictl edit-config` to make changes persistent.

**Mistake 5 — Forgetting sequence permissions for INSERT**

```sql
-- INSERT fails even with table INSERT permission if sequence is missing
GRANT INSERT ON ALL TABLES IN SCHEMA app_schema TO some_role;

-- Fix: also grant sequence usage
GRANT USAGE ON ALL SEQUENCES IN SCHEMA app_schema TO some_role;
```

---

## 6. Quick Reference

```sql
-- Create roles
CREATE ROLE role_name;
CREATE ROLE user_name WITH LOGIN PASSWORD 'password';

-- Grant permissions
GRANT CONNECT ON DATABASE dbname TO role_name;
GRANT USAGE ON SCHEMA schema_name TO role_name;
GRANT SELECT ON ALL TABLES IN SCHEMA schema_name TO role_name;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA schema_name TO role_name;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA schema_name TO role_name;
GRANT role_name TO user_name;

-- Future object permissions
ALTER DEFAULT PRIVILEGES IN SCHEMA schema_name
    GRANT SELECT ON TABLES TO role_name;
ALTER DEFAULT PRIVILEGES IN SCHEMA schema_name
    GRANT USAGE ON SEQUENCES TO role_name;

-- Revoke permissions
REVOKE privilege ON object FROM role_name;
REVOKE role_name FROM user_name;

-- Drop a role safely
REASSIGN OWNED BY role_name TO postgres;
DROP OWNED BY role_name;
DROP ROLE role_name;

-- Inspect
\du                          -- list all roles
\dp app_schema.*             -- show table permissions
SELECT * FROM pg_roles;      -- detailed role info
SELECT * FROM information_schema.role_table_grants WHERE table_schema = 'app_schema';

-- Row Level Security
ALTER TABLE tablename ENABLE ROW LEVEL SECURITY;
CREATE POLICY policy_name ON tablename USING (condition);
DROP POLICY policy_name ON tablename;
ALTER TABLE tablename DISABLE ROW LEVEL SECURITY;
```

---

*Environment: Rocky Linux 9.7 | PostgreSQL 16 | Patroni 4.x | HAProxy 2.8.x*
