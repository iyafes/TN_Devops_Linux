# MongoDB Complete Study Guide
### Architecture · Internals · Replica Set · Sharding · Tuning · Patterns

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

MongoDB হলো একটা **document-oriented NoSQL database**। Data store হয় **BSON documents** এ — JSON এর binary representation। Rows এবং columns এর বদলে flexible documents।

2009 সালে 10gen (পরে MongoDB Inc.) তৈরি করে। নাম এসেছে "humongous" থেকে — অনেক বড় data handle করার কথা মাথায় রেখে।

---

## কেন MongoDB তৈরি হয়েছিল — The Problem

2000s এর শেষের দিকে web applications এর data patterns বদলে যাচ্ছিল:

**Relational DB এর সমস্যা:**
- Schema rigid — নতুন field add করতে ALTER TABLE লাগে, large table এ এটা expensive
- Hierarchical বা nested data রাখতে অনেক join লাগে
- Horizontal scaling কঠিন — sharding manually করতে হয়
- Agile development এ schema frequent change করতে হয়

**MongoDB এর solution:**
- Schema flexible — document এ যেকোনো structure রাখা যায়
- Nested objects এবং arrays natively support করে
- Built-in horizontal scaling (sharding)
- Developer friendly — application এর objects directly document হিসেবে store হয়

---

## MongoDB কোথায় ভালো, কোথায় না

**MongoDB ভালো:**
- Varied বা evolving schema (product catalog, user profiles)
- Hierarchical বা nested data (blog posts with comments, orders with line items)
- High write throughput
- Horizontal scaling দরকার
- Rapid prototyping এবং agile development

**MongoDB ভালো না:**
- Complex multi-table transactions (banking, financial systems)
- Strong relational data with many joins
- Data warehousing এবং complex analytics (use columnar DB)
- Data যখন truly relational এবং normalized থাকা দরকার

---

## MongoDB vs PostgreSQL — মূল পার্থক্য

| বিষয় | PostgreSQL | MongoDB |
|------|-----------|---------|
| Data model | Tables, rows, columns | Collections, documents |
| Schema | Strict (DDL) | Flexible |
| Relationships | Foreign keys, JOINs | Embedded documents বা references |
| Transactions | Full ACID | Multi-document ACID (4.0+) |
| Scaling | Vertical primarily | Horizontal (sharding built-in) |
| Query language | SQL | MQL (MongoDB Query Language) |
| Joins | Native, efficient | $lookup (less efficient) |

---

<a name="chapter-2"></a>
# Chapter 2 — Core Concepts & Data Model

## Document

MongoDB এর fundamental unit হলো **document**। এটা BSON format এ store হয় — JSON এর মতো দেখতে কিন্তু binary encoded।

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
- Maximum size: **16MB**
- Key-value pairs, value যেকোনো BSON type হতে পারে
- Nested documents (embedded documents)
- Arrays
- Key order preserved (BSON এ)
- Keys case-sensitive

---

## BSON — Binary JSON

BSON হলো MongoDB এর wire format এবং storage format। JSON এর চেয়ে আলাদা কারণে:

**Extra data types যা JSON এ নেই:**
- `ObjectId` — 12-byte unique identifier
- `Date` — 64-bit integer (milliseconds since epoch)
- `BinData` — binary data
- `Decimal128` — high-precision decimal
- `Regular Expression`
- `Timestamp` — internal MongoDB use (replication)
- `Int32`, `Int64` — explicit integer types
- `Double` — 64-bit float

**JSON vs BSON:**
- JSON: text, human-readable, slow parse
- BSON: binary, fast parse/encode, more types, slightly larger size

---

## _id Field

প্রতিটা document এ `_id` field mandatory। Primary key এর মতো কাজ করে।

**Default: ObjectId**
```
ObjectId("507f1f77bcf86cd799439011")
```

ObjectId এর structure (12 bytes):
```
[4 bytes timestamp][5 bytes random][3 bytes incrementing counter]
```
- Timestamp থেকে creation time বের করা যায়
- Multiple servers এ collision ছাড়া generate করা যায় (random part এর কারণে)
- Roughly sortable by insertion time

**Custom _id:**
```json
{ "_id": "AWB123456", "status": "in_transit" }
{ "_id": 1001, "name": "Rahim" }
```
যেকোনো unique value দেওয়া যায়। Immutable — একবার set করলে change করা যায় না।

---

## Collection

Collection হলো documents এর group। Relational DB এর table এর মতো কিন্তু schema নেই।

**Collection এর বৈশিষ্ট্য:**
- Documents এর একই structure থাকতে হয় না
- Dynamically created — প্রথম document insert হলে collection তৈরি হয়
- Index store করে
- Namespace: `database_name.collection_name`

### Capped Collection

Fixed size collection। New documents পুরনোটাকে overwrite করে (circular buffer):

```javascript
db.createCollection("logs", {
  capped: true,
  size: 10485760,  // 10MB
  max: 1000        // maximum 1000 documents
})
```

Use case: Logs, event streams যেখানে শুধু recent data দরকার।

---

## Database

MongoDB instance এ multiple databases থাকতে পারে। প্রতিটা database আলাদা files এ store হয়।

**Special databases:**
- `admin` — administrative operations, user management
- `local` — replication data, এই DB replicate হয় না
- `config` — sharding metadata (sharded cluster এ)

```javascript
use myapp          // database switch করো বা তৈরি করো
db                 // current database দেখো
show dbs           // সব databases
db.dropDatabase()  // current database delete
```

---

## Data Modeling — Embedding vs Referencing

MongoDB এ data model করার দুটো approach।

### Embedding (Denormalization)

Related data একটাই document এ রাখো:

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
- "Contains" relationship (order contains items)
- Data একসাথে read হয়
- Data এর sub-part আলাদাভাবে access দরকার নেই
- Array এর size bounded এবং বড় হবে না

**Pros:** Single query তে সব data, fast read, atomic update
**Cons:** Document size বাড়ে, data duplication, large array slow হয়

### Referencing (Normalization)

Separate collections এ রাখো, reference দিয়ে link করো:

```json
// users collection
{ "_id": ObjectId("user1"), "name": "Rahim" }

// orders collection
{
  "_id": ObjectId("order1"),
  "user_id": ObjectId("user1"),  // reference
  "total": 1800
}
```

**কখন reference করবে:**
- Data অনেক জায়গায় reuse হয়
- Sub-document অনেক বড় হতে পারে
- Unbounded array (user এর orders)
- Data আলাদাভাবেও access হয়

**Pros:** Data duplication নেই, flexible
**Cons:** Multiple queries বা $lookup দরকার (JOIN এর equivalent)

### Practical Rule: "How data is accessed?"

Data model করার সময় প্রশ্ন করো: "এই data কীভাবে read করা হবে?" Access pattern অনুযায়ী model করো, not normalization theory অনুযায়ী।

---

## Schema Validation

MongoDB flexible schema হলেও validation enforce করা যায়:

```javascript
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email"],
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
          pattern: "^.+@.+$"
        }
      }
    }
  },
  validationAction: "error"  // "warn" বা "error"
})
```

