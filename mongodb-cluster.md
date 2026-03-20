# MongoDB Replica Set Setup Guide

A complete guide to setting up a 3-node MongoDB Replica Set (PRIMARY + 2 SECONDARY) on Ubuntu 24.04 LTS with password authentication, keyfile-based inter-node security, and systemd auto-start.

---

## Environment

| | |
|---|---|
| OS | Ubuntu 24.04 LTS |
| Virtualization | VMware (local machine) |
| MongoDB Version | 8.0.x |
| Nodes | 3 VMs |

| VM | IP Address | Port |
|----|-----------|------|
| mongo-vm1 | 172.16.93.149 | 27017 |
| mongo-vm2 | 172.16.93.150 | 27017 |
| mongo-vm3 | 172.16.93.151 | 27017 |

---

## Architecture

```mermaid
graph TD
    Client["Client Application"]

    subgraph rs0["Replica Set: rs0"]
        VM1["mongo-vm1\n172.16.93.149:27017\nPRIMARY"]
        VM2["mongo-vm2\n172.16.93.150:27017\nSECONDARY"]
        VM3["mongo-vm3\n172.16.93.151:27017\nSECONDARY"]
    end

    Client -->|"reads + writes"| VM1
    VM1 -->|"replicates oplog"| VM2
    VM1 -->|"replicates oplog"| VM3
```

> PRIMARY is elected — any node can become PRIMARY after a failover event.

---

## Replication Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant P as PRIMARY
    participant S1 as SECONDARY-1
    participant S2 as SECONDARY-2

    C->>P: Write operation
    P->>P: Write to oplog
    P-->>C: Acknowledge write
    P->>S1: Replicate oplog entry
    P->>S2: Replicate oplog entry
    S1->>S1: Apply operation
    S2->>S2: Apply operation
```

---

## Failover Flow

```mermaid
sequenceDiagram
    participant P as PRIMARY (VM1)
    participant S1 as SECONDARY (VM2)
    participant S2 as SECONDARY (VM3)

    P->>S1: Heartbeat
    P->>S2: Heartbeat
    Note over P: PRIMARY goes down
    S1->>S2: No heartbeat from PRIMARY
    S2->>S1: No heartbeat from PRIMARY
    S1->>S2: Start election
    S2-->>S1: Vote
    Note over S1: Wins election, becomes new PRIMARY
    Note over P: VM1 restarts, rejoins as SECONDARY
```

---

## Key Concepts Before You Begin

**Replica Set** — A group of MongoDB instances that maintain the same dataset. One node is PRIMARY (handles all writes), the rest are SECONDARY (replicate from PRIMARY). If PRIMARY goes down, the remaining nodes automatically elect a new one. A minimum of 3 nodes is recommended so that a majority quorum (2 out of 3) can always be reached for elections.

**oplog (operations log)** — A special capped collection that exists on every Replica Set member. Every write operation on the PRIMARY is recorded here. SECONDARY nodes continuously tail the PRIMARY's oplog and replay those operations on themselves to stay in sync. If a SECONDARY falls behind, it catches up by replaying missed oplog entries on restart.

**Election and Quorum** — When the PRIMARY becomes unreachable, the remaining nodes hold an election. A node needs votes from a majority of the total members to win. With 3 nodes, that means 2 votes are needed. This is why 3 nodes is the minimum useful Replica Set — a 2-node set cannot elect a new PRIMARY if one goes down, because neither node alone has majority.

**Keyfile** — A shared secret file used for internal authentication between Replica Set nodes. Every node must have the exact same keyfile. This is separate from user password authentication — keyfile handles node-to-node trust, password handles client-to-server access.

**`authorization: enabled`** — Enforces username and password authentication for all client connections. Without this, anyone who can reach port 27017 can access all data with no credentials.

**Localhost Exception** — When MongoDB starts for the very first time with `authorization: enabled` but no users exist yet, it permits exactly one unauthenticated connection from localhost. This window is used to create the first admin user. It closes permanently after the first user is created and cannot be reopened without disabling auth and restarting.

**Read Preference** — By default, clients only read from the PRIMARY. SECONDARY nodes do not serve reads unless the client explicitly sets a read preference such as `secondaryPreferred`. This default prevents applications from accidentally reading stale data due to replication lag.

---

## Step 1 — System Preparation

> Run all commands in this step on **all 3 VMs** unless stated otherwise.

### 1.1 — Update the system

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 — Add hostname entries to `/etc/hosts`

Replica Set nodes need to resolve each other by name or IP. Adding entries to `/etc/hosts` ensures resolution works without depending on external DNS.

```bash
sudo tee -a /etc/hosts <<EOF

