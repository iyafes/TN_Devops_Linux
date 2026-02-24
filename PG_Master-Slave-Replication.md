# PostgreSQL Master-Slave Streaming Replication — Complete Step-by-Step Guide

## Overview

This guide walks through setting up a **PostgreSQL Master-Slave Streaming Replication** cluster across two servers. The goal is:

- **Master Server** — accepts both READ and WRITE operations
- **Slave Server** — accepts READ only; data is automatically synced from Master in real time
- Any data written to Master is immediately replicated to Slave via WAL streaming
- Slave cannot accept any write operations (INSERT, UPDATE, DELETE)

---

## What is a PostgreSQL Cluster?

In PostgreSQL, a **"cluster"** does not mean a group of machines. It refers to a **single PostgreSQL instance managing a single data directory**. When you install PostgreSQL, it automatically creates one cluster.

The data directory (the cluster) lives here on Ubuntu:

```
/var/lib/postgresql/16/main/
```

Inside this directory:

```
/var/lib/postgresql/16/main/
├── base/             ← actual database files
├── global/           ← cluster-wide data (roles, users)
├── pg_wal/           ← Write-Ahead Log files
├── postgresql.conf   ← main configuration file
├── pg_hba.conf       ← authentication/access rules
├── PG_VERSION        ← version number
└── postmaster.pid    ← running process ID
```

> **One Cluster = One Data Directory = One PostgreSQL Instance**

---

## What is Streaming Replication?

Every change made in PostgreSQL (INSERT, UPDATE, DELETE) is first written to a **WAL (Write-Ahead Log)** file before being applied to actual data. This is a safety mechanism to prevent data loss.

In **Streaming Replication**:

1. Master writes changes to WAL
2. WAL is **streamed in real time** to the Slave
3. Slave **replays** the WAL (applies the same changes to its own data)
4. Slave stays in sync with Master continuously

```
Client writes data
       ↓
  Master Server
  (WAL is created)
       ↓
  WAL streamed ──────────────────► Slave Server
                                   (WAL is replayed)
                                   (Data stays in sync)
```

---

## Architecture Diagram

```
┌─────────────────────────────────┐         ┌──────────────────────────────────┐
│         MASTER SERVER           │         │          SLAVE SERVER            │
│         (VM1)                   │         │          (VM2)                   │
│                                 │         │                                  │
│  ┌──────────────────────────┐   │         │  ┌───────────────────────────┐   │
│  │ PostgreSQL Instance      │   │  WAL    │  │ PostgreSQL Instance       │   │
│  │                          │───┼─Stream──┼─►│ (Standby / Recovery Mode) │   │
│  │ READ  ✅                 │   │         │  │                           │   │
│  │ WRITE ✅                 │   │         │  │ READ  ✅                  │   │
│  │                          │   │         │  │ WRITE ❌ (ERROR)          │   │
│  └──────────────────────────┘   │         │  └───────────────────────────┘   │
│                                 │         │                                  │
│  IP: 172.16.93.132              │         │  IP: 172.16.93.133               │
│  Port: 5432                     │         │  Port: 5432                      │
└─────────────────────────────────┘         └──────────────────────────────────┘
```

---

## Prerequisites

- Two Ubuntu Server VMs (VM1 = Master, VM2 = Slave)
- PostgreSQL installed on **both** VMs
- Both VMs on the same network (can ping each other)
- SSH or terminal access to both VMs

Check PostgreSQL version on both VMs:

```bash
psql --version
```

> Note the version number (e.g., `16`). You will use it in directory paths throughout this guide.

---

## Environment Setup

| Role   | VM  | IP Address      |
|--------|-----|-----------------|
| Master | VM1 | 172.16.93.132   |
| Slave  | VM2 | 172.16.93.133   |

To find your IP on each VM:

```bash
ip a
```

Look under `ens33` (or similar) for the `inet` value. **Do not use `lo` (127.0.0.1)** — that is the loopback address and only works locally. You need the actual network interface IP.

---

---

# MASTER SERVER Configuration