---

<a name="chapter-3"></a>
# Chapter 3 — Internal Architecture

## MongoDB Architecture Overview

```
Client Application
      ↓
MongoDB Driver (BSON encode/decode, connection pool)
      ↓
mongod process
      ↓
Query Engine (parse, plan, optimize)
      ↓
Storage Engine (WiredTiger)
      ↓
OS File System Cache
      ↓
Disk (data files, journal, oplog)
```

---

## mongod — The Server Process

`mongod` হলো MongoDB এর main server daemon। প্রতিটা MongoDB instance এ একটা `mongod` process চলে।

কাজ:
- Client connections handle করা
- Query processing
- Data read/write (storage engine এর মাধ্যমে)
- Replication
- Background operations (TTL index cleanup, etc.)

---

## mongos — The Query Router

Sharded cluster এ `mongos` হলো routing layer। Client `mongos` এ connect করে, `mongos` সঠিক shard এ route করে। এটা stateless।

---

## Connection Handling

MongoDB প্রতিটা connection এর জন্য একটা thread তৈরি করে (thread-per-connection model)।

**Connection limits:**
```
ulimit -n 65536  # OS level file descriptor limit
```

```yaml
# mongod.conf
net:
  maxIncomingConnections: 1000000
```

**Connection Pool (Driver side):**
Driver connection pool maintain করে। Default pool size 100।

```javascript
// Node.js example
MongoClient.connect(uri, {
  maxPoolSize: 100,
  minPoolSize: 10,
  maxIdleTimeMS: 30000
})
```

---

## Memory Architecture

MongoDB heavily relies on OS page cache।

```
RAM
├── WiredTiger Cache (default: 50% of RAM - 1GB, minimum 256MB)
│   ├── Hot data (frequently accessed documents, indexes)
│   ├── Dirty pages (modified, not yet written to disk)
│   └── Internal data structures
└── OS File System Cache
    └── Memory-mapped files (data files)
```

**WiredTiger cache** এবং **OS page cache** দুটো আলাদা। Data উভয় cache এ থাকতে পারে।

Working set (frequently accessed data) RAM এ থাকলে performance ভালো। Working set RAM এর চেয়ে বড় হলে disk I/O বাড়ে।

---

## Journaling

Data corruption থেকে protect করতে MongoDB **journal** use করে। Write operation প্রথমে journal এ যায়, তারপর data files এ।

Crash হলে journal replay করে consistent state এ ফিরে আসা যায়।

WiredTiger এ journal = write-ahead log (WAL)। Default এ enabled।

```yaml
storage:
  journal:
    enabled: true
    commitIntervalMs: 100  # journal flush interval
```

---

## Oplog — Operations Log

Oplog (Operations Log) হলো Replica Set এর core mechanism। `local.oplog.rs` collection এ থাকে। Capped collection।

প্রতিটা write operation oplog এ record হয়:
```json
{
  "ts": Timestamp(1711123200, 1),
  "op": "i",          // i=insert, u=update, d=delete
  "ns": "mydb.users",
  "o": { "_id": ObjectId("..."), "name": "Rahim" }
}
```

Replica members oplog থেকে operations replay করে sync করে।

**Oplog size:**
```yaml
replication:
  oplogSizeMB: 10240  # 10GB (default: 5% of free disk, max 50GB)
```

Oplog বড় হলে:
- Replica recovery window বড় হয় (network issue এ পিছিয়ে পড়লে recover করার সুযোগ বেশি)
- Disk বেশি লাগে

---

<a name="chapter-4"></a>
# Chapter 4 — CRUD Operations

## Database এবং Collection Operations

```javascript
// Database
use mydb                    // switch বা create
db.getName()
db.stats()
db.dropDatabase()

// Collection
db.createCollection("users")
db.getCollectionNames()
db.users.drop()
db.users.stats()
db.users.count()            // deprecated, use countDocuments
db.users.estimatedDocumentCount()  // fast, approximate
db.users.countDocuments({ age: { $gt: 18 } })
```

---

## Insert

```javascript
// Single document
db.users.insertOne({
  name: "Rahim",
  age: 28,
  city: "Dhaka"
})
// Returns: { acknowledged: true, insertedId: ObjectId("...") }

// Multiple documents
db.users.insertMany([
  { name: "Karim", age: 25 },
  { name: "Sakib", age: 30 },
  { name: "Nadia", age: 22 }
])
// Returns: { acknowledged: true, insertedIds: { 0: ObjectId, 1: ObjectId, 2: ObjectId } }

// insertMany options
db.users.insertMany(docs, {
  ordered: false  // error হলেও বাকিগুলো continue করবে (default true = stop on error)
})
```

---

## Read (Find)

```javascript
// সব documents
db.users.find()
db.users.find({})

// Filter
db.users.find({ city: "Dhaka" })
db.users.find({ age: 28, city: "Dhaka" })  // AND condition

// Single document
db.users.findOne({ name: "Rahim" })

// Projection — কোন fields দেখাবে
db.users.find({ city: "Dhaka" }, { name: 1, age: 1 })     // শুধু name, age (+ _id)
db.users.find({ city: "Dhaka" }, { name: 1, _id: 0 })     // _id বাদ দাও
db.users.find({ city: "Dhaka" }, { password: 0 })          // password বাদ সব

// Comparison operators
db.users.find({ age: { $gt: 25 } })          // greater than
db.users.find({ age: { $gte: 25 } })         // greater than or equal
db.users.find({ age: { $lt: 30 } })          // less than
db.users.find({ age: { $lte: 30 } })
db.users.find({ age: { $eq: 28 } })          // equal
db.users.find({ age: { $ne: 28 } })          // not equal
db.users.find({ age: { $in: [25, 28, 30] } }) // in array
db.users.find({ age: { $nin: [25, 28] } })   // not in array
db.users.find({ age: { $gt: 20, $lt: 30 } }) // range

// Logical operators
db.users.find({ $and: [{ age: { $gt: 20 } }, { city: "Dhaka" }] })
db.users.find({ $or: [{ city: "Dhaka" }, { city: "Chittagong" }] })
db.users.find({ age: { $not: { $gt: 30 } } })
db.users.find({ $nor: [{ city: "Dhaka" }, { city: "Sylhet" }] })

// Element operators
db.users.find({ email: { $exists: true } })   // field আছে কিনা
db.users.find({ age: { $type: "int" } })      // type check

// Array operators
db.users.find({ hobbies: "coding" })           // array তে "coding" আছে
db.users.find({ hobbies: { $all: ["coding", "chess"] } })  // সব আছে
db.users.find({ hobbies: { $size: 3 } })       // array size = 3
db.users.find({ "hobbies.0": "reading" })      // first element = "reading"

// Nested document
db.users.find({ "address.city": "Dhaka" })
db.users.find({ "address.zip": { $in: ["1216", "1207"] } })

// Regular expression
db.users.find({ name: /^Rah/ })
db.users.find({ name: { $regex: "^Rah", $options: "i" } })  // case insensitive

// Cursor methods
db.users.find().sort({ age: 1 })        // ascending
db.users.find().sort({ age: -1 })       // descending
db.users.find().limit(10)               // first 10
db.users.find().skip(20).limit(10)      // pagination (page 3, 10 per page)
db.users.find().count()                 // deprecated
db.users.find().sort({ age: -1 }).limit(5).skip(10)

// Explain — query plan দেখো
db.users.find({ age: { $gt: 25 } }).explain("executionStats")
```