# MongoDB Replica Set
172.16.93.149   mongo-vm1
172.16.93.150   mongo-vm2
172.16.93.151   mongo-vm3
EOF
```

`tee -a` appends to the file without overwriting existing content, and also prints to the terminal so you can confirm what was written.

`<<EOF ... EOF` is a heredoc — a way to pass multi-line text as stdin to a command without creating a temporary file.

Verify:

```bash
cat /etc/hosts | grep mongo
```

All three lines should appear.

### 1.3 — Open port 27017 in the UFW firewall

MongoDB's default port is 27017. Replica Set nodes communicate over this port, so it must be open on all VMs.

```bash
sudo ufw allow 27017/tcp
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

> **Important:** Always run `sudo ufw allow OpenSSH` **before** `sudo ufw enable`. Enabling UFW without allowing SSH first will block your SSH session and lock you out of the VM.

Expected output:

```
Status: active

To                         Action      From
--                         ------      ----
27017/tcp                  ALLOW       Anywhere
OpenSSH                    ALLOW       Anywhere
```

### 1.4 — Verify network connectivity between VMs

From VM1:

```bash
ping -c 3 mongo-vm2
ping -c 3 mongo-vm3
```

Repeat from VM2 and VM3 toward the other two nodes. All pings must succeed before proceeding. If any ping fails, check the network adapter settings in VMware and confirm all VMs are on the same virtual network.

---

## Step 2 — MongoDB Installation

> Run all commands in this step on **all 3 VMs**.

Ubuntu's default APT repositories do not include MongoDB. MongoDB's official repository must be added manually.

### 2.1 — Import the MongoDB GPG key

APT verifies downloaded packages against a cryptographic signing key to ensure they are legitimate and have not been tampered with. This step imports MongoDB's official public key.

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor
```

The `\` at the end of the first line is a line continuation character — this is a single command split across two lines for readability. You can paste both lines together.

`--dearmor` converts the key from ASCII-armored text format (the `.asc` file) to binary format (`.gpg`) that APT understands.

### 2.2 — Add the MongoDB APT repository

```bash
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] \
https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | \
sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
```

`signed-by=` tells APT to only accept packages from this repository if they are signed by that specific key, preventing installation of tampered packages.

`noble` is the codename for Ubuntu 24.04 LTS. If you are on a different Ubuntu version, this value must change accordingly.

### 2.3 — Install MongoDB

```bash
sudo apt update
sudo apt install -y mongodb-org
```

`mongodb-org` is a meta-package that installs:

| Package | Purpose |
|---------|---------|
| `mongod` | The database server process |
| `mongosh` | The MongoDB shell for running queries and admin commands |
| `mongos` | The sharding query router (not used in this setup) |
| `mongodb-org-tools` | Import/export utilities (`mongodump`, `mongorestore`, etc.) |

### 2.4 — Verify installation

```bash
mongod --version
mongosh --version
```

Both should report version `8.0.x`.

### 2.5 — Enable the systemd service (do not start yet)

```bash
sudo systemctl enable mongod
```

`enable` registers `mongod` to start automatically whenever the VM boots. The service is intentionally not started yet — the configuration file must be updated first. Starting now would require an unnecessary restart later.

---

## Step 3 — Configuration

### 3.1 — Create the keyfile (VM1 only)

The keyfile is a shared secret used by Replica Set nodes to authenticate each other at the network level. Every node must have the exact same file content. Generate it on VM1 and then copy it to the other nodes.

On **VM1 only**:

```bash
openssl rand -base64 756 | sudo tee /etc/mongodb-keyfile
sudo chmod 400 /etc/mongodb-keyfile
sudo chown mongodb:mongodb /etc/mongodb-keyfile
```

`openssl rand -base64 756` generates 756 bytes of cryptographically random data and encodes it as base64 text. MongoDB recommends this size — long enough to be secure, within the maximum keyfile size limit.

`chmod 400` makes the file readable only by its owner. MongoDB enforces this permission strictly — if the keyfile has group or world read permissions, `mongod` will refuse to start.

`chown mongodb:mongodb` transfers ownership to the `mongodb` user and group. The `mongod` process runs as this user and must be able to read the keyfile at startup.

### 3.2 — Copy the keyfile to VM2 and VM3

On **VM1**, print the keyfile content:

```bash
sudo cat /etc/mongodb-keyfile
```

Copy the entire output to your clipboard.

On **VM2** and **VM3**, open a new file in the text editor and paste:

```bash
sudo nano /etc/mongodb-keyfile
```

Paste the copied content, then save and exit: `Ctrl+X` → `Y` → `Enter`.

Then set correct permissions on **VM2** and **VM3**:

```bash
sudo chmod 400 /etc/mongodb-keyfile
sudo chown mongodb:mongodb /etc/mongodb-keyfile
```

> **Why not use `tee` with heredoc here?**
> The keyfile content begins immediately after the `<<'EOF'` marker, and in some terminal emulators the first line of pasted content ends up on the same line as `EOF`. This causes `tee` to interpret that line as a filename rather than data — producing errors like `tee: ZSszYXs...: No such file or directory`. Using `nano` avoids this problem entirely.

Verify that all three nodes have identical keyfile content:

```bash
sudo md5sum /etc/mongodb-keyfile
```

Run this on all three VMs. The MD5 hash must be identical on all of them. If any hash differs, the nodes will fail to authenticate with each other.

### 3.3 — Configure `mongod.conf`

The default `/etc/mongod.conf` listens only on localhost with no authentication and no Replica Set configuration. It must be replaced on each VM.

The only difference between the three configs is the IP address in `bindIp` — each VM uses its own IP.

**VM1** (`172.16.93.149`):

```bash
sudo tee /etc/mongod.conf <<'EOF'
storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 127.0.0.1,172.16.93.149

