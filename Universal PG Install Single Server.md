# PostgreSQL Installation & Configuration SOP
## Rocky Linux 9 — Binary Install (No Docker)

---

## How to Use This Document

This is a general-purpose SOP. Every project will have different values for the items listed below. Before starting, fill these in:

| Variable | Description | Example |
|----------|-------------|---------|
| `PG_VERSION` | PostgreSQL major version to install | `18` |
| `PG_FULL_VERSION` | Full version for verification | `18.3` |
| `DATA_DIR` | Custom data directory path | `/data/pgsql/18/data` |
| `LOG_DIR` | Custom log directory path | `/log/postgresql` |
| `ARCHIVE_DIR` | WAL archive directory path | `/log/archive` |
| `DB_NAME` | Application database name | `academydb` |
| `DB_USER` | Application database user | `tn_academy` |
| `DB_PASS` | Application database user password | *(generate a strong one)* |
| `APP_SERVER_IP` | Backend/application server IP | `172.31.50.51` |
| `VPN_SUBNET` | VPN gateway IP or subnet for DBA access | `172.31.30.0/24` |
| `LOG_PREFIX` | Log filename prefix | `pg-db1` |

> **Note on VPN IP:** When you SSH into the DB server through a VPN, the server sees connections coming from the VPN gateway IP — not your laptop's IP. Check `Last login: ... from X.X.X.X` in your terminal after login to find this IP. If it varies slightly across sessions, use a `/24` subnet instead of `/32`.

> **Note on DB_USER casing:** If the username has uppercase letters (e.g., `TN_Academy`), PostgreSQL will fold it to lowercase unless you wrap it in double quotes everywhere. To avoid errors, use all-lowercase usernames (e.g., `tn_academy`).

---

## Server Requirements

This guide works on any Rocky Linux 9 server. Configuration parameters in Phase 8 must be adjusted based on server specs. A hardware-to-config reference table is provided at the end of this document.

**Minimum recommended layout:**
- Separate mount points for `/data` and `/log` (can be on LVM, separate disks, or separate partitions)
- SELinux in **Enforcing** mode — do not disable it

---

## Pre-checks (Run These First)

```bash
# Confirm OS
cat /etc/rocky-release

# Confirm SELinux is Enforcing
getenforce

# Check disk layout and available space
df -h
lsblk

# Confirm no existing PostgreSQL packages
rpm -qa | grep postgresql
```

Expected: `getenforce` returns `Enforcing`. `rpm -qa | grep postgresql` returns nothing.

If a previous PostgreSQL version exists, remove it completely before proceeding (see Appendix A).

---

## Phase 1 — Add PGDG Repository

Rocky Linux ships with a built-in PostgreSQL module that conflicts with the official PGDG packages. Disable it first, then add the official repo.

```bash
# Add the official PostgreSQL Global Development Group repo for EL9
sudo dnf install -y \
  https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable the built-in module to prevent conflicts
sudo dnf module disable postgresql -y
```

---

## Phase 2 — Install PostgreSQL

```bash
# Replace PG_VERSION with your target version (e.g., 18)
sudo dnf install -y postgresql{PG_VERSION} postgresql{PG_VERSION}-server postgresql{PG_VERSION}-contrib
```

**Example for PG 18:**
```bash
sudo dnf install -y postgresql18 postgresql18-server postgresql18-contrib
```

Verify:
```bash
/usr/pgsql-{PG_VERSION}/bin/postgres --version
# Expected: postgres (PostgreSQL) {PG_FULL_VERSION}
```

---

## Phase 3 — Create Custom Directories

Do not use PostgreSQL's default paths. Use the dedicated mount points for data and logs.

```bash
# Data directory — PostgreSQL will store all database files here
sudo mkdir -p {DATA_DIR}
sudo chown -R postgres:postgres {DATA_DIR}
sudo chmod 700 {DATA_DIR}

# Log directory — PostgreSQL log files
sudo mkdir -p {LOG_DIR}
sudo chown -R postgres:postgres {LOG_DIR}
sudo chmod 700 {LOG_DIR}

# WAL archive directory — archived WAL segments for backup/recovery
sudo mkdir -p {ARCHIVE_DIR}
sudo chown -R postgres:postgres {ARCHIVE_DIR}
sudo chmod 700 {ARCHIVE_DIR}
```