---

## Update

```javascript
// Single document update
db.users.updateOne(
  { name: "Rahim" },           // filter
  { $set: { age: 29 } }        // update
)

// Multiple documents update
db.users.updateMany(
  { city: "Dhaka" },
  { $set: { country: "Bangladesh" } }
)

// Replace entire document (_id ছাড়া)
db.users.replaceOne(
  { name: "Rahim" },
  { name: "Rahim", age: 29, city: "Dhaka" }
)

// Upsert — না থাকলে insert, থাকলে update
db.users.updateOne(
  { email: "rahim@example.com" },
  { $set: { name: "Rahim", age: 28 } },
  { upsert: true }
)

// Update operators
$set    // field set করো
$unset  // field remove করো
$inc    // numeric field increment/decrement
$mul    // numeric field multiply
$rename // field rename
$min    // current value এর চেয়ে argument ছোট হলে update
$max    // current value এর চেয়ে argument বড় হলে update

// Array update operators
$push   // array তে element add
$pull   // array থেকে element remove (condition based)
$pop    // array এর first (-1) বা last (1) element remove
$addToSet // array তে add, duplicate হবে না

// উদাহরণ
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

// Array element update (positional operator)
db.orders.updateOne(
  { "items.product": "Shirt" },
  { $set: { "items.$.price": 600 } }   // $ = matched element
)

// findOneAndUpdate — update করে old/new document return করে
db.users.findOneAndUpdate(
  { name: "Rahim" },
  { $inc: { age: 1 } },
  { returnDocument: "after" }  // "before" বা "after"
)
```

---

## Delete

```javascript
// Single
db.users.deleteOne({ name: "Rahim" })

// Multiple
db.users.deleteMany({ city: "Sylhet" })

// সব delete
db.users.deleteMany({})

// Find and delete
db.users.findOneAndDelete({ name: "Rahim" })
```

---

## Bulk Operations

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
], { ordered: false })
```

Bulk operations single round trip এ অনেক operations করে — much more efficient।

---

<a name="chapter-5"></a>
# Chapter 5 — Indexing

Index ছাড়া MongoDB প্রতিটা query তে **collection scan** করে — সব documents check করতে হয়, O(n)। Index দিয়ে O(log n) বা O(1) এ পৌঁছানো যায়।

---

## Index Internals — B-Tree

MongoDB default index হলো **B-Tree** (B+ Tree actually)। Balanced tree structure।

```
                [28, 35]
               /    |    \
         [22, 25]  [30, 32]  [40, 45]
```

B-Tree এ:
- Range query efficient — একবার tree traverse করলে range এর সব values পাওয়া যায়
- Equality query efficient — O(log n)
- Sorted output — index already sorted

---

## Index Types

### Single Field Index

```javascript
db.users.createIndex({ age: 1 })         // ascending
db.users.createIndex({ age: -1 })        // descending
// Single field এ ascending/descending same performance
```

### Compound Index

Multiple fields এর উপর একটা index:

```javascript
db.users.createIndex({ city: 1, age: 1 })
```

**ESR Rule (Equality, Sort, Range):** Compound index এ field order matter করে। Best practice:
1. **Equality** fields আগে
2. **Sort** fields মাঝে
3. **Range** fields শেষে

```javascript
// Query: city = "Dhaka" AND age > 25 ORDER BY name
// Optimal index:
db.users.createIndex({ city: 1, name: 1, age: 1 })
//                      equality  sort    range
```

**Prefix rule:** Compound index `{a, b, c}` এটা এই queries কে support করে:
- `{a}`
- `{a, b}`
- `{a, b, c}`

কিন্তু `{b}` বা `{b, c}` কে support করে না।

### Multikey Index (Array Index)

Array field এ index করলে automatically multikey হয়। Array এর প্রতিটা element index হয়:

```javascript
db.users.createIndex({ hobbies: 1 })
// hobbies: ["coding", "chess"] হলে দুটো index entry তৈরি হয়
```

**Limitation:** Compound index এ দুটো multikey field একসাথে index করা যায় না।

### Text Index

Full-text search এর জন্য:

```javascript
db.articles.createIndex({ title: "text", body: "text" })

// Text search
db.articles.find({ $text: { $search: "mongodb index" } })

// Relevance score সহ
db.articles.find(
  { $text: { $search: "mongodb" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

**Limitation:** একটা collection এ শুধু একটা text index।

### Geospatial Index

```javascript
// 2dsphere — GeoJSON objects এর জন্য (sphere surface)
db.locations.createIndex({ loc: "2dsphere" })

// Near query
db.locations.find({
  loc: {
    $near: {
      $geometry: { type: "Point", coordinates: [90.4125, 23.8103] },
      $maxDistance: 5000  // meters
    }
  }
})
```

### Hashed Index

Equality queries এর জন্য। Sharding এ hash-based partitioning এ use হয়।

```javascript
db.users.createIndex({ _id: "hashed" })
```

Range queries support করে না।

### Wildcard Index

Dynamic বা unknown field names এর জন্য:

```javascript
db.products.createIndex({ "attributes.$**": 1 })
// attributes এর যেকোনো nested field index হবে
```

---

## Index Options

```javascript
db.users.createIndex(
  { email: 1 },
  {
    unique: true,              // duplicate value allow করবে না
    sparse: true,              // field না থাকলে index এ include করবে না
    expireAfterSeconds: 3600,  // TTL index — document automatically delete হবে
    name: "email_unique_idx",  // custom index name
    background: false,         // deprecated in 4.2+, এখন সব background এ হয়
    partialFilterExpression: { age: { $gt: 18 } }  // partial index
  }
)
```

### TTL Index

Document automatically delete করার জন্য। Date field এ index করতে হবে:

```javascript
db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 86400 }  // 24 hours পর delete
)
```

Background job প্রতি 60 seconds এ expired documents delete করে।

### Partial Index

শুধু filter condition match করা documents index করে। Smaller index, better performance:

```javascript
// শুধু active users index করো
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { status: "active" } }
)
```

---

## Index Management

```javascript
// সব indexes দেখো
db.users.getIndexes()
db.users.indexStats()

// Index delete করো
db.users.dropIndex({ age: 1 })
db.users.dropIndex("age_1")        // index name দিয়ে
db.users.dropIndexes()             // সব drop (_id ছাড়া)

// Index rebuild করো
db.users.reIndex()
```

---

## Query Plan Analysis — EXPLAIN

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
        "stage": "IXSCAN",        // index scan — ভালো
        "indexName": "city_1_age_1"
      }
    },
    "rejectedPlans": [...]
  },
  "executionStats": {
    "nReturned": 150,             // কতটা document return হয়েছে
    "totalKeysExamined": 155,     // কতটা index key examine হয়েছে
    "totalDocsExamined": 155,     // কতটা document examine হয়েছে
    "executionTimeMillis": 2
  }
}
```

**Stage names:**
- `COLLSCAN` — collection scan, index নেই (খারাপ)
- `IXSCAN` — index scan (ভালো)
- `FETCH` — document fetch করছে index থেকে
- `SORT` — in-memory sort (index sort নেই)
- `PROJECTION` — field projection

**Efficiency check:**
- `nReturned` ≈ `totalKeysExamined` হওয়া উচিত
- বড় gap মানে index selective না

---

## Index Strategy — Common Mistakes

**1. Over-indexing:**
প্রতিটা write এ সব indexes update করতে হয়। বেশি index মানে slow write।

**2. Unused indexes:**
```javascript
db.users.aggregate([{ $indexStats: {} }])
// accesses.ops = 0 মানে index use হচ্ছে না
```

**3. Low cardinality field এ index:**
`gender` field এ index করা useless — শুধু দুটো value।

**4. Compound index এর order ভুল:**
ESR rule follow না করলে index efficient হয় না।

---

<a name="chapter-6"></a>
# Chapter 6 — Aggregation Framework

Aggregation হলো MongoDB এর data processing pipeline। Complex data transformation, grouping, calculation করার জন্য।

---

## Pipeline Concept

Aggregation pipeline হলো stages এর sequence। প্রতিটা stage আগের stage এর output নিয়ে কাজ করে:

```
Collection → Stage 1 → Stage 2 → Stage 3 → Result
```

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },      // Stage 1: filter
  { $group: { _id: "$city", total: { $sum: "$amount" } } },  // Stage 2: group
  { $sort: { total: -1 } },                 // Stage 3: sort
  { $limit: 5 }                             // Stage 4: limit
])
```