replication:
  replSetName: "rs0"

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile
EOF
```

**VM2** (`172.16.93.150`):

```bash
sudo tee /etc/mongod.conf <<'EOF'
storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 127.0.0.1,172.16.93.150

replication:
  replSetName: "rs0"

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile
EOF
```

**VM3** (`172.16.93.151`):

```bash
sudo tee /etc/mongod.conf <<'EOF'
storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 127.0.0.1,172.16.93.151

replication:
  replSetName: "rs0"

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile
EOF
```

**Config parameter explanation:**

| Parameter | Value | Explanation |
|-----------|-------|-------------|
| `storage.dbPath` | `/var/lib/mongodb` | Where MongoDB stores its data files. Default path, pre-owned by the `mongodb` user. |
| `systemLog.logAppend` | `true` | On restart, append to the existing log file instead of overwriting it. Keeps a continuous history across restarts. |
| `systemLog.path` | `/var/log/mongodb/mongod.log` | Log file location. |
| `net.port` | `27017` | MongoDB's default port. |
| `net.bindIp` | `127.0.0.1,<VM_IP>` | Network interfaces `mongod` listens on. Localhost is required for local `mongosh` access; the VM's own IP is required so other nodes can connect. Using `0.0.0.0` would expose MongoDB on all interfaces unnecessarily. |
| `replication.replSetName` | `rs0` | The Replica Set name. **Must be identical on all three nodes.** Nodes with different names will not recognize each other. |
| `security.authorization` | `enabled` | Requires valid username and password for all client connections. |
| `security.keyFile` | `/etc/mongodb-keyfile` | Path to the shared keyfile for inter-node authentication. Required whenever `authorization` is enabled in a Replica Set. |

### 3.4 — Start MongoDB on all 3 VMs

```bash
sudo systemctl start mongod
sudo systemctl status mongod
```

Expected output includes `Active: active (running)`.

If a node shows `failed`, inspect the logs:

```bash
sudo journalctl -u mongod --no-pager | tail -30
```

Common causes of startup failure are covered in the [Troubleshooting](#troubleshooting) section.

---

## Step 4 — Replica Set Initialization

### 4.1 — Create the admin user (VM1 only)

When MongoDB starts for the first time with `authorization: enabled` but no users exist, it allows one unauthenticated connection from localhost. This is the **localhost exception**. It closes permanently after the first user is created.

Connect without credentials:

```bash
mongosh --host 127.0.0.1 --port 27017
```

Create the admin user:

```javascript
use admin

db.createUser({
  user: "admin",
  pwd: "mongo123",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "clusterAdmin", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})
```

Expected response: `{ ok: 1 }`

**Role explanation:**

| Role | Purpose |
|------|---------|
| `userAdminAnyDatabase` | Create, modify, and delete users on any database |
| `clusterAdmin` | Manage the Replica Set: initiate, add/remove members, trigger elections |
| `readWriteAnyDatabase` | Read and write data on any database |

Exit the shell:

```javascript
exit
```

### 4.2 — Initialize the Replica Set (VM1 only)

Connect with admin credentials:

```bash
mongosh --host 127.0.0.1 --port 27017 \
  --username admin --password "mongo123" \
  --authenticationDatabase admin