> All steps in this section are performed on **VM1 (Master)**

---

## Step 1 — Create Replication User

Switch to the PostgreSQL system user:

```bash
sudo -i -u postgres
```

> **`sudo -i -u postgres`** — runs a login shell as the `postgres` OS user, which is the owner of all PostgreSQL data files. PostgreSQL requires this for administrative tasks.

Open the PostgreSQL interactive terminal:

```bash
psql
```

> **`psql`** — the PostgreSQL command-line client. It connects to the local PostgreSQL instance as the current OS user (`postgres`).

Create a dedicated replication user:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replica123';
```

> - **`CREATE ROLE`** — creates a new database role (user)
> - **`WITH REPLICATION`** — grants permission to initiate WAL streaming (required for replication)
> - **`LOGIN`** — allows this role to log in (connect to the database)
> - **`PASSWORD 'replica123'`** — sets the password for authentication

Verify the user was created:

```sql
\du
```

Expected output:

```
 Role name  |         Attributes
------------+------------------------------------
 postgres   | Superuser, Create role, Create DB
 replicator | Replication
```

Exit psql:

```sql
\q
```

Exit the postgres user:

```bash
exit
```

---

## Step 2 — Configure postgresql.conf

This is the **main PostgreSQL configuration file**. It controls how the PostgreSQL instance behaves.

Open the file:

```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```

> Replace `16` with your actual PostgreSQL version number.

> **`nano`** — a terminal-based text editor. Use `Ctrl+W` to search for text inside the file.

Make the following changes (find each line, uncomment it by removing `#`, and set the value):

---

### `listen_addresses`

Find:
```
#listen_addresses = 'localhost'
```

Change to:
```
listen_addresses = '*'
```

> **`listen_addresses`** — defines which network interfaces PostgreSQL listens on for incoming connections.
> - `'localhost'` (default) — only accepts connections from the same machine
> - `'*'` — accepts connections from any IP address, allowing the Slave to connect

---

### `wal_level`

Find:
```
#wal_level = replica
```

Change to:
```
wal_level = replica
```

> **`wal_level`** — controls how much information is written to the WAL (Write-Ahead Log).
> - `minimal` — minimum info, not enough for replication
> - `replica` — includes all info needed for streaming replication ✅
> - `logical` — needed for logical replication (more detailed, not needed here)

---

### `max_wal_senders`

Find:
```
#max_wal_senders = 10
```

Change to:
```
max_wal_senders = 5
```

> **`max_wal_senders`** — the maximum number of simultaneous WAL sender processes. Each Slave connected to this Master uses one WAL sender process. Setting `5` allows up to 5 Slave servers.

---

### `wal_keep_size`

Find:
```
#wal_keep_size = 0
```

Change to:
```
wal_keep_size = 64
```

> **`wal_keep_size`** — the minimum amount of WAL files (in MB) to keep on disk. If a Slave falls behind, it can catch up using these retained WAL files instead of requiring a full re-sync. `64` means keep at least 64 MB of WAL history.

Save and exit: `Ctrl+X` → `Y` → `Enter`

---

## Step 3 — Configure pg_hba.conf

This file controls **who can connect to PostgreSQL, from where, and how they authenticate**.

Open the file:

```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

Go to the very bottom of the file (`Ctrl+End`) and add this line:

```
host    replication     replicator      172.16.93.133/32        md5
```

> Each field explained:
>
> | Field | Value | Meaning |
> |-------|-------|---------|
> | Connection type | `host` | TCP/IP connection (network) |
> | Database | `replication` | Special keyword for replication connections |
> | User | `replicator` | Only this username is allowed |
> | IP Address | `172.16.93.133/32` | Only from this exact Slave IP |
> | Auth method | `md5` | Password-based authentication |
>
> **Why `/32` and not `/24`?**
> - `/24` means allow any IP in the range `172.16.93.0` to `172.16.93.255` (the entire subnet)
> - `/32` means allow **only this single exact IP**
> - `/32` is more secure — only your Slave can connect, not any machine on the network

Save and exit: `Ctrl+X` → `Y` → `Enter`

---

## Step 4 — Open Firewall Port

Allow incoming connections on PostgreSQL's default port:

```bash
sudo ufw allow 5432/tcp
sudo ufw reload
```

> - **`ufw`** — Uncomplicated Firewall, Ubuntu's firewall manager
> - **`5432`** — the default port PostgreSQL listens on
> - **`/tcp`** — TCP protocol (PostgreSQL uses TCP)
> - **`reload`** — applies the new rule without restarting the firewall

> If `ufw` is inactive/disabled, you can skip this step. Run `sudo ufw status` to check.

---

## Step 5 — Restart Master PostgreSQL

Apply all configuration changes by restarting the service:

```bash
sudo systemctl restart postgresql
```

> **`systemctl restart postgresql`** — stops and starts the PostgreSQL service, loading all changed configuration files.

Check cluster status:

```bash
sudo pg_lsclusters
```

> **`pg_lsclusters`** — Ubuntu-specific command that lists all PostgreSQL clusters and their status.

Expected output:

```
Ver  Cluster  Port  Status  Owner     Data directory
16   main     5432  online  postgres  /var/lib/postgresql/16/main
```

> `online` means the cluster is running correctly. On Ubuntu, `systemctl status postgresql` may show `active (exited)` — this is **normal behavior** for the wrapper service. What matters is that `pg_lsclusters` shows `online`.

---

## Step 6 — Verify Master Configuration

Confirm your settings are active:

```bash
sudo -i -u postgres
psql -c "SHOW wal_level;"
```

Expected output:
```
 wal_level
-----------
 replica
```

```bash
psql -c "SHOW listen_addresses;"
```

Expected output:
```
 listen_addresses
------------------
 *
```

```bash
psql -c "SHOW max_wal_senders;"
```

Expected output:
```
 max_wal_senders
-----------------
 5