---

## Important Stages

### $match — Filter

```javascript
{ $match: { status: "active", age: { $gt: 18 } } }
```

Pipeline এর শুরুতে $match দিলে index use করতে পারে।

### $project — Shape Output

```javascript
{ $project: {
  name: 1,
  age: 1,
  _id: 0,
  fullName: { $concat: ["$firstName", " ", "$lastName"] },  // computed field
  ageGroup: { $cond: { if: { $gte: ["$age", 18] }, then: "adult", else: "minor" } }
}}
```

### $group — Aggregate

```javascript
{ $group: {
  _id: "$city",                    // group by field
  count: { $sum: 1 },              // count
  totalAmount: { $sum: "$amount" },
  avgAmount: { $avg: "$amount" },
  maxAmount: { $max: "$amount" },
  minAmount: { $min: "$amount" },
  names: { $push: "$name" },       // array তে collect
  uniqueNames: { $addToSet: "$name" }  // unique array
}}

// সব documents group (overall aggregate)
{ $group: { _id: null, total: { $sum: "$amount" } } }
```

### $sort

```javascript
{ $sort: { amount: -1, name: 1 } }
```

### $limit এবং $skip

```javascript
{ $limit: 10 }
{ $skip: 20 }
```

### $unwind — Array Flatten

Array কে individual documents এ expand করে:

```javascript
// Input: { name: "Rahim", hobbies: ["coding", "chess"] }
{ $unwind: "$hobbies" }
// Output:
// { name: "Rahim", hobbies: "coding" }
// { name: "Rahim", hobbies: "chess" }

// null/missing array handle
{ $unwind: { path: "$hobbies", preserveNullAndEmptyArrays: true } }
```

### $lookup — JOIN

```javascript
// orders collection তে users collection join করো
{ $lookup: {
  from: "users",           // join করার collection
  localField: "user_id",   // orders এর field
  foreignField: "_id",     // users এর field
  as: "user_info"          // result array field name
}}

// Pipeline-based lookup (complex conditions)
{ $lookup: {
  from: "products",
  let: { orderId: "$_id", minQty: "$min_quantity" },
  pipeline: [
    { $match: { $expr: { $and: [
      { $eq: ["$$orderId", "$order_id"] },
      { $gte: ["$quantity", "$$minQty"] }
    ]}}}
  ],
  as: "matching_products"
}}
```

### $addFields / $set

```javascript
{ $addFields: {
  totalWithTax: { $multiply: ["$amount", 1.15] },
  processed: true
}}
```

### $facet — Multiple Pipelines

একটা pipeline এ multiple sub-pipelines চালানো:

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
```

### $bucket / $bucketAuto

```javascript
// Manual ranges
{ $bucket: {
  groupBy: "$age",
  boundaries: [0, 18, 30, 50, 100],
  default: "other",
  output: { count: { $sum: 1 } }
}}

// Auto buckets
{ $bucketAuto: { groupBy: "$price", buckets: 5 } }
```

---

## Aggregation Expressions

```javascript
// Arithmetic
$add, $subtract, $multiply, $divide, $mod, $pow, $sqrt, $abs, $ceil, $floor, $round

// String
$concat, $toUpper, $toLower, $substr, $split, $trim, $strLenBytes, $indexOfBytes

// Date
$year, $month, $dayOfMonth, $hour, $minute, $second, $dayOfWeek
$dateToString: { format: "%Y-%m-%d", date: "$created_at" }

// Conditional
$cond: { if: condition, then: val1, else: val2 }
$ifNull: ["$field", "default_value"]
$switch: { branches: [{ case: ..., then: ... }], default: ... }

// Array
$size, $slice, $arrayElemAt, $first, $last, $filter, $map, $reduce, $zip
$in: [value, array]

// Comparison
$eq, $ne, $gt, $gte, $lt, $lte, $cmp
```

---

## Aggregation Pipeline Optimization

**1. $match আগে রাখো** — data কমাও early, index use করার সুযোগ দাও

**2. $project আগে রাখো** — unnecessary fields early remove করো

**3. $sort + $limit একসাথে** — MongoDB automatically top-K sort করে (সব sort করার দরকার নেই)

**4. $lookup minimize করো** — expensive operation, possible হলে embed করো

**5. allowDiskUse:**
```javascript
db.orders.aggregate([...], { allowDiskUse: true })
// 100MB memory limit exceed করলে disk use করবে
```

---

<a name="chapter-7"></a>
# Chapter 7 — Transactions

MongoDB 4.0 থেকে multi-document ACID transactions। 4.2 থেকে sharded cluster এও।

---

## কখন Transaction দরকার

MongoDB এর single document operation naturally atomic। একটাই document এ সব data থাকলে transaction দরকার নেই।

Transaction দরকার:
- Multiple documents atomically update করতে হবে
- Multiple collections এ একসাথে write করতে হবে
- "All or nothing" guarantee দরকার

---

## Transaction Basics

```javascript
const session = client.startSession()

