# PostgreSQL Installation & Configuration SOP
## Ubuntu (20.04 / 22.04 / 24.04) — Binary Install (No Docker)

---

## How to Use This Document

This is a general-purpose SOP. Every project will have different values for the items listed below. Before starting, fill these in:

| Variable | Description | Example |
|----------|-------------|---------|
| `PG_VERSION` | PostgreSQL major version to install | `18` |
| `PG_FULL_VERSION` | Full version for verification | `18.3` |
| `UBUNTU_CODENAME` | Ubuntu release codename | `noble` (24.04), `jammy` (22.04), `focal` (20.04) |
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

> **Ubuntu codename reference:**
> - Ubuntu 24.04 LTS → `noble`
> - Ubuntu 22.04 LTS → `jammy`
> - Ubuntu 20.04 LTS → `focal`

---

## Key Differences from Rocky Linux

If you have used the Rocky Linux version of this SOP, note the following differences on Ubuntu:

| Area | Rocky Linux 9 | Ubuntu |
|------|--------------|--------|
| Package manager | `dnf` | `apt` |
| PGDG repo setup | `.rpm` file via `dnf install` | GPG key + `.list` file via `apt` |
| Built-in PG module conflict | `dnf module disable postgresql` | Not needed — Ubuntu does not have this |
| Mandatory access control | SELinux (Enforcing) | AppArmor (active but PG-aware by default) |
| systemd service file location | `/usr/lib/systemd/system/` | `/lib/systemd/system/` |
| Default `postgres` OS user shell | `/bin/bash` | `/bin/bash` |
| `sudo -u postgres psql` | Works the same | Works the same |

---

## Pre-checks (Run These First)

```bash
# Confirm OS and codename
cat /etc/os-release
lsb_release -cs

# Check AppArmor status
sudo aa-status | head -20

# Check disk layout and available space
df -h
lsblk

# Confirm no existing PostgreSQL packages
dpkg -l | grep postgresql
```

Expected: `lsb_release -cs` returns your Ubuntu codename (e.g., `noble`). `dpkg -l | grep postgresql` returns nothing.

If a previous PostgreSQL version exists, remove it completely before proceeding (see Appendix A).

---

## Phase 1 — Add PGDG Repository

Ubuntu's default apt repositories include older PostgreSQL versions. Use the official PGDG apt repository to get the latest releases.

```bash
# Install prerequisites
sudo apt install -y curl ca-certificates

# Create the keyring directory if it doesn't exist
sudo install -d /usr/share/postgresql-common/pgdg

# Download and add the PGDG signing key
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
  --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

# Add the PGDG apt repository
# Replace {UBUNTU_CODENAME} with your Ubuntu codename (e.g., noble, jammy, focal)
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] \
  https://apt.postgresql.org/pub/repos/apt {UBUNTU_CODENAME}-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'

# Update package lists
sudo apt update
```

Verify the repo was added:
```bash
cat /etc/apt/sources.list.d/pgdg.list
# Expected: deb [...] https://apt.postgresql.org/pub/repos/apt {UBUNTU_CODENAME}-pgdg main
```

---

## Phase 2 — Install PostgreSQL

```bash
# Replace {PG_VERSION} with your target version (e.g., 18)
sudo apt install -y postgresql-{PG_VERSION} postgresql-contrib-{PG_VERSION}
```

**Example for PG 18:**
```bash
sudo apt install -y postgresql-18 postgresql-contrib-18
```

> **Important:** The PGDG apt package automatically starts PostgreSQL after install and initializes a default cluster at `/var/lib/postgresql/{PG_VERSION}/main`. We will not use this default cluster — we will create our own at the custom `DATA_DIR`. Stop the service immediately after install.

```bash
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

Verify the binary is installed:
```bash
/usr/lib/postgresql/{PG_VERSION}/bin/postgres --version
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

## Phase 4 — AppArmor (Ubuntu's Access Control)