Verify ownership and permissions:
```bash
ls -la /data/pgsql/{PG_VERSION}/
ls -la /log/
```

Expected: directories owned by `postgres:postgres` with `drwx------` permissions.

---

## Phase 4 — Apply SELinux Contexts

SELinux on Rocky Linux 9 runs in Enforcing mode. Custom mount points (especially on separate disks or LVM volumes) start with `unlabeled_t` context. PostgreSQL cannot read or write to these directories without the correct SELinux labels.

**Why two steps?** The parent directories (`/data`, `/log`) must first be given a generic valid type (`var_t`, `var_log_t`). Without this, `restorecon` on child paths may not propagate correctly. Then PostgreSQL-specific types are applied to the actual subdirectories.

**Why both `semanage` and `chcon`?** `chcon` applies the label immediately (takes effect right now). `semanage fcontext` makes it persistent across reboots. Both are needed.

```bash
# Step 1: Fix parent directory contexts
sudo semanage fcontext -a -t var_t "/data(/.*)?"
sudo restorecon -Rv /data

sudo semanage fcontext -a -t var_log_t "/log(/.*)?"
sudo restorecon -Rv /log

# Step 2: Apply PostgreSQL-specific contexts
sudo semanage fcontext -a -t postgresql_db_t "/data/pgsql(/.*)?"
sudo semanage fcontext -a -t postgresql_log_t "{LOG_DIR}(/.*)?"
sudo semanage fcontext -a -t postgresql_log_t "{ARCHIVE_DIR}(/.*)?"

sudo chcon -Rv -t postgresql_db_t /data/pgsql
sudo chcon -Rv -t postgresql_log_t {LOG_DIR}
sudo chcon -Rv -t postgresql_log_t {ARCHIVE_DIR}
```

Verify (requires `sudo` because directories are `700` and owned by `postgres`):
```bash
sudo ls -laZ {DATA_DIR}
sudo ls -laZ {LOG_DIR}
sudo ls -laZ {ARCHIVE_DIR}
```

Expected contexts:
- Data directory: `postgresql_db_t`
- Log and archive directories: `postgresql_log_t`

If the context still shows `unlabeled_t` after these steps, see Troubleshooting at the end of this document.

---

## Phase 5 — Initialize the Database Cluster (initdb)

`initdb` creates the initial database cluster — the directory structure, system catalogs, and configuration files that PostgreSQL needs to start.

```bash
sudo -u postgres /usr/pgsql-{PG_VERSION}/bin/initdb \
  -D {DATA_DIR} \
  --encoding=UTF8 \
  --locale=en_US.UTF-8
```

**Note on `--data-checksums`:** Starting from PostgreSQL 18, data checksums are enabled by default during `initdb`. You do not need to pass `--data-checksums` explicitly. For PostgreSQL 17 and earlier, add `--data-checksums` to the command above — checksums allow PostgreSQL to detect disk-level corruption and are strongly recommended for production.

Expected output ends with:
```
Success. You can now start the database server using: ...
```

Verify the cluster files were created:
```bash
sudo ls -la {DATA_DIR}
# Expected: pg_hba.conf, postgresql.conf, pg_wal/, base/, global/
```

---

## Phase 6 — Update systemd Service File

The default systemd service file points to a different `PGDATA` path. Update it to point to your custom data directory.

```bash
sudo sed -i 's|Environment=PGDATA=.*|Environment=PGDATA={DATA_DIR}/|' \
  /usr/lib/systemd/system/postgresql-{PG_VERSION}.service

# Verify the change
grep PGDATA /usr/lib/systemd/system/postgresql-{PG_VERSION}.service
# Expected: Environment=PGDATA={DATA_DIR}/

sudo systemctl daemon-reload
```

---