try {
  session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" }
  })

  // Transaction এর মধ্যে operations
  await db.accounts.updateOne(
    { _id: "account1" },
    { $inc: { balance: -500 } },
    { session }
  )

  await db.accounts.updateOne(
    { _id: "account2" },
    { $inc: { balance: 500 } },
    { session }
  )

  await session.commitTransaction()
} catch (error) {
  await session.abortTransaction()
} finally {
  session.endSession()
}
```

---

## Read/Write Concerns

### Read Concern

কোন data পড়বে — কতটা "committed":

| Level | কাজ |
|-------|-----|
| `local` | Latest data, durability guarantee নেই (default) |
| `available` | Sharding এ local এর মতো কিন্তু আরো loose |
| `majority` | Majority replica তে written data — more consistent |
| `snapshot` | Transaction start এর snapshot — repeatable read |
| `linearizable` | Strongest — সব previous write reflect করবে |

### Write Concern

Write কতটা "durable" হবে:

```javascript
{ w: 1 }           // primary acknowledge করলেই (default)
{ w: "majority" }  // majority replica acknowledge করলে
{ w: 0 }           // fire and forget
{ j: true }        // journal এ লেখার পর acknowledge
{ wtimeout: 5000 } // 5 seconds এর মধ্যে না হলে error
```

---

## Transaction Limitations

- Maximum **60 seconds** (default 1 minute)
- Transaction এ 1000 এর বেশি documents modify করা avoid করো (performance)
- DDL operations (createCollection, createIndex) transaction এ করা যায় না (4.4 এ limited support)
- Sharded cluster এ cross-shard transaction expensive

---

<a name="chapter-8"></a>
# Chapter 8 — Replication — Replica Set

Replica Set হলো MongoDB এর high availability mechanism। একই data এর multiple copies different servers এ।

---

## Replica Set Architecture

```
Primary (Read + Write)
    ├── Secondary 1 (Read, Replication)
    ├── Secondary 2 (Read, Replication)
    └── Arbiter (Voting only, no data)
```

**Minimum recommended: 3 nodes** (1 Primary + 2 Secondary, বা 2 data + 1 Arbiter)

**Maximum voting members: 7**

---

## Roles

### Primary

- সব write operations receive করে
- Oplog এ operations record করে
- Default read target

### Secondary

- Primary থেকে oplog replicate করে
- Async replication (replication lag possible)
- Read queries serve করতে পারে (configured হলে)
- Failover এ Primary হওয়ার candidate

### Arbiter

- Data store করে না
- শুধু election এ vote দেয়
- Tie-breaking এর জন্য
- Resource কম লাগে
- **Production এ arbiter avoid করো** — data redundancy কমে

### Hidden Secondary

Replica থেকে replication করে কিন্তু client থেকে দেখা যায় না। Reporting বা backup এর জন্য:

```javascript
rs.reconfig({
  members: [
    ...
    { _id: 3, host: "hidden:27017", hidden: true, priority: 0 }
  ]
})
```

### Delayed Secondary

নির্দিষ্ট সময় পিছিয়ে থাকে। Accidental data deletion থেকে recovery:

```javascript
{ _id: 4, host: "delayed:27017", priority: 0, hidden: true, secondaryDelaySecs: 3600 }
// 1 hour পিছিয়ে থাকবে
```

---

## Replication কীভাবে কাজ করে

```
1. Client → Primary তে write করে
2. Primary → data store করে + oplog এ record করে
3. Secondary → Primary এর oplog poll করে (বা push receive করে)
4. Secondary → নিজের oplog এ apply করে
5. Secondary → Primary কে acknowledge পাঠায়
```

**Asynchronous** — Primary Secondary এর acknowledge এর জন্য wait করে না (write concern w:1 হলে)।

---

## Election — Automatic Failover

Primary unreachable হলে election শুরু হয়:

```
1. Secondary heartbeat fail করলে election শুরু করে
2. নিজে candidate হয়, অন্য members এর কাছে vote চায়
3. Majority vote পেলে নতুন Primary হয়
4. পুরনো Primary ফিরে আসলে Secondary হিসেবে join করে
```

**Election time: ~12 seconds** (default)

**Priority:**
- Higher priority = election এ preferred
- Priority 0 = কখনো Primary হবে না

```javascript
rs.reconfig({
  members: [
    { _id: 0, host: "node1:27017", priority: 2 },   // preferred primary
    { _id: 1, host: "node2:27017", priority: 1 },
    { _id: 2, host: "node3:27017", priority: 0 }    // never primary
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
  bindIp: 0.0.0.0

replication:
  replSetName: "rs0"           # সব nodes এ same name

security:
  keyFile: /etc/mongodb/keyfile  # nodes এর মধ্যে authentication
```

### Keyfile তৈরি

```bash
openssl rand -base64 756 > /etc/mongodb/keyfile
chmod 400 /etc/mongodb/keyfile
chown mongodb:mongodb /etc/mongodb/keyfile
```

### Initialize

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
rs.status()                    // সব members এর status
rs.conf()                      // current configuration
rs.isMaster()                  // current node info (deprecated, use hello())
db.hello()                     // current node info
rs.printReplicationInfo()      // oplog info (Primary)
rs.printSecondaryReplicationInfo()  // replication lag (Secondary)

// Member add/remove
rs.add("node4:27017")
rs.addArb("arbiter:27017")
rs.remove("node4:27017")

// Stepdown Primary (graceful)
rs.stepDown(60)  // 60 seconds এর জন্য stepdown

// Manual failover
rs.stepDown()
```

---

## Read Preference

Secondary থেকে read করতে Read Preference configure করো:

| Mode | কাজ |
|------|-----|
| `primary` | শুধু Primary (default) |
| `primaryPreferred` | Primary, না হলে Secondary |
| `secondary` | শুধু Secondary |
| `secondaryPreferred` | Secondary, না হলে Primary |
| `nearest` | Network latency কম এমন node |

```javascript
db.users.find().readPref("secondaryPreferred")

// Connection string এ
mongodb://node1,node2,node3/?replicaSet=rs0&readPreference=secondary
```

**সতর্কতা:** Secondary তে stale data থাকতে পারে। Consistency critical হলে Primary ব্যবহার করো।

---

<a name="chapter-9"></a>
# Chapter 9 — Sharding

Sharding হলো MongoDB এর horizontal scaling mechanism। Data multiple servers এ distribute করা।

---

## কখন Sharding দরকার

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
Config Servers (Replica Set) — metadata, shard map
  ↓
Shard 1 (Replica Set) — data subset
Shard 2 (Replica Set) — data subset
Shard 3 (Replica Set) — data subset
```

**Config Servers:**
- Sharding metadata store করে (কোন shard এ কোন data)
- Replica Set হিসেবে deploy করতে হয় (3 nodes)

**mongos:**
- Client এর single entry point
- Config server থেকে routing table cache করে
- Stateless — restart safe

**Shard:**
- প্রতিটা shard একটা Replica Set
- Data এর subset রাখে

---

## Shard Key — সবচেয়ে Important Decision

Shard key হলো যে field(s) এর উপর ভিত্তি করে data shards এ distribute হবে। **এই decision অনেক গুরুত্বপূর্ণ এবং পরিবর্তন করা কঠিন।**

```javascript
sh.shardCollection("mydb.orders", { customer_id: 1 })
// customer_id হলো shard key
```

### Good Shard Key এর বৈশিষ্ট্য

**High cardinality:** অনেক unique values। Boolean বা gender এর মতো field ভালো shard key না।

**Low frequency:** কোনো একটা value অনেক বেশি frequent হলে সেই shard hot হয়ে যাবে।

**Non-monotonically increasing:** Timestamp বা auto-increment id shard key হিসেবে ব্যবহার করলে সব new data একটাই shard এ যাবে (hot shard problem)।

---

## Sharding Strategies

### Range-Based Sharding

Shard key এর value range অনুযায়ী data distribute হয়:

```
Shard 1: customer_id 0 - 1000
Shard 2: customer_id 1001 - 2000
Shard 3: customer_id 2001+
```

**Pros:** Range query efficient — সব data একটা shard এ
**Cons:** Monotonic key এ hot shard problem

### Hash-Based Sharding

Shard key এর hash value অনুযায়ী distribute:

```javascript
sh.shardCollection("mydb.orders", { customer_id: "hashed" })
```

**Pros:** Even distribution, hot shard problem নেই
**Cons:** Range query inefficient — সব shards এ broadcast করতে হয়

### Zone-Based Sharding

Geographic বা logical partitioning:

```javascript
// Shard কে zone assign করো
sh.addShardToZone("shard1", "DHAKA")
sh.addShardToZone("shard2", "CHITTAGONG")

// Data range কে zone এ assign করো
sh.updateZoneKeyRange("mydb.orders", { region: "DHAKA" }, { region: "DHAKB" }, "DHAKA")
```

---

## Chunks

MongoDB data কে **chunks** এ ভাগ করে (default 64MB)। প্রতিটা chunk একটা shard key range represent করে।

**Balancer:** Background process chunks এর মধ্যে even distribution maintain করে। Chunk এর সংখ্যা বেশি হলে migrate করে।

```javascript
sh.status()                    // sharding status
sh.getBalancerState()          // balancer on/off
sh.startBalancer()
sh.stopBalancer()

// Balancer window set করো (maintenance hours)
db.settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "01:00", stop: "05:00" } } },
  { upsert: true }
)
```

---

## Targeted vs Broadcast Queries

**Targeted query (efficient):** Query তে shard key থাকলে mongos সঠিক shard এ route করে।

**Broadcast query (inefficient):** Shard key না থাকলে mongos সব shards এ পাঠায়, merge করে return করে।

```javascript
// Targeted — shard key আছে
db.orders.find({ customer_id: 1001 })

