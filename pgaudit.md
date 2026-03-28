# pgaudit

## Concept First

### What is pgaudit?

pgaudit is a PostgreSQL extension that provides detailed audit logging. PostgreSQL has built-in logging (`log_statement = 'all'`), but it only records query text — it does not tell you *who* accessed *which object* with *what operation* in a structured, queryable format.

pgaudit fills this gap. Every database operation gets a structured log entry with the user, object, operation type, and full statement. This is what compliance standards like PCI-DSS, HIPAA, and SOX require.

### Built-in Logging vs pgaudit

| Aspect | Built-in logging | pgaudit |
|--------|-----------------|---------|
| What it captures | Raw query text | User, object, operation, statement |
| Structured format | No | Yes — parseable fields |
| Object-level filtering | No | Yes — specific tables/views |
| Role-based audit | No | Yes |
| Compliance ready | No | Yes |

### Two Types of Audit

**Session audit** — logs all operations of specified types within a session. Controlled by `pgaudit.log`. Example: log all DDL and write operations globally.

**Object audit** — logs operations on specific objects (tables, views, functions). Controlled by granting permissions to a special audit role. Example: log all SELECT and DELETE on the `employees` table only, regardless of who does it.

Both can be active simultaneously. When an operation matches both, one log entry is written.

### Why shared_preload_libraries?

pgaudit is a **shared library** — compiled C code that hooks into PostgreSQL's internal executor. It must be loaded when PostgreSQL starts, not at runtime. This is why adding it to `shared_preload_libraries` requires a restart, while most other parameter changes only need a reload.

If you try to `CREATE EXTENSION pgaudit` without loading it via `shared_preload_libraries` first, PostgreSQL returns:
```
ERROR: pgaudit must be loaded via shared_preload_libraries
```

---

## Lab Environment

| Host | IP | Role |
|------|----|------|
| pnode1/pnode4/pnode5 | .136/.142/.144 | Patroni cluster nodes |
| HAProxy | 172.16.93.139 | Port 5000 → primary, Port 5001 → replicas |

All queries run through HAProxy port 5000 (primary). Patroni config changes from any cluster node. Log inspection on whichever node is currently primary.

---

## Setup

### Step 1 — Verify cluster state

```bash
# On any cluster node
patronictl -c /etc/patroni/patroni.yml list
```

Note which node is primary — you will need to SSH into it later to read log files.

### Step 2 — Verify pgaudit package is installed

```bash
# From any cluster node or backup server
psql -h 172.16.93.139 -p 5000 -U postgres -c \
  "SELECT name, default_version, installed_version FROM pg_available_extensions WHERE name = 'pgaudit';"
```

`default_version` shows the available version. `installed_version` is NULL if `CREATE EXTENSION` has not been run yet — that is fine at this stage, the package just needs to be present.

If the row is missing entirely, the package is not installed:

```bash
# On each cluster node
sudo dnf install -y pgaudit16
```

### Step 3 — Add pgaudit to shared_preload_libraries

In a Patroni-managed cluster, `postgresql.conf` is managed by Patroni — editing it directly gets overwritten. All permanent parameter changes must go through `patronictl edit-config`, which stores them in the distributed configuration (etcd) and propagates to all nodes.

```bash
# On any cluster node
patronictl -c /etc/patroni/patroni.yml edit-config
```

This opens a YAML editor. Find or add the `parameters:` section and modify `shared_preload_libraries`:

```yaml
parameters:
  shared_preload_libraries: pg_stat_statements,pgaudit
```

**Why keep `pg_stat_statements`?**
It was already there — removing it would disable query statistics tracking. Always append pgaudit to the existing value, never replace it.

Save and exit (`:wq` in vim).

**Verify the config was saved:**

```bash
patronictl -c /etc/patroni/patroni.yml show-config | grep -E "shared_preload|log_line_prefix"
```

### Step 4 — Rolling restart

`shared_preload_libraries` requires a full PostgreSQL restart to take effect. Patroni's rolling restart restarts nodes one at a time — replicas first, then primary — so the cluster stays available throughout.

```bash
# On any cluster node
patronictl -c /etc/patroni/patroni.yml restart postgres-cluster
```

Patroni will ask for confirmation. After confirming, watch the progress:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

Nodes cycle through `stopped` → `starting` → `running`. Once all nodes are `running`, the restart is complete.