## Phase 7 — Configure pg_hba.conf

`pg_hba.conf` (Host-Based Authentication) controls who can connect to PostgreSQL, from which address, and with which authentication method.

> **Important:** Never use `trust` authentication in production. `trust` allows passwordless connections and is a critical security risk.

> **Authentication method:** Use `scram-sha-256` — it is the most secure password authentication method available in PostgreSQL. Never use `md5` in new setups.

```bash
sudo tee {DATA_DIR}/pg_hba.conf << 'EOF'
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local socket - postgres OS user (peer auth, no password needed)
# The OS already verified the user's identity via Unix socket ownership.
local   all             postgres                                peer

# Local socket - all other users require password
local   all             all                                     scram-sha-256

# Localhost TCP - IPv4
host    all             all             127.0.0.1/32            scram-sha-256

# Localhost TCP - IPv6
host    all             all             ::1/128                 scram-sha-256

# DBA access via VPN (replace with your VPN gateway IP or subnet)
host    all             all             {VPN_SUBNET}            scram-sha-256

# Application/Backend server
host    all             all             {APP_SERVER_IP}/32      scram-sha-256

# Replication - local only (future-safe even on standalone)
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            scram-sha-256
EOF
```

**Entry explanations:**

| Entry | Purpose |
|-------|---------|
| `local postgres peer` | DBA connecting via Unix socket as the `postgres` OS user. No password because the OS already verified identity. Used when you SSH into the server and run `sudo -u postgres psql`. |
| `local all scram-sha-256` | Any other local Unix socket connection requires a password. |
| `127.0.0.1/32` | Localhost TCP connections require a password. |
| `{VPN_SUBNET}` | DBA remote access through VPN. PostgreSQL sees connections coming from the VPN gateway IP, not your laptop. Check `Last login: ... from X.X.X.X` to find it. Use `/24` if the IP changes within the same subnet. |
| `{APP_SERVER_IP}/32` | The backend/application server. Use `/32` for a single specific IP. |

**How to find your VPN IP:** After SSH-ing into the server, the terminal shows `Last login: ... from X.X.X.X`. That `X.X.X.X` is what PostgreSQL will see as the source of your connection. Add that IP (or its `/24` subnet if it varies) to `pg_hba.conf`.

**After editing pg_hba.conf while the service is running**, reload without restarting:
```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

---

## Phase 8 — Configure postgresql.conf

Replace the default `postgresql.conf` with a tuned configuration. **Adjust parameter values based on your server's RAM and CPU.** See the hardware reference table at the end of this document.

```bash
sudo -u postgres tee {DATA_DIR}/postgresql.conf << 'EOF'
#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------
data_directory = '{DATA_DIR}'

#------------------------------------------------------------------------------
# CONNECTIONS
#------------------------------------------------------------------------------
listen_addresses = '*'
port = 5432
max_connections = {MAX_CONNECTIONS}
reserved_connections = 5
superuser_reserved_connections = 3

#------------------------------------------------------------------------------
# MEMORY
#------------------------------------------------------------------------------
shared_buffers = {SHARED_BUFFERS}
huge_pages = try
temp_buffers = 32MB
max_prepared_transactions = 0
work_mem = {WORK_MEM}
hash_mem_multiplier = 2.0
maintenance_work_mem = {MAINTENANCE_WORK_MEM}
logical_decoding_work_mem = 32MB
max_stack_depth = 2MB
dynamic_shared_memory_type = posix
vacuum_buffer_usage_limit = 4MB

#------------------------------------------------------------------------------
# I/O (NVMe)
#------------------------------------------------------------------------------
effective_io_concurrency = 200
maintenance_io_concurrency = 100
max_worker_processes = {MAX_WORKER_PROCESSES}
max_parallel_workers_per_gather = {MAX_PARALLEL_WORKERS_PER_GATHER}
max_parallel_maintenance_workers = {MAX_PARALLEL_MAINTENANCE_WORKERS}
max_parallel_workers = {MAX_WORKER_PROCESSES}