```

`--authenticationDatabase admin` — Specifies that the credentials belong to the `admin` database. MongoDB stores user accounts per-database, so the auth database must always be specified explicitly.

Initialize the Replica Set:

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "172.16.93.149:27017" },
    { _id: 1, host: "172.16.93.150:27017" },
    { _id: 2, host: "172.16.93.151:27017" }
  ]
})
```

`_id: "rs0"` — Must match the `replSetName` in `mongod.conf` on all nodes exactly.

`members` — Declares all three nodes at once with their IP and port. Using IPs instead of hostnames avoids DNS dependency.

`_id: 0, 1, 2` — Unique integer identifiers for each member within the Replica Set, used internally for election and oplog tracking.

Expected response: `{ ok: 1 }`

Wait a few seconds, then check the status:

```javascript
rs.status()
```

One node will show `PRIMARY` and the other two will show `SECONDARY`. Nodes may briefly show `STARTUP2` during initial sync — this is normal and resolves within seconds.

For a concise summary:

```javascript
rs.status().members.forEach(m => print(m.name, m.stateStr))
```

---

## Step 5 — Failover Test

This test confirms that the Replica Set automatically elects a new PRIMARY when the current one goes down, and that no data is lost in the process.

### 5.1 — Check current state

From any VM:

```bash
mongosh --host 172.16.93.149:27017 \
  --username admin --password "mongo123" \
  --authenticationDatabase admin \
  --eval "rs.status().members.forEach(m => print(m.name, m.stateStr))"
```

Note which node is currently PRIMARY before proceeding.

### 5.2 — Write test data to PRIMARY

Connect to the current PRIMARY:

```bash
mongosh --host <PRIMARY_IP>:27017 \
  --username admin --password "mongo123" \
  --authenticationDatabase admin
```

Insert test documents:

```javascript
use testdb
db.employees.insertMany([
  { name: "Masud", role: "Engineer" },
  { name: "Mahmud", role: "Manager" },
  { name: "Mamun", role: "Designer" }
])
db.employees.find()
```

`use testdb` — Switches to the `testdb` database. In MongoDB, databases are not created with an explicit command. The database comes into existence automatically when the first document is written to it.

`insertMany` — Inserts an array of documents in a single operation.

`find()` with no arguments returns all documents in the collection. Confirm all three records are visible before proceeding.

Useful database inspection commands:

```javascript
// List all databases (only those with at least one document appear)
show dbs

// Show which database you are currently in
db

// List all collections in the current database
show collections
```

### 5.3 — Stop the PRIMARY

On whichever VM is currently PRIMARY:

```bash
sudo systemctl stop mongod
```

### 5.4 — Verify automatic election

From one of the remaining running VMs:

```bash
mongosh --host <SECONDARY_IP>:27017 \
  --username admin --password "mongo123" \
  --authenticationDatabase admin \
  --eval "rs.status().members.forEach(m => print(m.name, m.stateStr))"
```

Within a few seconds, one of the SECONDARY nodes will show `PRIMARY`. The stopped node will appear as `(not reachable/healthy)`. This confirms automatic failover and leader election are working correctly.

### 5.5 — Verify data integrity on new PRIMARY

Connect to the new PRIMARY:

```bash
mongosh --host <NEW_PRIMARY_IP>:27017 \
  --username admin --password "mongo123" \
  --authenticationDatabase admin
```

```javascript
use testdb
db.employees.find()
```

All three documents (Masud, Mahmud, Mamun) should be present. This confirms that replication was complete before the failover occurred and no data was lost.

### 5.6 — Restore the stopped node

On the stopped VM:

```bash
sudo systemctl start mongod
```

After a short resync period, it rejoins the Replica Set as a SECONDARY. The node that won the election remains PRIMARY — MongoDB does not automatically transfer leadership back to the original node.

---

## Reading from SECONDARY Nodes

By default, all reads are routed to the PRIMARY. SECONDARY nodes do not serve read requests unless explicitly instructed. This default exists to prevent reading stale data caused by replication lag.

To enable reads on a SECONDARY within the shell:

```javascript
db.getMongo().setReadPref("secondaryPreferred")
use testdb
db.employees.find()
```

Or specify read preference in the connection string:

```bash
mongosh "mongodb://admin:mongo123@172.16.93.150:27017/testdb?authSource=admin&readPreference=secondaryPreferred"
```