**Verify pgaudit is loaded:**

```bash
psql -h 172.16.93.139 -p 5000 -U postgres -c "SHOW shared_preload_libraries;"
```

`pgaudit` should appear in the output.

### Step 5 — Create the pgaudit extension

Loading the library (Step 3-4) and creating the extension (this step) are two separate things:

- `shared_preload_libraries` → loads the C library into the PostgreSQL process at startup
- `CREATE EXTENSION` → creates the extension's SQL objects (functions, views, catalog entries) inside a specific database

The extension must be created in each database where you want audit logging.

```bash
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb -c \
  "CREATE EXTENSION IF NOT EXISTS pgaudit;"

# Verify
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb -c \
  "SELECT extname, extversion FROM pg_extension WHERE extname = 'pgaudit';"
```

---

## Session Audit

Session audit captures all operations of specified types — regardless of which object they touch.

### Step 6 — Configure pgaudit.log

`pgaudit.log` controls which operation classes are audited:

| Value | What it captures |
|-------|-----------------|
| `read` | SELECT, COPY FROM |
| `write` | INSERT, UPDATE, DELETE, TRUNCATE, COPY TO |
| `ddl` | CREATE, ALTER, DROP, TRUNCATE (schema changes) |
| `role` | GRANT, REVOKE, CREATE/ALTER/DROP ROLE |
| `function` | Function and procedure calls |
| `misc` | DISCARD, FETCH, CHECKPOINT, VACUUM |
| `all` | Everything above |
| `none` | Disable session audit |

```bash
patronictl -c /etc/patroni/patroni.yml edit-config
```

Add to `parameters:`:

```yaml
parameters:
  shared_preload_libraries: pg_stat_statements,pgaudit
  log_line_prefix: '%m [%p] %u@%d '
  pgaudit.log: ddl,write,role
  pgaudit.log_catalog: 'off'
  pgaudit.log_relation: 'on'
  pgaudit.log_statement_once: 'off'
```

**Why `log_line_prefix: '%m [%p] %u@%d'`?**
This adds timestamp (`%m`), process ID (`%p`), username (`%u`), and database name (`%d`) to every log line. pgaudit logs the user inside its own `AUDIT:` format, but having it in the line prefix makes grep-based filtering much easier — `grep "AUDIT:" | grep "app_user@"` works only if the prefix contains the username. This is a general logging improvement, not pgaudit-specific — but it makes the most sense to set it here alongside the audit configuration.

**Why `ddl,write,role` and not `all`?**
`read` (SELECT) generates enormous log volume in any active database — every application query gets logged. In practice, you audit writes and schema changes, not reads. Object audit (Step 9) handles sensitive read tracking on specific tables.

**Why `pgaudit.log_catalog: off`?**
PostgreSQL internally queries system catalogs (`pg_class`, `pg_attribute`, etc.) constantly — for every query it parses. With `log_catalog: on`, these internal queries flood the audit log with noise that has no security value.

**Why `pgaudit.log_relation: on`?**
When a statement touches multiple tables (a JOIN, for example), `log_relation: on` creates a separate audit entry for each table. This makes it possible to search audit logs by table name and find every query that touched it.

**Why `pgaudit.log_statement_once: off`?**
When `on`, for a statement touching multiple objects, the full SQL is logged only for the first object — subsequent objects just get the object name. Keeping it `off` logs the full statement every time, which is more useful for forensics but creates larger logs.

These parameters take effect with a reload (no restart needed):

```bash
patronictl -c /etc/patroni/patroni.yml reload postgres-cluster
```

**Verify:**

```bash
psql -h 172.16.93.139 -p 5000 -U postgres -c \
  "SHOW pgaudit.log; SHOW pgaudit.log_catalog; SHOW pgaudit.log_relation;"
```

### Step 7 — Test session audit

Open a log tail on the primary node in one terminal:

```bash
# On the primary node (whichever node is currently primary)
sudo tail -f /data/patroni/log/postgresql-$(date +%a).log | grep AUDIT
```

In another terminal, run test operations:

**DDL:**
```bash
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb << 'EOF'
CREATE TABLE audit_test (id SERIAL, data TEXT);
ALTER TABLE audit_test ADD COLUMN created_at TIMESTAMP;
DROP TABLE audit_test;
EOF
```