#------------------------------------------------------------------------------
# WAL
#------------------------------------------------------------------------------
wal_level = replica
fsync = on
synchronous_commit = local
full_page_writes = on
wal_compression = zstd
checkpoint_timeout = 10min
checkpoint_completion_target = 0.9
checkpoint_flush_after = 256kB
checkpoint_warning = 30s
max_wal_size = {MAX_WAL_SIZE}
min_wal_size = {MIN_WAL_SIZE}

#------------------------------------------------------------------------------
# ARCHIVING
#------------------------------------------------------------------------------
archive_mode = on
archive_command = 'test ! -f {ARCHIVE_DIR}/%f && cp %p {ARCHIVE_DIR}/%f'
archive_timeout = 0

#------------------------------------------------------------------------------
# REPLICATION (minimal — standalone)
#------------------------------------------------------------------------------
max_wal_senders = 3
max_replication_slots = 3
wal_keep_size = 256
max_slot_wal_keep_size = 200
wal_sender_timeout = 60s
hot_standby = on

#------------------------------------------------------------------------------
# QUERY TUNING
#------------------------------------------------------------------------------
seq_page_cost = 1.0
random_page_cost = 1.1
effective_cache_size = {EFFECTIVE_CACHE_SIZE}
enable_parallel_hash = on
enable_partition_pruning = on

#------------------------------------------------------------------------------
# LOGGING
#------------------------------------------------------------------------------
log_destination = 'stderr'
logging_collector = on
log_directory = '{LOG_DIR}'
log_filename = '{LOG_PREFIX}-%Y-%m-%d_%H%M%S.log'
log_file_mode = 0600
log_rotation_age = 1d
log_rotation_size = 50MB
log_truncate_on_rotation = off
log_min_messages = warning
log_min_error_statement = error
log_min_duration_statement = 3000
log_checkpoints = on
log_connections = off
log_disconnections = off
log_duration = off
log_error_verbosity = default
log_hostname = off
log_line_prefix = '%m [%p]: [%l-1] db=%d,user=%u,app=%a,client=%h '
log_lock_waits = on
log_statement = 'ddl'
log_temp_files = 1024
log_timezone = 'UTC'

#------------------------------------------------------------------------------
# AUTOVACUUM
#------------------------------------------------------------------------------
autovacuum = on
autovacuum_max_workers = {AUTOVACUUM_MAX_WORKERS}
autovacuum_naptime = 5min
autovacuum_vacuum_threshold = 500
autovacuum_vacuum_insert_threshold = 1000
autovacuum_analyze_threshold = 500
autovacuum_vacuum_scale_factor = 0.2
autovacuum_vacuum_insert_scale_factor = 0.2
autovacuum_analyze_scale_factor = 0.1
autovacuum_freeze_max_age = 200000000
autovacuum_multixact_freeze_max_age = 400000000
autovacuum_vacuum_cost_delay = 2ms
vacuum_cost_delay = 0
vacuum_cost_limit = {VACUUM_COST_LIMIT}
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0

#------------------------------------------------------------------------------
# CLIENT DEFAULTS
#------------------------------------------------------------------------------
statement_timeout = 0
lock_timeout = 5min
idle_in_transaction_session_timeout = 5min
idle_session_timeout = 10min
vacuum_freeze_table_age = 150000000
vacuum_freeze_min_age = 50000000
vacuum_failsafe_age = 1600000000
datestyle = 'iso, mdy'
timezone = 'UTC'
lc_messages = 'en_US.UTF-8'
lc_monetary = 'en_US.UTF-8'
lc_numeric = 'en_US.UTF-8'
lc_time = 'en_US.UTF-8'
default_text_search_config = 'pg_catalog.english'

#------------------------------------------------------------------------------
# LOCK MANAGEMENT
#------------------------------------------------------------------------------
deadlock_timeout = 1s
max_locks_per_transaction = 128
max_pred_locks_per_transaction = 128