**Read preference options:**

| Option | Behavior |
|--------|----------|
| `primary` (default) | Always read from PRIMARY |
| `primaryPreferred` | Read from PRIMARY; fall back to SECONDARY if PRIMARY is unavailable |
| `secondary` | Always read from a SECONDARY |
| `secondaryPreferred` | Read from SECONDARY if available; fall back to PRIMARY |
| `nearest` | Read from the node with the lowest network latency |

---

## Connecting via Replica Set URI

For applications and production use, always connect using a Replica Set URI that lists all nodes. The client driver automatically discovers which node is currently PRIMARY and routes writes to it — even after a failover, without any manual reconfiguration.

```bash
mongosh "mongodb://admin:mongo123@172.16.93.149:27017,172.16.93.150:27017,172.16.93.151:27017/testdb?authSource=admin&replicaSet=rs0"
```

`replicaSet=rs0` — Tells the client to treat these hosts as a Replica Set rather than independent nodes. The client will automatically discover the topology and track PRIMARY changes after failovers.

---

## Systemd Auto-Start Verification

The `systemctl enable` command from Step 2.5 ensures `mongod` starts on boot. To verify:

```bash
sudo systemctl is-enabled mongod
```

Expected output: `enabled`

To confirm it actually starts after a reboot:

```bash
sudo reboot
```

After the VM comes back up:

```bash
sudo systemctl status mongod
```

Should show `active (running)` without any manual intervention.

---

## Useful Commands Reference

```javascript
// Check full Replica Set status
rs.status()

// Concise node summary
rs.status().members.forEach(m => print(m.name, m.stateStr))

// View current Replica Set configuration
rs.conf()

// Manually step down PRIMARY (triggers a new election)
rs.stepDown()

// List all databases
show dbs

// Show current database
db

// List all collections in current database
show collections

// Count documents in a collection
db.employees.countDocuments()

// Query with a filter
db.employees.find({ role: "Engineer" })

// Drop a test database
use testdb
db.dropDatabase()
```

---

## Troubleshooting

### `mongod` fails to start after config change

The service was not restarted after editing `mongod.conf`. Always restart after any config change:

```bash
sudo systemctl restart mongod
```

If it still fails, check the logs:

```bash
sudo journalctl -u mongod --no-pager | tail -30
```

### `NoReplicationEnabled` during `rs.initiate()`

The `replication` section in `mongod.conf` was not loaded — this happens when `mongod` was already running before the config was updated. Restart and try again:

```bash
sudo systemctl restart mongod
```

### `NodeNotFound` / `Connection refused` during `rs.initiate()`

One or more nodes are not reachable on port 27017. Check all of the following:

- `mongod` is running on all three VMs: `sudo systemctl status mongod`
- UFW allows port 27017: `sudo ufw status`
- `mongod.conf` on each VM has the correct `bindIp` — each VM must list **its own IP**, not another VM's IP

### Keyfile permission error on startup

```bash
sudo chmod 400 /etc/mongodb-keyfile
sudo chown mongodb:mongodb /etc/mongodb-keyfile
```

MongoDB will not start if the keyfile is group-readable or world-readable.

### Keyfile content mismatch between nodes

Nodes will fail to authenticate with each other if the keyfile content differs. Symptoms include nodes staying in `STARTUP` or `UNKNOWN` state indefinitely. Verify:

```bash
sudo md5sum /etc/mongodb-keyfile
```

All three VMs must return the same hash. If they differ, re-copy the keyfile from VM1 to the affected nodes using `nano`.

### `tee` heredoc treats first line as filename

When pasting multi-line content using `sudo tee /path <<'EOF'`, if the first line of content ends up on the same line as `EOF` in the terminal, `tee` interprets it as a filename:

```
tee: ZSszYXsPBbxYor...: No such file or directory
```

**Fix:** Use `sudo nano /path/to/file` and paste the content directly into the editor.

### Cannot write from a SECONDARY connection

Writes are only accepted by the PRIMARY. Connecting to a SECONDARY and attempting a write produces:

```
MongoServerError: not primary
```

Connect to the PRIMARY directly, or use the Replica Set URI which automatically routes writes to the current PRIMARY.

### Data not visible after switching nodes

If `db.employees.find()` returns nothing after switching to a different node, check two things:

1. Make sure you ran `use testdb` first — without this, you are querying the default `test` database.
2. If connected to a SECONDARY, set read preference first: `db.getMongo().setReadPref("secondaryPreferred")`