// Broadcast — shard key নেই
db.orders.find({ status: "pending" })
```

**Design principle:** সব common queries যাতে shard key include করে।

---

## Sharding Setup

```javascript
// mongos এ
sh.enableSharding("mydb")

// Index তৈরি করো shard key তে (আগে)
db.orders.createIndex({ customer_id: 1 })

// Collection shard করো
sh.shardCollection("mydb.orders", { customer_id: 1 })

// Status check
sh.status()
db.orders.getShardDistribution()
```

---

<a name="chapter-10"></a>
# Chapter 10 — Storage Engine — WiredTiger

MongoDB 3.2 থেকে **WiredTiger** default storage engine।

---

## WiredTiger Architecture

```
MongoDB API
     ↓
WiredTiger API
     ↓
WiredTiger Cache (in-memory B-Trees)
     ↓
Write-Ahead Log (Journal)
     ↓
Data Files (compressed, on disk)
```

---

## Document-Level Concurrency

WiredTiger **document-level locking** use করে। আগে MongoDB collection-level lock ছিল।

এর মানে: দুটো operations একই collection এ কিন্তু আলাদা documents এ একসাথে চলতে পারে। Concurrency অনেক বেশি।

**MVCC (Multi-Version Concurrency Control):** Readers writers কে block করে না। Writer নতুন version তৈরি করে, reader পুরনো consistent version দেখে।

---

## Compression

WiredTiger data compress করে store করে। Disk usage কমে।

```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy    # snappy (default), zlib, zstd, none
    indexConfig:
      prefixCompression: true
```

| Compressor | Speed | Compression Ratio |
|-----------|-------|-------------------|
| `snappy` | Fast | Moderate (default) |
| `zlib` | Slower | Better |
| `zstd` | Fast | Best |
| `none` | Fastest | None |

---

## WiredTiger Cache Configuration

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4    # default: max(50% RAM - 1GB, 256MB)
```

**Rule of thumb:** RAM এর 50-70% WiredTiger cache কে দাও। OS এর জন্যও কিছু রাখো।

Cache eviction:
- Dirty data 20% হলে eviction শুরু হয়
- Cache 95% full হলে aggressive eviction

---

## Checkpoint

WiredTiger প্রতি 60 seconds এ (বা 2GB journal হলে) **checkpoint** তৈরি করে। Checkpoint মানে cache এর dirty data disk এ flush করা।

Checkpoint এর মধ্যে crash হলে last checkpoint থেকে journal replay করে recover হয়।

```yaml
storage:
  wiredTiger:
    engineConfig:
      checkpointSizeMB: 2048    # 2GB journal এ checkpoint
```

---

<a name="chapter-11"></a>
# Chapter 11 — Performance & Tuning

---

## Profiler — Slow Query Detection

MongoDB এর built-in query profiler:

```javascript
// Profiling levels:
// 0 = off
// 1 = slow queries only
// 2 = all queries

db.setProfilingLevel(1, { slowms: 100 })  // 100ms এর বেশি হলে log

// Profile data দেখো
db.system.profile.find().sort({ ts: -1 }).limit(10)
db.system.profile.find({ millis: { $gt: 100 } }).sort({ millis: -1 })
```

Profile entry এ important fields:
```json
{
  "op": "query",
  "ns": "mydb.users",
  "query": { "age": { "$gt": 25 } },
  "millis": 450,
  "keysExamined": 10000,
  "docsExamined": 10000,
  "nreturned": 150,
  "planSummary": "COLLSCAN"  // এখানে IXSCAN হওয়া উচিত ছিল
}
```

---

## currentOp — Running Operations

```javascript
db.currentOp({ active: true, secs_running: { $gt: 5 } })
// 5 seconds এর বেশি চলছে এমন operations

// Kill করো
db.killOp(opid)
```

---

## Connection Pool Tuning

```yaml
# mongod.conf
net:
  maxIncomingConnections: 65536
```

Application এ connection pool:
```javascript
MongoClient.connect(uri, {
  maxPoolSize: 100,      // default 100
  minPoolSize: 10,
  maxIdleTimeMS: 30000,  // idle connection timeout
  waitQueueTimeoutMS: 5000
})
```

---

## Read/Write Performance Tips

**1. Index সব common queries তে:**
```javascript
db.users.find({ age: { $gt: 25 }, city: "Dhaka" }).explain()
// COLLSCAN দেখলে index তৈরি করো
```

