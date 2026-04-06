# MongoDB 8.0 Installation, Configuration & Setup Guide
## Target: Rocky Linux 9 — Binary Install (No Docker)

---

## Overview

This document covers the complete process of installing MongoDB 8.0 on a fresh Rocky Linux 9 server with custom data and log directories on separate NVMe disks. It includes all issues encountered and their fixes so the process can be completed in one attempt.

**Server Specs (this setup was tested on):**
- OS: Rocky Linux 9
- RAM: 8GB
- CPU: 4 cores
- Disk layout:
  - `/` (root): 30GB NVMe (OS only)
  - `/data`: 50GB NVMe (database data)
  - `/logs`: 50GB NVMe (database logs)

**Note:** PostgreSQL 17 is also running on this server. MongoDB memory configuration accounts for shared RAM.

---

## Pre-requisites

- SELinux is **Enforcing** — do not disable it. Apply correct contexts instead (covered in detail below).
- The `/data` and `/logs` parent directories are already mounted (separate NVMe disks).
- Parent directory SELinux contexts must be fixed before MongoDB-specific contexts will work (see Phase 4).

---

## Phase 1 — Add MongoDB 8.0 Repository

> **Important:** The senior DBA's reference document used MongoDB 5.0 repository. This is outdated. Use the 8.0 repository below.
> The `$releasever` variable may not resolve correctly on Rocky Linux. Use hardcoded `9` instead.

```bash
sudo tee /etc/yum.repos.d/mongodb-org-8.0.repo << 'EOF'
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-8.0.asc
EOF
```

---

## Phase 2 — Install MongoDB

```bash
sudo dnf install -y mongodb-org
```

Verify installation:
```bash
mongod --version
# Expected: db version v8.0.x
```

---

## Phase 3 — Create Custom Directories

Do **not** use default MongoDB paths (`/var/lib/mongo`, `/var/log/mongodb`). Data goes to `/data`, logs go to `/logs`.

```bash
# MongoDB data directory
sudo mkdir -p /data/mongodb
sudo chown -R mongod:mongod /data/mongodb
sudo chmod 755 /data/mongodb

# MongoDB log directory
sudo mkdir -p /logs/mongodb
sudo chown -R mongod:mongod /logs/mongodb
sudo chmod 755 /logs/mongodb
```

Verify:
```bash
ls -la /data/mongodb/
ls -la /logs/mongodb/
# Expected: owned by mongod:mongod
```

---

## Phase 4 — SELinux Context (Critical on Rocky Linux)

SELinux is Enforcing on Rocky Linux. Without proper context labels, mongod cannot write to custom directories. **Do not disable SELinux.**

### The Core Problem

The `/data` and `/logs` directories are on separate NVMe disks mounted at boot. Their default SELinux context is `unlabeled_t` — which means SELinux has no policy for them and blocks all writes. This causes MongoDB to fail with `Permission denied` even if file ownership is correct.

### Fix Must Be Done in Two Steps

**Step 1: Fix parent directory contexts first.**

If parent contexts are `unlabeled_t`, applying MongoDB-specific contexts to subdirectories will not work — the parent overrides them during `restorecon`.

```bash
sudo semanage fcontext -a -t var_t "/data(/.*)?"
sudo restorecon -Rv /data

sudo semanage fcontext -a -t var_log_t "/logs(/.*)?"
sudo restorecon -Rv /logs
```

**Step 2: Apply MongoDB-specific contexts.**

```bash
sudo semanage fcontext -a -t mongod_var_lib_t "/data/mongodb(/.*)?"
sudo semanage fcontext -a -t mongod_log_t "/logs/mongodb(/.*)?"
```

Then use `chcon` to apply immediately (because `restorecon` alone may not override parent context):

```bash
sudo chcon -Rv -t mongod_var_lib_t /data/mongodb
sudo chcon -Rv -t mongod_log_t /logs/mongodb
```

> **Why both `semanage` and `chcon`?**
> - `semanage fcontext` makes the label persistent across reboots.
> - `chcon` applies it immediately to already-existing files/directories.
> - Using only `chcon` means contexts reset after reboot.
> - Using only `semanage` + `restorecon` may not override parent unlabeled_t correctly.
> - Both together guarantee correct and persistent labeling.

Verify:
```bash
ls -laZ /data/mongodb/
# Expected: mongod_var_lib_t

ls -laZ /logs/mongodb/
# Expected: mongod_log_t
```