Unlike Rocky Linux which uses SELinux, Ubuntu uses **AppArmor**. The good news: the PGDG package ships with an AppArmor profile for PostgreSQL that already allows access to custom directories — no manual labeling is needed in most cases.

Verify AppArmor is not blocking PostgreSQL:
```bash
sudo aa-status | grep postgres
# If postgres appears under "enforce mode profiles", check if it causes issues after starting
```

If PostgreSQL fails to start or write to custom directories due to AppArmor, add your custom paths to the profile:

```bash
# Edit the PostgreSQL AppArmor profile
sudo nano /etc/apparmor.d/usr.lib.postgresql.{PG_VERSION}.bin.postgres
```

Add these lines inside the profile (before the closing `}`):
```
{DATA_DIR}/ r,
{DATA_DIR}/** rwk,
{LOG_DIR}/ r,
{LOG_DIR}/** rw,
{ARCHIVE_DIR}/ r,
{ARCHIVE_DIR}/** rw,
```

Reload the profile:
```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.lib.postgresql.{PG_VERSION}.bin.postgres
```

> **In practice:** Most Ubuntu setups with custom directories will work without touching AppArmor, because the default PostgreSQL AppArmor profile is permissive about custom paths. Only modify it if you see AppArmor denials in `/var/log/syslog` after attempting to start PostgreSQL.

Check for AppArmor denials:
```bash
sudo grep apparmor /var/log/syslog | grep postgres | tail -20
```

---

## Phase 5 — Remove the Default Cluster

The PGDG apt package creates a default cluster at `/var/lib/postgresql/{PG_VERSION}/main`. Remove it — we will use our custom `DATA_DIR` instead.

```bash
# Drop the default cluster (this only removes the cluster data, not the binaries)
sudo pg_dropcluster --stop {PG_VERSION} main
```

Verify it is gone:
```bash
pg_lsclusters
# Expected: no clusters listed, or only the one you will create
```

---

## Phase 6 — Initialize the Database Cluster (initdb)

`initdb` creates the initial database cluster — the directory structure, system catalogs, and configuration files that PostgreSQL needs to start.

```bash
sudo -u postgres /usr/lib/postgresql/{PG_VERSION}/bin/initdb \
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

## Phase 7 — Create a systemd Service File

On Ubuntu, the PGDG package uses `pg_ctlcluster` by default rather than a direct systemd service per data directory. For production setups with a custom `DATA_DIR`, create a dedicated systemd service file instead — this gives you direct `systemctl start/stop/status` control.

```bash
sudo tee /etc/systemd/system/postgresql-{PG_VERSION}-custom.service << 'EOF'
[Unit]
Description=PostgreSQL {PG_VERSION} Database Server (custom)
After=network.target

[Service]
Type=forking
User=postgres
Group=postgres
Environment=PGDATA={DATA_DIR}
ExecStart=/usr/lib/postgresql/{PG_VERSION}/bin/pg_ctl start -D {DATA_DIR} -l {LOG_DIR}/startup.log
ExecStop=/usr/lib/postgresql/{PG_VERSION}/bin/pg_ctl stop -D {DATA_DIR} -m fast
ExecReload=/usr/lib/postgresql/{PG_VERSION}/bin/pg_ctl reload -D {DATA_DIR}
TimeoutSec=300

[Install]
WantedBy=multi-user.target
EOF
```

Reload systemd:
```bash
sudo systemctl daemon-reload
```

---

## Phase 8 — Configure pg_hba.conf

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

**After editing pg_hba.conf while the service is running**, reload without restarting:
```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

---

## Phase 9 — Configure postgresql.conf

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

## Phase 10 — Enable and Start PostgreSQL

```bash
sudo systemctl enable postgresql-{PG_VERSION}-custom
sudo systemctl start postgresql-{PG_VERSION}-custom
sudo systemctl status postgresql-{PG_VERSION}-custom
```

Expected: `Active: active (running)`