#------------------------------------------------------------------------------
# PRELOAD LIBRARIES
#------------------------------------------------------------------------------
shared_preload_libraries = 'pg_stat_statements'
max_files_per_process = 1000
EOF
```

### Key Parameter Explanations

| Parameter | Rule | Reason |
|-----------|------|--------|
| `shared_buffers` | 25% of total RAM | PostgreSQL's main memory cache. The single most impactful memory parameter. |
| `effective_cache_size` | 50% of total RAM | Tells the query planner how much memory is available for caching (including OS cache). Does not allocate memory — only affects planning decisions. |
| `work_mem` | See table below | Memory per sort/hash operation per query. With many concurrent connections, setting this too high causes OOM. Formula: `(RAM - shared_buffers) / (max_connections * 2)`. |
| `maintenance_work_mem` | 3–6% of RAM | Used by VACUUM, CREATE INDEX, ALTER TABLE. Higher = faster index builds. |
| `max_worker_processes` | = CPU core count | Background worker ceiling. Do not exceed core count. |
| `max_parallel_workers_per_gather` | CPU cores / 2 | Parallel query workers per query. Leave at least half the cores for other queries. On 2-core servers, set to 1. |
| `max_wal_size` | 20% of data disk | Maximum WAL accumulation before a forced checkpoint. Too small = frequent checkpoints. Too large = long crash recovery. |
| `random_page_cost` | `1.1` for NVMe, `4.0` for HDD | Tells the planner how expensive random reads are relative to sequential. NVMe has nearly equal random/sequential speed, so set close to `seq_page_cost`. |
| `wal_compression` | `zstd` (PG 14+) | Compresses WAL segments. Reduces I/O on the WAL path. `zstd` is faster and compresses better than `pglz`. |
| `log_duration` | `off` | Logs the duration of every completed statement. On busy servers this floods the log directory quickly. Use `log_min_duration_statement` instead (logs only slow queries). |
| `log_hostname` | `off` | DNS reverse lookup on every connection. Slow and unnecessary when IPs are already logged. |
| `log_statement` | `ddl` | Logs CREATE, ALTER, DROP statements. `all` would log every SELECT — avoid in production unless debugging. |
| `autovacuum_max_workers` | CPU cores - 1, min 1 | VACUUM workers. Too many compete with query workers for CPU. |

---

## Phase 9 — Enable and Start PostgreSQL

```bash
sudo systemctl enable postgresql-{PG_VERSION}
sudo systemctl start postgresql-{PG_VERSION}
sudo systemctl status postgresql-{PG_VERSION}
```

Expected: `Active: active (running)`

---

## Phase 10 — Set postgres Superuser Password

```bash
sudo -u postgres psql
```

```sql
ALTER USER postgres WITH PASSWORD 'your_strong_superuser_password';
\q
```

---

## Phase 11 — Create Application Database and User

```bash
sudo -u postgres psql
```

```sql
-- Create the application database
CREATE DATABASE {DB_NAME};

-- Create the application user
CREATE USER {DB_USER} WITH PASSWORD '{DB_PASS}';

-- Grant connection and usage rights on the database
GRANT ALL PRIVILEGES ON DATABASE {DB_NAME} TO {DB_USER};

-- Grant schema access (required in PostgreSQL 15+)
\c {DB_NAME}
GRANT ALL ON SCHEMA public TO {DB_USER};

\q
```

**Note:** `GRANT ALL PRIVILEGES ON DATABASE` alone is not enough in PostgreSQL 15 and later. The `GRANT ALL ON SCHEMA public` line is required for the user to create and access objects inside the database.

---

## Phase 12 — Verification

```bash
# Confirm data directory, log directory, and archive mode
sudo -u postgres psql -c "SHOW data_directory; SHOW log_directory; SHOW archive_mode;"

# Confirm data checksums are enabled
sudo -u postgres psql -c "SHOW data_checksums;"
# Expected: on

# Confirm log file is being written
ls -la {LOG_DIR}/

# Test WAL archiving
sudo -u postgres psql -c "SELECT pg_switch_wal();"
ls {ARCHIVE_DIR}/
# A WAL segment file should appear