---

## Phase 5 — Configure MongoDB

> **Important notes about the configuration:**
>
> - `storage.journal.enabled` has been **removed in MongoDB 8.0**. Do not include it — mongod will refuse to start with `Unrecognized option` error.
> - `cacheSizeGB` is set to `1.5` because PostgreSQL is also running on this server and takes 2GB for shared_buffers. MongoDB's default would claim 50% of RAM (4GB) which would conflict with PostgreSQL.
> - `slowOpThresholdMs: 3000` matches PostgreSQL's `log_min_duration_statement = 3000` for consistency — both databases log operations slower than 3 seconds.

```bash
sudo tee /etc/mongod.conf << 'EOF'
systemLog:
  destination: file
  logAppend: true
  path: /logs/mongodb/mongod.log
  logRotate: reopen

storage:
  dbPath: /data/mongodb
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1.5

net:
  port: 27017
  bindIp: 0.0.0.0

processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  pidFilePath: /var/run/mongodb/mongod.pid

security:
  authorization: enabled

operationProfiling:
  slowOpThresholdMs: 3000
  mode: slowOp
EOF
```

### Configuration Parameter Explanation

| Parameter | Value | Reason |
|-----------|-------|--------|
| `systemLog.path` | `/logs/mongodb/mongod.log` | Custom log directory on dedicated NVMe disk |
| `systemLog.logAppend` | true | Append to existing log file on restart instead of overwriting |
| `systemLog.logRotate` | reopen | Allows log rotation via `kill -USR1` or `mongod` signal |
| `storage.dbPath` | `/data/mongodb` | Custom data directory on dedicated NVMe disk |
| `wiredTiger.cacheSizeGB` | 1.5 | Conservative — PostgreSQL is also running. Default (50% RAM = 4GB) would starve PostgreSQL. |
| `net.bindIp` | 0.0.0.0 | Accept connections from all IPs. Access control is handled by `authorization: enabled`. |
| `net.port` | 27017 | Default MongoDB port |
| `security.authorization` | enabled | Requires authentication. Without this, anyone who can reach port 27017 can access all data. |
| `operationProfiling.mode` | slowOp | Only log operations slower than the threshold |
| `operationProfiling.slowOpThresholdMs` | 3000 | Log operations taking more than 3 seconds |

---

## Phase 6 — Enable and Start MongoDB

```bash
sudo systemctl enable mongod
sudo systemctl start mongod
sudo systemctl status mongod
```

Expected: `Active: active (running)`

If it fails, check logs:
```bash
sudo tail -50 /logs/mongodb/mongod.log
# or if log file not created yet:
sudo journalctl -u mongod --no-pager -n 50
```

---

## Phase 7 — Create Admin User

MongoDB starts with `authorization: enabled` but no users exist yet. You must temporarily disable authorization to create the first admin user.

### 7.1 Temporarily Disable Authorization

```bash
sudo systemctl stop mongod
sudo sed -i 's/  authorization: enabled/  authorization: disabled/' /etc/mongod.conf
sudo systemctl start mongod
```

### 7.2 Connect and Create Admin User

```bash
mongosh
```

Inside mongosh — **enter as a single line to avoid syntax errors:**

```javascript
use admin
db.createUser({ user: "admin", pwd: "your_strong_password_here", roles: [{ role: "root", db: "admin" }] })
exit
```

> **Common mistake:** Multi-line paste in mongosh can cause `SyntaxError: Unexpected token`. Always paste the `db.createUser(...)` call as a single line.

### 7.3 Re-enable Authorization

```bash
sudo systemctl stop mongod
sudo sed -i 's/  authorization: disabled/  authorization: enabled/' /etc/mongod.conf
sudo systemctl start mongod
```

---

## Phase 8 — Verification

### 8.1 Verify Service is Running

```bash
sudo systemctl status mongod
# Expected: Active: active (running)
```

### 8.2 Verify Authentication Works

```bash
mongosh --authenticationDatabase admin -u admin -p
# Enter password when prompted
```

Inside mongosh:
```javascript
show dbs
// Expected: admin, config, local
exit
```

### 8.3 Verify Log File

```bash
sudo tail -20 /logs/mongodb/mongod.log
# Expected: WiredTiger checkpoint messages appearing every minute — this is normal healthy behavior
```

