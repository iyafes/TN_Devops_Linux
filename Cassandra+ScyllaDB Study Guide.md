# Cassandra + ScyllaDB Complete Study Guide (Revised)
### Architecture · Data Model · Internals · Cluster · Tuning · Patterns
### সব technical terms সহজ ভাষায় in-place explain করা আছে

---

## Table of Contents

### Apache Cassandra
1. [Chapter 1 — What is Cassandra & Why It Exists](#chapter-1)
2. [Chapter 2 — Core Concepts & Data Model](#chapter-2)
3. [Chapter 3 — Internal Architecture](#chapter-3)
4. [Chapter 4 — CQL — Cassandra Query Language](#chapter-4)
5. [Chapter 5 — Partitioning & Consistent Hashing](#chapter-5)
6. [Chapter 6 — Replication & Consistency](#chapter-6)
7. [Chapter 7 — Write Path — কীভাবে Data লেখা হয়](#chapter-7)
8. [Chapter 8 — Read Path — কীভাবে Data পড়া হয়](#chapter-8)
9. [Chapter 9 — Compaction](#chapter-9)
10. [Chapter 10 — Cluster Setup & Operations](#chapter-10)
11. [Chapter 11 — Performance & Tuning](#chapter-11)
12. [Chapter 12 — Security](#chapter-12)
13. [Chapter 13 — Backup & Recovery](#chapter-13)
14. [Chapter 14 — Real Patterns & Use Cases](#chapter-14)

### ScyllaDB
15. [Chapter 15 — What is ScyllaDB & Why It Exists](#chapter-15)
16. [Chapter 16 — ScyllaDB Architecture — Shard-per-Core](#chapter-16)
17. [Chapter 17 — ScyllaDB vs Cassandra — Differences](#chapter-17)
18. [Chapter 18 — ScyllaDB Setup & Configuration](#chapter-18)
19. [Chapter 19 — ScyllaDB Specific Features](#chapter-19)
20. [Chapter 20 — ScyllaDB Tuning](#chapter-20)

---

# PART 1: APACHE CASSANDRA

---

<a name="chapter-1"></a>
# Chapter 1 — What is Cassandra & Why It Exists

## Cassandra কী?

Apache Cassandra হলো একটা **distributed, wide-column NoSQL database**।

**"Distributed" কী?**
একটা database যেটা একটাই server এ না থেকে multiple servers এ ছড়িয়ে থাকে এবং একসাথে কাজ করে। Users এর কাছে মনে হয় একটাই database, কিন্তু আসলে অনেক servers মিলে কাজ করছে।

**"Wide-column" কী?**
Data রাখার একটা model। Relational database এ প্রতিটা row এ fixed columns থাকে। Wide-column এ প্রতিটা row এর columns আলাদা হতে পারে এবং একটা row তে হাজার হাজার columns থাকতে পারে। Google এর Bigtable এই idea টা প্রথম introduce করে।

**"NoSQL" কী?**
"Not Only SQL" — SQL ছাড়া বা SQL এর পাশাপাশি অন্য query mechanism use করে এমন database। Relational (table-based) model follow করে না।

Facebook তৈরি করেছিল 2008 সালে inbox search এর জন্য, পরে open source করে Apache কে দেয়।

---

## কেন Cassandra তৈরি হয়েছিল — The Problem

Facebook এর inbox search এ প্রতিদিন billions of messages write হচ্ছিল। Requirements ছিল:
- Massive write throughput — প্রতি second এ লক্ষ লক্ষ write
- Data যেকোনো datacenter থেকে read করা যাবে
- কোনো downtime নেই
- **Linear scaling** — node add করলে capacity exactly সেই অনুপাতে বাড়বে

**"Linear scaling" কী?**
2টা server থাকলে 2x capacity, 3টা থাকলে 3x capacity। অনেক distributed systems এ scaling এত smooth হয় না — overhead থাকে।

Relational DB এবং MySQL এ এটা possible ছিল না। Amazon এর DynamoDB এবং Google এর Bigtable এর ideas combine করে Cassandra তৈরি হয়।

---

## Cassandra কোথায় ভালো, কোথায় না

**Cassandra ভালো:**
- **High write throughput** — millions of writes per second
- **Time-series data** — IoT sensor data, metrics, logs, events (সময়ের সাথে তৈরি হওয়া data)
- **Always-on applications** — downtime tolerate করতে পারে না
- **Multi-datacenter replication** — দুনিয়ার বিভিন্ন জায়গায় data রাখতে হবে
- **Predictable low latency** — read এবং write উভয়েই consistent single-digit milliseconds
- **Linear scalability** — node add করলে proportional capacity বাড়ে

**Cassandra ভালো না:**
- **Ad-hoc queries** — schema design এর বাইরে যেকোনো query করা কঠিন
  - "Ad-hoc" মানে আগে থেকে plan না করে হঠাৎ করা
- **Complex joins** — join নেই, relational data modeling কাজ করে না
- **ACID transactions** — lightweight transactions আছে কিন্তু full ACID নেই
  - ACID: Atomicity, Consistency, Isolation, Durability — database reliability এর চারটা property
- **Small dataset** — overhead বেশি, simple use case এ overkill
- **Aggregations** — COUNT, SUM ইত্যাদি expensive

---

## CAP Theorem এ Cassandra

**"CAP Theorem" কী?**
Distributed system এ তিনটা property আছে:
- **C**onsistency — সব nodes এ same data দেখা যায়
- **A**vailability — সবসময় response পাওয়া যায়
- **P**artition Tolerance — network এ দুটো part আলাদা হয়ে গেলেও (network partition) system চলে

Eric Brewer এর theorem: এই তিনটার মধ্যে যেকোনো দুটো একসাথে পাওয়া যায়, তিনটা একসাথে না।

**"Network partition" কী?**
Network problem এ cluster এর কিছু nodes অন্যদের সাথে communicate করতে পারছে না। যেন cluster দুটো isolated ভাগে ভেঙে গেছে।

Cassandra হলো **AP system** — Availability এবং Partition Tolerance।

কিন্তু Cassandra **tunable consistency** দেয় — per-query তুমি decide করতে পারো কতটা consistent হবে। Strong consistency চাইলে CP এর মতো behave করতে পারে।

---

## Cassandra vs MongoDB vs Redis

| বিষয় | Cassandra | MongoDB | Redis |
|------|-----------|---------|-------|
| Primary strength | Write throughput, time-series | Flexible schema, documents | Speed, caching |
| Data model | Wide-column | Document | Key-value |
| Joins | নেই | $lookup (limited) | নেই |
| Transactions | Limited | Multi-document ACID | Limited |
| Scaling | Linear horizontal | Sharding (complex) | Cluster |
| Best for | IoT, logs, time-series | Product catalog, CMS | Cache, sessions |

---

<a name="chapter-2"></a>
# Chapter 2 — Core Concepts & Data Model

## Wide-Column Store কী?

Cassandra এর data model এর তিনটা level:
```
Keyspace (database এর মতো)
  └── Table (collection এর মতো, কিন্তু schema আছে)
        └── Row (partition key দিয়ে identify করা হয়)
              └── Columns (values)
```

**Relational table:**
- Fixed schema — আগে থেকে columns define করতে হয়
- প্রতিটা row এ same columns

**Wide-column:**
- Row আছে, column আছে
- কিন্তু columns dynamically add করা যায়
- Clustering columns দিয়ে প্রতিটা row এর ভেতরে আলাদা data sort হয়

---

## Keyspace

Keyspace হলো Cassandra এর top-level namespace — SQL database এর মতো।

**"Namespace" কী?**
নাম conflict এড়াতে একটা bounded space। যেমন দুটো আলাদা school এ একই নামের student থাকতে পারে — school টাই namespace।

```sql
CREATE KEYSPACE my_app
  WITH REPLICATION = {
    'class': 'NetworkTopologyStrategy',
    'datacenter1': 3
  }
  AND DURABLE_WRITES = true;
```

**Replication strategy:**
- **SimpleStrategy** — single datacenter, development এ use করো
- **NetworkTopologyStrategy** — production, multi-datacenter এ use করো। প্রতিটা datacenter এ আলাদা replication factor দেওয়া যায়।

**"Replication factor" কী?**
Data কতটা nodes এ copy থাকবে। Factor 3 মানে 3টা nodes এ same data থাকবে।

**"DURABLE_WRITES" কী?**
Write হলে commit log এ লেখা হবে কিনা। true রাখো সবসময় — false করলে crash এ data হারাতে পারে।

---

## Table এবং Primary Key

Cassandra তে Primary Key অনেক বেশি important — এটাই data distribution এবং storage এর core।

```sql
CREATE TABLE orders (
  order_id    UUID,
  customer_id UUID,
  created_at  TIMESTAMP,
  status      TEXT,
  amount      DECIMAL,
  PRIMARY KEY (customer_id, created_at, order_id)
);
```

Primary Key এর দুটো অংশ:

### Partition Key

Primary Key এর প্রথম অংশ। এই key এর **hash** (একটা mathematical function এর output) দিয়ে কোন node এ data যাবে decide হয়।

**"Hash function" কী?**
Input থেকে fixed-length output তৈরি করার mathematical function। "rahim" → "a7b3c9..." এর মতো। Same input সবসময় same output। Output থেকে input বের করা যায় না। Data distribute করতে hash use করলে even distribution পাওয়া যায়।

```sql
PRIMARY KEY (customer_id)
-- customer_id হলো partition key

PRIMARY KEY ((customer_id, year), month, day)
-- (customer_id, year) হলো composite partition key
-- "composite" মানে দুটো fields মিলে একটা key তৈরি
```

**Partition key এর choice সবচেয়ে critical design decision।**
- **High cardinality** হওয়া উচিত — অনেক unique values থাকতে হবে
  - "Cardinality" মানে কতটা unique values একটা field এ আছে
- **Even distribution** — সব nodes এ সমান data যাওয়া উচিত
- Query তে সবসময় partition key থাকবে

### Clustering Columns

Primary Key এর বাকি অংশ। Partition এর ভেতরে data **sort** করে রাখে।

```sql
PRIMARY KEY (customer_id, created_at, order_id)
-- customer_id = partition key
-- created_at, order_id = clustering columns
-- এই order এ data disk এ sorted থাকে
```

Clustering columns এর order matter করে — এই order এ data sorted থাকে disk এ। Range query এই sort order follow করে হওয়া উচিত।

---

## Data Types

```sql
-- Numeric (সংখ্যা)
INT          -- 32-bit integer (-2 billion to +2 billion)
BIGINT       -- 64-bit integer (আরো বড় range)
SMALLINT     -- 16-bit integer
TINYINT      -- 8-bit integer
FLOAT        -- 32-bit floating point (decimal number, less precise)
DOUBLE       -- 64-bit floating point (more precise)
DECIMAL      -- exact decimal (financial calculations এ)
VARINT       -- arbitrary precision integer (যত বড় দরকার)

-- Text (লেখা)
TEXT, VARCHAR  -- Unicode text (same thing, interchangeable)
ASCII          -- শুধু ASCII characters (English letters, numbers)

-- Time (সময়)
TIMESTAMP    -- date + time (milliseconds precision)
DATE         -- শুধু date (2024-03-23)
TIME         -- শুধু time (10:30:00)
DURATION     -- time duration (1h30m)

-- UUID (Universally Unique Identifier)
UUID         -- random unique identifier
             -- "universally unique" মানে পুরো দুনিয়ায় unique হওয়ার guarantee
TIMEUUID     -- time-based UUID (version 1) — sortable by time
             -- UUID তে timestamp embedded থাকে, তাই sort করলে time order পাওয়া যায়

-- Binary
BLOB         -- Binary Large Object — raw bytes (image, file)

-- Boolean
BOOLEAN      -- true বা false

-- Collections (একটা field এ multiple values)
LIST<TEXT>   -- ordered list of text values
SET<INT>     -- unique integers (duplicates নেই)
MAP<TEXT, INT> -- key-value pairs

-- Special
INET         -- IP address (IPv4 বা IPv6)
COUNTER      -- distributed counter (বিশেষ type, নিচে বিস্তারিত)
```

---

## Collections Deep Dive

```sql
CREATE TABLE user_preferences (
  user_id    UUID PRIMARY KEY,
  tags       SET<TEXT>,
  scores     MAP<TEXT, INT>,
  history    LIST<TIMESTAMP>
);
```

**SET:** Unique values, unordered (কোনো নির্দিষ্ট order নেই)
**LIST:** Ordered (ক্রমানুসারে), duplicates allowed
**MAP:** Key-value pairs — একটা key এর জন্য একটা value

**Collections এর limitations:**
- Maximum 65535 elements (কিন্তু আরো কম রাখা উচিত — performance এর জন্য)
- Single element update efficiently করা যায় না
- Large collection থাকলে পুরো collection read হয় — slow

**Frozen collections:**
```sql
frozen<LIST<TEXT>>
```
**"Frozen" কী?**
Immutable (পরিবর্তন করা যায় না) হিসেবে treat করা হয়। পুরো collection একটাই cell হিসেবে store হয়। Update করতে হলে পুরো collection replace করতে হয়।

---

## Counter Table

**"Counter" কী?**
Distributed environment এ একটা number বাড়ানো বা কমানো — যেখানে multiple nodes একই counter update করতে পারে। Race condition ছাড়া এটা করা কঠিন।

```sql
CREATE TABLE page_views (
  page_id  TEXT PRIMARY KEY,
  views    COUNTER
);

UPDATE page_views SET views = views + 1 WHERE page_id = 'home';
```

**Counter এর সীমাবদ্ধতা:**
- Counter table এ শুধু counter columns থাকতে পারে (partition/clustering key ছাড়া)
- **Idempotent না** — retry করলে double count হতে পারে
  - "Idempotent" মানে একই operation বারবার করলেও same result। Counter increment idempotent না — বারবার করলে বারবার বাড়ে।
- Delete এবং re-insert করলে counter reset হয় না

---

## Materialized View

**"Materialized View" কী?**
একটা query এর result যেটা physically store করা হয় এবং automatically updated থাকে। Base table এ data change হলে view ও update হয়।

```sql
-- Base table
CREATE TABLE orders (
  order_id    UUID,
  customer_id UUID,
  status      TEXT,
  created_at  TIMESTAMP,
  PRIMARY KEY (customer_id, created_at, order_id)
);

-- Materialized view — status দিয়ে query করার জন্য
CREATE MATERIALIZED VIEW orders_by_status AS
  SELECT * FROM orders
  WHERE status IS NOT NULL AND customer_id IS NOT NULL
    AND created_at IS NOT NULL AND order_id IS NOT NULL
  PRIMARY KEY (status, customer_id, created_at, order_id);
```

**Caution:** Materialized views write performance কমায় (প্রতিটা write এ view ও update করতে হয়) এবং operational complexity বাড়ায়। Production এ carefully use করো।

---

<a name="chapter-3"></a>
# Chapter 3 — Internal Architecture

## Masterless Architecture — Ring

Cassandra এর সবচেয়ে unique বৈশিষ্ট্য: **কোনো master নেই।** সব nodes সমান (peer-to-peer)।

**"Master-slave" vs "Peer-to-peer" কী?**
Master-slave: একটা master node সব decisions নেয়, slave nodes follow করে। Master fail করলে system ভেঙে পড়ে।
Peer-to-peer: সব nodes সমান। কোনো একটা fail হলেও বাকিরা চলে।

Nodes একটা **virtual ring** তৈরি করে। প্রতিটা node ring এর একটা portion এর responsible।

```
        Node A (0-25%)
       /               \
Node D (75-100%)    Node B (25-50%)
       \               /
        Node C (50-75%)
```

Data কোন node এ যাবে — partition key এর hash দিয়ে determine হয়। Hash output টা ring এ কোন position এ পড়ে সেটা দেখে।

---

## Gossip Protocol

**"Gossip protocol" কী?**
মানুষ যেভাবে গুজব ছড়ায় সেভাবে — একজন দুজনকে বলে, তারা আরো দুজনকে বলে। দ্রুত সব জায়গায় ছড়িয়ে পড়ে।

Cassandra এ: প্রতি second এ প্রতিটা node 1-3টা অন্য node কে তার state জানায়। এভাবে পুরো cluster এর state সব nodes এ propagate হয়।

Gossip এ যা share হয়:
- Node কি alive আছে কিনা
- Node এর load (কতটা busy)
- Token assignment (কোন node কোন data range দেখে)
- Schema information

কোনো central coordinator নেই — পুরোপুরি decentralized।

---

## Snitch — Datacenter/Rack Awareness

**"Snitch" কী?**
Cassandra কে বলে কোন node কোন datacenter এ, কোন rack এ। এই information replication এবং read repair এ use হয়।

**"Rack" কী?**
Physical server rack — একটা cabinet যেখানে অনেক servers stack করা থাকে। একই rack এর সব servers একই power supply এবং network switch share করে। Rack fail হলে সব servers যায়। তাই replicas আলাদা racks এ রাখা ভালো।

```yaml
# cassandra.yaml
endpoint_snitch: GossipingPropertyFileSnitch
```

Types:
- `SimpleSnitch` — single DC, development
- `GossipingPropertyFileSnitch` — production recommended, `cassandra-rackdc.properties` থেকে config পড়ে
- `EC2Snitch`, `GoogleCloudSnitch` — cloud specific (automatically DC/rack detect করে)

```properties
# cassandra-rackdc.properties
dc=datacenter1
rack=rack1
```

---

## Token এবং Virtual Nodes (VNodes)

### Token

প্রতিটা node কে একটা **token range** assign করা হয়। Partition key এর hash সেই range এ পড়লে সেই node এ data যায়।

**Murmur3 hash function** use করে (default): range -2^63 থেকে 2^63-1।

**"Murmur3" কী?**
একটা fast, non-cryptographic hash function। Cryptographic মানে নয় (SHA256 এর মতো) — এটা security এর জন্য না, speed এবং even distribution এর জন্য।

### Virtual Nodes (VNodes)

আগে প্রতিটা physical node একটা continuous token range পেত। সমস্যা:
- নতুন node add করলে manual rebalancing দরকার
- Node failure এ একটাই node এর পুরো load অন্যরা নিত

VNodes এ প্রতিটা physical node অনেকগুলো (default 16) ছোট token range পায়, ring এ randomly ছড়িয়ে।

```yaml
num_tokens: 16  # cassandra.yaml
```

**VNodes এর সুবিধা:**
- নতুন node add করলে automatically অনেক nodes থেকে কিছু কিছু data নেয় — smooth rebalancing
- Even load distribution
- Node failure এর load অনেক nodes এ ছড়িয়ে পড়ে

---

## Coordinator Node

Client যেকোনো node এ connect করতে পারে। সেই node হয় **coordinator**। Coordinator জানে কোন partition কোন node এ — সে request forward করে।

```
Client → Node A (coordinator)
              ↓
         Hash(partition_key) → Token range → Node C, D, E (replicas)
              ↓
         Node C, D, E → data পাঠায় → Coordinator
              ↓
         Coordinator → Client কে response পাঠায়
```

---

## Commit Log

**"Commit Log" কী?**
Write আসলে প্রথমে Commit Log এ যায় — sequential (একটার পর একটা, ক্রমানুসারে) write, অনেক fast। Crash হলে recovery এর জন্য এই log থেকে replay করা হয়।

**কেন sequential write fast?**
Hard disk এ sequential write (data পরপর লেখা) random write (যেকোনো জায়গায় লেখা) এর চেয়ে অনেক দ্রুত। Disk head কে বারবার সরাতে হয় না।

```yaml
commitlog_directory: /var/lib/cassandra/commitlog
commitlog_sync: periodic          # periodic বা batch
commitlog_sync_period_in_ms: 10000  # 10 seconds এ একবার flush
```

---

## Memtable

Commit log এ লেখার পর data **Memtable** এ যায় — in-memory (RAM এ) write buffer।

**"Buffer" কী?**
Temporary storage। Data সাথে সাথে destination এ না পাঠিয়ে আগে buffer এ জমা করা হয়, তারপর batch এ পাঠানো হয়। Efficient।

প্রতিটা table এর জন্য একটা Memtable। Data clustering columns অনুযায়ী sorted থাকে।

---

## SSTable — Sorted String Table

Memtable full হলে বা flush trigger হলে disk এ **SSTable** হিসেবে লেখা হয়। **Immutable** — একবার লেখা হলে change হয় না।

**"Immutable" কী?**
পরিবর্তন করা যায় না। SSTable লেখা হলে সেটা কখনো modify হয় না। Update বা delete মানে নতুন SSTable তৈরি হয়।

SSTable এর components:
- **Data.db** — actual data
- **Index.db** — partition key → data file offset (কোন partition কোথায় আছে)
- **Summary.db** — Index.db এর in-memory summary (দ্রুত search এর জন্য)
- **Filter.db** — Bloom filter (partition key আছে কিনা quickly check করতে)
- **Statistics.db** — metadata, min/max values, tombstone info
- **CompressionInfo.db** — compression metadata

**"Offset" কী?**
File এর শুরু থেকে কতটা bytes দূরে আছে সেই position। Index.db তে partition key এর offset থাকে — সরাসরি সেই position এ jump করা যায়।

---

## Bloom Filter

**"Bloom Filter" কী?**
Probabilistic data structure — partition key কোন SSTable এ আছে কিনা quickly check করতে।

**"Probabilistic" মানে কী এখানে?**
- **False negative নেই** — "নেই" বললে সত্যিই নেই (100% নিশ্চিত)
- **False positive সম্ভব** — "আছে" বললে নাও থাকতে পারে (কিছুটা chance আছে ভুল)

False positive হলে শুধু extra SSTable read হয় — incorrect result আসে না।

Memory তে রাখা হয় — disk read ছাড়াই check করা যায়।

False positive rate tunable:
```yaml
bloom_filter_fp_chance: 0.01  # 1% false positive rate (default)
# কম করলে memory বেশি লাগে কিন্তু unnecessary SSTable read কমে
```

---

## Tombstone

**"Tombstone" কী?**
Cassandra তে delete করলে data immediately মুছে যায় না। একটা tombstone (delete marker) লেখা হয় — "এই data deleted" এর sign। Compaction এর সময় tombstone এবং actual data একসাথে পেলে data মুছে যায়।

**কেন immediately delete না করে tombstone?**
Cassandra distributed — অনেক nodes এ data আছে। সব nodes কে একসাথে delete করানো complex। Tombstone দিলে প্রতিটা node নিজে নিজে পরে delete করতে পারে।

Tombstone এর TTL default **10 days** (`gc_grace_seconds`)।

**"gc_grace_seconds" কী?**
"gc" = garbage collection। এই সময় পরে tombstone এবং সংশ্লিষ্ট data compaction এ মুছে যায়। এই সময়ের মধ্যে সব nodes এ repair হওয়া উচিত।

**Tombstone accumulation সমস্যা:**
অনেক tombstone জমলে read slow হয় — প্রতিটা read এ tombstones skip করতে হয়।

```sql
-- TTL দিয়ে insert করলে tombstone এড়ানো যায়
INSERT INTO events (id, data) VALUES (uuid(), 'click')
USING TTL 86400;  -- 24 hours পর automatically expire হবে
-- TTL expire হলে Cassandra নিজেই remove করে, explicit delete tombstone লাগে না
```

---

<a name="chapter-4"></a>
# Chapter 4 — CQL — Cassandra Query Language

CQL দেখতে SQL এর মতো কিন্তু ভেতরে অনেক আলাদা। SQL এ freely যা করা যায় তার অনেক কিছু CQL এ করা যায় না।

---

## Keyspace Operations

```sql
-- Create
CREATE KEYSPACE app
  WITH REPLICATION = {
    'class': 'NetworkTopologyStrategy',
    'dc1': 3,    -- dc1 তে 3 copies
    'dc2': 2     -- dc2 তে 2 copies
  };

-- Use
USE app;

-- Alter (modify করো)
ALTER KEYSPACE app
  WITH REPLICATION = { 'class': 'NetworkTopologyStrategy', 'dc1': 3 };

-- Drop (delete করো)
DROP KEYSPACE app;

-- List (দেখো)
DESCRIBE KEYSPACES;
```

---

## Table Operations

```sql
-- Create
CREATE TABLE IF NOT EXISTS user_events (
  user_id    UUID,
  event_time TIMESTAMP,
  event_id   TIMEUUID,
  event_type TEXT,
  data       MAP<TEXT, TEXT>,
  PRIMARY KEY (user_id, event_time, event_id)
) WITH CLUSTERING ORDER BY (event_time DESC, event_id DESC)
-- CLUSTERING ORDER BY = clustering columns কোন order এ sort হবে
-- DESC = descending (বড় থেকে ছোট, latest first)
  AND compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'DAYS',
    'compaction_window_size': 1
  }
-- compaction = কীভাবে SSTables merge হবে (Chapter 9 এ বিস্তারিত)
  AND default_time_to_live = 2592000;
-- default_time_to_live = সব rows এর default TTL (30 days = 2592000 seconds)

-- Describe (structure দেখো)
DESCRIBE TABLE user_events;

-- Alter — column add করো
ALTER TABLE user_events ADD ip_address INET;

-- Drop
DROP TABLE user_events;

-- Truncate (সব data delete, structure রাখো)
TRUNCATE user_events;
```

---

## Insert

```sql
-- Basic insert
INSERT INTO user_events (user_id, event_time, event_id, event_type)
VALUES (uuid(), toTimestamp(now()), now(), 'login');
-- uuid() = random UUID generate করে
-- now() = current timestamp
-- toTimestamp(now()) = TIMEUUID কে TIMESTAMP এ convert করে

-- TTL সহ
INSERT INTO sessions (session_id, user_id, data)
VALUES ('abc123', uuid(), 'session_data')
USING TTL 3600;  -- 1 hour পর automatically delete

-- Timestamp সহ (write time manually set করো)
INSERT INTO events (id, data) VALUES (uuid(), 'test')
USING TIMESTAMP 1711123200000000;
-- microseconds এ timestamp দিতে হয়

-- IF NOT EXISTS (lightweight transaction)
INSERT INTO users (user_id, email, name)
VALUES (uuid(), 'rahim@example.com', 'Rahim')
IF NOT EXISTS;
-- IF NOT EXISTS = যদি এই email দিয়ে user না থাকে তাহলেই insert করো
-- এটা lightweight transaction — compare-and-set operation
```

**CQL তে Upsert:**
Cassandra তে INSERT এবং UPDATE উভয়ই **upsert** — row already থাকলে update হয়, না থাকলে insert হয়।

**"Upsert" কী?**
Update + Insert এর combination। থাকলে update করো, না থাকলে insert করো।

---

## Select

```sql
-- Partition key দিয়ে query (সবসময় এটাই করা উচিত)
SELECT * FROM user_events WHERE user_id = 550e8400-e29b-41d4-a716-446655440000;

-- Partition key + clustering column range
SELECT * FROM user_events
WHERE user_id = 550e8400-e29b-41d4-a716-446655440000
  AND event_time >= '2024-03-01' AND event_time < '2024-03-24';

-- Specific columns
SELECT event_type, event_time FROM user_events
WHERE user_id = 550e8400-e29b-41d4-a716-446655440000;

-- Limit
SELECT * FROM user_events
WHERE user_id = 550e8400-e29b-41d4-a716-446655440000
LIMIT 10;

-- ALLOW FILTERING (dangerous — full scan)
SELECT * FROM user_events WHERE event_type = 'login' ALLOW FILTERING;
-- partition key ছাড়া query করতে ALLOW FILTERING লাগে
-- কিন্তু এটা সব partitions scan করে — production এ avoid করো

-- Token function (partition key range scan)
SELECT * FROM user_events WHERE token(user_id) > token(some_id);
-- token() = partition key এর hash value return করে
-- এটা দিয়ে ring এর একটা অংশ scan করা যায়

-- IN clause
SELECT * FROM users WHERE user_id IN (id1, id2, id3);
-- সতর্কতা: অনেক partition এ broadcast হয়, সীমিত ব্যবহার করো
```

---

## Update

```sql
-- Basic update
UPDATE user_events
SET event_type = 'logout'
WHERE user_id = 550e8400-e29b-41d4-a716-446655440000
  AND event_time = '2024-03-23 10:00:00'
  AND event_id = some_timeuuid;
-- primary key এর সব অংশ দিতে হবে

-- Counter update
UPDATE page_views SET views = views + 1 WHERE page_id = 'home';

-- Collection update
UPDATE users SET tags = tags + {'redis', 'mongodb'} WHERE user_id = uuid_here;
-- tags SET এ 'redis' এবং 'mongodb' add করো
UPDATE users SET tags = tags - {'redis'} WHERE user_id = uuid_here;
-- tags থেকে 'redis' remove করো
UPDATE users SET scores['math'] = 95 WHERE user_id = uuid_here;
-- scores MAP এ 'math' key এর value 95 set করো

-- TTL সহ update
UPDATE sessions USING TTL 1800 SET data = 'new_data' WHERE session_id = 'abc123';

-- Conditional update (lightweight transaction)
UPDATE users SET email = 'new@example.com'
WHERE user_id = uuid_here
IF email = 'old@example.com';
-- IF = current value check করো, match করলেই update করো
-- এটাই Cassandra তে compare-and-set
```

---

## Delete

```sql
-- Row delete
DELETE FROM user_events
WHERE user_id = 550e8400-e29b-41d4-a716-446655440000
  AND event_time = '2024-03-23 10:00:00'
  AND event_id = some_timeuuid;

-- Specific column delete
DELETE event_type FROM user_events
WHERE user_id = uuid_here AND event_time = timestamp_here AND event_id = id_here;

-- Partition delete (পুরো partition এর সব rows)
DELETE FROM user_events WHERE user_id = uuid_here;

-- Range delete (clustering column range)
DELETE FROM user_events
WHERE user_id = uuid_here
  AND event_time >= '2024-01-01' AND event_time < '2024-02-01';
```

---

## Batch

**"Batch" কী?**
Multiple operations একসাথে group করে পাঠানো। SQL এর transaction এর মতো দেখতে কিন্তু আলাদা।

```sql
BEGIN BATCH
  INSERT INTO users (user_id, name) VALUES (uuid(), 'Rahim');
  INSERT INTO user_audit (user_id, action) VALUES (uuid(), 'created');
APPLY BATCH;
-- logged batch = atomic — সব হবে অথবা কিছুই না

BEGIN UNLOGGED BATCH
  UPDATE counters SET count = count + 1 WHERE id = 'total';
  UPDATE counters SET count = count + 1 WHERE id = 'today';
APPLY BATCH;
-- unlogged = faster, not atomic
```

**Caution:**
- Batch কে performance optimization হিসেবে ব্যবহার করো না — এটা **atomicity** এর জন্য
  - "Atomicity" মানে সব হবে অথবা কিছুই না — মাঝপথে থামবে না
- Different partitions এ batch করলে coordinator এর load বাড়ে
- Large batch avoid করো

---

## Secondary Index

**"Secondary Index" কী?**
Primary key ছাড়া অন্য field এ index। Primary key দিয়ে directly partition এ যাওয়া যায়। Secondary index দিয়ে অন্য field এ query করা যায়।

```sql
-- Secondary index তৈরি করো
CREATE INDEX ON user_events (event_type);

-- এখন query করা যাবে
SELECT * FROM user_events WHERE event_type = 'login';
-- কিন্তু internally সব partitions এ broadcast হবে — slow
```

**Secondary index এর সমস্যা:**
- High cardinality field এ index করলে performance খারাপ
- Low selectivity — অনেক row match করলে অনেক nodes থেকে data আনতে হয়
  - "Selectivity" মানে কতটা percentage data match করে। কম match = high selectivity = better।
- Production এ সাবধানে use করো

---

## ALLOW FILTERING

```sql
SELECT * FROM users WHERE age > 25 ALLOW FILTERING;
```

Cassandra automatically এই query reject করে কারণ partition key নেই — full cluster scan করতে হবে।

`ALLOW FILTERING` দিয়ে force করা যায় কিন্তু **production এ avoid করো।** বরং table design পরিবর্তন করো।

---

<a name="chapter-5"></a>
# Chapter 5 — Partitioning & Consistent Hashing

## Consistent Hashing

**Traditional hashing এর সমস্যা:**
3টা servers আছে। Hash(key) % 3 দিয়ে কোন server এ যাবে বলো। এখন 4র্থ server add হলো। Hash(key) % 4 হবে — প্রায় সব keys এর server বদলে যাবে। সব data migrate করতে হবে।

**Consistent hashing এর সমাধান:**
- Hash space একটা circular ring (0 থেকে MAX, শেষে আবার 0 তে আসে)
- Servers এবং keys দুটোই এই ring এ map হয়
- Key টা ring এ কোথায় পড়ে, তার পরের (clockwise) server এ যায়

```
Ring: 0 -------- Server A -------- Server B -------- Server C -------- MAX
                  ↑                  ↑                  ↑
               Key1, Key2         Key3, Key4         Key5, Key6
```

Node add হলে: শুধু পরের node এর কিছু keys নতুন node এ যায়
Node remove হলে: শুধু সেই node এর keys পরের node এ যায়
বাকি সব data unchanged।

---

## Murmur3 Partitioner

Default partitioner। Partition key এর Murmur3 hash calculate করে। Even distribution নিশ্চিত করে।

```
token = murmur3_hash(partition_key)
-- output range: -2^63 থেকে 2^63-1
-- এই range ring এ map হয়
```

---

## Partition Size

একটা partition এর maximum recommended size **100MB** (hard limit নেই কিন্তু বড় হলে সমস্যা)।

**Large partition এর সমস্যা:**
- Read slow হয়
- Compaction expensive হয় (বেশি data process করতে হয়)
- Memory pressure বাড়ে — একসাথে অনেক data RAM এ আনতে হয়
- **GC (Garbage Collection)** এ spike হয়
  - "GC" = Java তে unused memory automatically free করার process। JVM (Java Virtual Machine) চালায়। বড় data এ GC pause বেশি।

**Check করো:**
```bash
nodetool tablehistograms keyspace.table
```

**Solution:** Partition key তে time bucket add করো।

```sql
-- Bad: user এর সব events একটাই partition এ (unbounded)
PRIMARY KEY (user_id, event_time, event_id)

-- Good: প্রতি মাসে আলাদা partition
PRIMARY KEY ((user_id, month), event_time, event_id)
-- month = '2024-03' format এ
-- এখন প্রতিটা partition শুধু এক মাসের data রাখে
```

---

## Hot Partition

একটা partition অনেক বেশি traffic পেলে সেটা **hot partition** হয়ে যায়। সেই node এর CPU, disk, network এ bottleneck।

**"Bottleneck" কী?**
System এর সবচেয়ে slow part যেটা পুরো system কে slow করে দেয়। Pipe এ সরু জায়গার মতো।

**Causes:**
- Low cardinality partition key (status, country)
- Monotonically increasing key (timestamp alone) — সব new data একটাই partition এ
  - "Monotonically increasing" মানে সবসময় বাড়তে থাকে, কখনো কমে না
- Famous entity (popular user এর data)

**Solution:**
- Composite partition key use করো
- Time-based bucketing
- Random suffix যোগ করো (bucketing) — `user_id + random_bucket_number`

---

<a name="chapter-6"></a>
# Chapter 6 — Replication & Consistency

## Replication Factor (RF)

RF মানে data কতটা nodes এ replicate হবে। RF=3 মানে প্রতিটা data 3টা node এ থাকবে।

```sql
CREATE KEYSPACE app WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3,    -- dc1 তে 3 copies
  'dc2': 2     -- dc2 তে 2 copies
};
```

**Production minimum RF=3।**
কেন? RF=1 হলে সেই node fail করলে data চলে যায়। RF=2 হলে একটা fail করলে ঠিক আছে কিন্তু repair এ ঝামেলা। RF=3 হলে 2টা fail হলেও data আছে।

---

## Tunable Consistency

Cassandra এর সবচেয়ে powerful feature। প্রতিটা read/write operation এ আলাদা consistency level specify করা যায়।

**"Consistency" কী এখানে?**
কতটা nodes থেকে একমত হলে operation successful বলা হবে। বেশি nodes = বেশি consistency = বেশি latency।

### Write Consistency Levels

| Level | কাজ |
|-------|-----|
| `ANY` | যেকোনো একটা node (hint এও চলে) — weakest |
| `ONE` | একটা replica acknowledge করলেই success |
| `TWO` | দুটো replica acknowledge করলে |
| `THREE` | তিনটা replica acknowledge করলে |
| `QUORUM` | (RF/2)+1 replicas — majority |
| `LOCAL_QUORUM` | Local datacenter এর quorum |
| `EACH_QUORUM` | প্রতিটা datacenter এর quorum |
| `ALL` | সব replicas — strongest, availability কমে |

**"Quorum" কী?**
Majority — অর্ধেকের বেশি। RF=3 এ quorum = 2। RF=5 এ quorum = 3।

**"Hinted Handoff" (ANY level এ):**
Node down থাকলে coordinator সেই node এর জন্য intended writes **hint** হিসেবে নিজের কাছে রাখে। Node ফিরে আসলে hint deliver করে। ANY level এ hint stored থাকলেও success বলে — সবচেয়ে loose।

### Read Consistency Levels

| Level | কাজ |
|-------|-----|
| `ONE` | একটা replica থেকে পড়ো — fastest, stale data possible |
| `TWO` | দুটো replica থেকে পড়ো, latest নাও |
| `QUORUM` | Quorum replica থেকে পড়ো |
| `LOCAL_QUORUM` | Local datacenter এর quorum |
| `ALL` | সব replicas থেকে পড়ো — strongest |
| `LOCAL_ONE` | Local datacenter এর একটা replica |
| `SERIAL` | Lightweight transaction এর জন্য |

---

## Strong Consistency Formula

**Write CL + Read CL > RF** হলে strong consistency।

**কেন এই formula?**
Write এ W টা nodes এ লেখা হয়। Read এ R টা nodes থেকে পড়া হয়। W + R > RF মানে read এর nodes এবং write এর nodes এর মধ্যে অন্তত একটা overlap আছে — সেই node এ latest data আছে।

উদাহরণ RF=3:
- Write=QUORUM (2), Read=QUORUM (2) → 2+2=4 > 3 ✓ Strong
- Write=ONE (1), Read=ONE (1) → 1+1=2 ≤ 3 ✗ Eventual

**Practical recommendation:**
- **Write: LOCAL_QUORUM, Read: LOCAL_QUORUM** — strong consistency, single DC
- **Write: ONE, Read: ONE** — best performance, eventual consistency

---

## Read Repair

Read এর সময় multiple replicas থেকে data এনে compare করে। Inconsistency পেলে সবচেয়ে recent (highest timestamp) দিয়ে fix করে।

**"Read repair" কী?**
Read করতে গিয়ে দেখা গেলো Replica 1 এ data আছে কিন্তু Replica 2 এ নেই বা পুরনো। Coordinator সাথে সাথে Replica 2 কে update করে দেয়।

- **Foreground read repair** — read এর সময় synchronously হয় (response delay হয়)
- **Background read repair** — asynchronously হয় (response এর পরে)

```yaml
read_repair_chance: 0.1       # 10% reads এ background repair trigger হবে
dclocal_read_repair_chance: 0.1
```

---

## Anti-Entropy Repair — nodetool repair

**"Anti-entropy" কী?**
Entropy মানে disorder। Anti-entropy মানে disorder কমানো — replicas এর মধ্যে data sync রাখা।

Manual repair trigger:

```bash
nodetool repair keyspace_name table_name

# Full repair
nodetool repair -full keyspace_name

# Incremental repair (faster, recommended)
nodetool repair keyspace_name
```

**Incremental repair কী?**
শুধু last repair এর পরে changed data repair করে। Full repair এর চেয়ে অনেক কম কাজ।

**Production best practice:** Regular repair schedule করো। `gc_grace_seconds` এর আগে প্রতিটা node এ repair চালাও।

**কেন gc_grace_seconds এর আগে repair দরকার?**
Node offline থাকলে সে tombstone miss করে। Repair না হলে সেই node তে deleted data আবার দেখাতে পারে (resurrection)।

---

<a name="chapter-7"></a>
# Chapter 7 — Write Path — কীভাবে Data লেখা হয়

Write path Cassandra এর সবচেয়ে optimized part।

```
Client → Coordinator Node
              ↓
         Consistency Level অনুযায়ী Replica Nodes কে পাঠায়
              ↓
         প্রতিটা Replica Node এ:
              ↓
    ┌─────────────────────────────┐
    │  1. Commit Log (sequential  │  ← Crash recovery
    │     disk write, very fast)  │
    │  2. Memtable (in-memory)    │  ← Fast write buffer
    └─────────────────────────────┘
              ↓
         Memtable full হলে:
              ↓
    ┌─────────────────────────────┐
    │  SSTable এ flush (disk)     │  ← Permanent storage
    │  Commit Log segment truncate│  ← পুরনো log free করা
    └─────────────────────────────┘
```

---

## Commit Log এর Detail

- **Sequential append-only write** — disk এর সবচেয়় fast operation
  - "Append-only" মানে শুধু শেষে লেখা যায়, মাঝখানে modify করা যায় না
- প্রতিটা mutation (write) এখানে আসে
- Crash হলে এখান থেকে replay করে recover হয়
- Memtable flush হলে সংশ্লিষ্ট commit log segment delete হয়
  - "Segment" মানে log file এর একটা portion। Log কে segments এ ভাগ করা হয়।

```yaml
commitlog_directory: /var/lib/cassandra/commitlog
# Separate fast disk এ রাখা ভালো (SSD বা dedicated disk)
commitlog_total_space_in_mb: 8192  # 8GB
```

---

## Memtable এর Detail

- **In-memory sorted data structure** — প্রতিটা table এর জন্য আলাদা
- Write এখানে হয় — RAM write, অনেক fast
- Data sorted by partition key + clustering columns
- Flush হওয়ার conditions:
  - Size threshold পৌঁছালে (`memtable_heap_space_in_mb`)
  - Commit log size threshold পৌঁছালে
  - Manual `nodetool flush`

**"Flush" কী?**
In-memory data কে disk এ write করা। Memtable flush মানে Memtable এর data SSTable হিসেবে disk এ লেখা।

---

## SSTable Flush

```bash
# Manual flush trigger
nodetool flush keyspace_name table_name
nodetool flush  # সব tables
```

Flush হলে:
1. Memtable **frozen** হয় — নতুন write নেওয়া বন্ধ
2. নতুন write নতুন Memtable এ যায়
3. Background এ frozen Memtable থেকে SSTable লেখা হয়
4. SSTable complete হলে commit log segment free হয়

---

## Write Amplification

**"Write amplification" কী?**
একটা single write — commit log + memtable + SSTable = অনেক জায়গায় লেখা হয়। একটা write এ disk এ আসলে কতগুণ বেশি write হচ্ছে সেটাই write amplification।

এটা intentional trade-off — write fast করতে sequential writes use করা হয়, কিন্তু data বেশি জায়গায় যায়। Compaction এ আরো amplification হয়।

---

<a name="chapter-8"></a>
# Chapter 8 — Read Path — কীভাবে Data পড়া হয়

Read path write path এর চেয়ে complex।

```
Client → Coordinator
              ↓
         Partition key থেকে token calculate করো
              ↓
         Token range কোন nodes এ — replica nodes identify করো
              ↓
         Consistency level অনুযায়ী কতটা nodes থেকে পড়তে হবে সেটা decide করো
              ↓
         Fastest replica কে full data request পাঠাও
         বাকিদের কে digest request পাঠাও
         ("digest" = data এর hash — আসল data না, শুধু verify করতে)
              ↓
    প্রতিটা Replica Node এ:
    ┌──────────────────────────────────────────────┐
    │ 1. Row Cache check (if enabled)              │
    │ 2. Memtable check (RAM এ আছে?)              │
    │ 3. SSTable check:                            │
    │    a. Bloom filter check (আছে কি এই SSTable এ?)│
    │    b. Key cache check (partition এর offset)  │
    │    c. Summary → Index → Data file read       │
    │ 4. Multiple SSTables এর result merge করো    │
    └──────────────────────────────────────────────┘
              ↓
         Coordinator: responses compare করে latest নাও
         Inconsistency পেলে read repair trigger করো
              ↓
         Client কে result পাঠাও
```

---

## Row Cache এবং Key Cache

### Key Cache (Default enabled)

Partition key → SSTable data file offset এর mapping RAM এ রাখে।

**কী সুবিধা?**
SSTable এ data খুঁজতে Index file পড়তে হয়। Key cache এ offset থাকলে Index file পড়তে হয় না — সরাসরি Data file এ সঠিক position এ যাওয়া যায়।

```yaml
key_cache_size_in_mb: 100  # 0 = auto (5% of heap)
key_cache_save_period: 14400  # seconds, restart এও cache থাকে
```

### Row Cache (Default disabled)

পুরো row cache করে — memory intensive।

```yaml
row_cache_size_in_mb: 0  # disabled by default
```

**কখন enable করবে?**
Read heavy, write light workload। Hot rows যেগুলো বারবার read হয়।

---

## Read এর Performance Factors

**1. SSTable count বেশি হলে read slow:**
অনেক SSTables মানে অনেক জায়গায় look করতে হয়। Compaction এ merge হলে read fast হয়।

**2. Tombstone:**
প্রতিটা tombstone skip করতে হয়। অনেক tombstone = slow read।

**3. Large partition:**
পুরো partition read করতে অনেক সময়।

**4. Consistency level:**
Higher CL = বেশি nodes থেকে পড়তে হয় = slow।

---

<a name="chapter-9"></a>
# Chapter 9 — Compaction

Cassandra তে SSTables immutable। Update/delete নতুন SSTable তৈরি করে। সময়ের সাথে অনেক SSTables জমে। **Compaction** এগুলো merge করে নতুন SSTable তৈরি করে।

**কেন Compaction দরকার?**
- অনেক SSTables থাকলে read slow হয় (অনেক জায়গায় look করতে হয়)
- Duplicate data (একই key এর পুরনো এবং নতুন version) জায়গা নষ্ট করে
- Tombstones জায়গা নষ্ট করে এবং read slow করে

---

## Compaction এ কী হয়

```
SSTable 1: [A:1, B:2, D:4(tombstone)]
SSTable 2: [B:3(newer version), C:5, E:6]
SSTable 3: [A:7(newest version), F:8]

After Compaction:
Merged SSTable: [A:7, B:3, C:5, E:6, F:8]
-- A এর পুরনো version (1) গেছে, newest (7) রেখেছে
-- D tombstone remove হয়েছে (gc_grace_seconds পার হলে)
```

Compaction এ:
- Duplicate keys — latest timestamp টা রাখে
- Expired TTL data remove করে
- Tombstone remove করে (gc_grace_seconds পার হলে)
- নতুন sorted SSTable তৈরি করে

---

## Compaction Strategies

### STCS — SizeTiered Compaction Strategy (Default)

Similar size এর SSTables একসাথে compact করে।

**"SizeTiered" কী?**
SSTables কে size অনুযায়ী tiers (স্তর) এ ভাগ করা হয়। একই tier এর enough SSTables হলে compact করে।

```sql
WITH compaction = {
  'class': 'SizeTieredCompactionStrategy',
  'min_threshold': 4,     -- কমপক্ষে 4টা SSTable compact করতে হবে
  'max_threshold': 32     -- একসাথে maximum 32টা compact করবে
}
```

**ভালো:** Write heavy workload, বড় datasets
**খারাপ:** Read latency spiky (compaction এর সময় slow), disk space temporarily 2x লাগে

### LCS — Leveled Compaction Strategy

SSTables কে levels এ organize করে।

**"Leveled" কীভাবে কাজ করে?**
- Level 0: নতুন SSTables (overlap হতে পারে)
- Level 1: ~10MB SSTables, non-overlapping
- Level 2: ~100MB SSTables, non-overlapping
- Level 3: ~1GB SSTables, non-overlapping

"Non-overlapping" মানে একই key দুটো SSTable এ নেই — read এ একটাই SSTable check করলেই হয়।

```sql
WITH compaction = {
  'class': 'LeveledCompactionStrategy',
  'sstable_size_in_mb': 160
}
```

**ভালো:** Read heavy workload, predictable read latency
**খারাপ:** Write amplification বেশি (একই data বারবার rewrite হয়), I/O heavy

### TWCS — TimeWindow Compaction Strategy

Time-series data এর জন্য। Time window অনুযায়ী compact করে।

**"TimeWindow" কীভাবে কাজ করে?**
Current time window (যেমন আজকের দিন) এর SSTables একসাথে compact হয়। পুরনো windows আর compact হয় না — ঐ data আর change হয় না।

```sql
WITH compaction = {
  'class': 'TimeWindowCompactionStrategy',
  'compaction_window_unit': 'DAYS',
  'compaction_window_size': 1
}
```

**ভালো:** Time-series, TTL heavy workload
**কারণ:** Same time window এর data একসাথে থাকে। Old data TTL expire হলে পুরো SSTable drop হয় — tombstone overhead নেই।

### কোনটা কখন?

| Workload | Strategy |
|---------|---------|
| Write heavy | STCS |
| Read heavy, random access | LCS |
| Time-series, TTL | TWCS |

---

## Compaction Management

```bash
# Manual compaction trigger
nodetool compact keyspace_name table_name

# Compaction progress দেখো
nodetool compactionstats

# Compaction history দেখো
nodetool compactionhistory

# Compaction stop করো
nodetool stop COMPACTION
```

---

<a name="chapter-10"></a>
# Chapter 10 — Cluster Setup & Operations

## Prerequisites

```bash
# Java (Cassandra 4.x এ Java 11 বা 17 দরকার)
java -version

# Python (nodetool command এর জন্য)
python3 --version
```

---

## Configuration — cassandra.yaml

```yaml
# Cluster name — সব nodes এ same হতে হবে
cluster_name: 'MyCluster'

# Seeds — gossip bootstrap এর জন্য
# "bootstrap" মানে নতুন node প্রথমে কার সাথে কথা বলবে
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "192.168.1.10,192.168.1.11"
# প্রতিটা datacenter থেকে অন্তত একটা seed রাখো

# Network
listen_address: 192.168.1.10    # inter-node communication এর জন্য IP
rpc_address: 0.0.0.0            # client connection এর জন্য
broadcast_rpc_address: 192.168.1.10  # client কে এই IP জানাবে

# Ports
storage_port: 7000              # inter-node data transfer
ssl_storage_port: 7001          # encrypted inter-node
native_transport_port: 9042     # client CQL connection port
jmx_port: 7199                  # monitoring এর জন্য JMX port
# "JMX" = Java Management Extensions — Java applications monitor করার standard

# Directories
data_file_directories:
  - /var/lib/cassandra/data
commitlog_directory: /var/lib/cassandra/commitlog
hints_directory: /var/lib/cassandra/hints
saved_caches_directory: /var/lib/cassandra/saved_caches

# Partitioner
partitioner: org.apache.cassandra.dht.Murmur3Partitioner
# "DHT" = Distributed Hash Table — consistent hashing implement করে

# Snitch
endpoint_snitch: GossipingPropertyFileSnitch

# Memory
memtable_heap_space_in_mb: 2048  # Memtable এর জন্য heap space

# Compaction
concurrent_compactors: 2       # parallel compaction threads সংখ্যা

# Timeouts
read_request_timeout_in_ms: 5000   # 5 seconds
write_request_timeout_in_ms: 2000  # 2 seconds
range_request_timeout_in_ms: 10000 # 10 seconds
```

---

## JVM Tuning — jvm.options

**"JVM" কী?**
Java Virtual Machine — Java code run করার environment। Cassandra Java তে লেখা তাই JVM এ run করে।

**"Heap" কী?**
JVM এর memory যেখানে objects store হয়। Java automatically এটা manage করে (Garbage Collection)।

```
# Heap size
-Xms8G    # minimum heap (শুরুতে এতটা allocate করো)
-Xmx8G    # maximum heap
# Min = Max করো — GC pause কমাতে (variable size হলে resize এ pause হয়)

# GC — Garbage Collection algorithm
-XX:+UseG1GC
# G1GC = Garbage First GC — large heap এ better pause time
-XX:MaxGCPauseMillis=500  # maximum GC pause 500ms
```

**"Garbage Collection (GC)" কী?**
Java তে unused objects automatically memory থেকে remove করা হয়। GC চলার সময় JVM pause করে — এই সময় Cassandra respond করে না। বড় heap এ GC pause বেশি।

**Heap sizing rule:** Total RAM এর 25-50%, maximum **32GB**।
**কেন 32GB limit?** JVM এ "compressed OOPs" (Ordinary Object Pointers) feature আছে — 32GB পর্যন্ত memory address 32-bit এ store করা যায়, কম memory নেয়। 32GB এর বেশি হলে 64-bit pointer লাগে — বেশি memory, কম efficient।

---

## nodetool — Cluster Management

**"nodetool" কী?**
Cassandra এর command-line management tool। Cluster status দেখা, operations trigger করা সব এই tool দিয়ে।

```bash
# Cluster status
nodetool status
# Output: UN = Up Normal, DN = Down Normal, UJ = Up Joining, UL = Up Leaving

nodetool info                 # current node এর detailed info
nodetool ring                 # token ring দেখো (কোন node কোন range)
nodetool describecluster      # cluster overview

# Node operations
nodetool drain                # write বন্ধ করো, flush করো (graceful shutdown আগে)
nodetool decommission         # node gracefully remove করো (data অন্যত্র move করে)
# "decommission" মানে একটা node কে পরিকল্পিতভাবে cluster থেকে সরানো
nodetool removenode <host-id> # dead node remove করো (already down এমন)
nodetool assassinate <address> # unresponsive node forcefully remove করো

# Data operations
nodetool flush keyspace table   # memtable flush করো
nodetool compact keyspace table # manual compaction trigger
nodetool repair keyspace        # anti-entropy repair
nodetool cleanup keyspace       # অন্য nodes এ চলে যাওয়া data delete করো
# cleanup = node add এর পরে old data যেটা এখন অন্য node এর দায়িত্ব সেটা মুছে দাও
nodetool rebuild -- <source-dc> # নতুন datacenter/node কে data দিয়ে bootstrap করো

# Performance monitoring
nodetool tpstats              # thread pool statistics
# "thread pool" = worker threads এর collection, বিভিন্ন কাজের জন্য আলাদা pool
nodetool tablehistograms keyspace.table  # read/write latency distribution
nodetool proxyhistograms      # coordinator level histograms
nodetool gcstats              # GC statistics দেখো

# Configuration
nodetool getlogginglevels
nodetool setlogginglevel org.apache.cassandra DEBUG
nodetool settraceprobability 0.01  # 1% requests trace করো
```

---

## Node Add করা

```bash
# নতুন node এ cassandra.yaml configure করো
# (same cluster_name, seeds set করো)
# তারপর Cassandra start করো — automatically bootstrap হবে

service cassandra start

# Progress দেখো
nodetool status  # UJ = Up Joining দেখাবে

# Bootstrap complete হলে UN হবে
# তারপর existing nodes এ cleanup চালাও
nodetool cleanup keyspace_name
```

---

## Node Remove করা

```bash
# Graceful removal (node alive আছে)
nodetool decommission
# Data অন্য nodes এ move হবে
# nodetool status এ UL = Up Leaving দেখাবে

# Dead node removal (node আর চলছে না)
nodetool status  # host-id নোট করো
nodetool removenode <host-id>
```

---

## Monitoring

**JMX Metrics:**
Cassandra সব metrics JMX এ expose করে। Prometheus + Grafana দিয়ে monitor করা যায়।

**"Prometheus" কী?**
Open-source monitoring system। Application থেকে metrics collect করে।

**"Grafana" কী?**
Metrics visualize করার tool। Prometheus এর data নিয়ে graphs বানায়।

**Key metrics:**
```
# Read/Write latency
ReadLatency, WriteLatency

# Pending tasks (কতটা কাজ queue এ আটকে আছে)
ThreadPools.*.PendingTasks

# Compaction queue
Compaction.PendingTasks

# GC
GarbageCollector.*.CollectionTime
```

---

<a name="chapter-11"></a>
# Chapter 11 — Performance & Tuning

## Data Model Optimization

**1. Query-first design:**
Cassandra তে schema design করো access pattern (কীভাবে data read হবে) অনুযায়ী — entity relationship (কোন data কোনটার সাথে related) অনুযায়ী নয়।

প্রতিটা query pattern এর জন্য আলাদা table তৈরি করো (intentional denormalization)।

```sql
-- Query 1: customer এর orders চাই (customer_id দিয়ে)
CREATE TABLE orders_by_customer (
  customer_id UUID,
  order_time  TIMESTAMP,
  order_id    UUID,
  amount      DECIMAL,
  PRIMARY KEY (customer_id, order_time, order_id)
) WITH CLUSTERING ORDER BY (order_time DESC);

-- Query 2: নির্দিষ্ট date এর সব orders চাই
CREATE TABLE orders_by_date (
  order_date  DATE,
  order_time  TIMESTAMP,
  order_id    UUID,
  customer_id UUID,
  amount      DECIMAL,
  PRIMARY KEY (order_date, order_time, order_id)
) WITH CLUSTERING ORDER BY (order_time DESC);
-- Same data, different partition key = different table
```

**2. Partition size control:**
```sql
PRIMARY KEY ((user_id, bucket), timestamp, event_id)
-- bucket = 'YYYY-MM' বা 'YYYY-MM-DD'
-- প্রতি মাসে বা প্রতিদিন আলাদা partition
```

**3. Avoid large IN queries:**
```sql
-- Bad — অনেক partitions এ broadcast হয়
SELECT * FROM users WHERE user_id IN (id1, id2, ..., id1000);
-- Better — async concurrent queries চালাও (application level এ)
```

---

## Caching Tuning

```sql
-- Table level cache configure করো
ALTER TABLE hot_data
  WITH caching = {
    'keys': 'ALL',           -- সব partition keys cache করো
    'rows_per_partition': '100'  -- প্রতিটা partition এর 100 rows cache করো
  };
```

---

## Compression Tuning

**"Compression" কী?**
Data কে smaller size এ encode করা। Disk এ কম জায়গা লাগে এবং I/O কম হয়।

```sql
ALTER TABLE my_table WITH compression = {
  'class': 'LZ4Compressor',
  'chunk_length_in_kb': 64
};
```

| Compressor | Speed | Ratio | কখন |
|-----------|-------|-------|-----|
| LZ4 | Fastest | Moderate | Default — good balance |
| Snappy | Fast | Moderate | Similar to LZ4 |
| Deflate | Slower | Better | Storage বাঁচাতে |
| Zstd | Fast | Best | Modern systems এ |

---

## Read Performance

```bash
# SSTable count দেখো (কম হলে read fast)
nodetool tablehistograms keyspace.table

# Compaction pending দেখো
nodetool compactionstats
```

**Read latency বাড়লে চেক করো:**
1. SSTable count বেশি? → compaction চালাও
2. Tombstone বেশি? → repair + compaction
3. Heap pressure বেশি? → JVM tuning
4. STCS use করছ? → LCS try করো read-heavy workload এ

---

## Write Performance

```bash
# Write pending দেখো
nodetool tpstats | grep MutationStage
# MutationStage = write operations handle করার thread pool
```

**Write latency বাড়লে:**
1. Commit log disk slow? → SSD use করো
2. Compaction করতে পারছে না? → concurrent_compactors বাড়াও
3. Memtable flush slow? → data directory SSD তে নাও

---

## Tracing

**"Tracing" কী?**
একটা request কোথায় কতটা সময় নিচ্ছে তার detailed log। Performance debug করতে কাজে আসে।

```sql
-- Specific query trace করো
TRACING ON;
SELECT * FROM users WHERE user_id = uuid_here;
TRACING OFF;

-- Probabilistic tracing (nodetool দিয়ে)
nodetool settraceprobability 0.01
-- 1% requests trace করো (production এ কম রাখো — overhead আছে)
```

---

## OS Tuning

```bash
# /etc/sysctl.conf
net.core.rmem_max = 16777216   # maximum receive buffer size
net.core.wmem_max = 16777216   # maximum send buffer size
vm.max_map_count = 1048575     # maximum memory map areas per process
vm.swappiness = 1              # swap প্রায় কখনো না করো

# Transparent Huge Pages disable করো
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# THP (Transparent Huge Pages) = OS automatically বড় memory pages use করার feature
# Cassandra এ এটা GC pause বাড়ায়

# ulimits — resource limits
cassandra soft nofile 100000   # maximum open file descriptors (= connections)
cassandra hard nofile 100000
cassandra soft nproc 32768     # maximum processes
cassandra hard nproc 32768
cassandra soft memlock unlimited  # memory lock — unlimited
cassandra hard memlock unlimited
```

---

<a name="chapter-12"></a>
# Chapter 12 — Security

## Authentication

**"Authentication" কী?**
"তুমি কে?" প্রমাণ করা। Username/password দিয়ে।

```yaml
# cassandra.yaml
authenticator: PasswordAuthenticator  # default: AllowAllAuthenticator (no auth)
authorizer: CassandraAuthorizer       # default: AllowAllAuthorizer (no restrictions)
```

```sql
-- Default superuser: cassandra/cassandra — প্রথমেই change করো
ALTER USER cassandra WITH PASSWORD 'new_strong_password';

-- নতুন user তৈরি
CREATE USER app_user WITH PASSWORD 'password' NOSUPERUSER;

-- Role (Cassandra 2.2+)
CREATE ROLE app_role WITH PASSWORD = 'password' AND LOGIN = true;
GRANT SELECT ON TABLE keyspace.table TO app_role;
-- SELECT permission = read করতে পারবে কিন্তু write না
GRANT app_role TO app_user;
-- user কে role assign করো
```

---

## Authorization

**"Authorization" কী?**
"তুমি এটা করতে পারো কিনা" check করা। Authentication এর পরে হয়।

```sql
-- Permissions দাও
GRANT ALL ON KEYSPACE my_app TO app_role;
-- ALL = সব permissions
GRANT SELECT ON TABLE my_app.users TO readonly_role;
GRANT MODIFY ON TABLE my_app.orders TO write_role;
-- MODIFY = INSERT, UPDATE, DELETE করতে পারবে

-- Revoke (permission তুলে নাও)
REVOKE SELECT ON TABLE my_app.users FROM readonly_role;

-- List permissions
LIST ALL PERMISSIONS;
LIST ALL PERMISSIONS OF app_user;
```

---

## Encryption

### Node-to-Node Encryption (SSL/TLS)

**"SSL/TLS" কী?**
Network এ data পাঠানোর সময় encrypt করার protocol। মাঝপথে কেউ intercept করলেও পড়তে পারবে না।

```yaml
# cassandra.yaml
server_encryption_options:
  internode_encryption: all     # none, dc (শুধু cross-DC), rack, all
  keystore: /path/to/keystore.jks
  keystore_password: password
  truststore: /path/to/truststore.jks
  truststore_password: password
  require_client_auth: true
```

**"Keystore" কী?**
নিজের certificate এবং private key রাখার secure file।

**"Truststore" কী?**
Trusted certificates রাখার file। কাদের certificates বিশ্বাস করবে সেটা এখানে থাকে।

### Client-to-Node Encryption

```yaml
client_encryption_options:
  enabled: true
  keystore: /path/to/keystore.jks
  keystore_password: password
  require_client_auth: false
```

---

<a name="chapter-13"></a>
# Chapter 13 — Backup & Recovery

## Snapshot — Physical Backup

**"Snapshot" কী?**
একটা নির্দিষ্ট মুহূর্তে data files এর exact copy। SSTable files এর hard links তৈরি হয়।

**"Hard link" কী?**
Linux এ একটা file এর একাধিক নাম। Actual file একটাই কিন্তু দুটো path দিয়ে access করা যায়। একটা path delete করলে file থাকে। Hard link মানে snapshot নেওয়া fast — file copy না করে শুধু link তৈরি।

```bash
# Snapshot নাও (সব keyspaces)
nodetool snapshot

# Specific keyspace
nodetool snapshot -t snapshot_name my_keyspace
# -t = tag/name

# Snapshot list দেখো
nodetool listsnapshots

# Snapshot clear করো
nodetool clearsnapshot -t snapshot_name
```

Snapshot location:
```
/var/lib/cassandra/data/<keyspace>/<table>/snapshots/<snapshot_name>/
```

---

## Snapshot Backup Strategy

```bash
# 1. Snapshot নাও
nodetool snapshot -t daily_backup

# 2. Schema backup করো
cqlsh -e "DESCRIBE SCHEMA" > /backup/schema.cql
# DESCRIBE SCHEMA = সব keyspaces, tables, indexes এর CREATE statements

# 3. Snapshot files copy করো (rsync দিয়ে)
rsync -avz /var/lib/cassandra/data/ backup_server:/backup/cassandra/
# rsync = efficient file sync tool

# 4. পুরনো snapshots clear করো
nodetool clearsnapshot -t old_backup
```

---

## Restore

```bash
# 1. Target node এ cassandra stop করো
service cassandra stop

# 2. Existing data clear করো
rm -rf /var/lib/cassandra/data/keyspace/table/*

# 3. Schema restore করো
cqlsh < /backup/schema.cql

# 4. Snapshot files copy করো data directory তে
# 5. Cassandra start করো
service cassandra start

# 6. nodetool refresh (restart ছাড়া SSTable load করো)
nodetool refresh keyspace table
```

---

## Incremental Backup

**"Incremental backup" কী?**
শুধু last backup এর পরে changed data backup করা। Full backup এর চেয়ে কম space এবং time লাগে।

```yaml
# cassandra.yaml
incremental_backups: true
```

SSTable flush এর সময় hard links তৈরি হয়:
```
/var/lib/cassandra/data/<keyspace>/<table>/backups/
```

Full snapshot + incremental backup = approximate Point-in-Time Recovery।

---

## Medusa — Backup Tool

**"Medusa" কী?**
DataStax এর open-source Cassandra backup tool। S3, GCS, Azure Blob support করে।

```bash
pip install cassandra-medusa

# Configure
medusa configure

# Backup করো
medusa backup --backup-name daily-2024-03-23

# Restore করো
medusa restore-node --backup-name daily-2024-03-23

# List backups
medusa list-backups
```

---

<a name="chapter-14"></a>
# Chapter 14 — Real Patterns & Use Cases

## 1. Time-Series Data — IoT / Metrics

সবচেয়ে common Cassandra use case।

```sql
CREATE TABLE sensor_readings (
  sensor_id   TEXT,
  bucket      TEXT,    -- 'YYYY-MM-DD' format
  reading_time TIMESTAMP,
  temperature  FLOAT,
  humidity     FLOAT,
  PRIMARY KEY ((sensor_id, bucket), reading_time)
) WITH CLUSTERING ORDER BY (reading_time DESC)
  AND compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'DAYS',
    'compaction_window_size': 1
  }
  AND default_time_to_live = 7776000;  -- 90 days

-- Insert
INSERT INTO sensor_readings (sensor_id, bucket, reading_time, temperature, humidity)
VALUES ('sensor_001', '2024-03-23', toTimestamp(now()), 28.5, 65.0);

-- Query — last hour readings
SELECT * FROM sensor_readings
WHERE sensor_id = 'sensor_001'
  AND bucket = '2024-03-23'
  AND reading_time >= '2024-03-23 09:00:00'
LIMIT 100;
```

---

## 2. User Activity Log

```sql
CREATE TABLE user_activity (
  user_id    UUID,
  month      TEXT,         -- 'YYYY-MM' format (time bucketing)
  event_time TIMEUUID,     -- time-based UUID, sortable by time
  action     TEXT,
  resource   TEXT,
  ip_address INET,
  PRIMARY KEY ((user_id, month), event_time)
) WITH CLUSTERING ORDER BY (event_time DESC)
  AND default_time_to_live = 7776000;  -- 90 days

-- Latest events
SELECT * FROM user_activity
WHERE user_id = user_uuid_here
  AND month = '2024-03'
LIMIT 20;
```

---

## 3. Session Management

```sql
CREATE TABLE sessions (
  session_id TEXT PRIMARY KEY,
  user_id    UUID,
  data       TEXT,
  created_at TIMESTAMP,
  last_access TIMESTAMP
) WITH default_time_to_live = 86400;  -- 24 hours

-- Upsert session
INSERT INTO sessions (session_id, user_id, data, created_at, last_access)
VALUES ('token123', user_uuid, 'session_data', toTimestamp(now()), toTimestamp(now()))
USING TTL 86400;
```

---

## 4. Messaging / Chat

```sql
CREATE TABLE messages (
  conversation_id UUID,
  sent_at         TIMEUUID,
  message_id      UUID,
  sender_id       UUID,
  content         TEXT,
  PRIMARY KEY (conversation_id, sent_at)
) WITH CLUSTERING ORDER BY (sent_at DESC);

-- Recent messages
SELECT * FROM messages
WHERE conversation_id = conv_uuid_here
LIMIT 50;

-- Messages before a specific time (pagination)
SELECT * FROM messages
WHERE conversation_id = conv_uuid_here
  AND sent_at < maxTimeuuid('2024-03-23 12:00:00')
  -- maxTimeuuid = এই timestamp এর সবচেয়ে বড় possible TIMEUUID
LIMIT 50;
```

---

## 5. Multi-Datacenter Replication

```sql
CREATE KEYSPACE app
WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc-us-east': 3,
  'dc-eu-west': 3,
  'dc-ap-southeast': 2
};
```

```properties
# cassandra-rackdc.properties (dc-us-east এর node এ)
dc=dc-us-east
rack=rack1
```

**LOCAL_QUORUM use করো:**
```
Write: LOCAL_QUORUM  -- local datacenter এর majority (fast, local)
Read: LOCAL_QUORUM   -- local datacenter থেকে পড়ো (low latency)
```

Cross-datacenter replication asynchronously হয়। Local DC তে strong consistency, cross-DC eventual।

---

---

# PART 2: SCYLLADB

---

<a name="chapter-15"></a>
# Chapter 15 — What is ScyllaDB & Why It Exists

## ScyllaDB কী?

ScyllaDB হলো **Apache Cassandra এর drop-in replacement**, C++ এ লেখা। 2015 সালে তৈরি।

**"Drop-in replacement" কী?**
Existing system এর জায়গায় রাখা যায় এমন alternative। Same interface, same protocol — application code change করতে হয় না। Cassandra এর driver দিয়েই ScyllaDB তে connect করা যায়।

Compatible মানে:
- Same CQL (Cassandra Query Language)
- Same drivers (Cassandra driver ScyllaDB তেও কাজ করে)
- Same data model
- Compatible with Cassandra cluster (mixed cluster possible — কিছু nodes Cassandra, কিছু ScyllaDB)

কিন্তু ভেতরে সম্পূর্ণ আলাদা।

---

## কেন ScyllaDB তৈরি হয়েছিল

Cassandra Java তে লেখা। Java এর কিছু সমস্যা production এ:

**JVM GC pauses:**
Java তে unused memory automatically free করা হয় — Garbage Collection (GC)। GC চলার সময় JVM pause করে। Large heap এ GC pause কয়েক seconds পর্যন্ত হতে পারে — এই সময় Cassandra respond করে না।

**Java memory overhead:**
JVM নিজে memory নেয়। Object header, references — সব কিছুতে extra memory লাগে। Actual data এর চেয়ে বেশি memory নষ্ট হয়।

**OS এর সাথে দূরত্ব:**
Java OS resources directly access করতে পারে না — JVM এর মাধ্যমে যেতে হয়। Native code (C/C++) এর চেয়ে কম efficient।

ScyllaDB C++ এ লেখা, **Seastar framework** এর উপর। ফলে:
- JVM নেই, GC নেই → no GC pauses
- Shard-per-core architecture → OS scheduling bypass করে
- io_uring, DPDK → OS bypass করে directly hardware access possible
- Memory নিজেই manage করে

---

## ScyllaDB এর Claims

- Cassandra এর চেয়ে **10x বেশি throughput** same hardware এ
- **Consistent low latency** — GC pause নেই, spiky latency নেই
- **Same hardware এ কম nodes** — operational cost কমে
- **Better resource utilization** — CPU এবং memory efficiently use করে

---

<a name="chapter-16"></a>
# Chapter 16 — ScyllaDB Architecture — Shard-per-Core

## Seastar Framework

ScyllaDB [Seastar](http://seastar.io/) framework এর উপর built। Seastar হলো high-performance async applications লেখার C++ framework।

**"Async" (Asynchronous) কী?**
Operation শেষ হওয়ার জন্য wait না করে পরের কাজে যাওয়া। Wait করতে হলে OS কে বলো "শেষ হলে জানাও।" এতে thread block হয় না, অন্য কাজ করতে পারে।

**Seastar এর key ideas:**
- **Shared-nothing architecture** — প্রতিটা core আলাদা, নিজের memory নিজে manage করে। Cores এর মধ্যে কোনো sharing নেই।
- **Cooperative scheduling** — OS scheduler bypass। Application নিজেই task schedule করে।
  - "OS scheduler" = OS decide করে কোন process/thread কখন CPU পাবে। Cooperative মানে application নিজে বলে "আমি এখন pause করছি, অন্যরা চলো।"
- **Async I/O** — io_uring (Linux 5.1+) বা aio দিয়ে non-blocking I/O

---

## Shard-per-Core Architecture

**Cassandra এর thread model:**
Thread pool থেকে threads তৈরি হয়। Threads shared memory access করে → locking দরকার। Context switching overhead আছে।

**ScyllaDB এর shard-per-core model:**
```
CPU Core 0  →  Shard 0  (নিজের memory, নিজের data subset)
CPU Core 1  →  Shard 1  (নিজের memory, নিজের data subset)
CPU Core 2  →  Shard 2  (নিজের memory, নিজের data subset)
...
```

প্রতিটা shard এর:
- Dedicated CPU core (share করে না)
- Dedicated memory pool (অন্য shards এর সাথে shared না)
- Dedicated disk I/O queue
- Dedicated network queue
- **অন্য shards এর সাথে কোনো locking নেই**

Request এলে partition key এর hash দিয়ে কোন shard handle করবে determine হয়। ভুল shard এ গেলে **inter-shard message** পাঠায় (lightweight, fast)।

---

## Reactor Model

প্রতিটা shard একটা **reactor** চালায়।

**"Reactor pattern" কী?**
Event-driven architecture। Single thread এ event loop চলে। Events আসলে handlers call হয়। কোনো blocking নেই।

```
Reactor Loop (প্রতিটা shard এ):
  1. I/O events check করো (network data আসছে? disk read শেষ?)
  2. Ready tasks execute করো
  3. Timer events check করো (TTL expire? scheduled tasks?)
  4. Repeat
```

**কেন blocking করা যাবে না?**
Single thread এ যদি একটা operation block করে (wait করে) তাহলে পুরো shard block হয়ে যাবে। তাই সব operations async।

---

## Memory Management — Seastar Allocator

ScyllaDB নিজস্ব **memory allocator** use করে।

**"Memory allocator" কী?**
Program কে memory দেওয়া এবং ফেরত নেওয়ার mechanism। Standard C library তে `malloc/free`। Seastar এর নিজস্ব allocator আছে যেটা per-shard।

**জানে কোন shard কোন memory use করছে।** Cross-shard memory access avoid করে। GC এর মতো automatic collection নেই — C++ RAII (Resource Acquisition Is Initialization) pattern এ manual management।

**"RAII" কী?**
C++ technique — object তৈরি হলে resource acquire করো, object destroy হলে resource release করো। Automatic, safe, no GC needed।

---

<a name="chapter-17"></a>
# Chapter 17 — ScyllaDB vs Cassandra — Differences

## Architecture Differences

| বিষয় | Cassandra | ScyllaDB |
|------|-----------|---------|
| Language | Java | C++ |
| Scheduling | JVM threads + OS scheduler | Shard-per-core, cooperative scheduling |
| GC | JVM GC (pauses possible) | নেই (C++ RAII) |
| Memory | JVM heap + off-heap | Shared-nothing per shard |
| I/O | Java NIO | io_uring / aio (more efficient) |
| Compaction | Mostly single thread | Per-shard parallel |

**"Java NIO" কী?**
Java New I/O — non-blocking I/O framework। Efficient কিন্তু io_uring (Linux kernel এর newer mechanism) এর চেয়ে কম efficient।

**"io_uring" কী?**
Linux kernel এর modern async I/O interface। System call overhead কমায়, batch operations support করে। Database এর জন্য অনেক efficient।

---

## Performance Differences

**Latency:**
- Cassandra: GC pause এ latency spike হয় (p99 latency high হতে পারে)
  - "p99 latency" = 99th percentile latency। 100টা request এর মধ্যে 99টার চেয়ে বেশি সময় নেওয়া request এর latency।
- ScyllaDB: Consistent low latency (GC নেই)

**Throughput:**
Same hardware এ ScyllaDB generally 3-10x বেশি throughput।

**Resource efficiency:**
ScyllaDB কম nodes এ same workload handle করে। কিন্তু প্রতিটা node এর সব resources (সব CPU cores) ScyllaDB use করে।

---

## Operational Differences

### ScyllaDB Shard Awareness

Driver **shard-aware** হলে সরাসরি সঠিক shard এ connect করতে পারে — unnecessary routing এড়ানো যায়।

```python
from cassandra.cluster import Cluster
from cassandra.policies import TokenAwarePolicy, DCAwareRoundRobinPolicy

cluster = Cluster(
  contact_points=['node1', 'node2'],
  load_balancing_policy=TokenAwarePolicy(DCAwareRoundRobinPolicy())
)
# TokenAwarePolicy = partition key দিয়ে সরাসরি সঠিক node এ যাবে
```

### Tablets (ScyllaDB 6.0+)

**"Tablets" কী?**
Cassandra এর virtual nodes (VNodes) এর পরিবর্তে ScyllaDB এর নিজস্ব data distribution mechanism।

VNodes এ প্রতিটা node fixed token ranges পায়। Tablets এ data dynamically distribute হয়।

Tablets এর সুবিধা:
- Node add/remove এ faster rebalancing
- Better load balancing
- Automatic split/merge

---

## CQL Compatibility

ScyllaDB Cassandra CQL compatible। কিছু পার্থক্য:

**ScyllaDB তে extra features:**
- `BYPASS CACHE` hint — specific query cache bypass করতে
- Per-partition rate limiting — partition level এ request limit করতে
- Workload prioritization — কোন queries বেশি priority পাবে
- Tablets (6.0+) — নতুন data distribution

**Cassandra features যা ScyllaDB তে different:**
- Materialized views এর implementation আলাদা
- Secondary indexes এর implementation আলাদা
- কিছু JMX metrics আলাদা

---

<a name="chapter-18"></a>
# Chapter 18 — ScyllaDB Setup & Configuration

## Installation

```bash
# Ubuntu
curl -sSf https://downloads.scylladb.com/deb/ubuntu/scylla-2024.2.list | \
  sudo tee /etc/apt/sources.list.d/scylla.list
sudo apt-get update
sudo apt-get install -y scylla

# ScyllaDB setup script — hardware optimizations automatically করে
sudo scylla_setup
# Interactive mode — RAID, NTP, network, CPU governor সব configure করে
```

---

## scylla.yaml — Configuration

Cassandra এর cassandra.yaml এর মতো, কিন্তু ScyllaDB specific options আছে:

```yaml
# Cluster
cluster_name: 'ScyllaCluster'

# Seeds
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "192.168.1.10,192.168.1.11"

# Network
listen_address: 192.168.1.10
rpc_address: 0.0.0.0
broadcast_rpc_address: 192.168.1.10
native_transport_port: 9042

# ScyllaDB specific options
developer_mode: false
# developer_mode: true করলে fsync disable হয়
# "fsync" = OS কে force করা disk এ লিখতে
# Development এ faster কিন্তু crash এ data হারাবে

# Memory limit
# Default: ScyllaDB automatically সব available memory নেয়
# Limit করতে:
# memory: 16G

# Commitlog
commitlog_directory: /var/lib/scylla/commitlog
commitlog_sync: periodic
commitlog_sync_period_in_ms: 10000

# Data
data_file_directories:
  - /var/lib/scylla/data
```

---

## ScyllaDB Setup Script এর কাজ

`scylla_setup` অনেক কিছু automatically করে:

```bash
# CPU governor performance mode set করে
cpupower frequency-set --governor performance
# "governor" = CPU clock speed control করার policy
# performance mode = সবসময় maximum speed

# NTP sync verify করে
# "NTP" = Network Time Protocol — সব servers এর clock sync রাখে
# Distributed system এ timestamp সব জায়গায় same হওয়া critical

# Disk setup — XFS filesystem recommend করে

# Network IRQ affinity set করে
# "IRQ" = Interrupt Request — hardware events OS কে interrupt করে জানায়
# "Affinity" = specific CPU core এ bind করা
# Network IRQ affinity = network interrupts specific CPUs এ handle করো
```

---

## scyllatop — ScyllaDB top

```bash
scyllatop   # real-time metrics, htop এর মতো
# CPU usage, memory, read/write ops — সব দেখা যায়
```

---

## nodetool

ScyllaDB এর nodetool Cassandra এর মতোই:

```bash
nodetool status     # cluster health
nodetool info       # node info
nodetool ring       # token ring
nodetool flush      # memtable flush
nodetool compact    # compaction
nodetool repair     # repair
nodetool drain      # graceful shutdown আগে
nodetool decommission  # node remove
nodetool tpstats    # thread pool statistics
nodetool tablehistograms  # latency histograms
```

---

<a name="chapter-19"></a>
# Chapter 19 — ScyllaDB Specific Features

## Workload Prioritization

ScyllaDB workloads এ priority assign করা যায়:

**Workload types:**
- **Interactive** — user-facing queries, low latency priority
- **Batch** — background processing, high throughput কিন্তু lower priority
- **Background** — compaction, repair

Interactive workload এ response দ্রুত দেওয়া হয়, batch এর সময় বেশি লাগলে চলে।

---

## Per-Partition Rate Limiting

ScyllaDB partition level এ rate limit করা যায়:

```sql
ALTER TABLE my_table WITH per_partition_rate_limit = {
  'max_reads_per_second': 100,
  'max_writes_per_second': 50
};
```

**কেন দরকার?**
Hot partition protect করতে। একটা partition এ অনেক বেশি requests আসলে সেই node এ pressure বাড়ে। Rate limit দিলে controlled load থাকে।

---

## BYPASS CACHE

Specific query cache bypass করতে:

```sql
SELECT * FROM my_table WHERE partition_key = 'value' BYPASS CACHE;
```

**কখন use করবে?**
Bulk scan বা analytics query তে। এই queries cache pollute করে (popular কিন্তু ভবিষ্যতে আর দরকার হবে না এমন data cache ভরে ফেলে)। BYPASS CACHE দিলে hot data cache থেকে বের হয় না।

---

## Consistent Topology Changes (Raft)

ScyllaDB 5.0+ এ internal metadata (schema changes, topology) এর জন্য **Raft consensus** use করে।

**"Raft consensus" কী?**
Distributed systems এ একটা decision নেওয়ার consensus algorithm। সব nodes agree করলেই action হয়। Leader election, log replication সব handle করে।

Cassandra এ gossip দিয়ে হয় — eventually consistent। Raft এ strongly consistent।

**Raft এর সুবিধা:**
- Schema changes atomic এবং consistent — partial schema change এর situation হয় না
- Split-brain scenario তে safer

---

## Tablets (ScyllaDB 6.0+)

```yaml
# scylla.yaml
tablets:
  enabled: true   # default in 6.0+
```

**Traditional VNodes vs Tablets:**

VNodes এ:
- প্রতিটা node N টা token range পায় (N = num_tokens)
- Range fixed — rebalancing এ range migrate করতে হয়

Tablets এ:
- Fine-grained (ছোট ছোট) data units
- Load imbalance detect হলে automatically tablet split বা move হয়
- Node add করলে faster rebalancing

---

<a name="chapter-20"></a>
# Chapter 20 — ScyllaDB Tuning

## CPU Tuning

ScyllaDB সব available CPUs use করে। Production এ dedicated server দাও।

```bash
# Performance CPU governor
sudo cpupower frequency-set --governor performance

# Verify
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# "performance" দেখালে ঠিক আছে
```

---

## Memory Tuning

```yaml
# scylla.yaml
# ScyllaDB automatically সব free RAM নেয়
# Limit করতে (GB তে):
memory: 32G
```

**Reserve করো:**
- OS এর জন্য minimum 4GB বা 5% রাখো
- ScyllaDB OS page cache ও use করে, তাই বেশি দেওয়া ভালো

---

## Disk Tuning

```bash
# XFS filesystem recommended
mkfs.xfs /dev/sdb
mount -o noatime,nodiratime /dev/sdb /var/lib/scylla/data
# noatime = file access time update করো না (performance)

# I/O scheduler
echo mq-deadline > /sys/block/sdb/queue/scheduler
# "I/O scheduler" = disk এ কোন order এ read/write হবে সেটা manage করে
# SSD তে: none বা mq-deadline
# HDD তে: mq-deadline বা bfq

# Read-ahead কমাও
blockdev --setra 4 /dev/sdb
# "Read-ahead" = sequential read এ আগে থেকে extra data load করা
# Random access database workload এ এটা waste
```

**Separate disk recommendation:**
- Data: Fast SSD (NVMe preferred)
- Commitlog: Separate fast disk বা same SSD
- আলাদা রাখলে data write এবং commitlog write compete করে না

---

## Network Tuning

```bash
# /etc/sysctl.conf
net.core.rmem_max = 16777216     # maximum receive buffer (bytes)
net.core.wmem_max = 16777216     # maximum send buffer (bytes)
net.core.netdev_max_backlog = 300000  # network packet queue size
net.ipv4.tcp_congestion_control = bbr
# "BBR" = Bottleneck Bandwidth and RTT — Google এর TCP congestion control algorithm
# High bandwidth networks এ better throughput

# ScyllaDB setup script এটা automatically করে
```

---

## Compaction Tuning

```sql
ALTER TABLE my_table WITH compaction = {
  'class': 'TimeWindowCompactionStrategy',
  'compaction_window_unit': 'DAYS',
  'compaction_window_size': 1
};
```

---

## Monitoring — Prometheus + Grafana

ScyllaDB Prometheus metrics natively expose করে:

```yaml
# scylla.yaml
prometheus_port: 9180
prometheus_address: 0.0.0.0
```

```yaml
# Prometheus config (prometheus.yml)
scrape_configs:
  - job_name: 'scylla'
    static_configs:
      - targets: ['node1:9180', 'node2:9180']
    # scrape = Prometheus নিজে গিয়ে metrics collect করে
```

---

## ScyllaDB Monitoring Stack

```bash
# Official monitoring stack (Docker Compose)
git clone https://github.com/scylladb/scylla-monitoring
cd scylla-monitoring
./start-all.sh -s node1,node2,node3
# Grafana: http://localhost:3000
# Pre-built dashboards আছে — সব important metrics দেখা যায়
```

---

## Migration from Cassandra to ScyllaDB

### Option 1: Rolling Migration (Zero Downtime)

**"Rolling migration" কী?**
একটা একটা করে nodes replace করা। পুরো cluster একসাথে down করতে হয় না।

```bash
# 1. ScyllaDB node Cassandra cluster এ add করো
#    (mixed cluster supported)
# 2. Data automatically replicate হয়
# 3. একটা একটা করে Cassandra node remove করো
# 4. শেষে সব ScyllaDB
```

### Option 2: SSTableloader (Offline)

```bash
# Cassandra থেকে SSTables snapshot নাও
nodetool snapshot my_keyspace

# ScyllaDB তে load করো
sstableloader -d scylla_node /path/to/snapshots/
```

---

# Quick Reference Summary

## Cassandra vs ScyllaDB Feature Matrix

| Feature | Cassandra | ScyllaDB |
|---------|-----------|---------|
| CQL | ✓ | ✓ (compatible) |
| Same drivers | ✓ | ✓ |
| VNodes | ✓ | ✓ |
| Tablets | ✗ | ✓ (6.0+) |
| Raft metadata | ✗ | ✓ (5.0+) |
| GC pauses | সম্ভব | নেই |
| Per-partition rate limit | ✗ | ✓ |
| BYPASS CACHE | ✗ | ✓ |
| Prometheus native | ✗ (JMX দরকার) | ✓ |
| Language | Java | C++ |

---

## Data Modeling Checklist

```
□ Partition key high cardinality আছে?
□ Partition size < 100MB থাকবে?
□ Hot partition avoid হয়েছে?
□ প্রতিটা query pattern এর জন্য আলাদা table আছে?
□ Clustering order, query এর sort order এর সাথে match করে?
□ TTL set করা আছে (যেখানে দরকার)?
□ ALLOW FILTERING নেই?
□ Large collection (>1000 elements) নেই?
□ Counter table আলাদা?
□ Time-series এ TWCS compaction আছে?
```

---

## Consistency Level Selection Guide

| Use Case | Write CL | Read CL |
|---------|---------|---------|
| Strong consistency | LOCAL_QUORUM | LOCAL_QUORUM |
| Best performance | ONE | ONE |
| Multi-DC strong | EACH_QUORUM | LOCAL_QUORUM |
| Analytics (stale ok) | ONE | ONE |
| Critical data | QUORUM | QUORUM |

---

## Common CQL Quick Reference

```sql
-- Keyspace
CREATE KEYSPACE ks WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy', 'dc1': 3
};
USE ks;

-- Table
CREATE TABLE t (
  pk UUID,
  ck TIMESTAMP,
  val TEXT,
  PRIMARY KEY (pk, ck)
) WITH CLUSTERING ORDER BY (ck DESC);

-- CRUD
INSERT INTO t (pk, ck, val) VALUES (uuid(), toTimestamp(now()), 'data')
USING TTL 86400;

SELECT * FROM t WHERE pk = uuid_here AND ck >= '2024-03-01' LIMIT 100;

UPDATE t SET val = 'new' WHERE pk = uuid_here AND ck = ts_here;

DELETE FROM t WHERE pk = uuid_here AND ck = ts_here;

-- Inspect
DESCRIBE TABLE t;
```

---

## nodetool Quick Reference

```bash
nodetool status              # cluster health (UN/DN)
nodetool info                # current node info
nodetool tpstats             # thread pool stats
nodetool tablehistograms ks.t  # latency histogram
nodetool compactionstats     # compaction progress
nodetool repair ks           # anti-entropy repair
nodetool flush ks t          # memtable flush
nodetool compact ks t        # manual compaction
nodetool drain               # graceful shutdown আগে
nodetool decommission        # node gracefully remove করো
```

---

## cassandra.yaml / scylla.yaml Key Settings

```yaml
cluster_name: 'MyCluster'
seeds: "node1,node2"
listen_address: <node-ip>
rpc_address: 0.0.0.0
broadcast_rpc_address: <node-ip>
endpoint_snitch: GossipingPropertyFileSnitch
num_tokens: 16
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
```

---

## OS Tuning Checklist

```bash
# sysctl.conf
vm.swappiness = 1
vm.max_map_count = 1048575
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# Transparent Huge Pages — MUST disable
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# ulimits
cassandra/scylla soft nofile 100000
cassandra/scylla hard nofile 100000
cassandra/scylla soft memlock unlimited
cassandra/scylla hard memlock unlimited

# CPU governor
cpupower frequency-set --governor performance
```

---

*Cassandra + ScyllaDB Complete Study Guide (Revised) | সব technical terms in-place explained*
*Architecture, Operations, and Administration*