```

All correct? ✅ Master configuration is complete. Exit:

```bash
exit
```

---

---

# SLAVE SERVER Configuration

> All steps in this section are performed on **VM2 (Slave)**

---

## Step 7 — Stop PostgreSQL on Slave

```bash
sudo systemctl stop postgresql
```

> **Why stop it?** The Slave's data directory must be completely replaced with a copy from the Master. This cannot be done while PostgreSQL is running because it holds locks on the files.

Verify it stopped:

```bash
sudo pg_lsclusters
```

Expected:
```
Ver  Cluster  Port  Status  Owner
16   main     5432  down    postgres
```

---

## Step 8 — Clear Slave Data Directory

Switch to the postgres user and delete all existing data:

```bash
sudo -i -u postgres
rm -rf /var/lib/postgresql/16/main/*
```

> - **`rm -rf`** — forcefully removes files and directories recursively
> - **`/var/lib/postgresql/16/main/*`** — the `*` deletes everything **inside** the directory but keeps the directory itself
>
> **Why delete?** When PostgreSQL was installed, it created an empty cluster here. We must remove it completely before copying the Master's data. If we don't, the replication setup will conflict with existing files.

Verify the directory is empty:

```bash
ls /var/lib/postgresql/16/main/
```

No output = empty = correct ✅

---

## Step 9 — Take Base Backup from Master

Still as the `postgres` user, run:

```bash
pg_basebackup -h 172.16.93.132 -U replicator -D /var/lib/postgresql/16/main/ -P -Xs -R
```

> **`pg_basebackup`** — PostgreSQL's built-in tool for creating a binary copy of a running cluster. It is specifically designed for setting up replication standbys.

> Each flag explained:
>
> | Flag | Value | Meaning |
> |------|-------|---------|
> | `-h` | `172.16.93.132` | Connect to this host (Master's IP) |
> | `-U` | `replicator` | Use this username to connect |
> | `-D` | `/var/lib/.../main/` | Write the backup to this directory (Slave's data dir) |
> | `-P` | *(none)* | Show progress percentage during backup |
> | `-X s` | `s` = stream | Stream WAL files during the backup (so no changes are missed) |
> | `-R` | *(none)* | **Automatically** create `standby.signal` and write connection info to `postgresql.auto.conf` |

When prompted for a password:

```
Password:
```

Type `replica123` and press Enter.

You will see a progress bar:

```
30264/30264 kB (100%), 1/1 tablespace
```

When it reaches 100%, the backup is complete ✅

> **What `-R` does (very important):**
> Without `-R`, you would have to manually create two things:
> 1. An empty file called `standby.signal` in the data directory
> 2. A `primary_conninfo` entry in `postgresql.auto.conf` telling the Slave how to reach the Master
>
> With `-R`, both are done automatically.

Exit postgres user:

```bash
exit
```

---

## Step 10 — Verify Backup Files

Check that critical replication files were created:

```bash
sudo -i -u postgres
```

Check `standby.signal` exists:

```bash
ls /var/lib/postgresql/16/main/standby.signal
```

> **`standby.signal`** — an empty file that tells PostgreSQL "you are a Slave, run in read-only standby mode." PostgreSQL checks for this file at startup. If it exists → Slave mode. If it doesn't exist → normal (Master) mode.

Check connection info was written:

```bash
cat /var/lib/postgresql/16/main/postgresql.auto.conf
```

Expected output:

```
# Do not edit this file manually!
primary_conninfo = 'user=replicator password=replica123 host=172.16.93.132 port=5432 ...'
```

> **`primary_conninfo`** — the connection string that tells the Slave where the Master is and how to connect to it for WAL streaming. This is automatically written by `pg_basebackup -R`.

Both files present ✅

```bash
exit
```

---

## Step 11 — Start PostgreSQL on Slave

```bash
sudo systemctl start postgresql
```

Check status:

```bash
sudo pg_lsclusters
```

Expected:
```
Ver  Cluster  Port  Status  Owner
16   main     5432  online  postgres
```

`online` = Slave is running ✅

---

---

# Verify Replication is Working

---

## Step 12 — Check Slave Connection on Master

Go back to **VM1 (Master)** and run:

```bash
sudo -i -u postgres
psql -c "SELECT * FROM pg_stat_replication;"
```

> **`pg_stat_replication`** — a PostgreSQL system view that shows information about all Slave servers currently connected and streaming WAL from this Master.

Expected output:

```
 pid  | usename    | client_addr    | state     | sync_state
------+------------+----------------+-----------+------------
 1234 | replicator | 172.16.93.133  | streaming | async
```

> Key fields:
> - **`client_addr`** — the Slave's IP address
> - **`state`** — `streaming` means WAL is actively being sent to the Slave ✅
> - **`sync_state`** — `async` means asynchronous replication (Master doesn't wait for Slave confirmation)

If this table is empty, see the [Troubleshooting](#troubleshooting) section.

```bash
exit
```

---

## Step 13 — Check Slave is in Standby Mode

On **VM2 (Slave)**:

```bash
sudo -i -u postgres
psql -c "SELECT pg_is_in_recovery();"
```

> **`pg_is_in_recovery()`** — a PostgreSQL function that returns `true` if the instance is running in recovery/standby mode (i.e., it is a Slave), and `false` if it is a normal primary (Master).

Expected output:

```
 pg_is_in_recovery
-------------------
 t
```

> `t` = `true` = this server is a Slave in standby mode ✅
> `f` = `false` = something went wrong, this server thinks it is a Master

```bash
exit
```

---

---

# Testing Read/Write & Real-Time Sync

---

## Step 14 — Write Data on Master

On **VM1 (Master)**:

```bash
sudo -i -u postgres
psql
```

Create a new database:

```sql
CREATE DATABASE testdb;
```

Connect to it:

```sql
\c testdb
```

> **`\c`** — psql command to switch connection to a different database.

Create a table:

```sql
CREATE TABLE employees (
    id         SERIAL PRIMARY KEY,
    name       VARCHAR(100),
    department VARCHAR(100)
);
```

> - **`SERIAL`** — auto-incrementing integer (1, 2, 3...) — no need to insert ID manually
> - **`PRIMARY KEY`** — uniquely identifies each row
> - **`VARCHAR(100)`** — variable-length text up to 100 characters

Insert data:

```sql
INSERT INTO employees (name, department) VALUES
    ('Alice', 'Engineering'),
    ('Bob', 'Marketing'),
    ('Charlie', 'Finance');
```

Verify the data:

```sql
SELECT * FROM employees;
```

Expected output:

```
 id |  name   | department
----+---------+-------------
  1 | Alice   | Engineering
  2 | Bob     | Marketing
  3 | Charlie | Finance
(3 rows)
```

Data exists on Master ✅

---

## Step 15 — Read Data from Slave

Open **VM2 (Slave)** terminal (keep Master terminal open too):

```bash
sudo -i -u postgres
psql -d testdb
```

> **`-d testdb`** — connect directly to the `testdb` database.

Run the same SELECT query:

```sql
SELECT * FROM employees;
```

Expected output:

```
 id |  name   | department
----+---------+-------------
  1 | Alice   | Engineering
  2 | Bob     | Marketing
  3 | Charlie | Finance
(3 rows)
```

> **The data written on Master is readable on Slave!** ✅
> This confirms WAL streaming replication is working. The Slave received and replayed the WAL from Master, resulting in identical data.

---

## Step 16 — Prove Slave is Read-Only

Still on **VM2 (Slave)**, try to write data:

```sql
INSERT INTO employees (name, department) VALUES ('Hacker', 'Unknown');
```

Expected output:

```
ERROR:  cannot execute INSERT in a read-only transaction
```

> This error proves the Slave is correctly operating in **read-only mode**. The `standby.signal` file and WAL recovery mode prevent any write operations. This is the expected and correct behavior for a replication Slave.

Try UPDATE and DELETE — they also fail:

```sql
UPDATE employees SET name = 'X' WHERE id = 1;
-- ERROR: cannot execute UPDATE in a read-only transaction

DELETE FROM employees WHERE id = 1;
-- ERROR: cannot execute DELETE in a read-only transaction
```

Slave is truly read-only ✅

---

## Step 17 — Real-Time Sync Test

This test shows replication happening **live** as you type.

Open **two terminals side by side** — one for Master, one for Slave.

**On Slave terminal** — run this query and note only 3 rows exist:

```sql
SELECT * FROM employees;
-- Shows 3 rows: Alice, Bob, Charlie
```

**On Master terminal** — insert a new row:

```sql
INSERT INTO employees (name, department) VALUES ('David', 'HR');
```

**On Slave terminal** — immediately run the SELECT again:

```sql
SELECT * FROM employees;
```

Expected output:

```
 id |  name   | department
----+---------+-------------
  1 | Alice   | Engineering
  2 | Bob     | Marketing
  3 | Charlie | Finance
  4 | David   | HR
```

> **David appeared on Slave almost instantly!** ✅ This is real-time WAL streaming replication in action. The moment Master commits the INSERT, the WAL record is streamed to Slave and replayed.

---

---

# How It All Works — Full Flow Summary

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  1. Client sends INSERT to Master                                       │
│                   ↓                                                     │
│  2. Master writes change to WAL buffer (RAM)                           │
│                   ↓                                                     │
│  3. WAL is flushed to disk (pg_wal/ directory)                         │
│                   ↓                                                     │
│  4. WAL Sender process streams WAL record to Slave ──────────────────► │
│                                                                         │
│                                              ┌────────────────────────┐│
│                                              │ 5. WAL Receiver gets   ││
│                                              │    the WAL record      ││
│                                              │         ↓              ││
│                                              │ 6. WAL is replayed     ││
│                                              │    (same INSERT runs)  ││
│                                              │         ↓              ││
│                                              │ 7. Slave data is now   ││
│                                              │    identical to Master ││
│                                              └────────────────────────┘│
│                                                                         │
│  8. Client SELECT on Slave → sees the new data ✅                      │
│  9. Client INSERT on Slave → ERROR: read-only ❌                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