**Write:**
```bash
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb << 'EOF'
INSERT INTO app_schema.employees (name, department_id, salary)
  VALUES ('Test User', 1, 50000);
UPDATE app_schema.employees SET salary = 60000 WHERE name = 'Test User';
DELETE FROM app_schema.employees WHERE name = 'Test User';
EOF
```

**Role:**
```bash
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb << 'EOF'
CREATE ROLE audit_test_role;
GRANT SELECT ON app_schema.employees TO audit_test_role;
DROP ROLE audit_test_role;
EOF
```

### Step 8 — Read the audit log format

A session audit log entry looks like this:

```
2026-03-26 18:00:01 +06 [1234] postgres@practicedb LOG:  AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.audit_test,"CREATE TABLE audit_test (id SERIAL, data TEXT)",<not logged>
```

Breaking it down:

```
AUDIT: SESSION , 1 , 1 , DDL , CREATE TABLE , TABLE , public.audit_test , "CREATE TABLE..." , <not logged>
         |       |   |    |         |            |           |                    |                  |
         |       |   |    |         |            |           |                    |                  └─ bind parameters
         |       |   |    |         |            |           |                    └─ full SQL statement
         |       |   |    |         |            |           └─ schema.objectname
         |       |   |    |         |            └─ object type (TABLE/INDEX/ROLE...)
         |       |   |    |         └─ specific command (CREATE TABLE/INSERT/GRANT...)
         |       |   |    └─ audit class (DDL/WRITE/READ/ROLE/FUNCTION)
         |       |   └─ statement ID within audit entry
         |       └─ audit entry number within session
         └─ SESSION (vs OBJECT for object audit)
```

The line prefix (from `log_line_prefix`) adds timestamp, PID, user, and database before the `AUDIT:` keyword — this is why `log_line_prefix` was set in Step 3.

---

## Object Audit

Object audit logs operations on specific objects regardless of `pgaudit.log` settings. It works through PostgreSQL's permission system:

1. Create a dedicated audit role
2. Grant that role the permissions you want to audit on specific objects
3. Set `pgaudit.role` to that role name
4. pgaudit checks — for every operation — whether the audit role *would* have permission to perform it. If yes, it logs the operation.

This means: you do not need to log all SELECTs globally to track who reads the `employees` table.

### Step 9 — Configure object audit

```bash
# Create the audit role — NOLOGIN because it is never used to connect, only for permission checking
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb -c \
  "CREATE ROLE auditor NOLOGIN;"

# Grant SELECT and DELETE on employees to the auditor role
# This means: audit SELECT and DELETE on this table
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb -c \
  "GRANT SELECT, DELETE ON app_schema.employees TO auditor;"
```

Now tell pgaudit which role to use for object audit:

```bash
patronictl -c /etc/patroni/patroni.yml edit-config
```

Add to `parameters:`:

```yaml
pgaudit.role: auditor
```

```bash
patronictl -c /etc/patroni/patroni.yml reload postgres-cluster
```

### Step 10 — Test object audit

```bash
# SELECT — will be logged (auditor has SELECT on employees)
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb -c \
  "SELECT * FROM app_schema.employees LIMIT 2;"

# SELECT with count — also logged
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb -c \
  "SELECT count(*) FROM app_schema.employees;"

# INSERT — NOT logged (auditor does not have INSERT on employees)
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb -c \
  "INSERT INTO app_schema.employees (name, department_id, salary) VALUES ('Test', 1, 1000);"
```

The SELECT entries show `OBJECT` instead of `SESSION`:

```
AUDIT: OBJECT,1,1,READ,SELECT,TABLE,app_schema.employees,"SELECT * FROM app_schema.employees LIMIT 2",<not logged>
```

**Why SELECT is logged here but not in session audit:**
`pgaudit.log` is set to `ddl,write,role` — SELECT (`read` class) is excluded from session audit. But object audit bypasses `pgaudit.log` entirely — it only checks whether the `auditor` role has the relevant permission on the object. Since `auditor` has `SELECT` on `employees`, every SELECT on that table is logged regardless of session audit settings.

---

## Per-database and Per-user Audit

`pgaudit.log` can be overridden at the database or user level using `ALTER DATABASE` and `ALTER ROLE`. This allows fine-grained control without changing the global config.