If the service fails to start, check logs:
```bash
sudo journalctl -u postgresql-{PG_VERSION}-custom -n 50
sudo cat {LOG_DIR}/startup.log
```

---

## Phase 11 — Set postgres Superuser Password

```bash
sudo -u postgres psql
```

```sql
ALTER USER postgres WITH PASSWORD 'your_strong_superuser_password';
\q
```

---

## Phase 12 — Create Application Database and User

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

## Phase 13 — Verification

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

- [ ] No existing PostgreSQL packages (`dpkg -l | grep postgresql` returns empty before install)
- [ ] PGDG repo added and `apt update` completed
- [ ] PG installed and version verified
- [ ] Default cluster stopped, disabled, and dropped (`pg_dropcluster`)
- [ ] Custom data, log, archive directories created with correct ownership (`postgres:postgres`, `700`)
- [ ] AppArmor not blocking access (check `/var/log/syslog` if issues arise)
- [ ] `initdb` completed successfully
- [ ] Custom systemd service file created and daemon reloaded
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
Then apply: `sudo sysctl -p`

**`huge_pages = try`** means PostgreSQL will use huge pages if the OS has them configured, and silently fall back if not. For servers with `shared_buffers` above 4GB, configure huge pages at the OS level for better performance.

**Servers running other services** (e.g., MongoDB, Redis, application servers on the same machine): reduce `shared_buffers` to 15–20% of RAM and `effective_cache_size` to 30–40% of RAM to leave room for the other processes.

---

## Troubleshooting Reference

### PostgreSQL fails to start — check logs first

```bash
sudo journalctl -u postgresql-{PG_VERSION}-custom -n 50 --no-pager
sudo cat {LOG_DIR}/startup.log
```

---

### AppArmor blocking PostgreSQL access to custom directories

Check for denials:
```bash
sudo grep apparmor /var/log/syslog | grep postgres | tail -30
```

If you see `DENIED` entries for your custom paths, add those paths to the AppArmor profile as described in Phase 4, then reload:
```bash
sudo apparmor_parser -r /etc/apparmor.d/usr.lib.postgresql.{PG_VERSION}.bin.postgres
```

---

### Default cluster interfering after install

If `pg_lsclusters` shows a `main` cluster still running after `pg_dropcluster`:
```bash
sudo pg_ctlcluster {PG_VERSION} main stop
sudo pg_dropcluster {PG_VERSION} main
```

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

### idle_session_timeout disconnects psql during setup

PostgreSQL disconnects idle sessions after `idle_session_timeout` (set to 10 minutes in this config). If you are cut off mid-setup, psql will attempt to reconnect automatically (`Attempting reset: Succeeded`). After reconnection, check whether your last command completed before re-running it.

To check if a database was created:
```sql
\l {DB_NAME}
```

---

## Appendix A — Removing an Existing PostgreSQL Installation

Only needed if a previous version is present (`dpkg -l | grep postgresql` returns output).

> **Warning:** Verify there is no production data before removing anything.

```bash
# Check what databases exist before removing
sudo -u postgres /usr/lib/postgresql/{OLD_VERSION}/bin/psql -c "\l+"

# Stop and drop the existing cluster
sudo pg_ctlcluster {OLD_VERSION} main stop
sudo pg_dropcluster {OLD_VERSION} main

# Remove packages (--purge removes config files too)
sudo apt purge -y postgresql-{OLD_VERSION} postgresql-contrib-{OLD_VERSION} postgresql-client-{OLD_VERSION}
sudo apt autoremove -y

# Remove any leftover data (confirm no production data first)
sudo rm -rf /var/lib/postgresql/{OLD_VERSION}/
sudo rm -rf /etc/postgresql/{OLD_VERSION}/

# Remove the PGDG repo entry if switching repo versions
# (usually not needed — the same repo serves all PG versions)

# Verify clean removal
dpkg -l | grep postgresql
# Expected: no output
```