# Confirm pg_stat_statements extension
sudo -u postgres psql -c "CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"
sudo -u postgres psql -c "SELECT count(*) FROM pg_stat_statements;"

# Confirm database and user were created
sudo -u postgres psql -c "\l {DB_NAME}"
sudo -u postgres psql -c "\du {DB_USER}"
```

---

## Optional — Add Swap

Production servers should have swap to prevent OOM kills when memory is exhausted. A common rule is swap = 50% of RAM for servers above 4GB, or = RAM for servers at 4GB and below.

```bash
# Adjust size as needed (e.g., 2G, 4G, 8G)
sudo fallocate -l {SWAP_SIZE} /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Post-Setup Checklist

- [ ] No existing PostgreSQL packages (`rpm -qa | grep postgresql` returns empty)
- [ ] PG installed and version verified
- [ ] Custom data, log, archive directories created with correct ownership (`postgres:postgres`, `700`)
- [ ] SELinux contexts applied — `postgresql_db_t` and `postgresql_log_t` confirmed
- [ ] `initdb` completed successfully
- [ ] systemd service file updated with correct `PGDATA`
- [ ] `postgresql.conf` applied with tuned parameters
- [ ] `pg_hba.conf` configured — no `trust` entries, VPN IP and app server IP included
- [ ] `postgres` superuser password set
- [ ] Service enabled and running
- [ ] `data_directory`, `log_directory`, `archive_mode` verified via `SHOW`
- [ ] `data_checksums = on`
- [ ] Log file appearing in log directory
- [ ] WAL archive working — files appearing in archive directory
- [ ] Application database and user created
- [ ] Schema permissions granted
- [ ] Swap added

---

## Hardware Reference — postgresql.conf Parameters by Server Size

Use this table to pick the right values for `postgresql.conf` based on your server's RAM and CPU.

> These are starting-point recommendations for general-purpose OLTP workloads on NVMe storage. Adjust based on actual query patterns and monitoring data over time.

| Parameter | 2 CPU / 2GB RAM | 2 CPU / 4GB RAM | 4 CPU / 8GB RAM | 4 CPU / 16GB RAM | 8 CPU / 32GB RAM | 16 CPU / 64GB RAM |
|-----------|-----------------|-----------------|-----------------|------------------|------------------|-------------------|
| `max_connections` | 100 | 200 | 300 | 500 | 800 | 1000 |
| `shared_buffers` | 512MB | 1GB | 2GB | 4GB | 8GB | 16GB |
| `effective_cache_size` | 1GB | 2GB | 3GB | 8GB | 16GB | 48GB |
| `work_mem` | 2MB | 4MB | 8MB | 16MB | 32MB | 64MB |
| `maintenance_work_mem` | 64MB | 128MB | 256MB | 512MB | 1GB | 2GB |
| `max_worker_processes` | 2 | 2 | 4 | 4 | 8 | 16 |
| `max_parallel_workers_per_gather` | 1 | 1 | 2 | 2 | 4 | 8 |
| `max_parallel_maintenance_workers` | 1 | 1 | 2 | 2 | 4 | 8 |
| `max_parallel_workers` | 2 | 2 | 4 | 4 | 8 | 16 |
| `max_wal_size` | 1GB | 2GB | 4GB | 8GB | 16GB | 32GB |
| `min_wal_size` | 128MB | 256MB | 512MB | 1GB | 2GB | 4GB |
| `autovacuum_max_workers` | 1 | 2 | 3 | 3 | 4 | 5 |
| `vacuum_cost_limit` | 200 | 400 | 600 | 800 | 1000 | 1200 |
| `bgwriter_lru_maxpages` | 50 | 100 | 200 | 300 | 500 | 1000 |

### Notes on the Table

**`work_mem`** is the trickiest parameter. The values above assume a moderate number of active queries. If your application runs many concurrent complex queries with sorts or hash joins, lower this. If it mostly runs simple queries, you can raise it. The practical ceiling is: `(RAM - shared_buffers) / max_connections`. Going above this risks OOM.