**2. Projection use করো:**
```javascript
// Bad — সব fields
db.users.find({ city: "Dhaka" })

// Good — শুধু দরকারী fields
db.users.find({ city: "Dhaka" }, { name: 1, email: 1 })
```

**3. Covered Query:**
Query র সব fields index এ থাকলে document fetch করতে হয় না (index only scan):
```javascript
db.users.createIndex({ city: 1, name: 1, email: 1 })
db.users.find({ city: "Dhaka" }, { name: 1, email: 1, _id: 0 })
// সব fields index এ আছে — covered query
```

**4. $limit early:**
```javascript
db.orders.find().sort({ amount: -1 }).limit(10)
// MongoDB optimize করে — সব sort করে না
```

**5. Large collection এ count() avoid:**
```javascript
// Slow — actual count করে
db.users.countDocuments({ status: "active" })

// Fast — approximate (metadata থেকে)
db.users.estimatedDocumentCount()
```

**6. Avoid $where এবং $regex without anchors:**
```javascript
// Bad — JavaScript execution, slow
db.users.find({ $where: "this.age > 25" })

// Bad — full string scan
db.users.find({ name: { $regex: "rahim" } })

// Good — anchored regex, can use index
db.users.find({ name: { $regex: "^Rahim" } })
```

---

## Memory Tuning

```javascript
// Working set size দেখো
db.serverStatus().wiredTiger.cache

// page faults দেখো — বেশি হলে RAM কম
db.serverStatus().extra_info.page_faults
```

**Rule:** Working set (hot data + indexes) RAM এ fit করা উচিত।

---

## mongotop এবং mongostat

```bash
# Top — collection level activity
mongotop 5          # প্রতি 5 seconds

# Stat — server level metrics
mongostat 5

# mongostat output:
# insert/query/update/delete/getmore/command — ops/sec
# dirty — WiredTiger cache dirty %
# used — WiredTiger cache used %
# res — resident memory
# conn — connections
```

---

## Aggregation Pipeline Optimization Tips

```javascript
// Bad — $match শেষে
db.orders.aggregate([
  { $lookup: {...} },
  { $unwind: "$items" },
  { $match: { status: "active" } }  // অনেক data process হওয়ার পর filter
])

// Good — $match আগে
db.orders.aggregate([
  { $match: { status: "active" } },  // আগেই filter করো
  { $lookup: {...} },
  { $unwind: "$items" }
])
```

---

## OS Level Tuning

```bash
# /etc/sysctl.conf
vm.swappiness = 1                    # swap minimize
net.core.somaxconn = 65535
net.ipv4.tcp_keepalive_time = 120

# Transparent Huge Pages disable
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# ulimits (/etc/security/limits.conf)
mongod soft nofile 64000
mongod hard nofile 64000
mongod soft nproc 64000
mongod hard nproc 64000
```

---

<a name="chapter-12"></a>
# Chapter 12 — Security

---

## Authentication

MongoDB এর authentication system এ **users** থাকে database level এ।

```javascript
// Admin user তৈরি
use admin
db.createUser({
  user: "admin",
  pwd: "strongpassword",
  roles: [{ role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase"]
})

// Application user তৈরি
use mydb
db.createUser({
  user: "appuser",
  pwd: "apppassword",
  roles: [{ role: "readWrite", db: "mydb" }]
})
```

### Authentication Enable

```yaml
# mongod.conf
security:
  authorization: enabled
```

### Authentication Mechanisms

- **SCRAM-SHA-256** (default) — password-based
- **x.509** — certificate-based (production recommended)
- **LDAP** — enterprise feature
- **Kerberos** — enterprise feature

---

## Built-in Roles

| Role | Access |
|------|--------|
| `read` | Collection read |
| `readWrite` | Collection read + write |
| `dbAdmin` | DB administration (indexes, stats) |
| `userAdmin` | User management in DB |
| `dbOwner` | dbAdmin + userAdmin + readWrite |
| `clusterAdmin` | Cluster management |
| `root` | Superuser |
| `readAnyDatabase` | All DBs read |
| `readWriteAnyDatabase` | All DBs read + write |
| `userAdminAnyDatabase` | All DBs user management |

---

## Role Management

```javascript
// Custom role তৈরি
db.createRole({
  role: "readOrdersOnly",
  privileges: [{
    resource: { db: "mydb", collection: "orders" },
    actions: ["find"]
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
  bindIp: 127.0.0.1,192.168.1.10  # specific IPs তে bind করো
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem
    CAFile: /etc/ssl/ca.pem
```

---

## Encryption

### Encryption at Rest (WiredTiger Encrypted Storage Engine)

Enterprise feature। Community edition এ filesystem-level encryption use করো।

### Encryption in Transit (TLS)

```yaml
net:
  tls:
    mode: requireTLS               # allowTLS, preferTLS, requireTLS
    certificateKeyFile: /path/to/cert.pem
    CAFile: /path/to/ca.pem
    allowInvalidHostnames: false
    allowInvalidCertificates: false
```

---

## Auditing

```yaml
# mongod.conf (Enterprise)
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ atype: { $in: ["authenticate", "createUser", "dropUser"] } }'
```

---

<a name="chapter-13"></a>
# Chapter 13 — Backup & Recovery

---

## Backup Strategies

### mongodump / mongorestore — Logical Backup

BSON format এ export করে:

```bash
# Full backup
mongodump --host localhost --port 27017 \
  --username admin --password password \
  --authenticationDatabase admin \
  --out /backup/mongodb/$(date +%Y%m%d)

# Specific database
mongodump --db mydb --out /backup/

# Specific collection
mongodump --db mydb --collection users --out /backup/

# Compressed
mongodump --db mydb --archive=/backup/mydb.gz --gzip

# Restore
mongorestore --host localhost --port 27017 \
  --username admin --password password \
  --authenticationDatabase admin \
  /backup/mongodb/20240323

# Restore specific collection
mongorestore --db mydb --collection users /backup/mydb/users.bson

# Drop existing data before restore
mongorestore --drop --db mydb /backup/mydb
```

**Limitations:**
- Large dataset এ slow
- Point-in-time recovery নেই (oplog backup সহ হলে possible)
- Backup চলাকালীন data change হতে পারে (Replica Secondary থেকে নাও)

### Oplog Backup — Point-in-Time Recovery

```bash
# Oplog backup
mongodump --host localhost --port 27017 \
  --db local --collection oplog.rs \
  --out /backup/oplog

# Restore with oplog replay
mongorestore --oplogReplay --oplogLimit "1711123200:1" /backup/
```

### mongodump Best Practices

```bash
# Replica Set Secondary থেকে backup নাও (Primary load কমাবে)
mongodump --host "rs0/secondary1:27017" \
  --readPreference secondary \
  --out /backup/
```

---

## File System Snapshot — Physical Backup

Production এ preferred। Fast এবং consistent।

### LVM Snapshot

