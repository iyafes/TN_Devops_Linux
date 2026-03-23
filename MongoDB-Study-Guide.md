# MongoDB Complete Study Guide (Revised)
### Architecture · Internals · Replica Set · Sharding · Tuning · Patterns
### সব technical terms সহজ ভাষায় in-place explain করা আছে

---

## Table of Contents

1. [Chapter 1 — What is MongoDB & Why It Exists](#chapter-1)
2. [Chapter 2 — Core Concepts & Data Model](#chapter-2)
3. [Chapter 3 — Internal Architecture](#chapter-3)
4. [Chapter 4 — CRUD Operations](#chapter-4)
5. [Chapter 5 — Indexing](#chapter-5)
6. [Chapter 6 — Aggregation Framework](#chapter-6)
7. [Chapter 7 — Transactions](#chapter-7)
8. [Chapter 8 — Replication — Replica Set](#chapter-8)
9. [Chapter 9 — Sharding](#chapter-9)
10. [Chapter 10 — Storage Engine — WiredTiger](#chapter-10)
11. [Chapter 11 — Performance & Tuning](#chapter-11)
12. [Chapter 12 — Security](#chapter-12)
13. [Chapter 13 — Backup & Recovery](#chapter-13)
14. [Chapter 14 — Real Patterns & Use Cases](#chapter-14)

---

<a name="chapter-1"></a>
# Chapter 1 — What is MongoDB & Why It Exists

## MongoDB কী?

MongoDB হলো একটা **document-oriented NoSQL database**।

**"Document-oriented" মানে কী?**
Relational database (PostgreSQL, MySQL) এ data রাখো rows এবং columns এ — একটা spreadsheet এর মতো। MongoDB তে data রাখো **documents** এ — JSON এর মতো দেখতে। প্রতিটা document নিজের মতো structure রাখতে পারে।

**"NoSQL" মানে কী?**
"Not Only SQL" — SQL ছাড়া বা SQL এর পাশাপাশি অন্য query mechanism use করে এমন database। Relational model follow করে না। NoSQL databases বিভিন্ন ধরনের: document (MongoDB), key-value (Redis), wide-column (Cassandra), graph (Neo4j)।

Data store হয় **BSON documents** এ — JSON এর binary (0 এবং 1) representation।

**"Binary" কী?**
Computer এ সব কিছু শেষ পর্যন্ত 0 এবং 1 এ store হয়। Text based JSON এর চেয়ে binary format parse করা দ্রুত এবং বেশি data type support করে।

2009 সালে 10gen (পরে MongoDB Inc.) তৈরি করে।

---

## কেন MongoDB তৈরি হয়েছিল — The Problem

2000s এর শেষে web applications এর data patterns বদলে যাচ্ছিল।

**Relational DB এর সমস্যা:**
- **Schema rigid** — "Schema" মানে data এর structure এর definition। Relational DB তে আগে থেকে বলে দিতে হয় কোন columns থাকবে। নতুন field add করতে `ALTER TABLE` লাগে — large table এ এটা hours নিতে পারে।
- **Hierarchical data তে অনেক join** — "Join" মানে দুটো table এর data একসাথে আনা। Complex nested data (যেমন order এর ভেতরে items, প্রতিটা item এর ভেতরে details) রাখতে অনেক tables এবং joins দরকার।
- **Horizontal scaling কঠিন** — "Horizontal scaling" মানে বেশি servers যোগ করা। Relational DB তে এটা manually configure করতে হয়।

**MongoDB এর solution:**
- **Schema flexible** — document এ যেকোনো structure রাখা যায়, আগে define করতে হয় না
- Nested objects এবং arrays natively support করে
- Built-in horizontal scaling (sharding — data multiple servers এ ভাগ করা)
- Developer friendly — application এর objects directly document হিসেবে store হয়

---

## MongoDB কোথায় ভালো, কোথায় না

**MongoDB ভালো:**
- **Varied বা evolving schema** — product catalog (প্রতিটা product এর আলাদা attributes), user profiles
- **Hierarchical বা nested data** — blog posts with comments, orders with line items
- **High write throughput** — অনেক বেশি write operation
- **Rapid prototyping** — দ্রুত development, schema বারবার change হয়

**MongoDB ভালো না:**
- **Complex multi-table transactions** — banking, financial systems যেখানে অনেক tables এ একসাথে atomic change দরকার
- **Strongly relational data** — data গুলো inherently related এবং normalize থাকা দরকার
- **Data warehousing** — complex analytics এর জন্য columnar database better

---

## MongoDB vs PostgreSQL — মূল পার্থক্য

| বিষয় | PostgreSQL | MongoDB |
|------|-----------|---------|
| Data model | Tables, rows, columns | Collections, documents |
| Schema | Strict (define করতে হয়) | Flexible (optional) |
| Relationships | Foreign keys, JOINs | Embedded documents বা references |
| Transactions | Full ACID | Multi-document ACID (4.0+) |
| Scaling | Vertical primarily | Horizontal (sharding built-in) |
| Query language | SQL | MQL (MongoDB Query Language) |
| Joins | Native, efficient | $lookup (less efficient) |

**"ACID" কী?**
- **A**tomicity: সব operations হবে অথবা কোনোটাই হবে না
- **C**onsistency: data সবসময় valid state এ থাকবে
- **I**solation: একটা transaction অন্যটাকে affect করবে না
- **D**urability: committed data হারাবে না

---

<a name="chapter-2"></a>
# Chapter 2 — Core Concepts & Data Model

## Document

MongoDB এর fundamental (মূল) unit হলো **document**। BSON format এ store হয়।

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "Rahim",
  "age": 28,
  "city": "Dhaka",
  "address": {
    "street": "Mirpur Road",
    "zip": "1216"
  },
  "hobbies": ["reading", "coding", "chess"],
  "created_at": ISODate("2024-03-23T10:00:00Z")
}
```

**Document এর বৈশিষ্ট্য:**
- Maximum size: **16MB** per document
- Key-value pairs — key হলো field name, value হলো data
- **Nested documents** — document এর ভেতরে document (address field এর মতো)
- **Arrays** — একটা field এ multiple values (hobbies এর মতো)
- Keys case-sensitive ("Name" এবং "name" আলাদা)

---

## BSON — Binary JSON

BSON হলো MongoDB এর wire format (network এ পাঠানোর format) এবং storage format (disk এ রাখার format)।

**"Wire format" কী?**
Client এবং server এর মধ্যে network এ data পাঠানোর format। যেমন চিঠি পাঠাতে envelope এ রাখতে হয় — সেটাই wire format।

**JSON এর চেয়ে BSON ভালো কেন:**
- Extra data types যা JSON এ নেই:
  - `ObjectId` — 12-byte unique identifier (নিচে বিস্তারিত)
  - `Date` — 64-bit integer (milliseconds since epoch) — JSON এ date শুধু string হিসেবে
  - `BinData` — binary data (file, image)
  - `Decimal128` — high-precision decimal (financial calculations এ দরকার)
  - `Int32`, `Int64` — explicit integer types (JSON এ সব number same)
- Binary format দ্রুত parse হয়
- Size calculation efficient

---

## _id Field

প্রতিটা document এ `_id` field mandatory। Primary key এর মতো কাজ করে।

**Default: ObjectId**
```
ObjectId("507f1f77bcf86cd799439011")
```

**ObjectId এর structure (12 bytes):**
```
[4 bytes timestamp][5 bytes random][3 bytes incrementing counter]
```
- **Timestamp থেকে creation time** বের করা যায়: `ObjectId("...").getTimestamp()`
- **Multiple servers এ collision ছাড়া** generate করা যায় — random part এর কারণে দুটো server একই সময়ে আলাদা ObjectId তৈরি করবে
- **Roughly sortable by insertion time** — ObjectId sort করলে insertion order পাওয়া যায় (approximate)

**"Collision" কী?**
দুটো আলাদা item এর same identifier generate হওয়া। এটা primary key হিসেবে ব্যবহারের সময় মারাত্মক সমস্যা।

**Custom _id:**
```json
{ "_id": "AWB123456", "status": "in_transit" }
{ "_id": 1001, "name": "Rahim" }
```
যেকোনো unique value দেওয়া যায়। `_id` **immutable** — একবার set করলে change করা যায় না।

---

## Collection

Collection হলো documents এর group। SQL database এর **table** এর মতো, কিন্তু schema নেই।

**Collection এর বৈশিষ্ট্য:**
- Documents এর একই structure থাকতে হয় না — একটা document এ 3টা field, আরেকটায় 10টা
- **Dynamically created** — প্রথম document insert হলে collection তৈরি হয়, আগে create করতে হয় না
- Index store করে
- Namespace format: `database_name.collection_name` (যেমন `myapp.users`)

### Capped Collection

Fixed size collection। Circular buffer এর মতো — নতুন documents পুরনোটাকে overwrite করে।

**"Circular buffer" কী?**
মনে করো একটা গোলাকার track। গাড়ি চলছে। track ভরে গেলে পুরনো গাড়িগুলো সরে যায়। নতুন গাড়ি জায়গা পায়।

```javascript
db.createCollection("logs", {
  capped: true,
  size: 10485760,  // 10MB maximum size
  max: 1000        // maximum 1000 documents
})
```

**Use case:** Logs, event streams — শুধু recent data দরকার, পুরনো automatically চলে যাক।

---

## Database

MongoDB instance এ multiple databases থাকতে পারে।

**Special databases (built-in):**
- `admin` — administrative operations, user management। Superuser এখানে থাকে।
- `local` — replication data store হয় এখানে। এই DB কখনো replicate হয় না।
- `config` — sharding metadata (কোন data কোন server এ) store হয়। Sharded cluster এ।

```javascript
use myapp          // database switch করো (না থাকলে তৈরি হবে প্রথম document insert এ)
db                 // current database এর নাম দেখো
show dbs           // সব databases দেখো
db.dropDatabase()  // current database delete করো
```

---

## Data Modeling — Embedding vs Referencing

MongoDB এ data model করার দুটো approach। এটা MongoDB এর সবচেয়ে important design decision।

### Embedding (Denormalization)

Related data একটাই document এ রাখো।

**"Denormalization" কী?**
Relational DB তে normalization মানে data duplicate না রেখে আলাদা tables এ রাখা। Denormalization এর উল্টো — data duplicate হলেও একজায়গায় রাখো। MongoDB তে এটা intentional।

```json
{
  "_id": ObjectId("..."),
  "order_id": "AWB123",
  "customer": {
    "name": "Rahim",
    "phone": "01711..."
  },
  "items": [
    { "product": "Shirt", "qty": 2, "price": 500 },
    { "product": "Pants", "qty": 1, "price": 800 }
  ],
  "total": 1800
}
```

**কখন embed করবে:**
- "Contains" relationship — order contains items (item আলাদা exist করে না)
- Data একসাথে read হয় — order দেখলে items ও দেখতে হয়
- Sub-part আলাদাভাবে access দরকার নেই
- Array এর size bounded এবং বড় হবে না (যেমন maximum 10টা items)

**Pros:** Single query তে সব data, fast read, atomic update
**Cons:** Document size বাড়ে, data duplication হতে পারে

### Referencing (Normalization)

Separate collections এ রাখো, reference (অন্যের `_id`) দিয়ে link করো।

```json
// users collection
{ "_id": ObjectId("user1"), "name": "Rahim" }

// orders collection
{
  "_id": ObjectId("order1"),
  "user_id": ObjectId("user1"),  // reference — user এর _id store করা আছে
  "total": 1800
}
```

**কখন reference করবে:**
- Data অনেক জায়গায় reuse হয় — user info অনেক orders এ দরকার
- Sub-document অনেক বড় হতে পারে
- **Unbounded array** — user এর orders সংখ্যা অসীম হতে পারে
  - "Unbounded" মানে কোনো upper limit নেই
- Data আলাদাভাবেও access হয় — user কে orders ছাড়াও query করা হয়

**Pros:** Data duplication নেই, flexible
**Cons:** Multiple queries বা `$lookup` (join এর মতো) দরকার

### Practical Rule: "How data is accessed?"

MongoDB এ এই প্রশ্নটাই সব: **"এই data কীভাবে read করা হবে?"**

Access pattern অনুযায়ী model করো, normalization theory অনুযায়ী নয়।

---

## Schema Validation

MongoDB flexible schema হলেও validation enforce করা যায় — কোনো field mandatory, কোন type হবে:

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email"],   // এই fields থাকতেই হবে
      properties: {
        name: {
          bsonType: "string",
          description: "must be a string and is required"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        email: {
          bsonType: "string",
          pattern: "^.+@.+$"   // regex pattern — email format validate
        }
      }
    }
  },
  validationAction: "error"  // "error" = reject করো, "warn" = allow কিন্তু log করো
})
```

**"Regex" বা "Regular Expression" কী?**
String pattern match করার জন্য একটা mini language। `^.+@.+$` মানে: শুরুতে কিছু characters, তারপর @, তারপর আরো কিছু। Email validate করতে ব্যবহার।

---

<a name="chapter-3"></a>
# Chapter 3 — Internal Architecture

## MongoDB Architecture Overview

```
Client Application
      ↓
MongoDB Driver
(BSON encode/decode করে, connection pool manage করে)
      ↓
mongod process (main server)
      ↓
Query Engine (query parse করে, plan বানায়, optimize করে)
      ↓
Storage Engine — WiredTiger (actual data read/write)
      ↓
OS File System Cache (OS এর নিজস্ব memory cache)
      ↓
Disk (data files, journal, oplog)
```

---

## mongod — The Server Process

`mongod` হলো MongoDB এর main server **daemon** (background process)।

**"Daemon" কী?**
Background এ চলা process। Terminal বন্ধ করলেও চলতে থাকে। Linux এ `-d` suffix দিয়ে daemon বোঝানো হয় (mongod, sshd, nginx)।

`mongod` এর কাজ:
- Client connections handle করা
- Query processing (parse, plan, execute)
- Data read/write (storage engine এর মাধ্যমে)
- Replication manage করা
- Background operations (TTL index cleanup, etc.)

---

## mongos — The Query Router

Sharded cluster এ `mongos` হলো routing layer।

**"Router" কী?**
Traffic কোথায় পাঠাবে সেটা decide করে। Internet router এর মতো — data packet কোন path এ যাবে বলে দেয়।

Client `mongos` এ connect করে। `mongos` জানে কোন data কোন shard (server) এ আছে, সে সঠিক জায়গায় forward করে।

`mongos` **stateless** — নিজে কোনো data রাখে না। Restart করা safe।

---

## Connection Handling

MongoDB প্রতিটা connection এর জন্য একটা **thread** তৈরি করে।

**"Thread-per-connection model" কী?**
প্রতিটা client connection এর জন্য আলাদা thread। 1000 clients = 1000 threads। Thread মানে execution এর একটা path — একটা কাজ করার জন্য একটা worker।

**Connection limits:**
```
ulimit -n 65536  # OS level — একটা process কতটা file descriptor (connection) খুলতে পারবে
```

**"File descriptor" কী?**
Linux এ সব কিছু file হিসেবে treat হয় — network connection ও। প্রতিটা open connection একটা file descriptor নেয়।

```yaml
# mongod.conf
net:
  maxIncomingConnections: 1000000
```

**Connection Pool (Driver side):**
**"Connection pool" কী?**
নতুন connection তৈরি করতে time লাগে (TCP handshake, authentication)। Pool এ আগে থেকে কিছু connections তৈরি রাখা হয়। Request এলে pool থেকে নেওয়া হয়, কাজ শেষে return করা হয়।

```javascript
MongoClient.connect(uri, {
  maxPoolSize: 100,        // maximum connection pool size
  minPoolSize: 10,         // minimum — সবসময় এতটা ready থাকবে
  maxIdleTimeMS: 30000     // idle connection 30 seconds পর close করো
})
```

---

## Memory Architecture

MongoDB heavily relies on **OS page cache**।

**"OS page cache" কী?**
OS disk থেকে data read করলে RAM এ রেখে দেয়। পরে আবার same data দরকার হলে disk এ না গিয়ে RAM থেকে দেয়। এটাই page cache। OS automatically manage করে।

```
RAM
├── WiredTiger Cache (default: max(50% RAM - 1GB, 256MB) পর্যন্ত)
│   ├── Hot data (frequently accessed documents, indexes)
│   ├── Dirty pages (modified, disk এ লেখা হয়নি এখনো)
│   └── Internal data structures
└── OS File System Cache
    └── Memory-mapped data files
```

**"Hot data" কী?**
Frequently (বারবার) access হওয়া data। যেটা বেশি use হয় সেটা cache এ থাকলে fast।

**"Dirty pages" কী?**
Cache এ modify করা হয়েছে কিন্তু disk এ এখনো লেখা হয়নি এমন data। "Dirty" মানে modified।

**"Memory-mapped files" কী?**
OS data files কে directly RAM এর একটা portion এ map করে। File read করা মানে সেই RAM portion পড়া। OS automatically disk থেকে load করে এবং sync রাখে।

**Working set বোঝো:**
Working set মানে frequently access হওয়া data এর total size। এটা যদি RAM এ fit করে — performance excellent। RAM এর চেয়ে বড় হলে disk I/O বাড়ে — performance কমে।

---

## Journaling

Data corruption থেকে protect করতে MongoDB **journal** use করে।

**"Journal" বা "Write-Ahead Log (WAL)" কী?**
Write operation প্রথমে journal file এ লেখা হয়। তারপর actual data files এ। Crash হলে incomplete operations journal থেকে replay করে consistent state এ ফিরে আসা যায়।

**কেন এটা দরকার?**
Crash হলে data file অর্ধেক লেখা থাকতে পারে — corrupted। Journal থেকে বোঝা যায় কোন operation complete হয়েছিল, কোনটা হয়নি।

WiredTiger এ journal = write-ahead log (WAL)। Default এ enabled।

```yaml
storage:
  journal:
    enabled: true
    commitIntervalMs: 100  # journal কতক্ষণ পর পর disk এ flush হবে
```

---

## Oplog — Operations Log

Oplog (Operations Log) হলো Replica Set এর core mechanism।

**"Oplog" কী?**
`local.oplog.rs` collection এ থাকে। Capped collection — fixed size, পুরনো entries overwrite হয়। প্রতিটা write operation এখানে record হয়।

```json
{
  "ts": Timestamp(1711123200, 1),  // কখন হয়েছে
  "op": "i",          // operation type: i=insert, u=update, d=delete
  "ns": "mydb.users", // namespace (database.collection)
  "o": { "_id": ObjectId("..."), "name": "Rahim" }  // operation এর data
}
```

Replica members oplog থেকে operations replay করে sync করে। Primary তে কী হয়েছে Secondary সেটা oplog দেখে নিজে apply করে।

**Oplog size:**
```yaml
replication:
  oplogSizeMB: 10240  # 10GB
  # default: 5% of free disk, max 50GB
```

Oplog বড় হলে: Replica recovery window বড় হয়। Replica অনেকক্ষণ offline থাকলেও reconnect এ full resync দরকার না — oplog থেকে missed operations পড়ে নিতে পারে।

---

<a name="chapter-4"></a>
# Chapter 4 — CRUD Operations

**"CRUD" কী?**
**C**reate, **R**ead, **U**pdate, **D**elete — database এর চারটা basic operation।

---

## Database এবং Collection Operations

```javascript
// Database
use mydb                    // switch বা create (first write এ create হবে)
db.getName()                // current database এর নাম
db.stats()                  // database statistics
db.dropDatabase()           // delete করো

// Collection
db.createCollection("users")
db.getCollectionNames()     // সব collection এর list
db.users.drop()             // collection delete করো
db.users.stats()            // collection statistics

// Document count
db.users.count()            // deprecated — পুরনো method
db.users.estimatedDocumentCount()  // fast, approximate (metadata থেকে)
db.users.countDocuments({ age: { $gt: 18 } })  // exact count with filter
```

---

## Insert

```javascript
// Single document insert
db.users.insertOne({
  name: "Rahim",
  age: 28,
  city: "Dhaka"
})
// Returns: { acknowledged: true, insertedId: ObjectId("...") }
// "acknowledged" = server confirm করেছে

// Multiple documents insert
db.users.insertMany([
  { name: "Karim", age: 25 },
  { name: "Sakib", age: 30 },
  { name: "Nadia", age: 22 }
])

// insertMany options
db.users.insertMany(docs, {
  ordered: false
  // ordered: true (default) = error হলে stop করো
  // ordered: false = error হলেও বাকিগুলো continue করো
})
```

---

## Read (Find)

```javascript
// সব documents
db.users.find()
db.users.find({})

// Filter দিয়ে
db.users.find({ city: "Dhaka" })
db.users.find({ age: 28, city: "Dhaka" })  // AND condition (উভয়ই match করতে হবে)

// Single document (প্রথমটা return করে)
db.users.findOne({ name: "Rahim" })

// Projection — কোন fields দেখাবে, কোনটা বাদ দেবে
// 1 = include করো, 0 = exclude করো
db.users.find({ city: "Dhaka" }, { name: 1, age: 1 })      // name, age দেখাও (+ _id automatically)
db.users.find({ city: "Dhaka" }, { name: 1, _id: 0 })      // _id বাদ দাও
db.users.find({ city: "Dhaka" }, { password: 0 })           // শুধু password বাদ, বাকি সব

// Comparison operators (তুলনার জন্য)
// "$" prefix মানে operator — MongoDB এর query language এ
db.users.find({ age: { $gt: 25 } })          // $gt = greater than (বেশি)
db.users.find({ age: { $gte: 25 } })         // $gte = greater than or equal (বেশি বা সমান)
db.users.find({ age: { $lt: 30 } })          // $lt = less than (কম)
db.users.find({ age: { $lte: 30 } })         // $lte = less than or equal
db.users.find({ age: { $eq: 28 } })          // $eq = equal
db.users.find({ age: { $ne: 28 } })          // $ne = not equal
db.users.find({ age: { $in: [25, 28, 30] } }) // $in = এই list এর যেকোনো একটা
db.users.find({ age: { $nin: [25, 28] } })   // $nin = not in list
db.users.find({ age: { $gt: 20, $lt: 30 } }) // range: 20 এর বেশি এবং 30 এর কম

// Logical operators
db.users.find({ $and: [{ age: { $gt: 20 } }, { city: "Dhaka" }] })
// $and = উভয়ই true হতে হবে
db.users.find({ $or: [{ city: "Dhaka" }, { city: "Chittagong" }] })
// $or = যেকোনো একটা true হলেই চলবে
db.users.find({ age: { $not: { $gt: 30 } } })
// $not = condition এর বিপরীত
db.users.find({ $nor: [{ city: "Dhaka" }, { city: "Sylhet" }] })
// $nor = কোনোটাই true না হলে match

// Element operators
db.users.find({ email: { $exists: true } })   // field আছে কিনা
db.users.find({ age: { $type: "int" } })      // field এর BSON type check

// Array operators
db.users.find({ hobbies: "coding" })           // array তে "coding" আছে কিনা
db.users.find({ hobbies: { $all: ["coding", "chess"] } })  // সব elements আছে কিনা
db.users.find({ hobbies: { $size: 3 } })       // array এ exactly 3টা element
db.users.find({ "hobbies.0": "reading" })      // array এর first element (index 0)

// Nested document query — dot notation দিয়ে
db.users.find({ "address.city": "Dhaka" })
// "address.city" মানে address field এর ভেতরে city field

// Regular expression — pattern matching
db.users.find({ name: /^Rah/ })
// /^Rah/ মানে "Rah" দিয়ে শুরু

db.users.find({ name: { $regex: "^Rah", $options: "i" } })
// $options: "i" = case insensitive (বড়-ছোট হাতের পার্থক্য নেই)

// Cursor methods — result কে shape করো
// "Cursor" = database থেকে আসা result এর একটা pointer
db.users.find().sort({ age: 1 })        // 1 = ascending (ছোট থেকে বড়)
db.users.find().sort({ age: -1 })       // -1 = descending (বড় থেকে ছোট)
db.users.find().limit(10)               // প্রথম 10টা
db.users.find().skip(20).limit(10)      // 20টা skip করে পরের 10টা (pagination এর জন্য)

// Explain — query plan দেখো (debug করতে)
db.users.find({ age: { $gt: 25 } }).explain("executionStats")
```

---

## Update

```javascript
// Single document update
db.users.updateOne(
  { name: "Rahim" },           // filter — কোন document update করবো
  { $set: { age: 29 } }        // update — কী change করবো
)
// $set = নির্দিষ্ট field এর value set করো

// Multiple documents update
db.users.updateMany(
  { city: "Dhaka" },
  { $set: { country: "Bangladesh" } }
)

// Replace entire document (_id ছাড়া সব replace হয়)
db.users.replaceOne(
  { name: "Rahim" },
  { name: "Rahim", age: 29, city: "Dhaka" }
  // পুরনো document এর অন্য সব fields চলে যাবে
)

// Upsert — না থাকলে insert, থাকলে update
db.users.updateOne(
  { email: "rahim@example.com" },
  { $set: { name: "Rahim", age: 28 } },
  { upsert: true }
  // "upsert" = update + insert এর combination
)

// Update operators
// $set    = field এর value set করো (নতুন field ও add করতে পারে)
// $unset  = field remove করো  { $unset: { old_field: "" } }
// $inc    = numeric field এর value বাড়াও বা কমাও  { $inc: { login_count: 1 } }
// $mul    = numeric field কে multiply করো  { $mul: { price: 1.1 } }
// $rename = field এর নাম change করো  { $rename: { "old": "new" } }
// $min    = current value এর চেয়ে argument ছোট হলে update করো
// $max    = current value এর চেয়ে argument বড় হলে update করো

// Array update operators
// $push   = array তে element add করো
// $pull   = array থেকে condition match element remove করো
// $pop    = array এর first (-1) বা last (1) element remove করো
// $addToSet = array তে add করো, duplicate হবে না (Set এর মতো)

// Example — multiple operators একসাথে
db.users.updateOne(
  { _id: ObjectId("...") },
  {
    $set: { city: "Dhaka" },
    $unset: { old_field: "" },
    $inc: { login_count: 1 },
    $push: { hobbies: "swimming" },
    $pull: { hobbies: "chess" }
  }
)

// Array element update — positional operator "$"
db.orders.updateOne(
  { "items.product": "Shirt" },  // items array এ "Shirt" product আছে এমন document
  { $set: { "items.$.price": 600 } }
  // "$" = matched array element এর position — "Shirt" এর price update করো
)

// findOneAndUpdate — update করে document return করে
db.users.findOneAndUpdate(
  { name: "Rahim" },
  { $inc: { age: 1 } },
  { returnDocument: "after" }   // "before" = update আগের document, "after" = update পরের
)
```

---

## Delete

```javascript
// Single document delete
db.users.deleteOne({ name: "Rahim" })

// Multiple documents delete
db.users.deleteMany({ city: "Sylhet" })

// সব documents delete (collection structure রাখো)
db.users.deleteMany({})

// Find করে delete করো (deleted document return করে)
db.users.findOneAndDelete({ name: "Rahim" })
```

---

## Bulk Operations

একটা network round trip এ অনেক operations করা যায়:

```javascript
db.users.bulkWrite([
  { insertOne: { document: { name: "Ali", age: 25 } } },
  { updateOne: {
    filter: { name: "Rahim" },
    update: { $set: { age: 29 } }
  }},
  { deleteOne: { filter: { name: "Old User" } } },
  { replaceOne: {
    filter: { name: "Karim" },
    replacement: { name: "Karim", age: 26 }
  }}
], { ordered: false })  // false = error হলেও বাকিগুলো চলবে
```

**"Round trip" কী?**
Client থেকে server এ request পাঠানো এবং response পাওয়া — এই পুরো journey। Network এ এটা time নেয়। Bulk operations এ একটা round trip এ অনেক কাজ হয়।

---

<a name="chapter-5"></a>
# Chapter 5 — Indexing

Index ছাড়া MongoDB প্রতিটা query তে **collection scan** করে — সব documents check করতে হয়।

**"Collection scan" কী?**
মনে করো একটা বই এ কোনো index নেই। "Redis" শব্দ খুঁজতে হলে প্রথম page থেকে শেষ পর্যন্ত পড়তে হবে — O(n) (n = total pages)। Index থাকলে সরাসরি সেই page এ যাওয়া যায় — O(log n)।

---

## Index Internals — B-Tree

MongoDB default index হলো **B-Tree** (actually B+ Tree)।

**"B-Tree" কী?**
Balanced tree data structure।

```
                [28, 35]             ← root node
               /    |    \
         [22, 25]  [30, 32]  [40, 45]  ← internal nodes
```

**"Balanced" কী?**
Tree এর সব leaves (শেষ nodes) same depth এ থাকে। এতে worst case এও search time predictable — O(log n)।

**B-Tree এ query কীভাবে fast:**
- **Equality query** (age = 28): Root থেকে শুরু করে সঠিক leaf এ পৌঁছাও — O(log n)
- **Range query** (age > 25): B+ Tree তে leaf nodes linked — range efficiently traverse করা যায়
- **Sorted output**: Index already sorted — ORDER BY এর জন্য extra sort লাগে না

---

## Index Types

### Single Field Index

```javascript
db.users.createIndex({ age: 1 })         // 1 = ascending order
db.users.createIndex({ age: -1 })        // -1 = descending order
// Single field এ ascending/descending same performance
```

### Compound Index

Multiple fields এর উপর একটা index:

```javascript
db.users.createIndex({ city: 1, age: 1 })
```

**ESR Rule (Equality, Sort, Range) — Compound Index এর সবচেয়ে important concept:**

Compound index এ field order matter করে। Best practice:
1. **Equality** fields আগে — যেটায় exact match করো
2. **Sort** fields মাঝে — যেটায় ORDER BY করো
3. **Range** fields শেষে — যেটায় >, <, between করো

```javascript
// Query: city = "Dhaka" AND age > 25 ORDER BY name
// Optimal index:
db.users.createIndex({ city: 1, name: 1, age: 1 })
//                      equality  sort    range
```

**Prefix rule:**
Compound index `{a, b, c}` এটা এই queries support করে:
- `{a}` alone
- `{a, b}` together
- `{a, b, c}` together

কিন্তু `{b}` alone বা `{b, c}` together support করে না। Leftmost prefix থেকে শুরু করতে হবে।

### Multikey Index (Array Index)

Array field এ index করলে automatically multikey হয়। Array এর প্রতিটা element আলাদা index entry পায়:

```javascript
db.users.createIndex({ hobbies: 1 })
// hobbies: ["coding", "chess"] হলে দুটো index entry তৈরি হয়
// "coding" এর জন্য একটা, "chess" এর জন্য একটা
```

**Limitation:** Compound index এ দুটো array field একসাথে index করা যায় না।

### Text Index

Full-text search এর জন্য। সাধারণ index এ "mongodb index" লিখে search করলে exact match দরকার। Text index word-by-word search করে।

```javascript
db.articles.createIndex({ title: "text", body: "text" })

// Text search
db.articles.find({ $text: { $search: "mongodb index" } })
// "mongodb" OR "index" যেকোনোটা আছে এমন documents

// Relevance score সহ (কতটা relevant সেই score)
db.articles.find(
  { $text: { $search: "mongodb" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
// $meta = computed metadata, textScore = relevance score
```

**Limitation:** একটা collection এ শুধু একটা text index।

### Geospatial Index

Location-based query এর জন্য:

```javascript
// 2dsphere — spherical surface এ (পৃথিবীর মতো গোল) GeoJSON objects এর জন্য
db.locations.createIndex({ loc: "2dsphere" })

// Near query — কাছের locations খোঁজো
db.locations.find({
  loc: {
    $near: {
      $geometry: { type: "Point", coordinates: [90.4125, 23.8103] },
      // GeoJSON Point: [longitude, latitude]
      $maxDistance: 5000  // meters এ maximum distance
    }
  }
})
```

**"GeoJSON" কী?**
Geographic data represent করার JSON standard। `{ type: "Point", coordinates: [lon, lat] }` format এ।

### Hashed Index

Partition key এর hash value দিয়ে index। Sharding এ use হয়।

**"Hash" কী?**
একটা function যেটা input থেকে fixed length output তৈরি করে। "rahim" → "a7b3c9..." এর মতো। Same input সবসময় same output দেয়।

```javascript
db.users.createIndex({ _id: "hashed" })
// Range queries support করে না, শুধু equality
```

### Wildcard Index

Dynamic (runtime এ তৈরি) বা unknown field names এর জন্য:

```javascript
db.products.createIndex({ "attributes.$**": 1 })
// attributes এর যেকোনো nested field index হবে
// $** = wildcard — সব sub-fields
```

---

## Index Options

```javascript
db.users.createIndex(
  { email: 1 },
  {
    unique: true,
    // unique = duplicate value allow করবে না
    // email already আছে এমন document insert করলে error

    sparse: true,
    // sparse = field না থাকলে index এ include করবে না
    // email নেই এমন documents index এ থাকবে না

    expireAfterSeconds: 3600,
    // TTL index — document automatically delete হবে
    // Date field এ index করতে হবে

    name: "email_unique_idx",
    // custom index name (default: field_order এর combination)

    partialFilterExpression: { age: { $gt: 18 } }
    // partial index — শুধু এই condition match করা documents index হবে
  }
)
```

### TTL Index

Document automatically delete করার জন্য:

```javascript
db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 86400 }  // 24 hours = 86400 seconds
)
```

**কীভাবে কাজ করে:** MongoDB এর background job (TTL monitor) প্রতি 60 seconds এ run করে এবং expired documents delete করে।

### Partial Index

শুধু condition match করা documents index করে — smaller index, better performance:

```javascript
// শুধু active users index করো — inactive users index এ নেই
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { status: "active" } }
)
```

**সুবিধা:** Index ছোট → read fast, write এ index update কম।

---

## Index Management

```javascript
// সব indexes দেখো
db.users.getIndexes()
db.users.indexStats()   // কতটা accesses হয়েছে প্রতিটা index এ

// Index delete করো
db.users.dropIndex({ age: 1 })
db.users.dropIndex("age_1")        // index name দিয়ে
db.users.dropIndexes()             // সব drop করো (_id index ছাড়া)

// Index rebuild করো
db.users.reIndex()
```

---

## Query Plan Analysis — EXPLAIN

**"Query plan" কী?**
MongoDB query execute করার আগে plan তৈরি করে — কোন index use করবে, কীভাবে data access করবে। Query optimizer এই plan বানায়।

```javascript
db.users.find({ age: { $gt: 25 }, city: "Dhaka" }).explain("executionStats")
```

Output এ important fields:

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",        // IXSCAN = index scan — index use হচ্ছে (ভালো)
        "indexName": "city_1_age_1"
      }
    }
  },
  "executionStats": {
    "nReturned": 150,             // কতটা document return হয়েছে
    "totalKeysExamined": 155,     // কতটা index key check হয়েছে
    "totalDocsExamined": 155,     // কতটা document check হয়েছে
    "executionTimeMillis": 2      // কতটা সময় লেগেছে (milliseconds)
  }
}
```

**Stage names:**
- `COLLSCAN` — collection scan (খারাপ — index নেই বা use হচ্ছে না)
- `IXSCAN` — index scan (ভালো — index use হচ্ছে)
- `FETCH` — document fetch করছে index থেকে
- `SORT` — in-memory sort (index sort নেই — extra memory লাগছে)
- `PROJECTION` — field projection

**Efficiency check:**
`nReturned` ≈ `totalKeysExamined` হওয়া উচিত। বড় gap মানে index selective না — অনেক examine করে কম পাচ্ছে।

---

## Index Strategy — Common Mistakes

**1. Over-indexing:**
প্রতিটা write এ সব indexes update হয়। বেশি index মানে write slow।

**2. Unused indexes:**
```javascript
db.users.aggregate([{ $indexStats: {} }])
// accesses.ops = 0 মানে কেউ এই index use করছে না
```

**3. Low cardinality field এ index:**
**"Cardinality" কী?**
একটা field এ কতটা unique values আছে। "gender" field এ শুধু 2টা value — low cardinality। Index করা useless।

**4. Compound index এর wrong order:**
ESR rule follow না করলে index efficient হয় না।

---

<a name="chapter-6"></a>
# Chapter 6 — Aggregation Framework

**"Aggregation" কী?**
Data process করে summary বা transformed result বের করা। GROUP BY, SUM, AVG — এগুলো aggregation। MongoDB তে এটার জন্য Aggregation Framework আছে।

---

## Pipeline Concept

Aggregation pipeline হলো stages এর sequence। প্রতিটা stage আগের stage এর output নিয়ে কাজ করে।

**"Pipeline" কী?**
Data একটা pipe এর মধ্যে দিয়ে যাচ্ছে। প্রতিটা stage একটা filter বা transformer — data shape করে পরের stage এ পাঠায়।

```
Collection → Stage 1 (filter) → Stage 2 (group) → Stage 3 (sort) → Result
```

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  // Stage 1: filter — শুধু completed orders রাখো

  { $group: { _id: "$city", total: { $sum: "$amount" } } },
  // Stage 2: group by city, amount এর sum করো

  { $sort: { total: -1 } },
  // Stage 3: total এর উপর descending sort

  { $limit: 5 }
  // Stage 4: top 5 রাখো
])
```

---

## Important Stages

### $match — Filter

```javascript
{ $match: { status: "active", age: { $gt: 18 } } }
// Pipeline এর শুরুতে $match দিলে index use করতে পারে — early filtering
```

### $project — Shape Output

কোন fields রাখবো, কোনটা বাদ দেবো, computed fields তৈরি করবো:

```javascript
{ $project: {
  name: 1,
  age: 1,
  _id: 0,
  fullName: { $concat: ["$firstName", " ", "$lastName"] },
  // $concat = strings জোড়া লাগানো
  ageGroup: {
    $cond: {
      if: { $gte: ["$age", 18] },
      then: "adult",
      else: "minor"
    }
  }
  // $cond = conditional (if-else) — if ageGroup >= 18 then "adult" else "minor"
}}
```

### $group — Aggregate করো

Data group করো এবং aggregation করো:

```javascript
{ $group: {
  _id: "$city",                    // এই field অনুযায়ী group করো
  count: { $sum: 1 },              // প্রতিটা document এর জন্য 1 যোগ করো = count
  totalAmount: { $sum: "$amount" }, // amount field এর sum
  avgAmount: { $avg: "$amount" },   // average
  maxAmount: { $max: "$amount" },   // maximum
  minAmount: { $min: "$amount" },   // minimum
  names: { $push: "$name" },        // সব names array তে collect করো
  uniqueNames: { $addToSet: "$name" } // unique names collect করো
}}

// সব documents একসাথে group করো (overall stats)
{ $group: { _id: null, total: { $sum: "$amount" } } }
// _id: null মানে কোনো grouping নেই, সব একসাথে
```

### $sort, $limit, $skip

```javascript
{ $sort: { amount: -1, name: 1 } }  // amount descending, name ascending
{ $limit: 10 }                       // প্রথম 10টা
{ $skip: 20 }                        // 20টা skip করো
```

### $unwind — Array Flatten করো

Array কে individual documents এ expand করে:

```javascript
// Input document:
// { name: "Rahim", hobbies: ["coding", "chess"] }

{ $unwind: "$hobbies" }

// Output (2টা documents হবে):
// { name: "Rahim", hobbies: "coding" }
// { name: "Rahim", hobbies: "chess" }
```

**"Flatten" কী?**
Nested বা array data কে flat (এক level) এ নিয়ে আসা।

```javascript
// null/missing array handle করো
{ $unwind: { path: "$hobbies", preserveNullAndEmptyArrays: true } }
// hobbies না থাকলেও document রাখো
```

### $lookup — JOIN

দুটো collection join করো:

```javascript
{ $lookup: {
  from: "users",           // কোন collection এ join করবো
  localField: "user_id",   // current collection এর field (foreign key)
  foreignField: "_id",     // joined collection এর field
  as: "user_info"          // result এই field name এ array হিসেবে আসবে
}}
```

**"Join" কী?**
দুটো table/collection এর data একসাথে আনা। SQL এ native। MongoDB তে `$lookup` দিয়ে। Performance SQL join এর চেয়ে কম।

**Pipeline-based lookup (complex conditions):**
```javascript
{ $lookup: {
  from: "products",
  let: { orderId: "$_id" },
  // let = local variables define করো
  pipeline: [
    { $match: { $expr: { $eq: ["$$orderId", "$order_id"] } } }
    // $$orderId = let variable (double $ দিয়ে)
    // $order_id = foreign collection এর field (single $ দিয়ে)
  ],
  as: "matching_products"
}}
```

### $addFields / $set

Existing documents এ নতুন fields add করো:

```javascript
{ $addFields: {
  totalWithTax: { $multiply: ["$amount", 1.15] },
  // $multiply = গুণ করো
  processed: true
}}
```

### $facet — Multiple Pipelines একসাথে

একটাই pass এ multiple aggregations করো:

```javascript
{ $facet: {
  byCity: [
    { $group: { _id: "$city", count: { $sum: 1 } } }
  ],
  priceStats: [
    { $group: { _id: null, avg: { $avg: "$price" }, max: { $max: "$price" } } }
  ],
  topProducts: [
    { $sort: { sales: -1 } },
    { $limit: 5 }
  ]
}}
// Result:
// { byCity: [...], priceStats: [...], topProducts: [...] }
```

**"Pass" কী?**
Data কে একবার scan করা। $facet এ একবার scan করেই তিনটা আলাদা result পাওয়া যায়।

### $bucket / $bucketAuto

Data কে ranges (buckets) এ ভাগ করো:

```javascript
// Manual ranges
{ $bucket: {
  groupBy: "$age",
  boundaries: [0, 18, 30, 50, 100],
  // 0-18, 18-30, 30-50, 50-100 — এই buckets এ ভাগ হবে
  default: "other",
  output: { count: { $sum: 1 } }
}}

// Auto buckets — MongoDB নিজে ranges decide করবে
{ $bucketAuto: { groupBy: "$price", buckets: 5 } }
```

---

## Aggregation Pipeline Optimization

**1. $match আগে রাখো:** Data কমাও early। Index use করার সুযোগ দাও।

**2. $project আগে রাখো:** Unnecessary fields early remove করো।

**3. $sort + $limit একসাথে:** MongoDB automatically top-K sort করে (সব sort করার দরকার নেই — memory বাঁচে)।

**4. $lookup minimize করো:** Expensive operation। Possible হলে embed করো।

**5. allowDiskUse:**
```javascript
db.orders.aggregate([...], { allowDiskUse: true })
// 100MB memory limit exceed করলে disk এ temporary files লিখে চালাবে
```

---

<a name="chapter-7"></a>
# Chapter 7 — Transactions

MongoDB 4.0 থেকে multi-document ACID transactions। 4.2 থেকে sharded cluster এও।

---

## কখন Transaction দরকার

MongoDB এর single document operation naturally atomic (পুরোটা হবে অথবা কিছুই হবে না)। একটাই document এ সব data থাকলে transaction দরকার নেই।

Transaction দরকার:
- Multiple documents atomically update করতে হবে (সব update হবে বা কোনোটাই না)
- Multiple collections এ একসাথে write করতে হবে
- "All or nothing" guarantee দরকার (যেমন: bank transfer — একজনের account থেকে কাটা এবং অন্যজনের account এ যোগ করা — উভয়ই হতে হবে)

---

## Transaction Basics

```javascript
const session = client.startSession()
// session = transaction এর context, এটার মধ্যে সব operations করতে হবে

try {
  session.startTransaction({
    readConcern: { level: "snapshot" },
    // snapshot = transaction শুরুর সময়ের data দেখবে (consistent view)
    writeConcern: { w: "majority" }
    // majority = majority replicas এ write হলে success
  })

  await db.accounts.updateOne(
    { _id: "account1" },
    { $inc: { balance: -500 } },
    { session }  // session pass করতে হবে
  )

  await db.accounts.updateOne(
    { _id: "account2" },
    { $inc: { balance: 500 } },
    { session }
  )

  await session.commitTransaction()  // সব operations commit করো (permanent করো)
} catch (error) {
  await session.abortTransaction()   // কোনো error হলে সব rollback করো (বাতিল)
} finally {
  session.endSession()
}
```

**"Commit" কী?**
Transaction এর সব changes permanent করা। এরপর অন্য clients এই changes দেখতে পারবে।

**"Rollback" কী?**
Transaction এর সব changes বাতিল করা। যেন transaction হয়নি।

---

## Read/Write Concerns

### Read Concern

কোন data পড়বে — কতটা "committed":

**"Concern" কী?**
MongoDB কে বলা "আমি কতটা data safety চাই।" বেশি safety মানে বেশি latency।

| Level | কাজ |
|-------|-----|
| `local` | Latest data, durability guarantee নেই (default) — fastest |
| `majority` | Majority replica তে written data — more consistent |
| `snapshot` | Transaction start এর snapshot — repeatable read |
| `linearizable` | Strongest — সব previous write reflect করবে — slowest |

**"Durability" কী?**
Data permanently saved কিনা তার guarantee। Durability ছাড়া data cache এ থাকতে পারে — crash এ হারাতে পারে।

### Write Concern

Write কতটা "durable" হবে:

```javascript
{ w: 1 }           // শুধু primary acknowledge করলেই success (default, fast)
{ w: "majority" }  // majority replicas acknowledge করলে success (safer)
{ w: 0 }           // fire and forget — কোনো confirmation নেই (fastest, unsafe)
{ j: true }        // journal এ লেখার পর acknowledge (disk এ গেছে confirm)
{ wtimeout: 5000 } // 5 seconds এর মধ্যে acknowledge না হলে error
```

---

## Transaction Limitations

- Maximum **60 seconds** (default) — এর বেশি চললে abort হয়
- Transaction এ 1000 এর বেশি documents modify করা avoid করো (performance)
- DDL operations (createCollection, createIndex) transaction এ করা যায় না সাধারণত
- Sharded cluster এ cross-shard transaction (দুটো আলাদা shard এ) expensive

---

<a name="chapter-8"></a>
# Chapter 8 — Replication — Replica Set

Replica Set হলো MongoDB এর high availability mechanism।

**"High Availability (HA)" কী?**
System যাতে সবসময় চলে — downtime minimum হয়। কোনো server fail হলেও service চালু থাকে।

---

## Replica Set Architecture

```
Primary (Read + Write)
    ├── Secondary 1 (Read, Replication)
    ├── Secondary 2 (Read, Replication)
    └── Arbiter (Voting only, no data)
```

**Minimum recommended: 3 nodes** — 1 Primary + 2 Secondary, অথবা 2 data nodes + 1 Arbiter

**Maximum voting members: 7**

---

## Roles

### Primary

- সব write operations receive করে
- Oplog এ operations record করে
- Default read target (client সাধারণত Primary থেকে পড়ে)

### Secondary

- Primary থেকে oplog replicate করে (copy করে apply করে)
- **Async replication** — Primary Secondary এর জন্য wait করে না
  - "Async" মানে একসাথে না, পরে হলেও চলবে
- Replication lag থাকতে পারে — Secondary কিছুটা পিছিয়ে
- Read queries serve করতে পারে (configured হলে)
- Failover এ Primary হওয়ার candidate

### Arbiter

- Data store করে না
- শুধু election এ **vote** দেয় (tie-breaking)
- Odd number of voting members রাখতে Arbiter useful
- Resource কম লাগে (data রাখে না বলে)
- **Production এ arbiter avoid করো** — data redundancy কমে যায়

**"Election" কী?**
Primary fail করলে কোন Secondary নতুন Primary হবে সেটা vote এর মাধ্যমে decide হয়।

### Hidden Secondary

Replica থেকে replication করে কিন্তু client থেকে দেখা যায় না। Reporting বা backup এর জন্য:

```javascript
rs.reconfig({
  members: [
    ...
    { _id: 3, host: "hidden:27017", hidden: true, priority: 0 }
    // priority: 0 মানে কখনো Primary হবে না
    // hidden: true মানে client এর কাছে invisible
  ]
})
```

### Delayed Secondary

নির্দিষ্ট সময় পিছিয়ে থাকে। Accidental data deletion এ কাজে আসে:

```javascript
{ _id: 4, host: "delayed:27017", priority: 0, hidden: true, secondaryDelaySecs: 3600 }
// 1 hour পিছিয়ে থাকবে
// কেউ ভুলে data delete করলে এখান থেকে recover করা যাবে
```

---

## Replication কীভাবে কাজ করে

```
1. Client → Primary তে write করে
2. Primary → data store করে + oplog এ record করে
3. Secondary → Primary এর oplog পড়ে (poll করে বা push receive করে)
4. Secondary → নিজের data তে apply করে
5. Secondary → acknowledgement পাঠায় (write concern এ দরকার হলে)
```

**Asynchronous** — Primary Secondary এর জন্য wait করে না (write concern w:1 হলে)।

---

## Election — Automatic Failover

Primary unreachable হলে election শুরু হয়:

```
1. Secondary heartbeat fail করে — Primary কে ping করে কোনো response নেই
2. Secondary নিজে candidate হয়, অন্য members এর কাছে vote চায়
3. Majority vote পেলে নতুন Primary হয়
4. অন্য Secondaries নতুন Primary এর সাথে sync শুরু করে
5. পুরনো Primary ফিরে আসলে Secondary হিসেবে join করে
```

**"Heartbeat" কী?**
Regular interval এ ছোট ছোট message পাঠানো — "আমি alive আছি।" কেউ heartbeat না পাঠালে সে down বলে ধরা হয়।

**Election time: ~12 seconds** (default)। এই সময় writes fail করবে।

**Priority:**
```javascript
rs.reconfig({
  members: [
    { _id: 0, host: "node1:27017", priority: 2 },   // preferred primary (বেশি priority)
    { _id: 1, host: "node2:27017", priority: 1 },
    { _id: 2, host: "node3:27017", priority: 0 }    // কখনো Primary হবে না
  ]
})
```

---

## Replica Set Setup

### Configuration File

```yaml
# mongod.conf (প্রতিটা node এ)
storage:
  dbPath: /var/lib/mongodb

net:
  port: 27017
  bindIp: 0.0.0.0   # সব IPs এ listen করো

replication:
  replSetName: "rs0"           # সব nodes এ same name থাকতে হবে

security:
  keyFile: /etc/mongodb/keyfile
  # keyFile = nodes এর মধ্যে authentication এর জন্য shared secret key
```

### Keyfile তৈরি

```bash
openssl rand -base64 756 > /etc/mongodb/keyfile
# random 756 bytes generate করে base64 encode করো
chmod 400 /etc/mongodb/keyfile   # শুধু owner read করতে পারবে
chown mongodb:mongodb /etc/mongodb/keyfile
```

**"openssl" কী?**
Cryptography tools এর collection। `rand` command random bytes generate করে।

**"base64" কী?**
Binary data কে printable text এ convert করার encoding। Email attachment, keyfile ইত্যাদিতে use হয়।

### Initialize করো

```javascript
// Node 1 এ
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "node1:27017" },
    { _id: 1, host: "node2:27017" },
    { _id: 2, host: "node3:27017" }
  ]
})
```

---

## Replica Set Commands

```javascript
rs.status()                    // সব members এর current status দেখো
rs.conf()                      // current configuration দেখো
db.hello()                     // current node এর info (primary/secondary কিনা)
rs.printReplicationInfo()      // oplog info (Primary এ চালাও)
rs.printSecondaryReplicationInfo()  // replication lag কতটা (Secondary এ চালাও)

// Member add/remove
rs.add("node4:27017")
rs.addArb("arbiter:27017")
rs.remove("node4:27017")

// Stepdown — Primary কে gracefully Secondary বানাও
rs.stepDown(60)  // 60 seconds এর জন্য stepdown, election হবে

// Manual failover force করো
rs.stepDown()
```

---

## Read Preference

Secondary থেকে read করতে Read Preference configure করো:

| Mode | কাজ | কখন use করবে |
|------|-----|-------------|
| `primary` | শুধু Primary (default) | Strong consistency দরকার |
| `primaryPreferred` | Primary, না থাকলে Secondary | Primary ভালো কিন্তু availability important |
| `secondary` | শুধু Secondary | Read-heavy, stale data acceptable |
| `secondaryPreferred` | Secondary, না থাকলে Primary | Load বিতরণ করতে |
| `nearest` | Network latency সবচেয়ে কম এমন node | Low latency priority |

```javascript
db.users.find().readPref("secondaryPreferred")
```

**সতর্কতা:** Secondary তে stale (পুরনো) data থাকতে পারে। Consistency critical হলে Primary ব্যবহার করো।

---

<a name="chapter-9"></a>
# Chapter 9 — Sharding

Sharding হলো MongoDB এর horizontal scaling mechanism।

**কেন Sharding দরকার:**
- Dataset single server এর RAM বা disk capacity ছাড়িয়ে যাচ্ছে
- Write throughput single server এ handle করা যাচ্ছে না
- Horizontal scaling দরকার

**Sharding complex।** Single server বা Replica Set এ কাজ চললে Sharding avoid করো।

---

## Sharded Cluster Architecture

```
Client
  ↓
mongos (Query Router) — stateless, multiple থাকতে পারে
  ↓
Config Servers (Replica Set) — sharding metadata রাখে
  ↓
Shard 1 (Replica Set) — data এর একটা subset
Shard 2 (Replica Set) — data এর আরেকটা subset
Shard 3 (Replica Set) — data এর আরেকটা subset
```

**Config Servers:**
- Sharding metadata store করে — কোন shard এ কোন data আছে এই map
- নিজেও Replica Set হিসেবে deploy করতে হয় (3 nodes) — highly available
- mongos config servers থেকে routing table cache করে

**mongos:**
- Client এর single entry point — client শুধু mongos এর address জানে
- Stateless — নিজে data রাখে না। Restart করা safe।
- Multiple mongos থাকলে load balancing হয়

**Shard:**
- প্রতিটা shard একটা Replica Set — HA আছে প্রতিটা shard এ
- Data এর subset রাখে

---

## Shard Key — সবচেয়ে Important Decision

Shard key হলো সেই field(s) যার উপর ভিত্তি করে data shards এ distribute হবে।

**এই decision অনেক গুরুত্বপূর্ণ এবং পরিবর্তন করা কঠিন।**

```javascript
sh.shardCollection("mydb.orders", { customer_id: 1 })
// customer_id হলো shard key
```

### Good Shard Key এর বৈশিষ্ট্য

**High cardinality:**
অনেক unique values। Boolean ("true/false") বা gender এর মতো field ভালো shard key না — মাত্র 2-3টা unique value মানে মাত্র 2-3টা shard এ data যাবে।

**Low frequency:**
কোনো একটা value অনেক বেশি frequent হলে সেই shard **hot** হয়ে যাবে — সেই shard এ অনেক বেশি load।

**Non-monotonically increasing:**
"Monotonically increasing" মানে সবসময় বাড়তে থাকে — যেমন auto-increment id বা timestamp। এগুলো shard key হিসেবে ব্যবহার করলে সব new data সবসময় একটাই shard এ যাবে (last shard)। Hot shard problem।

---

## Sharding Strategies

### Range-Based Sharding

Shard key এর value range অনুযায়ী data distribute হয়:

```
Shard 1: customer_id 0 - 1000
Shard 2: customer_id 1001 - 2000
Shard 3: customer_id 2001+
```

**Pros:** Range query efficient — সব data একটা shard এ, single shard query
**Cons:** Monotonic key এ hot shard problem

### Hash-Based Sharding

Shard key এর hash value অনুযায়ী distribute:

```javascript
sh.shardCollection("mydb.orders", { customer_id: "hashed" })
```

**Hash** করলে monotonic value ও randomly distribute হয়।

**Pros:** Even distribution, hot shard problem নেই
**Cons:** Range query inefficient — hash এর পর range meaningful না, সব shards এ broadcast করতে হয়

### Zone-Based Sharding

Geographic বা logical partitioning:

```javascript
// Shard কে zone assign করো
sh.addShardToZone("shard1", "DHAKA")
sh.addShardToZone("shard2", "CHITTAGONG")

// Data range কে zone এ assign করো
sh.updateZoneKeyRange("mydb.orders",
  { region: "DHAKA" },
  { region: "DHAKB" },
  "DHAKA"
)
// "DHAKA" region এর orders সবসময় "DHAKA" zone এর shard এ থাকবে
```

---

## Chunks

MongoDB data কে **chunks** এ ভাগ করে (default 64MB)। প্রতিটা chunk একটা shard key range represent করে।

**"Chunk" কী?**
Data এর একটা continuous (একটানা) range। Chunk = shard key range এর একটা ভাগ।

**Balancer:**
Background process যেটা chunks এর মধ্যে even distribution maintain করে। কোনো shard এ বেশি chunks থাকলে অন্য shard এ migrate করে।

```javascript
sh.status()                    // sharding status দেখো
sh.getBalancerState()          // balancer on/off কিনা
sh.startBalancer()
sh.stopBalancer()

// Balancer window set করো — শুধু রাতে চলুক (low traffic সময়)
db.settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "01:00", stop: "05:00" } } },
  { upsert: true }
)
```

---

## Targeted vs Broadcast Queries

**Targeted query (efficient):**
Query তে shard key থাকলে mongos সঠিক shard এ route করে — শুধু একটা shard এ query।

**Broadcast query (inefficient):**
Shard key না থাকলে mongos সব shards এ পাঠায়, সব থেকে result নিয়ে merge করে।

```javascript
// Targeted — shard key আছে, একটা shard এ যাবে
db.orders.find({ customer_id: 1001 })

// Broadcast — shard key নেই, সব shards এ যাবে
db.orders.find({ status: "pending" })
```

**Design principle:** সব common queries যাতে shard key include করে।

---

## Sharding Setup

```javascript
// mongos এ
sh.enableSharding("mydb")

// আগে index তৈরি করো shard key তে
db.orders.createIndex({ customer_id: 1 })

// Collection shard করো
sh.shardCollection("mydb.orders", { customer_id: 1 })

// Status check করো
sh.status()
db.orders.getShardDistribution()   // data কোন shard এ কতটা আছে
```

---

<a name="chapter-10"></a>
# Chapter 10 — Storage Engine — WiredTiger

MongoDB 3.2 থেকে **WiredTiger** default storage engine।

**"Storage engine" কী?**
Database এর একটা component যেটা actual data disk এ read/write করে। MongoDB এর উপরের layer (query engine) storage engine কে বলে "এই data দাও" বা "এই data রাখো।" Storage engine implement করে কীভাবে data physically store হবে।

---

## WiredTiger Architecture

```
MongoDB API (CRUD operations)
     ↓
WiredTiger API
     ↓
WiredTiger Cache (in-memory B-Trees — hot data এখানে)
     ↓
Write-Ahead Log / Journal (crash recovery)
     ↓
Data Files (compressed, on disk)
```

---

## Document-Level Concurrency

WiredTiger **document-level locking** use করে।

**"Locking" কী?**
Multiple users একই data access করলে conflict হতে পারে। Lock দিয়ে বলা হয় "আমি এটা change করছি, অন্যরা wait করো।"

আগে MongoDB তে **collection-level lock** ছিল — একটা collection এ write হলে পুরো collection lock হয়ে যেত। অনেক slow।

এখন **document-level lock** — শুধু সেই নির্দিষ্ট document lock হয়। আলাদা documents এ simultaneously কাজ করা যায়।

**MVCC (Multi-Version Concurrency Control):**

**"MVCC" কী?**
Readers writers কে block করে না। Writer নতুন version তৈরি করে (old version রেখে দেয়)। Reader পুরনো consistent version দেখে। Write শেষ হলে new version visible হয়।

এর ফলে: read করার সময় write হলে lock এর জন্য wait করতে হয় না। Concurrency (একসাথে অনেক operations) অনেক বেশি।

---

## Compression

WiredTiger data compress করে store করে — disk usage কমে।

**"Compression" কী?**
Data কে smaller size এ encode করা। যেমন "aaaaaa" কে "6a" বলা। Decompress করলে original পাওয়া যায়।

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy    # compression algorithm
    indexConfig:
      prefixCompression: true    # index এর common prefix share করা
```

| Compressor | Speed | Compression Ratio | কখন |
|-----------|-------|-------------------|-----|
| `snappy` | Fast | Moderate | Default — good balance |
| `zlib` | Slower | Better | Storage বাঁচাতে |
| `zstd` | Fast | Best | Modern systems এ |
| `none` | Fastest | None | CPU overhead যদি concern হয় |

**"Compression ratio" কী?**
Original size / compressed size। Ratio বেশি = বেশি compress হয়েছে।

---

## WiredTiger Cache Configuration

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4    # default: max(50% RAM - 1GB, 256MB)
```

**Rule of thumb:** RAM এর 50-70% WiredTiger cache কে দাও। OS এর page cache এর জন্যও কিছু রাখো।

**Cache eviction:**
- Cache এর dirty data 20% হলে eviction শুরু হয় (পুরনো data বের করে নতুনের জায়গা করা)
- Cache 95% full হলে aggressive eviction শুরু হয়

---

## Checkpoint

WiredTiger প্রতি **60 seconds** এ (বা 2GB journal এ) **checkpoint** তৈরি করে।

**"Checkpoint" কী?**
Cache এর dirty data (modified but not written to disk) পুরোটা disk এ flush করা। Checkpoint এর পরে journal এর পুরনো portion delete করা যায়।

Crash হলে last checkpoint থেকে journal replay করে recover হয়।

```yaml
storage:
  wiredTiger:
    engineConfig:
      checkpointSizeMB: 2048    # 2GB journal হলে checkpoint
```

---

<a name="chapter-11"></a>
# Chapter 11 — Performance & Tuning

---

## Profiler — Slow Query Detection

MongoDB এর built-in query profiler। Slow queries automatically capture করে।

**"Profiler" কী?**
Query এর performance measure করার tool। কোন query কতটা সময় নিচ্ছে, কতটা documents examine করছে — এসব track করে।

```javascript
// Profiling levels:
// 0 = off
// 1 = slow queries only (threshold এর বেশি সময় নেওয়া)
// 2 = all queries (সব — production এ avoid করো, অনেক overhead)

db.setProfilingLevel(1, { slowms: 100 })
// 100ms এর বেশি সময় নেওয়া queries log করো

// Profile data দেখো
db.system.profile.find().sort({ ts: -1 }).limit(10)
// ts = timestamp, -1 = latest first
```

Profile entry এ important fields:
```json
{
  "op": "query",             // operation type
  "ns": "mydb.users",       // namespace
  "millis": 450,             // কতটা সময় লেগেছে (milliseconds)
  "keysExamined": 10000,     // কতটা index key check হয়েছে
  "docsExamined": 10000,     // কতটা document check হয়েছে
  "nreturned": 150,          // কতটা return হয়েছে
  "planSummary": "COLLSCAN"  // query plan — COLLSCAN মানে index নেই!
}
```

---

## currentOp — Running Operations

```javascript
// এখন কোন operations চলছে দেখো
db.currentOp({ active: true, secs_running: { $gt: 5 } })
// 5 seconds এর বেশি চলছে এমন operations

// কোনো operation kill করো
db.killOp(opid)  // opid = operation id, currentOp থেকে পাবে
```

---

## Connection Pool Tuning

```yaml
# mongod.conf
net:
  maxIncomingConnections: 65536
```

```javascript
MongoClient.connect(uri, {
  maxPoolSize: 100,       // maximum connections pool এ
  minPoolSize: 10,        // সবসময় minimum এতটা ready থাকবে
  maxIdleTimeMS: 30000,   // idle connection 30 seconds পর close
  waitQueueTimeoutMS: 5000 // pool থেকে connection পেতে 5 seconds এর বেশি লাগলে error
})
```

---

## Read/Write Performance Tips

**1. সব common queries তে index দাও:**
```javascript
db.users.find({ age: { $gt: 25 }, city: "Dhaka" }).explain()
// "COLLSCAN" দেখলে index তৈরি করো
```

**2. Projection use করো:**
```javascript
// Bad — সব fields আনো (অনেক data transfer)
db.users.find({ city: "Dhaka" })

// Good — শুধু দরকারী fields আনো
db.users.find({ city: "Dhaka" }, { name: 1, email: 1 })
```

**3. Covered Query:**

**"Covered query" কী?**
Query র সব fields (filter + projection) index এ থাকলে document fetch করতে হয় না — শুধু index পড়লেই হয়। Index-only scan। অনেক fast।

```javascript
db.users.createIndex({ city: 1, name: 1, email: 1 })
db.users.find({ city: "Dhaka" }, { name: 1, email: 1, _id: 0 })
// সব fields (city, name, email) index এ আছে — document এ যেতে হয় না
```

**4. $limit early রাখো:**
```javascript
db.orders.find().sort({ amount: -1 }).limit(10)
// MongoDB optimize করে — শুধু top 10 find করার জন্য সব sort করে না
```

**5. Large collection এ count avoid করো:**
```javascript
// Slow — সব matching documents count করে
db.users.countDocuments({ status: "active" })

// Fast — approximate, metadata থেকে
db.users.estimatedDocumentCount()
```

**6. $where এবং anchored regex avoid করো:**
```javascript
// Bad — JavaScript execution, slow
db.users.find({ $where: "this.age > 25" })

// Bad — full string scan (শুরুতে anchor নেই)
db.users.find({ name: { $regex: "rahim" } })

// Good — anchored regex, index use করতে পারে
db.users.find({ name: { $regex: "^Rahim" } })
// "^" = string এর শুরু — Rahim দিয়ে শুরু হওয়া
```

---

## Memory Tuning

```javascript
// Working set size দেখো
db.serverStatus().wiredTiger.cache

// page faults দেখো — বেশি মানে RAM কম, disk থেকে আনতে হচ্ছে
db.serverStatus().extra_info.page_faults
```

**"Page fault" কী?**
OS কোনো data RAM এ পায়নি, disk থেকে আনতে হয়েছে। বেশি page fault = working set RAM এ fit করছে না = slow।

**Rule:** Working set (hot data + indexes) RAM এ fit করা উচিত।

---

## mongotop এবং mongostat

```bash
# mongotop — collection level activity দেখো
mongotop 5          # প্রতি 5 seconds update

# mongostat — server level metrics দেখো
mongostat 5         # প্রতি 5 seconds

# mongostat output:
# insert/query/update/delete — ops/sec (প্রতি সেকেন্ডে কতটা operation)
# dirty — WiredTiger cache এ dirty data এর %
# used — WiredTiger cache কতটা use হচ্ছে
# res — resident memory (RAM এ কতটা)
# conn — active connections
```

---

## Aggregation Pipeline Optimization Tips

```javascript
// Bad — $match শেষে (অনেক data process হওয়ার পর filter)
db.orders.aggregate([
  { $lookup: {...} },
  { $unwind: "$items" },
  { $match: { status: "active" } }
])

// Good — $match আগে (আগেই filter করো, পরের stages এ কম data)
db.orders.aggregate([
  { $match: { status: "active" } },   // early filter
  { $lookup: {...} },
  { $unwind: "$items" }
])
```

---

## OS Level Tuning

```bash
# /etc/sysctl.conf
vm.swappiness = 1
# swappiness = 0-100, OS কতটা swap use করবে
# 1 = প্রায় কখনো swap করে না (MongoDB এর জন্য swap খারাপ)

net.core.somaxconn = 65535
# TCP connection queue size

net.ipv4.tcp_keepalive_time = 120
# idle TCP connection কতক্ষণ পর keepalive পাঠাবে

# Transparent Huge Pages disable করো
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
# MongoDB fork করার সময় THP latency spike করে

# ulimits (resource limits)
mongod soft nofile 64000    # maximum file descriptors (connections)
mongod hard nofile 64000
mongod soft nproc 64000     # maximum processes
mongod hard nproc 64000
```

---

<a name="chapter-12"></a>
# Chapter 12 — Security

---

## Authentication

**"Authentication" কী?**
"তুমি কে?" প্রমাণ করা। Username/password দিয়ে বা certificate দিয়ে।

MongoDB এ users থাকে database level এ:

```javascript
// Admin user তৈরি (সবার আগে এটা করো)
use admin
db.createUser({
  user: "admin",
  pwd: "strongpassword",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    "readWriteAnyDatabase"
  ]
})

// Application user তৈরি (limited permissions)
use mydb
db.createUser({
  user: "appuser",
  pwd: "apppassword",
  roles: [{ role: "readWrite", db: "mydb" }]
  // শুধু mydb তে read/write করতে পারবে
})
```

### Authentication Enable করো

```yaml
# mongod.conf
security:
  authorization: enabled
  # এই ছাড়া কেউ password ছাড়া connect করতে পারে
```

### Authentication Mechanisms

**"Mechanism" কী?**
Authentication করার পদ্ধতি। কোন protocol দিয়ে verify হবে।

- **SCRAM-SHA-256** (default) — password-based, secure hashing
- **x.509** — certificate-based authentication (production এ recommend)
- **LDAP** — centralized user directory (enterprise)
- **Kerberos** — network authentication protocol (enterprise)

---

## Built-in Roles

**"Role" কী?**
Permissions এর একটা group। User কে role দিলে সেই role এর সব permissions পায়।

| Role | Access |
|------|--------|
| `read` | Collection read করতে পারবে |
| `readWrite` | Collection read + write |
| `dbAdmin` | DB administration (indexes, stats) কিন্তু data না |
| `userAdmin` | User management in DB |
| `dbOwner` | dbAdmin + userAdmin + readWrite (DB এর মালিক) |
| `clusterAdmin` | Cluster management |
| `root` | Superuser — সব পারে |
| `readAnyDatabase` | সব DB read |
| `readWriteAnyDatabase` | সব DB read + write |

---

## Role Management

```javascript
// Custom role তৈরি করো
db.createRole({
  role: "readOrdersOnly",
  privileges: [{
    resource: { db: "mydb", collection: "orders" },
    actions: ["find"]   // শুধু find (read) করতে পারবে
  }],
  roles: []
})

// User এ role add করো
db.grantRolesToUser("appuser", [{ role: "readOrdersOnly", db: "mydb" }])

// Role remove করো
db.revokeRolesFromUser("appuser", [{ role: "readOrdersOnly", db: "mydb" }])
```

---

## Network Security

```yaml
# mongod.conf
net:
  port: 27017
  bindIp: 127.0.0.1,192.168.1.10  # specific IPs তেই listen করো
  # 0.0.0.0 avoid করো — সব network interface এ open থাকে

  tls:
    mode: requireTLS        # সব connection এ TLS require করো
    certificateKeyFile: /etc/ssl/mongodb.pem
    CAFile: /etc/ssl/ca.pem
    # TLS = Transport Layer Security — network এ data encrypt করে
```

**"TLS/SSL" কী?**
Network এ data পাঠানোর সময় encrypt করার protocol। মাঝপথে কেউ intercept করলেও data পড়তে পারবে না।

---

## Encryption

### Encryption at Rest (ডিস্কে encrypt)

**"Encryption at rest" কী?**
Disk এ stored data encrypt করা। কেউ hard disk চুরি করলেও data পড়তে পারবে না।

Enterprise feature। Community edition এ filesystem-level encryption use করো।

### Encryption in Transit (নেটওয়ার্কে encrypt)

উপরে TLS section দেখো।

---

<a name="chapter-13"></a>
# Chapter 13 — Backup & Recovery

---

## Backup Strategies

### mongodump / mongorestore — Logical Backup

**"Logical backup" কী?**
Data কে BSON/JSON format এ export করা। Schema এবং data দুটোই export হয়। Cross-version restore করা যায়।

**"Physical backup" এর বিপরীত:** Physical backup মানে actual disk files copy করা।

```bash
# Full backup
mongodump --host localhost --port 27017 \
  --username admin --password password \
  --authenticationDatabase admin \
  --out /backup/mongodb/$(date +%Y%m%d)
# $(date +%Y%m%d) = আজকের তারিখ (e.g., 20240323)

# Specific database
mongodump --db mydb --out /backup/

# Compressed backup
mongodump --db mydb --archive=/backup/mydb.gz --gzip
# --gzip = gzip compression use করো

# Restore
mongorestore --host localhost --port 27017 \
  --username admin --password password \
  --authenticationDatabase admin \
  /backup/mongodb/20240323

# Drop করে restore (existing data মুছে)
mongorestore --drop --db mydb /backup/mydb
```

**Limitations:**
- Large dataset এ slow — document by document export
- Point-in-time recovery নেই (oplog সহ backup করলে possible)
- Backup চলাকালীন data change হতে পারে

---

## Oplog Backup — Point-in-Time Recovery

**"Point-in-time recovery" কী?**
ঠিক একটা নির্দিষ্ট সময়ের state এ restore করা। "কাল দুপুর 2টায় কেউ data delete করেছে, 1:59 এর state এ ফিরে যেতে চাই।"

```bash
# Oplog backup
mongodump --host localhost --port 27017 \
  --db local --collection oplog.rs \
  --out /backup/oplog

# Restore with oplog replay
mongorestore --oplogReplay --oplogLimit "1711123200:1" /backup/
# --oplogLimit = এই timestamp পর্যন্ত oplog replay করো
```

---

## File System Snapshot — Physical Backup

Production এ preferred। Fast এবং consistent।

**"Snapshot" কী?**
একটা নির্দিষ্ট মুহূর্তে disk এর exact copy। লেখার সময় পুরনো block copy করে রাখে (Copy-on-Write)।

### LVM Snapshot

**"LVM" কী?**
Logical Volume Manager — Linux এ disk partition এর flexible management। LVM দিয়ে online snapshot নেওয়া যায়।

```bash
# MongoDB pause করো (journal flush করো)
mongo admin --eval "db.fsyncLock()"
# fsyncLock = সব write বন্ধ করো এবং data disk এ flush করো

# LVM snapshot নাও
lvcreate -L 10G -s -n mongodb-snapshot /dev/vg0/mongodb-data
# -L 10G = snapshot এর size, -s = snapshot, -n = name

# MongoDB resume করো
mongo admin --eval "db.fsyncUnlock()"

# Snapshot mount করে backup নাও
mount /dev/vg0/mongodb-snapshot /mnt/snapshot
rsync -av /mnt/snapshot/ /backup/mongodb/

# Snapshot remove করো
lvremove /dev/vg0/mongodb-snapshot
```

### Replica Set এ Snapshot

Secondary তে data file সরাসরি copy করা যায়। Secondary stop করো, snapshot নাও, restart করো। Primary অপ্রভাবিত থাকে।

---

## mongodump vs Snapshot

| | mongodump | Snapshot |
|--|---------|---------|
| Speed | Slow (document by document) | Fast (block level) |
| Consistency | Good (Secondary তে) | Perfect |
| Point-in-time | Oplog সহ possible | হ্যাঁ |
| Space | BSON size (compressed) | Full disk size |
| Restore | Easy (mongorestore) | Complex |
| Cross-version | হ্যাঁ | Same version দরকার |

---

## Backup Verification

**Backup verify করা mandatory।** Unverified backup = no backup।

```bash
# Restore করে verify করো
mongorestore --db verify_db /backup/mydb

# Data check করো
mongo verify_db --eval "db.users.count()"

# Consistency check
db.runCommand({ dbCheck: "users" })
# dbCheck = data corruption check করে
```

---

<a name="chapter-14"></a>
# Chapter 14 — Real Patterns & Use Cases

---

## 1. E-Commerce Product Catalog

Product catalog এর জন্য MongoDB perfect — প্রতিটা product এর attributes আলাদা।

```json
{
  "_id": ObjectId("..."),
  "sku": "SHIRT-001",
  "name": "Cotton T-Shirt",
  "category": "clothing",
  "price": 500,
  "attributes": {
    "color": "blue",
    "size": ["S", "M", "L", "XL"],
    "material": "100% cotton"
  },
  "inventory": {
    "in_stock": true,
    "quantity": 150
  },
  "tags": ["casual", "summer", "cotton"]
}
```

```javascript
// Efficient queries এর জন্য index
db.products.createIndex({ category: 1, price: 1 })
db.products.createIndex({ tags: 1 })
db.products.createIndex({ "attributes.color": 1, "attributes.size": 1 })

// Query
db.products.find({
  category: "clothing",
  price: { $lte: 1000 },
  "attributes.color": "blue",
  "attributes.size": "M"
})
```

---

## 2. Real-time Analytics — Time Series

```json
{
  "device_id": "sensor_001",
  "timestamp": ISODate("2024-03-23T10:00:00Z"),
  "metrics": {
    "temperature": 28.5,
    "humidity": 65
  }
}
```

MongoDB 5.0 থেকে **Time Series Collections** — time-series data এর জন্য optimized:

```javascript
db.createCollection("sensor_data", {
  timeseries: {
    timeField: "timestamp",    // কোন field টা time
    metaField: "device_id",    // কোন field টা metadata (grouping)
    granularity: "minutes"     // data এর granularity: seconds, minutes, hours
  },
  expireAfterSeconds: 2592000  // 30 days পর automatically delete
})
```

**"Granularity" কী?**
Data এর time precision। "minutes" মানে per-minute data। এটা দিয়ে MongoDB internally optimize করে।

Time series collection internally data compress করে এবং time-based query optimize করে।

---

## 3. User Activity / Audit Log

**"Audit log" কী?**
কে কী করেছে তার record। Security এবং compliance এর জন্য।

```json
{
  "user_id": "user:1001",
  "action": "login",
  "resource": "/dashboard",
  "ip": "192.168.1.1",
  "timestamp": ISODate("2024-03-23T10:00:00Z")
}
```

```javascript
// TTL index — 90 days পর auto-delete
db.audit_logs.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 7776000 }  // 90 days = 7776000 seconds
)

// Query — user এর recent activity
db.audit_logs.find({
  user_id: "user:1001",
  timestamp: { $gte: ISODate("2024-03-01") }
}).sort({ timestamp: -1 })
```

---

## 4. Content Management — Blog

```json
{
  "_id": ObjectId("..."),
  "title": "Getting Started with MongoDB",
  "slug": "getting-started-mongodb",
  "content": "...",
  "author": {
    "_id": ObjectId("..."),
    "name": "Rahim",
    "avatar": "avatar.jpg"
  },
  "tags": ["mongodb", "database", "nosql"],
  "comments": [
    {
      "_id": ObjectId("..."),
      "author": "Karim",
      "text": "Great article!",
      "created_at": ISODate("...")
    }
  ],
  "views": 1250,
  "status": "published"
}
```

**"Slug" কী?**
URL এ use করার জন্য formatted string। "Getting Started with MongoDB" → "getting-started-mongodb"। Spaces এর বদলে hyphen, lowercase।

**সতর্কতা:** Comments array unbounded — যত মন্তব্য আসবে array তত বড় হবে। 16MB limit ছুঁতে পারে। বড় blog এ comments আলাদা collection এ রাখো।

---

## 5. Geospatial — Delivery Tracking

**"Geospatial" কী?**
Location (latitude/longitude) based data এবং queries।

```json
{
  "_id": "courier:1001",
  "name": "Rahim",
  "location": {
    "type": "Point",
    "coordinates": [90.4125, 23.8103]
  },
  "status": "active"
}
```

```javascript
db.couriers.createIndex({ location: "2dsphere" })

// নিকটতম active courier খোঁজো
db.couriers.find({
  location: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [90.4000, 23.8000]  // pickup point
      },
      $maxDistance: 5000  // 5km radius
    }
  },
  status: "active"
}).limit(5)
```

---

## 6. Change Streams — Real-time Data

**"Change stream" কী?**
Collection এর changes real-time এ subscribe করা। কোনো insert/update/delete হলে immediately notify পাওয়া।

MongoDB 3.6+। Oplog এর উপরে built।

```javascript
// orders collection এর changes watch করো
const changeStream = db.orders.watch([
  { $match: { "operationType": { $in: ["insert", "update"] } } }
])

changeStream.on("change", (change) => {
  console.log("Change:", change)
  // Real-time notification, cache invalidation, etc.
})
```

**"Subscribe" কী?**
কোনো event এর জন্য listen করা। Event হলে automatically notify পাওয়া — বারবার check করতে হয় না।

Replica Set বা Sharded Cluster এ কাজ করে।

---

## 7. Schema Migration Strategy

**"Schema migration" কী?**
Existing data এর structure change করা। যেমন `firstName + lastName` কে `fullName` এ merge করা।

```javascript
// Lazy migration — read এ old format handle করো, write এ নতুন format
db.users.find({ fullName: { $exists: false } }).forEach(user => {
  // fullName নেই এমন users update করো
  db.users.updateOne(
    { _id: user._id },
    {
      $set: { fullName: user.firstName + " " + user.lastName },
      $unset: { firstName: "", lastName: "" }
    }
  )
})
```

**Dual-write strategy:**
1. নতুন code উভয় format লেখে এবং পড়তে পারে
2. Background migration চলে — old format কে new তে convert
3. Migration শেষে old format support remove করো

---

# Quick Reference Summary

## Data Modeling Decision Tree

```
Data related?
  └── হ্যাঁ
      └── Data একসাথে access হয়?
          ├── হ্যাঁ → EMBED করো (nested document)
          └── না
              └── Array bounded? (small, fixed max size)
                  ├── হ্যাঁ → EMBED করো
                  └── না → REFERENCE করো (আলাদা collection, _id store করো)
```

## Index Selection Guide

| Query Pattern | Recommended Index |
|--------------|-------------------|
| Equality (exact match) | Single field বা Compound prefix |
| Range (>, <, between) | Compound (equality first, range last — ESR rule) |
| Sort (ORDER BY) | Include sort field in compound index |
| Array field | Multikey (automatic) |
| Full-text search | Text index |
| Location-based | 2dsphere |
| Auto-delete | TTL index (Date field) |
| Subset of documents | Partial index |

## Replica Set vs Sharding

| Need | Solution |
|------|---------|
| High availability | Replica Set (3 nodes minimum) |
| Read scaling | Replica Set + secondary read preference |
| Write scaling | Sharding |
| Data > single server | Sharding |

## Common mongod.conf Checklist

```yaml
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true           # crash recovery
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4        # RAM এর 50%

net:
  port: 27017
  bindIp: 127.0.0.1        # specific IP only

security:
  authorization: enabled   # authentication require করো
  keyFile: /etc/mongodb/keyfile  # Replica Set এ

replication:
  replSetName: "rs0"
  oplogSizeMB: 10240       # 10GB oplog

operationProfiling:
  slowOpThresholdMs: 100   # 100ms এর বেশি হলে log
  mode: slowOp
```

## OS Tuning Checklist

```bash
# /etc/sysctl.conf
vm.swappiness = 1                    # swap minimize
net.core.somaxconn = 65535           # TCP queue size

# Transparent Huge Pages — MUST disable
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# ulimits
mongod soft nofile 64000
mongod hard nofile 64000
```

---

*MongoDB Complete Study Guide (Revised) | সব technical terms in-place explained*
*Covers MongoDB Administration, Architecture, and Operations*