Example of normal log output:
```
"msg":"WiredTiger message","attr":{"message":{"msg":"saving checkpoint snapshot..."}}
```

These checkpoint messages confirm WiredTiger storage engine is running correctly.

### 8.4 Verify Data Directory

```bash
ls -la /data/mongodb/
# Expected: WiredTiger.*, mongod.lock, collection files etc.
```

---

## Troubleshooting Reference

### mongod fails — "Unrecognized option: storage.journal.enabled"

Cause: `storage.journal.enabled` was removed in MongoDB 8.0. It was valid in older versions (5.0, 6.0).

Fix: Remove the `journal` block from `/etc/mongod.conf`. The storage section should look like:
```yaml
storage:
  dbPath: /data/mongodb
  wiredTiger:
    engineConfig:
      cacheSizeGB: 1.5
```

### mongod fails — "Permission denied" on mongod.lock or log file

Cause: SELinux blocking access to custom directories with wrong context.

Diagnosis:
```bash
ls -laZ /data/mongodb/
ls -laZ /logs/mongodb/
# Look for unlabeled_t or wrong context type
```

Fix — follow this exact order:
```bash
# Step 1: Fix parent directories
sudo semanage fcontext -a -t var_t "/data(/.*)?"
sudo restorecon -Rv /data
sudo semanage fcontext -a -t var_log_t "/logs(/.*)?"
sudo restorecon -Rv /logs

# Step 2: Apply MongoDB-specific contexts
sudo semanage fcontext -a -t mongod_var_lib_t "/data/mongodb(/.*)?"
sudo semanage fcontext -a -t mongod_log_t "/logs/mongodb(/.*)?"
sudo chcon -Rv -t mongod_var_lib_t /data/mongodb
sudo chcon -Rv -t mongod_log_t /logs/mongodb

# Step 3: Start mongod
sudo systemctl start mongod
```

> **Why this order matters:** If parent directory is `unlabeled_t`, `restorecon` on subdirectory inherits the parent context and overwrites your MongoDB-specific labels. Always fix parent first.

### Authentication failed after creating user

Cause: User was created while authorization was disabled, but was not actually saved (mongod may have been restarted between creation and verification).

Fix:
```bash
# Disable auth again
sudo systemctl stop mongod
sudo sed -i 's/  authorization: enabled/  authorization: disabled/' /etc/mongod.conf
sudo systemctl start mongod

# Connect and check
mongosh
```
```javascript
use admin
show users
// If empty, create user again
db.createUser({ user: "admin", pwd: "your_password", roles: [{ role: "root", db: "admin" }] })
exit
```
```bash
# Re-enable auth
sudo systemctl stop mongod
sudo sed -i 's/  authorization: disabled/  authorization: enabled/' /etc/mongod.conf
sudo systemctl start mongod
```

### SyntaxError in mongosh when pasting createUser command

Cause: Multi-line paste in mongosh terminal interprets line breaks as submission.

Fix: Paste the entire `db.createUser(...)` call as a single line:
```javascript
db.createUser({ user: "admin", pwd: "your_password", roles: [{ role: "root", db: "admin" }] })
```

---

## Creating Application Database Users

After admin user is created, create specific users for each application database:

```bash
mongosh --authenticationDatabase admin -u admin -p
```

```javascript
// Create application database and user
use your_app_database_name
db.createUser({
  user: "app_user",
  pwd: "app_user_strong_password",
  roles: [{ role: "readWrite", db: "your_app_database_name" }]
})
exit
```

Use `readWrite` role for application users — never give application users the `root` role.

---

## Post-Setup Checklist

- [ ] MongoDB 8.0 repository added (not 5.0 or older)
- [ ] `mongod --version` shows 8.0.x
- [ ] Custom directories created with correct ownership (mongod:mongod)
- [ ] SELinux contexts applied — parent directories fixed first, then MongoDB-specific
- [ ] mongod.conf created — no `storage.journal.enabled` (removed in 8.0)
- [ ] `authorization: enabled` in mongod.conf
- [ ] `cacheSizeGB` set appropriately for shared server (1.5GB when running alongside PostgreSQL)
- [ ] Service enabled and running
- [ ] Admin user created and authentication verified
- [ ] Log file appearing in `/logs/mongodb/`
- [ ] Data files appearing in `/data/mongodb/`
- [ ] Application database users created with `readWrite` role only