```bash
# MongoDB pause করো (journal flush)
mongo admin --eval "db.fsyncLock()"

# LVM snapshot নাও
lvcreate -L 10G -s -n mongodb-snapshot /dev/vg0/mongodb-data

# MongoDB resume করো
mongo admin --eval "db.fsyncUnlock()"

# Snapshot mount করে backup নাও
mount /dev/vg0/mongodb-snapshot /mnt/snapshot
rsync -av /mnt/snapshot/ /backup/mongodb/

# Snapshot remove করো
lvremove /dev/vg0/mongodb-snapshot
```

### Replica Set এ Snapshot

Secondary তে data file সরাসরি copy করা যায়। Replica Set এ Secondary stop করো, snapshot নাও, restart করো।

---

## mongodump vs Snapshot

| | mongodump | Snapshot |
|--|---------|---------|
| Speed | Slow | Fast |
| Consistency | Good (Secondary তে) | Perfect |
| Point-in-time | Oplog সহ | হ্যাঁ |
| Space | BSON size | Disk size |
| Restore | Easy | Complex |
| Cross-version | হ্যাঁ | Same version |

---

## Backup Verification

```bash
# Restore করে verify করো
mongorestore --db verify_db /backup/mydb

# Data check করো
mongo verify_db --eval "db.users.count()"

# Consistency check
db.runCommand({ dbCheck: "users" })
```

---

## Monitoring Backup with Ops Manager / Cloud Manager

MongoDB এর official backup tool। Continuous backup, PITR, GUI।

Community alternative: **Percona Backup for MongoDB (pbm)**।

---

<a name="chapter-14"></a>
# Chapter 14 — Real Patterns & Use Cases

---

## 1. E-Commerce Product Catalog

Product catalog এর জন্য MongoDB perfect কারণ প্রতিটা product এর attributes আলাদা।

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
    "material": "100% cotton",
    "brand": "ABC"
  },
  "inventory": {
    "in_stock": true,
    "quantity": 150
  },
  "images": ["img1.jpg", "img2.jpg"],
  "tags": ["casual", "summer", "cotton"],
  "created_at": ISODate("2024-03-23")
}
```

```javascript
// Category + price range search
db.products.createIndex({ category: 1, price: 1 })
db.products.createIndex({ tags: 1 })
db.products.createIndex({ "attributes.color": 1, "attributes.size": 1 })

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
  "_id": ObjectId("..."),
  "device_id": "sensor_001",
  "timestamp": ISODate("2024-03-23T10:00:00Z"),
  "metrics": {
    "temperature": 28.5,
    "humidity": 65,
    "pressure": 1013
  }
}
```

MongoDB 5.0 থেকে **Time Series Collections:**

```javascript
db.createCollection("sensor_data", {
  timeseries: {
    timeField: "timestamp",
    metaField: "device_id",
    granularity: "minutes"   // seconds, minutes, hours
  },
  expireAfterSeconds: 2592000  // 30 days
})
```

Time series collection internally data compress করে, query optimize করে।

---

## 3. User Activity / Audit Log

```json
{
  "_id": ObjectId("..."),
  "user_id": "user:1001",
  "action": "login",
  "resource": "/dashboard",
  "ip": "192.168.1.1",
  "timestamp": ISODate("2024-03-23T10:00:00Z"),
  "metadata": {
    "browser": "Chrome",
    "os": "Windows"
  }
}
```

```javascript
// TTL index — 90 days পর auto-delete
db.audit_logs.createIndex({ timestamp: 1 }, { expireAfterSeconds: 7776000 })

// Query
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
  "status": "published",
  "views": 1250,
  "created_at": ISODate("..."),
  "updated_at": ISODate("...")
}
```

**সতর্কতা:** Comments array unbounded হলে document size 16MB ছাড়াতে পারে। বড় blog এর জন্য comments আলাদা collection এ রাখো।

---

## 5. Geospatial — Delivery Tracking

```json
{
  "_id": "courier:1001",
  "name": "Rahim",
  "location": {
    "type": "Point",
    "coordinates": [90.4125, 23.8103]  // [longitude, latitude]
  },
  "status": "active",
  "updated_at": ISODate("...")
}
```

```javascript
db.couriers.createIndex({ location: "2dsphere" })

// নিকটতম courier খোঁজো
db.couriers.find({
  location: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [90.4000, 23.8000]  // pickup point
      },
      $maxDistance: 5000  // 5km
    }
  },
  status: "active"
}).limit(5)
```

---

## 6. Change Streams — Real-time Data

MongoDB 3.6+। Collection এর changes real-time এ subscribe করা যায়:

```javascript
// orders collection এর changes watch করো
const changeStream = db.orders.watch([
  { $match: { "operationType": { $in: ["insert", "update"] } } }
])

changeStream.on("change", (change) => {
  console.log("Change detected:", change)
  // Real-time notification, cache invalidation, etc.
})
```

Replica Set বা Sharded Cluster এ কাজ করে (oplog use করে)।

---

## 7. Schema Migration Strategy

Schema change করার সময়:

```javascript
// Lazy migration — read এ old format handle করো, write এ নতুন format
db.users.find().forEach(user => {
  if (!user.fullName) {
    db.users.updateOne(
      { _id: user._id },
      {
        $set: { fullName: user.firstName + " " + user.lastName },
        $unset: { firstName: "", lastName: "" }
      }
    )
  }
})
```

**Dual-write strategy:** নতুন code উভয় format লেখে এবং পড়তে পারে। Background migration চলে। Migration শেষে পুরনো format support remove করো।

---

# Quick Reference Summary

## Data Modeling Decision Tree

```
Data related?
  └── হ্যাঁ
      └── Data একসাথে access হয়?
          ├── হ্যাঁ → EMBED করো
          └── না
              └── Array bounded? (small, fixed max)
                  ├── হ্যাঁ → EMBED করো
                  └── না → REFERENCE করো
```

## Index Selection Guide

| Query Pattern | Recommended Index |
|--------------|-------------------|
| Equality | Single field বা Compound prefix |
| Range | Compound (equality first, range last) |
| Sort | Include sort field in compound |
| Array | Multikey |
| Text search | Text index |
| Geo | 2dsphere |
| TTL | Date field with expireAfterSeconds |
| Full document | Covered query (all fields in index) |

## Replica Set vs Sharding

| Need | Solution |
|------|---------|
| High availability | Replica Set (3 nodes) |
| Read scaling | Replica Set + read preference |
| Write scaling | Sharding |
| Data > single server | Sharding |

## Common Configuration Checklist

```yaml
# mongod.conf
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4
    collectionConfig:
      blockCompressor: snappy

net:
  port: 27017
  bindIp: 127.0.0.1

security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile  # Replica Set এ

replication:
  replSetName: "rs0"
  oplogSizeMB: 10240

operationProfiling:
  slowOpThresholdMs: 100
  mode: slowOp
```

## OS Tuning Checklist

```bash
# sysctl.conf
vm.swappiness = 1
vm.overcommit_memory = 1
net.core.somaxconn = 65535

# Transparent Huge Pages
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# ulimits
mongod soft nofile 64000
mongod hard nofile 64000
```

---

*MongoDB Complete Study Guide | Covers MongoDB Administration, Architecture, and Operations*
