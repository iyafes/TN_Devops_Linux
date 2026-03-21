# MySQL 8.4 — Complete Study Guide

> **Environment:** Rocky Linux 9 | MySQL 8.4 Community Edition
> **Coverage:** Architecture · Configuration · Data Types · Indexes · Transactions · Query Optimization · Backup & Recovery · Monitoring · Security · High Availability

---

## Table of Contents

1. [Architecture](#1-architecture)
   - 1.1 Overall Architecture · 1.2 Connection Layer · 1.3 SQL Layer
   - 1.4 InnoDB Memory · 1.5 Background Threads · 1.6 File System · 1.7 LSN
   - 1.8 Read Path · 1.9 Write Path · 1.10 UPDATE · 1.11 DELETE · 1.12 ROLLBACK
   - 1.13 Crash Recovery · 1.14 MVCC · 1.15 Large Transactions · 1.16 Summary
2. [Configuration & Parameters](#2-configuration--parameters)
   - 2.1 my.cnf Structure · 2.2 Core · 2.3 Memory · 2.4 Connections
   - 2.5 InnoDB I/O · 2.6 Logging · 2.7 Replication · 2.8 Character Set
   - 2.9 Runtime Change · 2.10 Production my.cnf
3. [Data Types](#3-data-types)
   - 3.1 Integer · 3.2 Decimal · 3.3 String · 3.4 DateTime · 3.5 JSON · 3.6 Table Design
4. [Indexes](#4-indexes)
   - 4.1 Why Index · 4.2 B-Tree · 4.3 Clustered vs Secondary
   - 4.4 Index Types · 4.5 EXPLAIN · 4.6 Design Rules
5. [Transactions & Locking](#5-transactions--locking)
   - 5.1 Transaction · 5.2 ACID · 5.3 Commands · 5.4 Isolation Levels
   - 5.5 MVCC · 5.6 Locking · 5.7 Deadlock · 5.8 FOR UPDATE · 5.9 Monitoring
6. [Query Optimization](#6-query-optimization)
   - 6.1 Slow Query · 6.2 EXPLAIN · 6.3 Slow Patterns · 6.4 ORDER BY
   - 6.5 JOIN · 6.6 Hints · 6.7 Checklist
7. [Backup & Recovery](#7-backup--recovery)
   - 7.1 Types · 7.2 mysqldump · 7.3 Restore · 7.4 Binlog · 7.5 PITR
   - 7.6 XtraBackup · 7.7 Automation · 7.8 Best Practices
8. [Monitoring](#8-monitoring)
   - 8.1 OS Level · 8.2 Connections · 8.3 Buffer Pool · 8.4 Query Performance
   - 8.5 Replication · 8.6 Database Size · 8.7 InnoDB Diagnostic
   - 8.8 Index Usage · 8.9 Dashboard Script · 8.10 Thresholds
9. [Security](#9-security)
   - 9.1 Layers · 9.2 Authentication · 9.3 Privileges · 9.4 Roles
   - 9.5 Root Hardening · 9.6 SSL/TLS · 9.7 Encryption · 9.8 Network · 9.9 Checklist
10. [High Availability & Replication](#10-high-availability--replication)
    - 10.1 RPO/RTO · 10.2 How Replication Works · 10.3 Single Replica
    - 10.4 Multi-Replica · 10.5 Master-Master · 10.6 Install to Replication (E2E)
    - 10.7 Troubleshooting · 10.8 GTID · 10.9 ProxySQL · 10.10 Group Replication
    - 10.11 Topology Comparison · 10.12 Health Checklist · 10.13 Quick Reference

---

# 1. Architecture

## 1.1 Overall Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                │
│              Application / mysql CLI / JDBC / ORM                   │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ TCP/IP (port 3306) or Unix Socket
┌─────────────────────────────▼───────────────────────────────────────┐
│                      CONNECTION LAYER                               │
│         Thread Manager · Authentication · Authorization             │
│                    Thread Cache / Connection Pool                   │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                         SQL LAYER                                   │
│        Parser → Preprocessor → Optimizer → Execution Engine         │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                    STORAGE ENGINE LAYER (InnoDB)                    │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      InnoDB Memory                          │   │
│  │  ┌─────────────────────────────────────────────────────┐    │   │
│  │  │                   Buffer Pool                       │    │   │
│  │  │   Data Pages · Index Pages · Undo Pages             │    │   │
│  │  │   Free List · LRU List · Flush List                 │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  │  ┌──────────────┐  ┌───────────────┐  ┌────────────────┐    │   │
│  │  │  Log Buffer  │  │ Change Buffer │  │  Adaptive Hash │    │   │
│  │  └──────────────┘  └───────────────┘  │  Index (AHI)   │    │   │
│  │                                        └────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │               InnoDB Background Threads                     │   │
│  │  Master Thread · IO Threads · Purge Thread · Page Cleaner   │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                          FILE SYSTEM                                │
│   *.ibd · ibdata1 · ib_logfile · undo_001 · mysql-bin · mysql.sock  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1.2 Connection Layer

Client connect করলে Connection Layer এ প্রথম কাজ হয়।

**TCP/IP vs Unix Socket:**
```
TCP/IP  → network এর মাধ্যমে (remote বা localhost উভয়)
          mysql -u root -p -h 172.16.93.140

Socket  → same machine এ file এর মাধ্যমে (network stack bypass, faster)
          mysql -u root -p -S /data/mysql/mysql.sock
```

**Thread Lifecycle:**
```
Client connects
      │
      ▼
Thread Cache এ thread আছে? → Yes → Reuse thread
      │ No
      ▼
New OS thread তৈরি করো
      │
      ▼
Authentication: user exist? password ok? host allowed?
      │
      ▼
Authorization: এই database/table access করতে পারে?
      │
      ▼
Query execute করো
      │
      ▼
Client disconnects → Thread cache এ ফেরত (thread_cache_size পর্যন্ত)
```

```sql
SHOW STATUS LIKE 'Threads_connected';   -- এখন কতটা connection
SHOW STATUS LIKE 'Threads_created';     -- মোট নতুন thread তৈরি হয়েছে
SHOW STATUS LIKE 'Threads_cached';      -- Thread Cache এ কতটা
SHOW VARIABLES LIKE 'thread_cache_size';
SHOW VARIABLES LIKE 'max_connections';
```

---

## 1.3 SQL Layer

### Parser
SQL query কে Parse Tree এ convert করে। Syntax ভুল থাকলে এখানেই error।

```sql
SELECT name FROM users WHERE age > 25;
-- Parser বোঝে:
-- কী চাই    → name
-- কোথা থেকে → users table
-- Condition  → age > 25
```

### Preprocessor
Semantic check — table exist করে? column আছে? permission আছে?

### Optimizer
Best execution plan তৈরি করে।

```sql
EXPLAIN SELECT name FROM users WHERE age > 25;
-- Optimizer এর plan দেখো
-- type=ALL মানে full table scan (সমস্যা)
-- type=ref বা range মানে index use হচ্ছে (ভালো)
```

Optimizer statistics দেখে সিদ্ধান্ত নেয়:
- Table এ কতটা row আছে
- Index এর cardinality কতটা
- Data distribution কেমন

### Execution Engine
Optimizer এর plan অনুযায়ী query চালায় এবং Storage Engine এর সাথে কথা বলে।

---

## 1.4 InnoDB Memory Components

### Buffer Pool

InnoDB এর সবচেয়ে critical component। সব data এবং index এখানে page আকারে (default 16KB) cache হয়।

**Internal Structure:**
```
Buffer Pool
├── Free List       → Empty pages, ব্যবহারের জন্য ready
├── LRU List        → Modified LRU algorithm
│   ├── New Sublist (5/8)  → Recently accessed (Hot end)
│   └── Old Sublist (3/8) → Less recently accessed (Cold end)
└── Flush List      → Dirty pages যেগুলো disk এ যায়নি
```

**Modified LRU — কেন Midpoint Insertion?**
```
New page load হলে → Old Sublist এর HEAD এ যায় (midpoint)
Page access হলে  → New Sublist এর HEAD এ promote হয়
Old Sublist TAIL  → Evict হয়

Midpoint insertion এর কারণ:
Full table scan এ অনেক page load হয়
সরাসরি New Sublist HEAD এ গেলে hot pages evict হতো
Midpoint এ গেলে scan pages Old Sublist এ থাকে → দ্রুত evict
Hot pages New Sublist এ সুরক্ষিত থাকে
```

```sql
-- Buffer Pool hit rate (99%+ হওয়া উচিত)
SELECT ROUND(
    (1 - (SELECT VARIABLE_VALUE FROM performance_schema.global_status
          WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
         (SELECT VARIABLE_VALUE FROM performance_schema.global_status
          WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
    ) * 100, 4) AS hit_rate_pct;

SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool%';
```

### Log Buffer
Redo Log এর in-memory buffer। Transaction changes প্রথমে এখানে লেখা হয়।

```
Flush হয় যখন:
  1. Transaction COMMIT (innodb_flush_log_at_trx_commit=1)
  2. Log Buffer 1/2 full হলে
  3. প্রতি 1 সেকেন্ডে (background thread)
```

### Change Buffer
Secondary index এর changes buffer করে রাখে। Disk I/O কমায়।

```
কেন দরকার?
Secondary index pages random order এ disk এ থাকে
প্রতিটা INSERT/UPDATE এ সেই page disk থেকে আনা → expensive

Change Buffer এ রেখে দাও
পরে ওই page memory তে আসলে (read এর কারণে) merge করো
```

```sql
SHOW VARIABLES LIKE 'innodb_change_buffer_max_size'; -- Default: 25%
```

### Adaptive Hash Index (AHI)
Frequently accessed pages এর জন্য automatic hash index।

```
B-Tree lookup : Root → Branch → Leaf = 3-4 page reads → O(log n)
AHI lookup    : Hash(key) → Direct to page             → O(1)

MySQL নিজেই decide করে কোন pages এর জন্য AHI তৈরি করবে
Manual control নেই
```

```sql
SHOW VARIABLES LIKE 'innodb_adaptive_hash_index';
SHOW ENGINE INNODB STATUS\G  -- ADAPTIVE HASH INDEX section দেখো
```

---

## 1.5 InnoDB Background Threads

```
Master Thread (1টা):
  প্রতি 1 সেকেন্ডে:
    - Dirty pages flush (দরকার হলে)
    - Redo Log flush
    - Change Buffer merge
  প্রতি 10 সেকেন্ডে:
    - Dirty pages 10% flush
    - Undo Log purge

IO Threads (default 4 read + 4 write):
  - Async disk read/write
  innodb_read_io_threads  = 4
  innodb_write_io_threads = 4

Purge Threads (default 4):
  - Committed transaction এর Undo Log delete
  - Delete marked rows actually মুছে দেয়
  innodb_purge_threads = 4

Page Cleaner Threads (default 4):
  - Flush List থেকে dirty pages disk এ লেখে
  innodb_page_cleaners = 4
```

---

## 1.6 File System

```
/data/mysql/
│
├── mysql-bin.000001     → Binary Log (Replication + PITR)
├── mysql-bin.index      → Binlog files এর list
│
├── ib_logfile0          → Redo Log (circular, crash recovery)
├── ib_logfile1          → Redo Log
│
├── ibdata1              → System Tablespace
│                          (Data Dictionary, Change Buffer,
│                           Undo Log যদি separate না হয়)
│
├── undo_001             → Undo Tablespace 1 (MVCC, Rollback)
├── undo_002             → Undo Tablespace 2
│
├── mysql/               → mysql system database
├── performance_schema/  → Performance Schema data
│
└── myapp_db/
    ├── users.ibd        → users table data + index (file per table)
    └── orders.ibd       → orders table data + index
```

| File | কাজ |
|---|---|
| `*.ibd` | InnoDB table data এবং index |
| `mysql-bin.*` | Binary Log — replication এবং PITR |
| `ib_logfile*` | Redo Log — crash recovery |
| `ibdata1` | System tablespace |
| `undo_*` | Undo tablespace — MVCC এবং rollback |
| `mysql.sock` | Unix socket file |
| `mysqld.pid` | MySQL process ID |

---

## 1.7 LSN — Log Sequence Number

InnoDB এর internal clock। প্রতিটা Redo Log write এ LSN বাড়ে।

```
LSN ব্যবহার হয়:
  - Redo Log এ কোন position পর্যন্ত লেখা আছে
  - Buffer Pool page এ কোন LSN পর্যন্ত change আছে
  - Checkpoint: কোন LSN পর্যন্ত disk এ flush হয়েছে

Recovery তে:
  Checkpoint LSN থেকে শুরু করো
  সব Redo Log event reapply করো
```

```sql
SHOW ENGINE INNODB STATUS\G
-- Log sequence number XXXXX  ← current LSN
-- Log flushed up to   XXXXX  ← disk এ কতটুকু
-- Pages flushed up to XXXXX  ← dirty pages flush পর্যন্ত
-- Last checkpoint at  XXXXX  ← checkpoint LSN
```

---

## 1.8 Complete Read Path (Full Cycle)

**Query:** `SELECT * FROM users WHERE id = 5`

```
Step 1: Client → Connection Layer
  TCP/Socket connection
  Thread Cache থেকে thread reuse অথবা নতুন thread
  Authentication + Authorization check

Step 2: Parser
  "SELECT * FROM users WHERE id = 5"
  → Parse Tree তৈরি
  Syntax check

Step 3: Preprocessor
  users table exist করে?
  id column আছে?
  SELECT permission আছে?

Step 4: Optimizer
  id column এ PRIMARY KEY আছে → type=const
  Cost: 1 row, 1 page read
  Plan: PRIMARY KEY index দিয়ে 1 row fetch

Step 5: Execution Engine → InnoDB Handler API
  "id=5 এর row দাও"

Step 6: InnoDB — Buffer Pool Check
  ┌──────────────────────────────────────┐
  │ Buffer Pool এ page আছে?              │
  │                                      │
  │ YES → page থেকে row read (no disk)   │
  │                                      │
  │ NO  → Free List এ page আছে?          │
  │       YES → disk থেকে load           │
  │             Free → LRU Old end       │
  │                                      │
  │       NO  → LRU Old TAIL evict করো   │
  │             (dirty হলে flush করো)    │
  │             সেই slot এ নতুন page load│
  └──────────────────────────────────────┘

Step 7: MVCC — কোন Version দেখাবে?
  Row এর DB_TRX_ID দেখো (কোন transaction modify করেছে)
  Transaction এর snapshot time এর আগের version দেখাবে
  দরকার হলে Undo Log থেকে পুরনো version তৈরি করো

Step 8: Result → Client
  Row data → Execution Engine → SQL Layer → Network → Client

Component States during READ:
  Buffer Pool    : Page LRU New end এ যায়
  Undo Log       : Read হতে পারে (MVCC এর জন্য)
  Redo Log       : Untouched
  Lock           : Consistent snapshot read (REPEATABLE READ এ no lock)
```

---

## 1.9 Complete Write Path (Full Cycle)

**Query:** `INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')`

```
Step 1-4: Connection → Parser → Preprocessor → Optimizer
  Write permission check
  Target page বের করো

Step 5: Transaction শুরু
  TRX_ID assign (globally unique, monotonically increasing)
  Undo Log slot allocate

Step 6: Undo Log এ পুরনো state লেখো
  INSERT undo: "এই row ছিল না, rollback করলে delete করো"
  UPDATE undo: "এই row এর আগের value ছিল X"
  DELETE undo: "এই row ছিল, rollback করলে restore করো"

  কেন Undo Log?
    1. ROLLBACK এর জন্য
    2. MVCC — অন্য transaction পুরনো version দেখতে পারে

Step 7: Buffer Pool এ Page Modify
  Page Buffer Pool এ আছে?
    YES → সরাসরি modify
    NO  → Disk থেকে load, তারপর modify
  
  Page state: Clean → Dirty
  Dirty page → Flush List এ যায়
  Row update: DB_TRX_ID = current TRX_ID
              DB_ROLL_PTR = Undo Log pointer

Step 8: Redo Log Buffer এ লেখো
  Physical change: "Page X, offset Y তে এই bytes লেখো"
  Sequential memory write → অনেক fast
  এখনো disk এ নেই

Step 9: Secondary Index Update
  Index page Buffer Pool এ আছে?
    YES → সরাসরি update (Dirty page)
    NO  → Change Buffer এ রেখে দাও
           পরে page memory তে আসলে merge হবে

Step 10: COMMIT
  1. Redo Log Buffer → Disk flush (fsync)
     এই মুহূর্তে transaction durable হলো
     Crash হলেও Redo Log থেকে recover করা যাবে
  2. TRX_ID Commit List এ mark
  3. INSERT Undo entry obsolete (Purge Thread পরে clean করবে)
  4. Row Lock release
  5. Binary Log এ event লেখো (replication এর জন্য)

NOTE: COMMIT এর পরেও dirty page disk এ নেই
      Redo Log disk এ আছে — crash হলে এটা দিয়ে recover হবে
      Background Page Cleaner dirty page পরে disk এ লিখবে

Component States during WRITE:
  Buffer Pool    : Target page dirty, Flush List এ
  Undo Log       : New entry লেখা হয়
  Redo Log Buffer: Change লেখা হয়
  Redo Log Disk  : COMMIT এ flush
  Change Buffer  : Secondary index change buffer হতে পারে
  Row Lock       : Exclusive lock নেওয়া হয়
  Binary Log     : COMMIT এ event লেখা হয়
```

---

## 1.10 UPDATE Operation

**Query:** `UPDATE users SET salary = 50000 WHERE id = 5`

```
Before: id=5 → {name:'Alice', salary:40000, DB_TRX_ID:100, DB_ROLL_PTR:→UL1}

1. id=5 খোঁজো (PRIMARY KEY), Exclusive Lock নাও

2. Undo Log এ পুরনো version রাখো:
   TRX_ID: 200 | Type: UPDATE | Old: {salary:40000} | Prev: →UL1

3. Buffer Pool এ in-place modify:
   id=5 → {name:'Alice', salary:50000, DB_TRX_ID:200, DB_ROLL_PTR:→UL2}
   Page: Clean → Dirty → Flush List

4. Redo Log Buffer: "Page P, offset O: salary=50000"

5. COMMIT:
   Redo Log disk flush → Binary Log event → Lock release

After (Buffer Pool):
   id=5 → {salary:50000, DB_TRX_ID:200, DB_ROLL_PTR:→UL2}
   UL2 Undo entry আছে (MVCC এর জন্য)
   Purge Thread পরে UL2 delete করবে
```

---

## 1.11 DELETE Operation

**Query:** `DELETE FROM users WHERE id = 5`

```
1. id=5 খোঁজো, Exclusive Lock নাও

2. Soft Delete — row immediately মুছে যায় না!
   Row এ delete flag set:
     DB_DELETED = true
     DB_TRX_ID  = current TRX_ID

   কেন soft delete?
   অন্য transaction MVCC দিয়ে পুরনো version দেখতে পারে
   Purge Thread পরে actually delete করবে

3. Undo Log এ পুরো row রাখো
   Rollback করলে delete flag সরিয়ে row restore

4. Secondary Index: Purge Thread পরে delete করবে

5. COMMIT → Redo Log flush → Lock release

6. Purge Thread (Background):
   সব active transaction এর snapshot এর আগের
   delete marked row physically মুছে দেয়
   Secondary index entries মুছে দেয়
   Disk space free হয়
```

---

## 1.12 ROLLBACK Operation

```
ROLLBACK বা error হলে:

Undo Log পড়ো — সব change উল্টো order এ undo করো

INSERT undo → সেই row DELETE করো, secondary index entry মুছো
UPDATE undo → Row এর পুরনো value restore করো
DELETE undo → Delete flag সরাও, row visible হয়ে যাক

Buffer Pool এ changes undo হয়
Undo operation গুলোও Redo Log এ লেখা হয়
(Undo এর undo — crash recovery consistent রাখতে)

Lock release → অপেক্ষা করা transaction এগিয়ে যেতে পারে
Purge Thread পরে Undo Log clean করবে
```

---

## 1.13 Crash Recovery

```
Crash এর সময় possible states:
  1. Committed, dirty page disk এ যায়নি → Redo Log এ আছে
  2. Committed, Redo Log disk এ আছে     → Safe
  3. Uncommitted, Redo Log buffer এ ছিল  → Lost (ok, commit হয়নি)
  4. চলছিল, commit হয়নি               → Rollback করতে হবে

Recovery Phases (MySQL restart এ):
  Phase 1: Redo Log Scan
    Last checkpoint থেকে শেষ পর্যন্ত
    Committed transaction এর changes reapply

  Phase 2: Undo Phase
    Commit হয়নি এমন transaction Undo Log দিয়ে rollback

  Phase 3: Normal operation শুরু

Checkpoint:
  নিয়মিত checkpoint নেয়
  এই LSN পর্যন্ত সব dirty pages disk এ আছে
  Recovery তে শুধু checkpoint থেকে scan করলেই হয়
```

---

## 1.14 MVCC — Concurrent Read + Write

```
Timeline:
  T=0: Transaction A শুরু (TRX_ID=100), salary snapshot: 40000
  T=1: Transaction B শুরু (TRX_ID=200)
       UPDATE salary=50000 WHERE id=5 → COMMIT
  T=2: Transaction A আবার SELECT id=5

MVCC কী করে?
  Current row: DB_TRX_ID=200, salary=50000

  Transaction A এর snapshot time: TRX_ID=100 এর আগে
  TRX_ID=200 এর change দেখা যাবে না

  Undo Log traverse:
    Current: TRX_ID=200 → Too new, skip
    UL2:     TRX_ID=100 এর আগে → salary=40000 ← এটা দেখাও

  Transaction A দেখবে: salary=40000
  Transaction B commit করলেও Transaction A এর view change হয় না

Result:
  Writer, Reader কে block করে না
  Reader, Writer কে block করে না
  High concurrency সম্ভব
```

---

## 1.15 Large Transaction Considerations

```sql
-- 10 লক্ষ row update করলে সমস্যা:
UPDATE orders SET status = 'archived' WHERE created_at < '2020-01-01';

সমস্যা:
  1. Undo Log অনেক বড় → Undo Tablespace বাড়ে
  2. Buffer Pool এ অনেক dirty page → thrashing
  3. Redo Log overflow → force checkpoint
  4. 10 লক্ষ row exclusive lock → অন্যরা block
  5. COMMIT slow → Redo Log flush বড়

Solution — Batch করো:
```

```sql
-- Loop এ batch করো
START TRANSACTION;
UPDATE orders SET status = 'archived'
WHERE created_at < '2020-01-01' LIMIT 1000;
COMMIT;
-- Repeat যতক্ষণ ROW_COUNT() > 0
```

---

## 1.16 Component Summary

| Component | Role |
|---|---|
| Buffer Pool | Data/Index cache, Hot data in memory |
| LRU List | Eviction policy, hot vs cold pages |
| Flush List | Track dirty pages for disk write |
| Redo Log | Crash recovery, durability guarantee |
| Undo Log | Rollback support, MVCC old versions |
| Log Buffer | Redo log memory buffer before disk flush |
| Change Buffer | Defer secondary index I/O |
| AHI | Hash index for hot pages, O(1) lookup |
| Master Thread | Coordinate background work |
| IO Threads | Async disk read/write |
| Purge Thread | Clean old undo log, delete marked rows |
| Page Cleaner | Flush dirty pages to disk |
| Binary Log | Replication, Point-in-time recovery |
| Parser | SQL syntax → Parse Tree |
| Optimizer | Best execution plan |
| Execution Engine | Plan execute, Storage Engine call |
| Thread Cache | Connection thread reuse |

---

# 2. Configuration & Parameters

## 2.1 my.cnf Structure

```ini
[mysqld]        ← MySQL server
[client]        ← সব client tools (mysql, mysqldump)
[mysql]         ← শুধু mysql CLI
[mysqldump]     ← শুধু mysqldump
```

Parameter এর section গুরুত্বপূর্ণ — `[mysqld]` এ লেখা parameter server এ apply হয়।

## 2.2 Core / Identity Parameters

```ini
[mysqld]
datadir   = /data/mysql
socket    = /data/mysql/mysql.sock
log-error = /var/log/mysqld.log
pid-file  = /var/run/mysqld/mysqld.pid

[client]
socket    = /data/mysql/mysql.sock
```

| Parameter | কাজ |
|---|---|
| `datadir` | সব database, table, index এখানে |
| `socket` | Local connection এ TCP bypass করে file দিয়ে communicate। `[mysqld]` এবং `[client]` দুটোতেই same path দিতে হয় |
| `log-error` | সব error, warning, startup/shutdown message এখানে |
| `pid-file` | MySQL process ID — systemctl এই file দেখে |

```sql
SHOW VARIABLES LIKE 'datadir';
```

## 2.3 Memory Parameters

```ini
[mysqld]
innodb_buffer_pool_size      = 6G
innodb_buffer_pool_instances = 4
innodb_log_buffer_size       = 64M
tmp_table_size               = 64M
max_heap_table_size          = 64M
sort_buffer_size             = 4M
join_buffer_size             = 4M
```

### `innodb_buffer_pool_size` — সবচেয়ে গুরুত্বপূর্ণ
```
Rule of thumb:
  Dedicated MySQL server → RAM এর 70-80%
  Shared server         → RAM এর 40-50%

8GB RAM  → 6G
16GB RAM → 12G
32GB RAM → 24G
```

### `innodb_buffer_pool_instances`
```ini
# Buffer Pool >= 1GB হলে ব্যবহার করো
# সাধারণত CPU core সংখ্যার সমান, maximum 8
innodb_buffer_pool_instances = 4
# প্রতিটা instance এ আলাদা mutex, আলাদা LRU
# Concurrent access এ lock contention কমে
```

### `tmp_table_size` এবং `max_heap_table_size`
```ini
# দুটো সবসময় same value দাও
tmp_table_size      = 64M
max_heap_table_size = 64M
# এর বেশি হলে temporary table disk এ যায় → slow
```

```sql
SHOW STATUS LIKE 'Created_tmp_disk_tables';
SHOW STATUS LIKE 'Created_tmp_tables';
-- ratio বেশি হলে এই values বাড়াও
```

### `sort_buffer_size` এবং `join_buffer_size`
Per-thread buffer। সাবধান — 1000 connection × 4M = 4GB।

## 2.4 Connection Parameters

```ini
[mysqld]
max_connections     = 200
wait_timeout        = 600
interactive_timeout = 600
```

| Parameter | কাজ |
|---|---|
| `max_connections` | Maximum concurrent connections. বেশি হলে ERROR 1040 |
| `wait_timeout` | Idle non-interactive connection এর timeout (seconds) |
| `interactive_timeout` | Idle interactive connection (mysql CLI) এর timeout |

```sql
SHOW STATUS LIKE 'Max_used_connections'; -- সর্বোচ্চ কতটা ছিল
SHOW STATUS LIKE 'Threads_connected';    -- এখন কতটা
```

## 2.5 InnoDB I/O Parameters

```ini
[mysqld]
innodb_file_per_table          = ON
innodb_flush_method            = O_DIRECT
innodb_flush_log_at_trx_commit = 1
innodb_io_capacity             = 200
innodb_io_capacity_max         = 2000
```

### `innodb_flush_log_at_trx_commit` — Durability vs Performance

| Value | কী হয় | Speed | Safety |
|---|---|---|---|
| `0` | প্রতি 1 সেকেন্ডে flush | সবচেয়ে fast | Crash এ 1s data হারাতে পারে |
| `1` | প্রতি commit এ flush | সবচেয়ে slow | সম্পূর্ণ safe — ACID guarantee |
| `2` | OS cache, 1s flush | মধ্যম | OS crash এ data হারাতে পারে |

```ini
# Production (banking, e-commerce) → সবসময় 1
innodb_flush_log_at_trx_commit = 1
```

### `innodb_flush_method = O_DIRECT`
MySQL data disk এ লেখার সময় OS cache bypass করে। Double buffering এড়ানো যায় — InnoDB Buffer Pool এ আছে, OS cache এও রাখার দরকার নেই।

### `innodb_file_per_table = ON`
```
ON  → প্রতিটা table এর আলাদা .ibd file
      Table drop করলে disk space free হয়
      Per-table backup সহজ
      Corruption হলে শুধু সেই table affected

OFF → সব table ibdata1 এ (shared tablespace)
      Space কখনো free হয় না
```

## 2.6 Logging Parameters

```ini
[mysqld]
slow_query_log                = ON
slow_query_log_file           = /var/log/mysql/slow.log
long_query_time               = 2
log_queries_not_using_indexes = ON
general_log                   = OFF
```

### `slow_query_log`
Performance tuning এর সেরা tool। `long_query_time` সেকেন্ডের বেশি লাগলে log হয়।

```bash
sudo tail -100 /var/log/mysql/slow.log
sudo mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
# -s t = sort by time, -t 10 = top 10
```

`general_log` — সব query log করে। Production এ **OFF** রাখো।

## 2.7 Replication Parameters

```ini
# Master এ
server-id                  = 1
log_bin                    = /data/mysql/mysql-bin.log
binlog_format              = ROW
binlog_expire_logs_seconds = 604800
sync_binlog                = 1

# Replica তে
server-id          = 2
read_only          = ON
relay_log          = /data/mysql/relay-bin.log
relay_log_recovery = ON
```

### `binlog_format` — তিনটা Option

| Format | কী log হয় | সমস্যা |
|---|---|---|
| `STATEMENT` | SQL query টাই | `NOW()`, `RAND()` এ Replica তে ভুল result |
| `ROW` | প্রতিটা row এর আগে ও পরের value | Reliable, কিন্তু বেশি disk |
| `MIXED` | Auto-switch | Unpredictable |

```ini
binlog_format = ROW  # সবসময় এটা
```

### `sync_binlog = 1`
প্রতি commit এ binlog disk এ sync। Replication এ সবসময় 1 রাখো।

### `read_only = ON` (Replica তে)
Normal user রা Replica তে write করতে পারবে না। শুধু replication thread এবং root।

```sql
-- Super user কেও block করতে
SET GLOBAL super_read_only = ON;
```

## 2.8 Character Set Parameters

```ini
[mysqld]
character_set_server = utf8mb4
collation_server     = utf8mb4_unicode_ci
```

**`utf8` vs `utf8mb4`:**
MySQL এর `utf8` সত্যিকারের UTF-8 না — maximum 3 bytes। Emoji এবং অনেক special character 4 bytes নেয়। `utf8mb4` সত্যিকারের UTF-8 (4 bytes পর্যন্ত)। সবসময় `utf8mb4` ব্যবহার করো।

## 2.9 Runtime Parameter Change

```sql
-- Runtime এ change (restart ছাড়া, কিন্তু restart এ reset)
SET GLOBAL max_connections = 300;

-- Session level (শুধু এই connection এর জন্য)
SET SESSION sort_buffer_size = 8M;

-- Current value দেখো
SHOW VARIABLES LIKE 'innodb%';
SHOW VARIABLES LIKE '%timeout%';

-- Status দেখো
SHOW GLOBAL STATUS LIKE 'Threads%';
```

স্থায়ীভাবে change করতে `my.cnf` edit করে `systemctl restart mysqld`।

## 2.10 Production-ready my.cnf

```ini
[mysqld]
# Core
datadir   = /data/mysql
socket    = /data/mysql/mysql.sock
log-error = /var/log/mysqld.log
pid-file  = /var/run/mysqld/mysqld.pid

# Character Set
character_set_server = utf8mb4
collation_server     = utf8mb4_unicode_ci

# Memory (8GB RAM example)
innodb_buffer_pool_size      = 6G
innodb_buffer_pool_instances = 4
tmp_table_size               = 64M
max_heap_table_size          = 64M
sort_buffer_size             = 4M
join_buffer_size             = 4M

# Connections
max_connections     = 200
wait_timeout        = 600
interactive_timeout = 600

# InnoDB I/O
innodb_file_per_table          = ON
innodb_flush_method            = O_DIRECT
innodb_flush_log_at_trx_commit = 1
innodb_io_capacity             = 200
innodb_io_capacity_max         = 2000

# Logging
slow_query_log                = ON
slow_query_log_file           = /var/log/mysql/slow.log
long_query_time               = 2
log_queries_not_using_indexes = ON

# Replication (Master)
server-id                  = 1
log_bin                    = /data/mysql/mysql-bin.log
binlog_format              = ROW
binlog_expire_logs_seconds = 604800
sync_binlog                = 1

[client]
socket = /data/mysql/mysql.sock
```

---

# 3. Data Types

## 3.1 Integer Types

```
┌──────────────┬──────────┬────────────────────────────────┐
│ Type         │ Storage  │ Signed Range                   │
├──────────────┼──────────┼────────────────────────────────┤
│ TINYINT      │ 1 byte   │ -128 to 127                    │
│ SMALLINT     │ 2 bytes  │ -32,768 to 32,767              │
│ MEDIUMINT    │ 3 bytes  │ -8,388,608 to 8,388,607        │
│ INT          │ 4 bytes  │ -2,147,483,648 to 2,147,483,647│
│ BIGINT       │ 8 bytes  │ -9.2 quintillion to 9.2Q       │
└──────────────┴──────────┴────────────────────────────────┘
```

**Signed vs Unsigned:**
```sql
age TINYINT           -- -128 to 127 (signed)
age TINYINT UNSIGNED  -- 0 to 255 (unsigned, double range)
```

**কোনটা কখন:**
```sql
is_active   TINYINT(1) UNSIGNED  -- boolean: 0 or 1
age         TINYINT UNSIGNED     -- 0 to 255
user_id     INT UNSIGNED         -- 0 to ~4.2 billion
event_id    BIGINT UNSIGNED      -- very large IDs
```

**Common mistake:** সব জায়গায় INT। `age` এ INT (4 bytes) না দিয়ে TINYINT (1 byte) দাও — 1 কোটি row এ ~28MB বাঁচে।

**AUTO_INCREMENT:**
```sql
id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY
-- 2 billion+ row হবে? শুরু থেকে BIGINT UNSIGNED দাও
```

## 3.2 Decimal Types

```sql
FLOAT   -- 4 bytes, approximate
DOUBLE  -- 8 bytes, approximate
DECIMAL -- exact, variable storage
```

**FLOAT/DOUBLE সমস্যা:**
```sql
SELECT 0.1 + 0.2;
-- Result: 0.30000000000000004  ← ভুল!
```

**টাকা বা exact value → সবসময় DECIMAL:**
```sql
-- DECIMAL(precision, scale)
price   DECIMAL(10, 2)   -- 99999999.99 পর্যন্ত, exact
salary  DECIMAL(12, 2)
amount  DECIMAL(15, 2) NOT NULL DEFAULT 0.00
```

## 3.3 String Types

**CHAR vs VARCHAR:**
```
CHAR(10)    → সবসময় 10 bytes (fixed), trailing space padding
VARCHAR(10) → যতটুকু data ততটুকু + 1-2 bytes overhead
```

```sql
-- Fixed length → CHAR
country_code CHAR(2)     -- 'BD', 'US'
uuid         CHAR(36)    -- UUID সবসময় same length
status       CHAR(1)     -- 'A', 'I', 'D'

-- Variable length → VARCHAR
name    VARCHAR(100)
email   VARCHAR(255)
address VARCHAR(500)
```

**TEXT Types:**
```
TINYTEXT   → max 255 bytes
TEXT       → max 65,535 bytes (~64KB)
MEDIUMTEXT → max 16,777,215 bytes (~16MB)
LONGTEXT   → max 4GB
```

TEXT এর সীমাবদ্ধতা:
- Index দিতে prefix length লাগে
- Default value দেওয়া যায় না
- Row এর বাইরে store → Buffer Pool এ কম efficient

500 character এর মধ্যে → `VARCHAR(500)` ই ভালো।

**ENUM:**
```sql
status ENUM('active', 'inactive', 'pending', 'deleted')
-- String না, number হিসেবে store (1-2 bytes, efficient)
-- নতুন value যোগ করতে ALTER TABLE লাগে — বড় table এ expensive
```

## 3.4 Date and Time Types

```
DATE      → 3 bytes → '2024-03-08'
TIME      → 3 bytes → '14:30:00'
DATETIME  → 8 bytes → '2024-03-08 14:30:00'
TIMESTAMP → 4 bytes → '2024-03-08 14:30:00' (UTC stored)
YEAR      → 1 byte  → 2024
```

**DATETIME vs TIMESTAMP:**

| | DATETIME | TIMESTAMP |
|---|---|---|
| Range | 1000-9999 | 1970-2038 ⚠️ |
| Timezone | Stored as-is | UTC তে store, retrieve এ convert |
| Size | 8 bytes | 4 bytes |

**2038 Problem:** TIMESTAMP 4 bytes এ Unix timestamp store করে। 2038 সালে overflow। 2038 পরের date → `DATETIME` ব্যবহার করো।

```sql
-- Auto-manage timestamps
created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
-- ON UPDATE CURRENT_TIMESTAMP → row update এ automatically set
```

## 3.5 JSON Type

```sql
CREATE TABLE products (
    id       INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name     VARCHAR(200),
    metadata JSON
);

INSERT INTO products (name, metadata)
VALUES ('Laptop', '{"brand": "Dell", "ram": 16}');

-- JSON field access
SELECT metadata->>'$.brand' AS brand FROM products;
SELECT * FROM products WHERE metadata->>'$.ram' > 8;

-- JSON column directly index করা যায় না
-- Generated column দিয়ে index দাও:
ALTER TABLE products
ADD COLUMN brand VARCHAR(100) GENERATED ALWAYS AS (metadata->>'$.brand') VIRTUAL;
CREATE INDEX idx_brand ON products(brand);
```

## 3.6 Practical Table Design

```sql
CREATE TABLE users (
    id           INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    username     VARCHAR(50)  NOT NULL UNIQUE,
    email        VARCHAR(255) NOT NULL UNIQUE,
    full_name    VARCHAR(150) NOT NULL,
    bio          TEXT,
    country_code CHAR(2)      NOT NULL DEFAULT 'BD',
    status       ENUM('active','inactive','banned') NOT NULL DEFAULT 'active',
    age          TINYINT UNSIGNED,
    login_count  INT UNSIGNED NOT NULL DEFAULT 0,
    balance      DECIMAL(12, 2) NOT NULL DEFAULT 0.00,
    date_of_birth DATE,
    created_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at   TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_login_at DATETIME,
    is_verified  TINYINT(1) UNSIGNED NOT NULL DEFAULT 0
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

# 4. Indexes

## 4.1 Index কী এবং কেন?

```
Index ছাড়া (Full Table Scan):
  বইয়ের প্রথম পাতা থেকে শেষ পর্যন্ত পড়া — O(n)

Index দিয়ে (Index Scan):
  বইয়ের পেছনে Index section → সরাসরি page এ যাও — O(log n)
```

## 4.2 B-Tree Structure

MySQL এর default index structure।

```
              [50]
             /    \
       [25]          [75]
      /    \        /    \
  [10,20] [30,40] [60,70] [80,90]

সব value sorted order এ
যেকোনো value খুঁজতে root থেকে leaf এ আসতে হয়
Equality (=), Range (>, <, BETWEEN), Sort (ORDER BY) — সবে কাজ করে
```

## 4.3 Clustered vs Secondary Index

**Clustered Index (Primary Key):**
```
InnoDB এ Primary Key = Clustered Index
Actual row data Primary Key এর order এ disk এ সাজানো

Leaf Node: [id=1 | name='Alice' | age=25 | email='alice@...']
Leaf Node: [id=2 | name='Bob'   | age=30 | email='bob@...' ]

প্রতিটা InnoDB table এ exactly একটা Clustered Index
Primary Key না দিলে MySQL hidden clustered index তৈরি করে
→ সবসময় PRIMARY KEY দাও, INT/BIGINT AUTO_INCREMENT সবচেয়ে ভালো
```

**Secondary Index:**
```
Primary Key ছাড়া বাকি সব index = Secondary Index
Indexed column value + Primary Key store হয়

email Index Leaf: ['alice@...' | id=1]
                  ['bob@...'   | id=2]

Secondary Index দিয়ে query → Double Lookup:
  1. Secondary Index → Primary Key পাও
  2. Primary Key → Clustered Index থেকে full row আনো
```

## 4.4 Index Types

### Regular Index
```sql
CREATE INDEX idx_age ON users(age);
```

### Unique Index
```sql
CREATE UNIQUE INDEX idx_email ON users(email);
-- অথবা: email VARCHAR(255) UNIQUE
```

### Composite Index
```sql
CREATE INDEX idx_country_status ON users(country_code, status);
```

**Left-Prefix Rule — সবচেয়ে গুরুত্বপূর্ণ:**
```sql
-- (A, B, C) index মানে আসলে: (A), (A,B), (A,B,C) কাজ করে
-- (B), (C), (B,C) alone এ কাজ করে না

CREATE INDEX idx_a_b_c ON orders(country, status, created_at);

-- ✅ Index USE হবে
WHERE country = 'BD'
WHERE country = 'BD' AND status = 'active'
WHERE country = 'BD' AND status = 'active' AND created_at > '2024-01-01'

-- ❌ Index USE হবে না
WHERE status = 'active'                          -- leftmost নেই
WHERE created_at > '2024-01-01'                  -- leftmost নেই
WHERE status = 'active' AND created_at > '...'   -- leftmost নেই
```

```
Composite index (country, status, created_at) এর B-Tree:
'BD', 'active',   '2024-01-01'
'BD', 'active',   '2024-02-01'
'BD', 'inactive', '2024-01-01'
'US', 'active',   '2024-01-01'

প্রথমে country sort, তারপর status, তারপর created_at
country ছাড়া শুরু করলে index কাজে আসে না
```

### Covering Index
```sql
-- Query তে যা যা column দরকার সব index এ থাকলে
-- Secondary Index → Clustered Index এ যেতে হয় না (double lookup নেই)
SELECT name, email FROM users WHERE country_code = 'BD';

CREATE INDEX idx_covering ON users(country_code, name, email);
-- EXPLAIN Extra: Using index ← disk এ যেতে হয়নি
```

### Full-Text Index
```sql
CREATE FULLTEXT INDEX idx_ft ON articles(title, content);

-- LIKE '%keyword%' এর চেয়ে অনেক fast
SELECT * FROM articles
WHERE MATCH(title, content) AGAINST('MySQL replication' IN NATURAL LANGUAGE MODE);
```

`LIKE '%keyword%'` কখনো index use করতে পারে না — সবসময় full table scan।

## 4.5 EXPLAIN — Index Analysis

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com'\G
```

**`type` column — Best থেকে Worst:**
```
system  → Table এ exactly 1 row
const   → Primary Key / Unique Key, 1 row match
eq_ref  → JOIN এ Primary Key
ref     → Non-unique index lookup
range   → Index range scan (>, <, BETWEEN, IN)
index   → Full index scan
ALL     → Full Table Scan ← এটা দেখলেই সমস্যা
```

**`Extra` column:**
```
Using index          → Covering index ✅ (disk I/O নেই)
Using where          → WHERE filter apply
Using filesort       → Extra sort লাগছে ⚠️
Using temporary      → Temporary table ⚠️
```

**EXPLAIN ANALYZE (MySQL 8.0+):**
```sql
EXPLAIN ANALYZE SELECT ... \G
-- Actually query চালায়, real time দেখায়
-- নিচ থেকে উপরে পড়তে হয়
```

## 4.6 Index Design Rules

```sql
-- Rule 1: WHERE clause column এ
CREATE INDEX idx_status ON orders(status);

-- Rule 2: JOIN column এ
CREATE INDEX idx_user_id ON orders(user_id);

-- Rule 3: ORDER BY column এ (filesort এড়াতে)
CREATE INDEX idx_created ON products(created_at);

-- Rule 4: High Cardinality column এ বেশি effective
SELECT COUNT(DISTINCT status) / COUNT(*) AS cardinality_ratio FROM orders;
-- 0.01 → 1% unique — index কম কাজে আসবে
-- 0.99 → 99% unique — index অনেক effective

-- Rule 5: Composite index — equality আগে, range পরে
CREATE INDEX idx_user_status_created ON orders(user_id, status, created_at);

-- Rule 6: বেশি index দিও না (5-6 এর বেশি না)
-- প্রতিটা INSERT/UPDATE/DELETE এ সব index update হয় → write slow
```

**Unused Index খোঁজো:**
```sql
SELECT object_name AS table_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND index_name != 'PRIMARY'
  AND count_star = 0
  AND object_schema NOT IN ('mysql', 'performance_schema', 'information_schema');
-- count_star = 0 মানে কখনো use হয়নি → drop করো
```

---

# 5. Transactions & Locking

## 5.1 Transaction কী?

কয়েকটা SQL operation কে একটা single unit হিসেবে treat করা। হয় সবগুলো হবে, নয়তো কিছুই না।

```sql
-- ব্যাংক transfer — দুটো operation একসাথে হতে হবে
START TRANSACTION;
    UPDATE accounts SET balance = balance - 1000 WHERE user = 'Alice';
    UPDATE accounts SET balance = balance + 1000 WHERE user = 'Bob';
COMMIT;
-- মাঝে crash হলে দুটোই rollback → Alice এর টাকা নিরাপদ
```

## 5.2 ACID Properties

**Atomicity:** Transaction এর সব operation হয় পুরোটা হবে, নয়তো কিছুই না।

**Consistency:** Transaction শেষে database সবসময় valid state এ থাকবে। Constraints enforce হবে।

**Isolation:** একটা transaction অন্যের incomplete কাজ দেখতে পাবে না।

**Durability:** COMMIT হলে data permanently saved — crash বা power failure হলেও। InnoDB Redo Log দিয়ে guarantee করে।

## 5.3 Basic Commands

```sql
START TRANSACTION;   -- অথবা BEGIN
COMMIT;
ROLLBACK;

-- Partial rollback
SAVEPOINT my_savepoint;
ROLLBACK TO SAVEPOINT my_savepoint;
RELEASE SAVEPOINT my_savepoint;
```

**autocommit:**
```sql
SHOW VARIABLES LIKE 'autocommit';
-- Default: ON — প্রতিটা statement নিজেই একটা transaction

SET autocommit = 0;  -- Manual commit/rollback লাগবে
```

## 5.4 Isolation Levels

**তিনটা সমস্যা:**

| সমস্যা | মানে |
|---|---|
| Dirty Read | Uncommitted data পড়া |
| Non-Repeatable Read | Same query দুইবার চালালে আলাদা result |
| Phantom Read | Same range query তে নতুন row দেখা |

**Isolation Level Comparison:**

| Level | Dirty Read | Non-Repeatable | Phantom |
|---|---|---|---|
| READ UNCOMMITTED | হয় | হয় | হয় |
| READ COMMITTED | না | হয় | হয় |
| REPEATABLE READ | না | না | না* (InnoDB MVCC) |
| SERIALIZABLE | না | না | না |

*MySQL এর REPEATABLE READ MVCC দিয়ে Phantom Read ও prevent করে।

**MySQL Default: REPEATABLE READ**

```sql
-- Level দেখো
SHOW VARIABLES LIKE 'transaction_isolation';

-- Session এ change করো
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- my.cnf এ
transaction_isolation = REPEATABLE-READ
```

## 5.5 MVCC (Multi-Version Concurrency Control)

InnoDB Lock ছাড়াই consistent read দেওয়ার mechanism।

```
প্রতিটা row এর multiple version রাখা হয় Undo Log এ:
Version 3: balance=3000  ← latest (Transaction C committed)
Version 2: balance=2000  ← (Transaction B committed)
Version 1: balance=1000  ← original

Transaction A (শুরু হয়েছিল balance=1000 এর সময়):
→ Version 1 দেখবে — 1000

Transaction D (এখন শুরু):
→ Version 3 দেখবে — 3000

Reader কখনো Writer কে block করে না
Writer কখনো Reader কে block করে না
→ High concurrency
```

## 5.6 Locking

**Shared Lock (S Lock / Read Lock):**
```sql
SELECT * FROM accounts WHERE id = 1 LOCK IN SHARE MODE;
-- অনেকে একসাথে S Lock নিতে পারে
-- কিন্তু X Lock নেওয়া যাবে না
```

**Exclusive Lock (X Lock / Write Lock):**
```sql
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- একসাথে শুধু একটা transaction
-- অন্য কেউ S বা X lock নিতে পারবে না
```

```
Lock Compatibility:
         S Lock    X Lock
S Lock:  ✅ OK    ❌ Wait
X Lock:  ❌ Wait  ❌ Wait
```

**Row-level vs Table-level:**
```sql
-- Index আছে → শুধু সেই row lock
UPDATE users SET status = 'inactive' WHERE id = 5;  -- row lock ✅

-- Index নেই → full table scan → full table lock
UPDATE users SET status = 'inactive' WHERE name = 'Alice'; -- table lock ⚠️
-- name column এ index না থাকলে সব row lock হয়
```

## 5.7 Deadlock

দুটো transaction একে অপরের জন্য অপেক্ষা করতে করতে আটকে যাওয়া।

```
Transaction A:          Transaction B:
Lock row 1 ✅           Lock row 2 ✅
Wait for row 2 ⏳       Wait for row 1 ⏳
        └──── Deadlock ──────┘
```

MySQL automatically detect করে এবং একটাকে victim বানিয়ে rollback করে:
```
ERROR 1213: Deadlock found when trying to get lock
```

**Deadlock এড়ানোর উপায়:**
```sql
-- 1. সবসময় একই order এ rows access করো
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- always id=1 first
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- then id=2

-- 2. Transaction ছোট রাখো — শুধু DB কাজ transaction এ
-- API call, file write transaction এর বাইরে

-- 3. Deadlock log দেখো
SHOW ENGINE INNODB STATUS\G
-- LATEST DETECTED DEADLOCK section
```

## 5.8 FOR UPDATE — Pessimistic Locking

```sql
START TRANSACTION;
-- Stock পড়ো এবং সাথে lock নাও
SELECT stock FROM inventory WHERE product_id = 10 FOR UPDATE;
-- অন্য transaction এই row update করতে পারবে না
UPDATE inventory SET stock = stock - 1 WHERE product_id = 10;
INSERT INTO orders (product_id, user_id) VALUES (10, 5);
COMMIT;
-- FOR UPDATE ছাড়া: দুটো transaction same stock দেখতে পারে → oversell
```

## 5.9 Lock Monitoring

```sql
-- এখন কোন lock আছে
SELECT * FROM performance_schema.data_locks\G

-- কোন transaction কোন lock এর জন্য wait করছে
SELECT * FROM performance_schema.data_lock_waits\G

-- Long running transaction দেখো
SELECT trx_id, trx_state, trx_started, trx_mysql_thread_id, trx_query
FROM information_schema.innodb_trx
ORDER BY trx_started\G

SHOW VARIABLES LIKE 'innodb_lock_wait_timeout'; -- default 50s
```

---

# 6. Query Optimization

## 6.1 Slow Query খোঁজা

```ini
# my.cnf
slow_query_log          = ON
slow_query_log_file     = /var/log/mysql/slow.log
long_query_time         = 1
log_queries_not_using_indexes = ON
```

```bash
sudo mysqldumpslow -s t -t 10 /var/log/mysql/slow.log
# top 10 slowest by time
```

```sql
-- Performance Schema দিয়ে
SELECT
    DIGEST_TEXT                         AS query,
    COUNT_STAR                          AS exec_count,
    ROUND(AVG_TIMER_WAIT/1000000000, 3) AS avg_seconds,
    SUM_ROWS_EXAMINED                   AS total_rows_examined,
    SUM_ROWS_SENT                       AS total_rows_sent
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
-- total_rows_examined >> total_rows_sent → index দরকার
```

## 6.2 EXPLAIN এর সব Column

```sql
EXPLAIN SELECT * FROM orders
WHERE user_id = 5 AND status = 'pending'
ORDER BY created_at DESC LIMIT 10\G
```

| Column | মানে |
|---|---|
| `id` | Query step number |
| `type` | Access type (ALL=worst, const=best) |
| `possible_keys` | Use হতে পারে এমন index গুলো |
| `key` | Actually use হওয়া index |
| `rows` | Estimated rows to scan |
| `Extra` | Additional info |

**`type` values (best→worst):** `system` → `const` → `eq_ref` → `ref` → `range` → `index` → `ALL`

**`Extra` values:**
- `Using index` → Covering index ✅
- `Using filesort` → Extra sort ⚠️
- `Using temporary` → Temp table ⚠️

## 6.3 Common Slow Patterns এবং Fix

### Pattern 1 — Full Table Scan
```sql
-- ❌
SELECT * FROM orders WHERE user_id = 5;
-- type=ALL, rows=500000

-- ✅
CREATE INDEX idx_user_id ON orders(user_id);
-- type=ref, rows=47
```

### Pattern 2 — LIKE শুরুতে Wildcard
```sql
-- ❌ index কাজ করে না
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ শেষে wildcard হলে index কাজ করে
SELECT * FROM products WHERE name LIKE 'phone%';

-- ✅ Full-text search
CREATE FULLTEXT INDEX idx_ft ON products(name);
SELECT * FROM products WHERE MATCH(name) AGAINST('phone');
```

### Pattern 3 — Column এ Function
```sql
-- ❌ index কাজ করে না
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';

-- ✅
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
SELECT * FROM users WHERE email = 'alice@example.com';
```

### Pattern 4 — Implicit Type Conversion
```sql
-- ❌ user_id INT কিন্তু string দিলে index কাজ করে না
SELECT * FROM users WHERE user_id = '5';

-- ✅
SELECT * FROM users WHERE user_id = 5;
```

### Pattern 5 — SELECT *
```sql
-- ❌
SELECT * FROM users WHERE country_code = 'BD';

-- ✅ দরকারি column আনো → Covering index সম্ভব
SELECT id, name, email FROM users WHERE country_code = 'BD';

CREATE INDEX idx_covering ON users(country_code, id, name, email);
-- Extra: Using index ✅
```

### Pattern 6 — N+1 Query
```sql
-- ❌ N+1 — প্রতিটা user এর জন্য আলাদা query
SELECT id FROM users WHERE country = 'BD';
-- তারপর loop এ 100 বার:
SELECT COUNT(*) FROM orders WHERE user_id = ?;

-- ✅ একটা JOIN query
SELECT u.id, u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.country = 'BD'
GROUP BY u.id, u.name;
```

### Pattern 7 — Large OFFSET Pagination
```sql
-- ❌ 1 million + 10 row scan করে 10 return করে
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 1000000;

-- ✅ Keyset Pagination
SELECT * FROM orders
WHERE id > 1000000     -- আগের page এর last id
ORDER BY id LIMIT 10;
```

### Pattern 8 — Correlated Subquery
```sql
-- ❌ প্রতিটা row এর জন্য subquery
SELECT * FROM users u
WHERE (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) > 5;

-- ✅ JOIN দিয়ে
SELECT u.*
FROM users u
INNER JOIN (
    SELECT user_id FROM orders
    GROUP BY user_id HAVING COUNT(*) > 5
) active ON u.id = active.user_id;
```

## 6.4 Composite Index এ ORDER BY Optimization

```sql
-- এই query তে
SELECT * FROM orders WHERE user_id = 5 ORDER BY created_at DESC;

-- এই composite index WHERE এবং ORDER BY দুটোই cover করে
CREATE INDEX idx_user_created ON orders(user_id, created_at);
-- Extra: Using index condition (filesort নেই!)
```

## 6.5 JOIN Optimization

```sql
-- প্রতিটা JOIN column এ index লাগবে
CREATE INDEX idx_orders_user_id   ON orders(user_id);
CREATE INDEX idx_oi_order_id      ON order_items(order_id);
CREATE INDEX idx_oi_product_id    ON order_items(product_id);
```

MySQL smaller result set আগে join করে। `EXPLAIN` এ table order দেখলে বোঝা যায়।

## 6.6 Optimizer Hints

```sql
-- Specific index force করো
SELECT * FROM orders FORCE INDEX (idx_user_id) WHERE user_id = 5;

-- Index ignore করো
SELECT * FROM orders IGNORE INDEX (idx_status) WHERE status = 'pending';
```

## 6.7 Optimization Checklist

```
Query লেখার সময়:
  ✅ SELECT * এড়াও
  ✅ WHERE column এ function ব্যবহার করো না
  ✅ Correct data type দিয়ে compare করো
  ✅ N+1 query এড়াও, JOIN ব্যবহার করো
  ✅ বড় OFFSET এর বদলে Keyset Pagination
  ✅ Correlated Subquery এড়াও

Index দেওয়ার সময়:
  ✅ WHERE, JOIN, ORDER BY, GROUP BY column এ index
  ✅ Composite index এ equality column আগে, range পরে
  ✅ Covering index দিলে disk I/O শূন্য
  ✅ Low cardinality column এ index এড়াও

Verify করার সময়:
  ✅ EXPLAIN এ type=ALL নেই
  ✅ EXPLAIN Extra তে Using filesort নেই
  ✅ EXPLAIN ANALYZE দিয়ে actual time confirm
  ✅ Buffer Pool hit rate 99%+
```

---

# 7. Backup & Recovery

## 7.1 Backup Types

```
Logical Backup (mysqldump):
  SQL statements export
  Portable, human-readable
  Slow for large databases

Physical Backup (XtraBackup):
  Raw data files copy
  Fast, not portable across versions
  Best for large databases (100GB+)

Full Backup:    সব data, বেশি space, restore সহজ
Incremental:    Last backup এর পর যা change → binlog
```

## 7.2 mysqldump — Production Command

```bash
mysqldump \
  -u root -p \
  --all-databases \
  --single-transaction \
  --master-data=2 \
  --flush-logs \
  --routines \
  --triggers \
  --events \
  --hex-blob \
  | gzip > /backup/full_backup_$(date +%Y%m%d_%H%M%S).sql.gz
```

| Option | কাজ |
|---|---|
| `--single-transaction` | InnoDB snapshot, table lock হয় না |
| `--master-data=2` | Dump এ binlog position comment হিসেবে লেখে |
| `--flush-logs` | Dump আগে নতুন binlog শুরু করে |
| `--routines` | Stored procedures ও include করে |
| `--hex-blob` | BLOB data corruption এড়ায় |

## 7.3 Restore

```bash
# Full restore
mysql -u root -p < /backup/full_backup.sql

# নির্দিষ্ট database
mysql -u root -p database_name < backup.sql

# Progress দেখতে
pv /backup/full_backup.sql | mysql -u root -p
```

## 7.4 Binary Log Backup

```bash
# Binlog files copy করো
cp /data/mysql/mysql-bin.* /backup/binlog/

# Readable format এ export
mysqlbinlog \
  /data/mysql/mysql-bin.000003 \
  /data/mysql/mysql-bin.000004 \
  > /backup/binlog/binlog_003_004.sql
```

## 7.5 Point-in-Time Recovery (PITR)

**Scenario:** সকাল ১০টায় ভুলে `DELETE FROM orders` হয়েছে। রাত ২টার backup আছে।

```bash
# Step 1: Full backup restore
mysql -u root -p < /backup/full_backup_02:00.sql

# Step 2: Backup এর binlog position দেখো
grep "CHANGE REPLICATION" /backup/full_backup_02:00.sql
# CHANGE REPLICATION SOURCE TO SOURCE_LOG_FILE='mysql-bin.000003', SOURCE_LOG_POS=1234;

# Step 3: DELETE হওয়ার ঠিক আগের position খোঁজো
mysqlbinlog --start-datetime="2024-03-08 02:00:00" \
  /data/mysql/mysql-bin.000003 \
  | grep -i "DELETE FROM orders" -A 5 -B 5
# at 45678 ← এই position এর ঠিক আগে stop

# Step 4: DELETE এর আগ পর্যন্ত apply করো
mysqlbinlog \
  --start-position=1234 \
  --stop-position=45677 \
  /data/mysql/mysql-bin.000003 \
  | mysql -u root -p

# অথবা time দিয়ে
mysqlbinlog \
  --start-datetime="2024-03-08 02:00:00" \
  --stop-datetime="2024-03-08 09:59:59" \
  /data/mysql/mysql-bin.000003 \
  | mysql -u root -p
```

## 7.6 XtraBackup (Large Database)

```bash
# Install
sudo dnf install -y percona-xtrabackup-84

# Full backup
xtrabackup --backup --user=root --password=xxx \
  --target-dir=/backup/xtrabackup/full_$(date +%Y%m%d)

# Prepare (restore এর আগে লাগে)
xtrabackup --prepare --target-dir=/backup/xtrabackup/full_20240308

# Restore
sudo systemctl stop mysqld
sudo rm -rf /data/mysql/*
xtrabackup --copy-back --target-dir=/backup/xtrabackup/full_20240308
sudo chown -R mysql:mysql /data/mysql
sudo systemctl start mysqld
```

## 7.7 Automated Backup Script

```bash
#!/bin/bash
BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/mysql_backup.log"

mkdir -p $BACKUP_DIR/daily $BACKUP_DIR/binlog

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE; }

log "Starting full backup..."
mysqldump \
    -u root -p"your_password" \
    --all-databases \
    --single-transaction \
    --master-data=2 \
    --flush-logs \
    --routines --triggers --events \
    2>>$LOG_FILE \
    | gzip > $BACKUP_DIR/daily/full_backup_${DATE}.sql.gz

[ $? -eq 0 ] && log "Backup completed" || { log "ERROR: Backup failed!"; exit 1; }

cp /data/mysql/mysql-bin.* $BACKUP_DIR/binlog/

# 7 দিনের পুরনো backup delete করো
find $BACKUP_DIR/daily -name "*.sql.gz" -mtime +7 -delete
find $BACKUP_DIR/binlog -name "mysql-bin.*" -mtime +7 -delete
```

```bash
# Cron — প্রতিদিন রাত ২টায়
0 2 * * * /usr/local/bin/mysql_backup.sh
```

## 7.8 Backup Best Practices

```
Strategy:
  ✅ 3-2-1 Rule: 3 copies, 2 different media, 1 offsite
  ✅ Daily full backup + continuous binlog backup
  ✅ Monthly restore test (আলাদা server এ)
  ✅ Backup এর পর verify করো

Security:
  ✅ Password credential file এ রাখো (~/.my.cnf, chmod 600)
  ✅ Backup encrypt করো

Performance:
  ✅ Replica থেকে backup নাও — Master এ load নেই
  ✅ Off-peak time এ (রাত ২-৩টা)
  ✅ Compress করো (gzip)
```

---

# 8. Monitoring

## 8.1 OS Level

```bash
# CPU
top -u mysql

# Memory
free -h

# Disk
df -h /data/mysql
iostat -x 1 5

# MySQL process
ps aux | grep mysqld
```

**Critical threshold:** Data directory 80% ভরলে সতর্ক, 90% এ MySQL crash করতে পারে।

## 8.2 Connection Monitoring

```sql
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';
SHOW VARIABLES LIKE 'max_connections';

-- Connection utilization
SELECT
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME = 'Max_used_connections') AS max_used,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_variables
     WHERE VARIABLE_NAME = 'max_connections') AS max_allowed;
-- 80%+ হলে max_connections বাড়াও

-- Active connections দেখো
SELECT user, host, db, command, time, state, LEFT(info, 100) AS query
FROM information_schema.processlist
ORDER BY time DESC;
```

## 8.3 Buffer Pool Monitoring

```sql
-- Hit rate (99%+ হওয়া উচিত)
SELECT ROUND(
    (1 - (SELECT VARIABLE_VALUE FROM performance_schema.global_status
          WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
         NULLIF((SELECT VARIABLE_VALUE FROM performance_schema.global_status
          WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'), 0)
    ) * 100, 2) AS hit_rate_pct;

-- Usage
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_pages%';
```

## 8.4 Query Performance Monitoring

```sql
-- Slow query count
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- Full table scan count
SHOW GLOBAL STATUS LIKE 'Select_scan';

-- Top slow queries
SELECT
    DIGEST_TEXT                         AS query,
    COUNT_STAR                          AS exec_count,
    ROUND(AVG_TIMER_WAIT/1000000000, 3) AS avg_sec,
    SUM_ROWS_EXAMINED                   AS rows_examined,
    SUM_ROWS_SENT                       AS rows_returned
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
```

## 8.5 Replication Monitoring

```sql
-- Replica তে
SHOW REPLICA STATUS\G
```

| Field | Healthy | সমস্যা |
|---|---|---|
| `Replica_IO_Running` | `Yes` | Master connection নেই |
| `Replica_SQL_Running` | `Yes` | SQL error আছে |
| `Seconds_Behind_Source` | `0` | Replica lag করছে |
| `Last_IO_Error` | empty | Connection/Auth error |
| `Last_SQL_Error` | empty | Data conflict |

```bash
#!/bin/bash
# Replication health check script
LAG=$(mysql -u root -p"pass" -e "SHOW REPLICA STATUS\G" 2>/dev/null \
      | grep "Seconds_Behind_Source" | awk '{print $2}')
IO=$(mysql -u root -p"pass" -e "SHOW REPLICA STATUS\G" 2>/dev/null \
     | grep "Replica_IO_Running" | awk '{print $2}')
SQL=$(mysql -u root -p"pass" -e "SHOW REPLICA STATUS\G" 2>/dev/null \
     | grep "Replica_SQL_Running" | awk '{print $2}')

[ "$IO" != "Yes" ]  && echo "CRITICAL: IO Thread not running!"
[ "$SQL" != "Yes" ] && echo "CRITICAL: SQL Thread not running!"
[ "$LAG" -gt 60 ]   && echo "WARNING: Replication lag ${LAG}s"
```

## 8.6 Database Size Monitoring

```sql
-- Database sizes
SELECT
    table_schema AS db,
    ROUND(SUM(data_length + index_length)/1024/1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
GROUP BY table_schema
ORDER BY size_mb DESC;

-- Table sizes
SELECT
    table_name,
    ROUND(data_length/1024/1024, 2)                  AS data_mb,
    ROUND(index_length/1024/1024, 2)                 AS index_mb,
    ROUND((data_length+index_length)/1024/1024, 2)   AS total_mb,
    table_rows
FROM information_schema.tables
WHERE table_schema = 'your_database'
ORDER BY (data_length+index_length) DESC;
```

## 8.7 InnoDB Deep Diagnostic

```sql
SHOW ENGINE INNODB STATUS\G
-- TRANSACTIONS      → Long running transactions
-- BUFFER POOL       → Hit rate, dirty pages
-- ROW OPERATIONS    → rows/sec insert/update/delete
-- LATEST DEADLOCK   → শেষ deadlock details
```

## 8.8 Index Usage Monitoring

```sql
-- কোন index কখনো use হয়নি (drop candidate)
SELECT object_schema AS db, object_name AS table_name, index_name
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL
  AND index_name != 'PRIMARY'
  AND count_star = 0
  AND object_schema NOT IN ('mysql','performance_schema','information_schema')
ORDER BY object_schema, object_name;

-- কোন table এ সবচেয়ে বেশি read হচ্ছে
SELECT object_schema AS db, object_name AS table_name,
       count_read, count_write
FROM performance_schema.table_io_waits_summary_by_table
WHERE object_schema NOT IN ('mysql','performance_schema','information_schema')
ORDER BY count_read DESC LIMIT 10;
```

## 8.9 Monitoring Dashboard Script

```bash
#!/bin/bash
# /usr/local/bin/mysql_health_check.sh
MYSQL="mysql -u root -p'your_password' -e"

echo "========================================="
echo " MySQL Health Check - $(date)"
echo "========================================="

echo "--- CONNECTIONS ---"
$MYSQL "SELECT
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME='Threads_connected') AS connected,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_variables
     WHERE VARIABLE_NAME='max_connections') AS max_allowed,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME='Max_used_connections') AS peak;" 2>/dev/null

echo "--- BUFFER POOL HIT RATE ---"
$MYSQL "SELECT ROUND((1 - (
    SELECT VARIABLE_VALUE FROM performance_schema.global_status
    WHERE VARIABLE_NAME='Innodb_buffer_pool_reads') /
    NULLIF((SELECT VARIABLE_VALUE FROM performance_schema.global_status
    WHERE VARIABLE_NAME='Innodb_buffer_pool_read_requests'),0)) * 100, 2)
    AS hit_rate_pct;" 2>/dev/null

echo "--- REPLICATION STATUS ---"
$MYSQL "SHOW REPLICA STATUS\G" 2>/dev/null \
    | grep -E "Replica_IO_Running|Replica_SQL_Running|Seconds_Behind_Source|Last_IO_Error|Last_SQL_Error"

echo "--- DISK USAGE ---"
df -h /data/mysql | tail -1

echo "--- TOP 5 LARGEST TABLES ---"
$MYSQL "SELECT table_schema AS db, table_name,
    ROUND((data_length+index_length)/1024/1024, 2) AS total_mb, table_rows
FROM information_schema.tables
WHERE table_schema NOT IN ('mysql','information_schema','performance_schema','sys')
ORDER BY (data_length+index_length) DESC LIMIT 5;" 2>/dev/null
```

```bash
chmod +x /usr/local/bin/mysql_health_check.sh
# Cron এ প্রতি 5 মিনিটে
*/5 * * * * /usr/local/bin/mysql_health_check.sh >> /var/log/mysql_health.log 2>&1
```

## 8.10 Key Metrics Thresholds

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| Buffer Pool Hit Rate | > 99% | 95-99% | < 95% |
| Connection Utilization | < 70% | 70-85% | > 85% |
| Replication Lag | 0-5s | 5-60s | > 60s |
| Disk Usage | < 70% | 70-85% | > 85% |
| Slow Query Rate | < 0.1% | 0.1-1% | > 1% |
| Tmp Disk Table Rate | < 5% | 5-20% | > 20% |

---

# 9. Security

## 9.1 Security Layers

```
Layer 4 — Audit: কে কখন কী করেছে track
Layer 3 — Data Security: Encryption at rest, SSL/TLS in transit
Layer 2 — Access Control: Privileges, Roles
Layer 1 — Authentication: কে connect করতে পারবে
```

## 9.2 Authentication

**MySQL User = username + host:**
```sql
'root'@'localhost'        -- শুধু same machine
'root'@'%'               -- যেকোনো IP (dangerous!)
'appuser'@'10.0.0.%'     -- 10.0.0.x range
'replica'@'172.16.93.141' -- specific IP only

-- PostgreSQL এ pg_hba.conf আলাদা file
-- MySQL এ host restriction user definition এ @'host' দিয়েই হয়
```

```sql
-- User তৈরি
CREATE USER 'appuser'@'10.0.0.5' IDENTIFIED BY 'AppPass@2024!';

-- Password policy
SHOW VARIABLES LIKE 'validate_password%';

-- Password expiry
ALTER USER 'appuser'@'10.0.0.5' PASSWORD EXPIRE INTERVAL 90 DAY;
ALTER USER 'replica'@'172.16.93.141' PASSWORD EXPIRE NEVER;

-- Account lock
ALTER USER 'appuser'@'10.0.0.5' ACCOUNT LOCK;
ALTER USER 'appuser'@'10.0.0.5' ACCOUNT UNLOCK;

-- Failed login auto-lock
ALTER USER 'appuser'@'10.0.0.5'
FAILED_LOGIN_ATTEMPTS 5
PASSWORD_LOCK_TIME 1;
```

## 9.3 Privileges

**Hierarchy:** GLOBAL → DATABASE → TABLE → COLUMN → ROUTINE

```sql
-- Application user — specific database
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp_db.* TO 'appuser'@'10.0.0.5';

-- Read-only user
GRANT SELECT ON myapp_db.* TO 'reporter'@'10.0.0.%';

-- Specific table
GRANT SELECT, INSERT ON myapp_db.orders TO 'order_service'@'10.0.0.6';

-- Column level (sensitive data restrict করো)
GRANT SELECT (id, name, email) ON myapp_db.users TO 'limited'@'localhost';
-- salary column দেখতে পারবে না

-- Revoke
REVOKE DELETE ON myapp_db.* FROM 'appuser'@'10.0.0.5';
REVOKE ALL PRIVILEGES ON myapp_db.* FROM 'appuser'@'10.0.0.5';

-- Privileges দেখো
SHOW GRANTS FOR 'appuser'@'10.0.0.5';
```

## 9.4 Roles (MySQL 8.0+)

```sql
-- Role তৈরি
CREATE ROLE 'app_read', 'app_write', 'app_admin';

-- Role এ privilege
GRANT SELECT ON myapp_db.* TO 'app_read';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp_db.* TO 'app_write';

-- User কে role
GRANT 'app_write' TO 'appuser'@'10.0.0.5';

-- Default role (login এ auto-active)
SET DEFAULT ROLE 'app_write' TO 'appuser'@'10.0.0.5';

-- সুবিধা: নতুন developer → GRANT 'app_write' TO 'newdev'@'...'
-- আলাদাভাবে সব privilege দিতে হয় না
```

## 9.5 Root User Hardening

```sql
-- Remote root access বন্ধ করো
DELETE FROM mysql.user WHERE user = 'root' AND host != 'localhost';

-- Anonymous user remove করো
DELETE FROM mysql.user WHERE user = '';

-- Test database remove করো
DROP DATABASE IF EXISTS test;

FLUSH PRIVILEGES;
```

## 9.6 SSL/TLS

```sql
SHOW VARIABLES LIKE 'have_ssl';
SHOW STATUS LIKE 'Ssl_cipher';  -- Empty মানে SSL নেই

-- User এ SSL require করো
ALTER USER 'appuser'@'10.0.0.5' REQUIRE SSL;
```

## 9.7 Data Encryption

```sql
-- Table level encryption
CREATE TABLE sensitive_data (
    id     INT AUTO_INCREMENT PRIMARY KEY,
    ssn    VARCHAR(20)
) ENCRYPTION='Y';

ALTER TABLE sensitive_data ENCRYPTION='Y';
```

**Application level:** Sensitive data (password, card) database এ plain text রাখবে না। Application এ hash/encrypt করো।

## 9.8 Network Security (my.cnf)

```ini
[mysqld]
bind-address   = 172.16.93.140  -- specific interface
symbolic-links = 0              -- symlink attack বন্ধ
local-infile   = 0              -- local file read বন্ধ
```

```bash
# Firewall — শুধু trusted IP allow
sudo firewall-cmd --permanent --add-rich-rule='
  rule family=ipv4 source address=10.0.0.0/24
  port port=3306 protocol=tcp accept'
sudo firewall-cmd --reload
```

## 9.9 Security Checklist

```
Authentication:
  ✅ root শুধু localhost
  ✅ Anonymous user নেই
  ✅ Strong password policy
  ✅ Password expiry policy
  ✅ Wildcard host (%) এ unnecessary user নেই

Privileges:
  ✅ Least privilege principle
  ✅ Application user এর শুধু দরকারি privilege
  ✅ Unnecessary SUPER privilege নেই
  ✅ Role দিয়ে privilege manage

Network:
  ✅ bind-address specific IP
  ✅ Firewall trusted IP only
  ✅ local-infile বন্ধ

Data:
  ✅ Sensitive column encrypt
  ✅ Application level password hashing
  ✅ Backup encrypt

Audit:
  ✅ Audit/Binary logging enabled
  ✅ Regular privilege audit
```

---

# 10. High Availability & Replication

## 10.1 Core Concepts

**RPO (Recovery Point Objective):** কতটুকু data loss acceptable? → Backup frequency এবং replication lag এটা নির্ধারণ করে।

**RTO (Recovery Time Objective):** কতক্ষণের মধ্যে service restore হবে? → Failover mechanism এটা নির্ধারণ করে।

## 10.2 Replication কীভাবে কাজ করে

```
Master এ:
  INSERT/UPDATE/DELETE
      │
      ▼
  Binary Log (mysql-bin.000001)
      │
      │ streams over Port 3306
      ▼
Replica এ:
  I/O Thread → reads binlog → Relay Log
  SQL Thread → reads Relay Log → applies to database
```

## 10.3 Topology 1 — Master + Single Replica

তুমি এটাই করেছো। Simplest setup।

```
Master (172.16.93.140)  ──────►  Replica (172.16.93.141)
Read/Write                       Read Only
```

**Manual Failover:**
```sql
-- Replica এ
STOP REPLICA;
SET GLOBAL read_only = OFF;
SET GLOBAL super_read_only = OFF;
-- Application এ connection এই নতুন Master এ point করো
```

## 10.4 Topology 2 — Multi-Replica

```
                  Master (172.16.93.140)
                  Read/Write
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
     Replica 1      Replica 2      Replica 3
     Read Only      Read Only      Backup Only
```

Read traffic Replica গুলোতে distribute করো। Backup Replica থেকে নাও — Master এ load নেই।

## 10.5 Topology 3 — Master-Master (Active-Passive HA)

দুটো server একে অপরের Master এবং Replica।

```
┌─────────────────┐  ◄────────────►  ┌─────────────────┐
│    Master 1     │                   │    Master 2     │
│  Active         │                   │  Passive Standby│
│ 172.16.93.140   │                   │ 172.16.93.141   │
│ server-id=1     │                   │ server-id=2     │
│ offset=1, inc=2 │                   │ offset=2, inc=2 │
└─────────────────┘                   └─────────────────┘
```

**AUTO_INCREMENT conflict এড়াতে:**
```ini
# Master 1
auto_increment_offset    = 1   # generates: 1, 3, 5, 7... (odd)
auto_increment_increment = 2

# Master 2
auto_increment_offset    = 2   # generates: 2, 4, 6, 8... (even)
auto_increment_increment = 2
```

দুটো server এ একই id কখনো হবে না।

```sql
-- Master 2 তে Master 1 কে source করো
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='172.16.93.140', SOURCE_USER='replica',
  SOURCE_PASSWORD='Replica@123', SOURCE_LOG_FILE='mysql-bin.000001',
  SOURCE_LOG_POS=4, GET_SOURCE_PUBLIC_KEY=1;
START REPLICA;

-- Master 1 তে Master 2 কে source করো
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='172.16.93.141', SOURCE_USER='replica',
  SOURCE_PASSWORD='Replica@123', SOURCE_LOG_FILE='mysql-bin.000001',
  SOURCE_LOG_POS=4, GET_SOURCE_PUBLIC_KEY=1;
START REPLICA;
```

Production এ সাধারণত: একটা Active (write নেয়), অন্যটা Passive (শুধু sync হয়, failover ready)। দুটোতেই write করলে conflict risk বাড়ে।

## 10.6 MySQL Install থেকে Replication Setup — End-to-End

### উভয় Server এ (Install + Secure + Data Directory Change)

```bash
# Step 1: System update
sudo dnf update -y

# Step 2: MySQL repository
sudo dnf install -y https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm

# Step 3: Install
sudo dnf install -y mysql-community-server

# Step 4: Start
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Step 5: Temporary password
sudo grep 'temporary password' /var/log/mysqld.log

# Step 6: Secure installation
sudo mysql_secure_installation
# → নতুন root password দাও
# → Anonymous users remove (Y)
# → Remote root disallow (Y)
# → Test database remove (Y)
# → Reload privileges (Y)

# Step 7: Data directory change
sudo systemctl stop mysqld
sudo dnf install -y rsync
sudo mkdir -p /data/mysql
sudo rsync -av /var/lib/mysql/ /data/mysql/
sudo chown -R mysql:mysql /data/mysql
sudo chmod 750 /data/mysql

# Step 8: SELinux (Rocky Linux এ অবশ্যই)
sudo dnf install -y policycoreutils-python-utils
sudo semanage fcontext -a -t mysqld_db_t "/data/mysql(/.*)?"
sudo restorecon -Rv /data/mysql
```

```ini
# Step 9: /etc/my.cnf update করো
[mysqld]
datadir=/data/mysql
socket=/data/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

[client]
socket=/data/mysql/mysql.sock
```

```bash
sudo systemctl start mysqld

# Verify
mysql -u root -p -e "SHOW VARIABLES LIKE 'datadir';"
```

### Master Server এ (172.16.93.140)

```ini
# /etc/my.cnf এ [mysqld] section এ add করো
server-id                  = 1
log_bin                    = /data/mysql/mysql-bin.log
binlog_format              = ROW
binlog_expire_logs_seconds = 604800
sync_binlog                = 1
```

```bash
sudo systemctl restart mysqld
mysql -u root -p -e "SHOW VARIABLES LIKE 'log_bin';"
# log_bin = ON হলে সফল
```

```sql
-- Replication user তৈরি (MySQL 8.4 এ caching_sha2_password)
CREATE USER 'replica'@'172.16.93.141'
IDENTIFIED WITH caching_sha2_password BY 'Replica@123';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'172.16.93.141';
FLUSH PRIVILEGES;

-- User verify
SELECT user, host, plugin FROM mysql.user WHERE user = 'replica';

-- Binlog position নোট করো (এই দুটো value দরকার)
SHOW BINARY LOG STATUS\G
-- File: mysql-bin.000001
-- Position: 887

-- Firewall
```

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=172.16.93.141 port port=3306 protocol=tcp accept'
sudo firewall-cmd --reload
```

### Replica Server এ (172.16.93.141)

```ini
# /etc/my.cnf এ add করো
server-id          = 2
read_only          = ON
relay_log          = /data/mysql/relay-bin.log
relay_log_recovery = ON
```

```bash
sudo systemctl restart mysqld
```

```sql
-- Master এর সাথে connect করো
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST           = '172.16.93.140',
  SOURCE_USER           = 'replica',
  SOURCE_PASSWORD       = 'Replica@123',
  SOURCE_LOG_FILE       = 'mysql-bin.000001',
  SOURCE_LOG_POS        = 887,
  GET_SOURCE_PUBLIC_KEY = 1;
-- GET_SOURCE_PUBLIC_KEY = 1 → MySQL 8.4 এ caching_sha2_password SSL ছাড়া কাজ করাতে

START REPLICA;

-- Verify
SHOW REPLICA STATUS\G
-- Replica_IO_Running: Yes
-- Replica_SQL_Running: Yes
-- Seconds_Behind_Source: 0
```

### Live Test

```sql
-- Master এ
CREATE DATABASE repl_test;
USE repl_test;
CREATE TABLE t1 (id INT AUTO_INCREMENT PRIMARY KEY, msg VARCHAR(100));
INSERT INTO t1 (msg) VALUES ('Replication works!');

-- Replica তে
USE repl_test;
SELECT * FROM t1;
-- Row দেখা গেলে Replication সফল ✅
```

## 10.7 Troubleshooting

### Replica_IO_Running: No / Connecting

```sql
SHOW REPLICA STATUS\G  -- Last_IO_Error দেখো
```

| Error | Cause | Solution |
|---|---|---|
| `Authentication requires secure connection` | GET_SOURCE_PUBLIC_KEY নেই | `GET_SOURCE_PUBLIC_KEY = 1` যোগ করো |
| `Access denied for user 'replica'` | Wrong password/user | Master এ user recreate করো |
| `Can't connect to MySQL server` | Network/Firewall | Master এ port 3306 open করো |

```sql
-- Reset এবং reconfigure
STOP REPLICA;
RESET REPLICA ALL;
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='172.16.93.140', SOURCE_USER='replica',
  SOURCE_PASSWORD='Replica@123', SOURCE_LOG_FILE='mysql-bin.000001',
  SOURCE_LOG_POS=887, GET_SOURCE_PUBLIC_KEY=1;
START REPLICA;
```

### Replica_SQL_Running: No

```sql
SHOW REPLICA STATUS\G  -- Last_SQL_Error দেখো

-- Error skip করো (সাবধানে!)
STOP REPLICA;
SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;
START REPLICA;
-- Repeat যতক্ষণ SQL_Running = Yes
```

### Plugin 'mysql_native_password' is not loaded

MySQL 8.4 এ `mysql_native_password` সম্পূর্ণ removed।

```sql
-- caching_sha2_password ব্যবহার করো
CREATE USER 'replica'@'172.16.93.141'
IDENTIFIED WITH caching_sha2_password BY 'Replica@123';
-- Replica তে GET_SOURCE_PUBLIC_KEY = 1 দাও
```

## 10.8 GTID Based Replication

Traditional: binlog file + position দিয়ে track।
GTID: প্রতিটা transaction এর globally unique ID।

```
Traditional:  FILE=mysql-bin.000003, POS=1234
GTID:         3a09760c-1aaf-11f1-ba7c-000c29822ac7:1-100
              └──────────────────────┘ └──────────┘
                   Server UUID           Transaction range
```

**GTID এর সুবিধা:** Failover এ manually position খুঁজতে হয় না। MySQL নিজেই জানে কোথা থেকে শুরু করবে।

```ini
# উভয় server এ my.cnf
[mysqld]
gtid_mode                = ON
enforce_gtid_consistency = ON
log_bin                  = /data/mysql/mysql-bin.log
binlog_format            = ROW
```

```sql
-- GTID তে SOURCE_LOG_FILE/POS লাগে না
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST           = '172.16.93.140',
  SOURCE_USER           = 'replica',
  SOURCE_PASSWORD       = 'Replica@123',
  SOURCE_AUTO_POSITION  = 1,
  GET_SOURCE_PUBLIC_KEY = 1;
-- SOURCE_AUTO_POSITION = 1 → Replica নিজেই বুঝে নেবে কোথা থেকে শুরু করবে
```

## 10.9 ProxySQL — Read/Write Split

Application এবং MySQL এর মধ্যে intelligent proxy।

```
Application
    │
    ▼
ProxySQL (port 6033)
    │
    ├── SELECT          → Replica (hostgroup 20)
    └── INSERT/UPDATE   → Master  (hostgroup 10)
```

```sql
-- ProxySQL Admin (port 6032) এ
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES
(10, '172.16.93.140', 3306),
(20, '172.16.93.141', 3306);

INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup)
VALUES (1, 1, '^SELECT', 20);

LOAD MYSQL SERVERS TO RUNTIME; SAVE MYSQL SERVERS TO DISK;
LOAD MYSQL QUERY RULES TO RUNTIME; SAVE MYSQL QUERY RULES TO DISK;
```

## 10.10 Group Replication

Automatic Primary election, fault tolerant।

```
        ┌──────────┐
        │  Node 1  │ Primary (Read/Write)
        │          │
        └─────┬────┘
              │ Paxos Consensus
   ┌──────────┴──────────┐
   ▼                     ▼
┌──────────┐       ┌──────────┐
│  Node 2  │       │  Node 3  │
│Secondary │       │Secondary │
└──────────┘       └──────────┘

3 node → 1 failure tolerate করে
5 node → 2 failure tolerate করে
```

```ini
[mysqld]
plugin_load_add                 = group_replication.so
group_replication_group_name    = "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
group_replication_local_address = "172.16.93.140:33061"
group_replication_group_seeds   = "172.16.93.140:33061,172.16.93.141:33061"
```

## 10.11 Replication Topology Comparison

| Topology | Setup | Failover | Use Case |
|---|---|---|---|
| Master + Replica | Easy | Manual | Small app |
| Multi-Replica | Medium | Manual | Read heavy |
| Master-Master | Medium | Fast | Active-Passive HA |
| GTID Replication | Medium | Easier | Better HA |
| Group Replication | Hard | Auto | HA required |
| InnoDB Cluster | Hard | Auto | Production HA |

## 10.12 Replication Health Checklist

```
Master:
  ✅ log_bin = ON
  ✅ server-id = 1
  ✅ Replication user created with correct IP
  ✅ SHOW BINARY LOG STATUS — File and Position noted
  ✅ Firewall allows port 3306 from Replica IP

Replica:
  ✅ server-id = 2 (different from Master)
  ✅ read_only = ON
  ✅ CHANGE REPLICATION SOURCE TO done with GET_SOURCE_PUBLIC_KEY = 1
  ✅ START REPLICA executed
  ✅ Replica_IO_Running = Yes
  ✅ Replica_SQL_Running = Yes
  ✅ Seconds_Behind_Source = 0
  ✅ Live test: Master data appears on Replica
```

## 10.13 Quick Reference Commands

**Master:**

| Command | কাজ |
|---|---|
| `SHOW BINARY LOG STATUS\G` | Current binlog file and position |
| `SHOW BINARY LOGS` | All binlog files |
| `SHOW VARIABLES LIKE 'log_bin'` | Binary logging ON/OFF |
| `SHOW VARIABLES LIKE 'server_id'` | Server ID confirm |

**Replica:**

| Command | কাজ |
|---|---|
| `SHOW REPLICA STATUS\G` | Full replication status |
| `START REPLICA` | Start replication |
| `STOP REPLICA` | Stop replication |
| `RESET REPLICA ALL` | Clear all replication config |
| `SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1` | Skip one failed transaction |

**Both:**

| Command | কাজ |
|---|---|
| `SHOW VARIABLES LIKE 'datadir'` | Active data directory |
| `SHOW PROCESSLIST` | Active threads including replication |
| `SHOW ENGINE INNODB STATUS\G` | Deep InnoDB diagnostic |
| `EXPLAIN query\G` | Query execution plan |
| `EXPLAIN ANALYZE query\G` | Actual execution with timing |

---

*MySQL 8.4 | Rocky Linux 9 | Complete Study Reference*