```bash
# Log everything in practicedb specifically
psql -h 172.16.93.139 -p 5000 -U postgres -c \
  "ALTER DATABASE practicedb SET pgaudit.log = 'all';"

# Log read and write for app_user everywhere they connect
psql -h 172.16.93.139 -p 5000 -U postgres -c \
  "ALTER ROLE app_user SET pgaudit.log = 'read,write';"
```

**Priority order (highest to lowest):**
```
user-level (ALTER ROLE) → database-level (ALTER DATABASE) → global (postgresql.conf/patroni)
```

A user-level setting always wins. If `app_user` connects to `practicedb`, the user-level `read,write` applies, not the database-level `all`.

**Reset to inherit from global config:**

```bash
psql -h 172.16.93.139 -p 5000 -U postgres -c \
  "ALTER DATABASE practicedb RESET pgaudit.log;"

psql -h 172.16.93.139 -p 5000 -U postgres -c \
  "ALTER ROLE app_user RESET pgaudit.log;"
```

---

## Audit Log Analysis

In production, audit logs grow large quickly. Useful grep patterns:

```bash
# On the primary node
LOG=/data/patroni/log/postgresql-$(date +%a).log

# All audit entries
sudo grep "AUDIT:" $LOG

# Only DDL operations
sudo grep "AUDIT:.*DDL" $LOG

# All DROP statements — highest risk operations
sudo grep "AUDIT:.*DROP" $LOG

# All operations on the employees table
sudo grep "AUDIT:.*employees" $LOG

# Object audit entries only (not session)
sudo grep "AUDIT: OBJECT" $LOG

# Operations by a specific user (requires log_line_prefix with %u)
sudo grep "AUDIT:" $LOG | grep "app_user@"

# Last hour of audit entries
sudo grep "AUDIT:" $LOG | \
  awk -v cutoff="$(date -d '1 hour ago' '+%Y-%m-%d %H:%M')" '$0 >= cutoff'
```

---

## Troubleshooting

### pgaudit must be loaded via shared_preload_libraries

```
ERROR: pgaudit must be loaded via shared_preload_libraries
```

`CREATE EXTENSION pgaudit` was attempted before the library was loaded. Add pgaudit to `shared_preload_libraries` via `patronictl edit-config` and do a rolling restart first.

### Extension not available

```
ERROR: extension "pgaudit" is not available
```

The pgaudit RPM package is not installed on the node. Install it:

```bash
sudo dnf install -y pgaudit16
```

This must be done on **every cluster node**, then rolling restart.

### AUDIT entries not appearing in log

Check that `pgaudit.log` is set and the extension exists:

```bash
psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb -c \
  "SHOW pgaudit.log;"

psql -h 172.16.93.139 -p 5000 -U postgres -d practicedb -c \
  "SELECT extname FROM pg_extension WHERE extname = 'pgaudit';"
```

Also verify you are tailing the correct log file on the correct (primary) node — replicas do not execute write queries, so their logs will not show write audit entries.

### Patroni edit-config changes not applying

After `edit-config`, run:

```bash
patronictl -c /etc/patroni/patroni.yml show-config
```

If your changes are not there, the save did not complete. Re-run `edit-config`.

For parameters requiring restart (like `shared_preload_libraries`), reload is not enough:

```bash
# Check if any node needs restart
patronictl -c /etc/patroni/patroni.yml list
# Nodes with pending_restart = true need a restart
```

---

## Key Concepts Summary

```
pgaudit essentials:

1. pgaudit provides structured audit logging — user, object, operation, statement
2. Two audit types:
   - Session audit → pgaudit.log controls which operation classes are logged globally
   - Object audit  → pgaudit.role controls which objects are audited via permission check
3. shared_preload_libraries → must include pgaudit → requires restart (not just reload)
4. CREATE EXTENSION → must be run per database → only needs reload
5. In Patroni clusters → all config changes via patronictl edit-config
   Never edit postgresql.conf directly — Patroni overwrites it
6. pgaudit.log values: read, write, ddl, role, function, misc, all, none
7. pgaudit.log_catalog: off → prevents system catalog query noise
8. pgaudit.log_relation: on → one log entry per table in multi-table statements
9. Object audit bypasses pgaudit.log — works independently via role permissions
10. Priority: ALTER ROLE > ALTER DATABASE > global config
11. log_line_prefix with %u@%d adds user and database to every log line
12. AUDIT: SESSION vs AUDIT: OBJECT distinguishes audit type in log entries
```