**`max_wal_size`** should be sized against your data disk. As a rule, keep WAL under 20–25% of the data disk size. On small disks (20GB), `2–4GB` is the safe range.

**`shared_buffers` above 8GB** may require increasing the OS shared memory limit. Check with:
```bash
sysctl kernel.shmmax
sysctl kernel.shmall
```
If `shared_buffers` exceeds `kernel.shmmax`, PostgreSQL will fail to start. Add to `/etc/sysctl.conf`:
```bash
kernel.shmmax = 17179869184   # example for 16GB
kernel.shmall = 4194304
```

**`huge_pages = try`** means PostgreSQL will use huge pages if the OS has them configured, and silently fall back if not. For servers with `shared_buffers` above 4GB, configure huge pages at the OS level for better performance.

**Servers running other services** (e.g., MongoDB, Redis, application servers on the same machine): reduce `shared_buffers` to 15–20% of RAM and `effective_cache_size` to 30–40% of RAM to leave room for the other processes.

---

## Troubleshooting Reference

### PostgreSQL fails to start — Permission denied on data or log directory

**Cause:** SELinux is blocking access. Happens most often when the directories are on separate disks or LVM volumes that start with `unlabeled_t` context.

**Diagnosis:**
```bash
sudo ausearch -m avc -ts recent | grep postgres
# AVC denied messages will show which path and operation is being blocked
```

**Fix:** Re-apply the SELinux context from Phase 4. Make sure to fix the parent directory first (`var_t` for `/data`, `var_log_t` for `/log`) before applying the PostgreSQL-specific types.

---

### "no pg_hba.conf entry for host X.X.X.X"

**Cause:** The connecting IP is not listed in `pg_hba.conf`.

**Fix:** Add the IP or subnet to `pg_hba.conf`, then reload:
```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

To find the correct IP to add, check the error message — the IP in `for host X.X.X.X` is exactly what needs to be allowed. Or, after SSH-ing in, check `Last login: ... from X.X.X.X` in your terminal.

---

### "failed to resolve host" error in a client tool (e.g., pgAdmin, DBeaver)

**Cause:** A subnet (e.g., `172.31.30.0/24`) was entered in the Host field of the client instead of the DB server's actual IP.

**Fix:** The Host field must contain the DB server's IP or hostname (e.g., `10.0.130.206`), not a subnet. Subnets belong only in `pg_hba.conf`.

---

### `dnf module disable postgresql` fails or has no effect

```bash
dnf module list | grep postgres
dnf module disable postgresql:16   # or whichever version is listed
dnf list installed | grep postgresql
dnf remove postgresql*
dnf clean all
dnf makecache
```

---

### idle_session_timeout disconnects psql during setup

PostgreSQL disconnects idle sessions after `idle_session_timeout` (set to 10 minutes in this config). If you are cut off mid-setup, psql will attempt to reconnect automatically (`Attempting reset: Succeeded`). After reconnection, check whether your last command completed before re-running it.

To check if a database was created:
```sql
\l {DB_NAME}
```

---

## Appendix A — Removing an Existing PostgreSQL Installation

Only needed if a previous version is present (`rpm -qa | grep postgresql` returns output).

> **Warning:** Verify there is no production data before removing anything.

```bash
# Check what databases exist before removing
sudo -u postgres /usr/pgsql-{OLD_VERSION}/bin/psql -c "\l+"

# Stop and disable the service
sudo systemctl stop postgresql-{OLD_VERSION}
sudo systemctl disable postgresql-{OLD_VERSION}

# Remove data directory (confirm it has no production data first)
sudo rm -rf /data/pgsql/{OLD_VERSION}/

# Remove packages
sudo dnf remove -y postgresql{OLD_VERSION} postgresql{OLD_VERSION}-server \
  postgresql{OLD_VERSION}-libs postgresql{OLD_VERSION}-contrib 2>/dev/null
sudo rm -rf /usr/pgsql-{OLD_VERSION}/
sudo dnf clean all

# Verify clean removal
rpm -qa | grep postgresql
# Expected: no output
```
