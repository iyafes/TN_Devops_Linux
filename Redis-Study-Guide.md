# Redis Complete Study Guide (Revised)
### Layer 1 → Layer 8 | Architecture · Internals · Cluster · Tuning · Patterns
### সব technical terms সহজ ভাষায় in-place explain করা আছে

---

## Table of Contents

1. [Layer 1 — Core Architecture & Internals](#layer-1)
2. [Layer 2 — Data Structures: Deep Internals](#layer-2)
3. [Layer 3 — Persistence](#layer-3)
4. [Layer 4 — Replication](#layer-4)
5. [Layer 5 — Redis Sentinel](#layer-5)
6. [Layer 6 — Redis Cluster](#layer-6)
7. [Layer 7 — Performance & Tuning](#layer-7)
8. [Layer 8 — Real Patterns](#layer-8)

---

<a name="layer-1"></a>
# Layer 1 — Core Architecture & Internals

## Redis কী এবং কেন exist করে?

Redis হলো একটা **in-memory data structure store**।

**"In-memory" মানে কী?**
সাধারণ database (যেমন PostgreSQL, MySQL) data রাখে hard disk এ। তুমি কিছু read করতে চাইলে disk থেকে আনতে হয়। Disk অনেক slow — data আনতে milliseconds (1 millisecond = 1 second এর 1000 ভাগের 1 ভাগ) লাগে।

Redis data রাখে RAM এ (Random Access Memory — তোমার computer এর primary memory, যেটা বন্ধ করলে data চলে যায়)। RAM disk এর চেয়ে **100-1000 গুণ দ্রুত**। তাই Redis এর response time হয় microseconds এ (1 microsecond = 1 millisecond এর 1000 ভাগের 1 ভাগ)।

Redis তিনটা role এ কাজ করতে পারে:
- **Database** — primary data store হিসেবে (সব data এখানেই থাকবে)
- **Cache** — frequently accessed data দ্রুত serve করতে (database এর সামনে একটা fast layer)
- **Message Broker** — systems এর মধ্যে message pass করতে (একটা system message দেয়, আরেকটা receive করে)

**"Cache" মানে কী?**
ধরো তোমার একটা প্রশ্নের উত্তর বের করতে 10 minute লাগে। পরেরবার কেউ একই প্রশ্ন করলে আবার 10 minute লাগানো বোকামি। উত্তরটা কাছে লিখে রাখলে সাথে সাথে দেওয়া যায়। এটাই cache। Database এর ক্ষেত্রে — একবার database থেকে data এনে Redis এ রাখো, পরের বার Redis থেকে দাও।

**Tradeoff:** RAM expensive এবং limited (disk এর চেয়ে অনেক কম)। Server restart হলে data হারিয়ে যেতে পারে — এটা configure করা যায়, Layer 3 এ দেখবো।

---

## Single-Threaded Event Loop

Redis **single-threaded** — মানে একটাই thread সব command execute করে।

**"Thread" মানে কী?**
Thread হলো একটা program এর execution এর একটা path। তুমি যখন হাঁটছো সেটা একটা thread। তুমি একইসাথে দুই দিকে হাঁটতে পারবে না — কিন্তু তুমি আর তোমার বন্ধু একইসাথে দুই দিকে হাঁটতে পারবে (দুটো thread)।

এটা শুনে মনে হতে পারে Redis slow হওয়ার কথা। কিন্তু Redis প্রতি second এ **millions of operations** handle করে। কারণটা হলো Redis এর কাজই RAM এ read/write। RAM operation এতটাই fast যে CPU কখনো bottleneck (আটকে যাওয়ার জায়গা) হয় না।

**Single thread এর সুবিধা:**
- **Locking লাগে না** — Locking মানে হলো "এই data টা আমি use করছি, অন্যরা wait করো।" Multiple threads থাকলে একই data একসাথে change করার সমস্যা হয়। Single thread এ এই ঝামেলা নেই।
- **Context switching নেই** — Context switching মানে OS (Operating System) একটা thread থেকে অন্য thread এ যাওয়া। এতে সময় নষ্ট হয়। Single thread এ এটা নেই।
- **Race condition নেই** — Race condition মানে দুটো thread একই সময়ে একই resource change করার চেষ্টা করে সমস্যা তৈরি হওয়া।

**Redis 6.0 থেকে পরিবর্তন:** Network I/O (data আসা-যাওয়া) এর জন্য multiple thread use করে, কিন্তু actual command execute করা এখনো single thread এ। এই architecture কে বলে **I/O threading with single-threaded command processing।**

---

## Event Loop — I/O Multiplexing

Redis এর heart হলো এর **event loop**।

**"Event loop" কী?**
Event loop হলো একটা infinite loop যেটা বারবার চেক করে — "কোনো নতুন কাজ আছে? থাকলে করো।" Browser এও event loop আছে — তুমি button click করলে browser সেটা detect করে action নেয়।

Redis Linux এর `epoll` mechanism use করে।

**`epoll` কী?**
ধরো তোমার 10,000 জন student আছে এবং তুমি জানতে চাও কে হাত তুলেছে। দুটো উপায়:
1. প্রতিটা student কে একে একে জিজ্ঞেস করো — "তুমি হাত তুলেছ?" (এটা `select` — পুরনো, slow)
2. বলো "হাত তুললে আমাকে জানাও" এবং অপেক্ষা করো (এটা `epoll` — modern, fast)

`epoll` তে Redis বলে রাখে "কোনো client data পাঠালে আমাকে জানাও।" Data আসলে epoll সাথে সাথে Redis কে notify করে। Redis তখন সেটা process করে।

```
epoll_wait() → কোনো client data পাঠালে event আসে
     ↓
Event loop সেটা detect করে
     ↓
Command parse করে
     ↓
Execute করে RAM এ
     ↓
Response পাঠায়
     ↓
আবার epoll_wait() এ ফিরে যায়
```

এই cycle এতটাই দ্রুত যে practically সব client simultaneously (একইসাথে) serve হচ্ছে মনে হয়।

---

## Redis Object (robj) — Memory তে Data কীভাবে থাকে

Redis এ সব কিছু internally **Redis Object (robj)** হিসেবে represent হয়।

**"Internally" মানে কী?**
তুমি Redis এ `SET name Rahim` বললে বাইরে থেকে মনে হয় simple। কিন্তু ভেতরে Redis এটাকে একটা structured object (data + metadata একসাথে) হিসেবে রাখে।

```c
typedef struct redisObject {
    unsigned type;      // STRING, LIST, SET, HASH, ZSET — কোন ধরনের data
    unsigned encoding;  // ভেতরে actual কীভাবে store হচ্ছে
    unsigned lru;       // last access time — কতদিন আগে use হয়েছিল
    int refcount;       // কতটা জায়গায় এই object use হচ্ছে
    void *ptr;          // actual data এর memory address
} robj;
```

**"`encoding` field" কেন important?**
একই type এর data এর জন্য Redis পরিস্থিতি অনুযায়ী আলাদা internal format use করে memory বাঁচাতে। এটাকে বলে **encoding optimization।**

উদাহরণ: String value যদি integer (যেমন 100) এবং ছোট হয়, Redis সেটাকে actual string হিসেবে store না করে directly integer হিসেবে রাখে। "100" as string = 3 bytes, কিন্তু as integer = much less।

---

## Memory Management — jemalloc

Redis নিজে memory manage করে `jemalloc` দিয়ে।

**"Memory management" কী?**
Program চলার সময় data রাখতে memory (RAM) নিতে হয়। কাজ শেষে সেই memory ছেড়ে দিতে হয়। এই নেওয়া-ছাড়ার process কে memory management বলে।

**`jemalloc` কী?**
`malloc` (memory allocate) করার একটা library। System এর default malloc এর চেয়ে বেশি efficient।

**"Memory fragmentation" কী?**
ধরো তোমার কাছে 10টা box আছে। তুমি 3টায় জিনিস রাখলে, 2টা খালি করলে, আবার 4টায় রাখলে। এখন boxes এর মাঝে মাঝে ফাঁকা জায়গা আছে — কিন্তু বড় কিছু রাখতে চাইলে ফাঁকা জায়গা নেই কারণ সেগুলো ছড়িয়ে আছে। এটাই fragmentation।

Redis তে fragmentation মানে — Redis কে 1GB memory দিলে, কিন্তু `INFO memory` বলছে সে 1.3GB use করছে। Extra 300MB allocate হয়েছে কিন্তু actually use হচ্ছে না।

`mem_fragmentation_ratio` দিয়ে এটা measure হয়:
- **1.0 - 1.5** → Healthy (ঠিকঠাক)
- **1.5 এর বেশি** → Fragmentation সমস্যা, restart বা active defrag দরকার
- **1.0 এর কম** → Redis swap use করছে (নিচে explain আছে), মারাত্মক সমস্যা

**"Swap" কী?**
RAM ভরে গেলে OS কিছু RAM এর data hard disk এ সরিয়ে রাখে — এটাকে বলে swap। Disk RAM এর চেয়ে অনেক slow, তাই swap use হলে performance ভেঙে পড়ে। Redis এর জন্য swap একদম ভালো না।

**Active defragmentation:** Redis 4.0 থেকে runtime এ fragmentation কমানোর feature। Memory এর ছড়িয়ে থাকা data গুলো reorganize করে।

---

## RESP Protocol — Client-Server Communication

Redis client আর server এর মধ্যে communication হয় **RESP (REdis Serialization Protocol)** দিয়ে।

**"Protocol" কী?**
দুটো system এর মধ্যে communication এর নিয়মকানুন। যেমন মানুষ কথা বলার সময় ভাষার নিয়ম follow করে — protocol সেটাই কিন্তু computers এর জন্য।

**"Serialization" কী?**
তোমার কাছে complex data আছে (object, array, etc.)। এটাকে network এ পাঠাতে হলে bytes এর sequence এ convert করতে হয়। এই conversion কে serialization বলে। Receive করে আবার original format এ ফিরিয়ে আনাকে deserialization বলে।

RESP text-based এবং human readable।

`SET name Rahim` command টা wire (network) এ যায় এভাবে:
```
*3\r\n          ← 3টা element আছে এই command এ
$3\r\n          ← প্রথম element 3 bytes লম্বা
SET\r\n
$4\r\n          ← দ্বিতীয় element 4 bytes লম্বা
name\r\n
$5\r\n          ← তৃতীয় element 5 bytes লম্বা
Rahim\r\n
```

**`\r\n` কী?** Carriage Return + Line Feed — line ending। Windows এ এভাবে line শেষ হয়।

Server response:
```
+OK\r\n         ← '+' মানে simple string response
```

Error response:
```
-ERR wrong number of arguments\r\n   ← '-' মানে error
```

Integer response:
```
:42\r\n         ← ':' মানে integer
```

**RESP3** (Redis 6.0+) — নতুন version। আরো বেশি data types support করে।

---

## Command Execution Flow — শুরু থেকে শেষ পর্যন্ত

```
Client → TCP connection establish করে
      ↓
Redis এ connection register হয় (epoll এ add হয়)
      ↓
Client "SET name Rahim" পাঠায়
      ↓
Network buffer এ bytes আসে
("buffer" = temporary storage, পানির পাইপে যেমন পানি জমে)
      ↓
RESP parser bytes কে command এ convert করে
      ↓
Command lookup table এ "SET" খোঁজে
(lookup table = একটা map যেখানে command name → handler function)
      ↓
SET handler call হয়
      ↓
Key-value RAM এ write হয়
      ↓
AOF/RDB এ write (persistence configure থাকলে — Layer 3 এ বিস্তারিত)
      ↓
Replication buffer এ command যায় (replica থাকলে — Layer 4 এ বিস্তারিত)
      ↓
"+OK\r\n" client কে পাঠায়
```

পুরো এই flow microseconds এ শেষ হয়।

---

## Redis Database (keyspace)

**"Keyspace" কী?**
Redis এ সব keys এর collection কে keyspace বলে। যেমন একটা ঘরের সব জিনিসের তালিকা।

একটা Redis instance এ by default **16টা logical database** থাকে (index 0-15)। এগুলো আলাদা keyspace। এক database এর key অন্য database এ দেখা যায় না।

```bash
SELECT 0    # database 0 এ যাও (default)
SELECT 1    # database 1 এ যাও
```

**Production এ এটা avoid করা উচিত।** আলাদা database দরকার হলে আলাদা Redis instance ব্যবহার করো। কারণ:
- Cluster mode এ শুধু database 0 support করে
- আলাদা instance এ আলাদা memory limit, eviction policy set করা যায়

---

## Redis Configuration File — redis.conf

Redis start করার সময় config file দেওয়া যায়:
```bash
redis-server /etc/redis/redis.conf
```

Runtime এ (চলা অবস্থায়) config change করা যায়:
```bash
CONFIG SET maxmemory 2gb
CONFIG GET maxmemory
CONFIG REWRITE    # running config কে redis.conf এ save করে (disk এ permanent করে)
```

Important initial config:
```conf
bind 127.0.0.1              # কোন IP address এ listen (শুনবে) করবে
port 6379                   # default port (যে port এ connection নেবে)
daemonize yes               # background এ run করবে (terminal বন্ধ করলেও চলবে)
loglevel notice             # log কতটা verbose হবে: debug, verbose, notice, warning
logfile /var/log/redis/redis.log
databases 16                # logical database এর সংখ্যা
requirepass yourpassword    # authentication password (ছাড়া connect করা যাবে না)
maxmemory 2gb               # maximum memory limit
```

---

<a name="layer-2"></a>
# Layer 2 — Data Structures: Deep Internals

Redis এর data structures শুধু "কী কী আছে" জানলে হবে না। প্রতিটা structure ভেতরে কীভাবে implement হয়েছে, কোন situation এ কোনটা use করবে, performance implications — এটা বুঝতে হবে।

---

## 1. String

Redis এর সবচেয়ে basic type। ভেতরে C এর সাধারণ string না — Redis নিজস্ব implementation use করে যার নাম **SDS (Simple Dynamic String)।**

**"Dynamic" কী মানে?**
Fixed size না — প্রয়োজন অনুযায়ী size বাড়তে পারে।

### SDS কেন?

**C string এ সমস্যা:**
- Length জানতে O(n) — পুরো string এর শেষ পর্যন্ত যেতে হয় (n = string এর length)
- **"O(n)" কী?** Big O notation — কাজের সময় input size এর সাথে কীভাবে বাড়ে। O(n) মানে string 1000 character হলে 1000 step, O(1) মানে যেকোনো size এ একটাই step।
- Binary safe না — null byte (`\0`) পেলে string শেষ ভাবে, তাই binary data (image, file) রাখা যায় না

**SDS structure:**
```c
struct sdshdr {
    int len;      // বর্তমান length — O(1) এ জানা যায়
    int free;     // allocated কিন্তু এখনো unused space
    char buf[];   // actual data
};
```

SDS এর সুবিধা:
- Length O(1) এ জানা যায় (len field directly পড়লেই হয়)
- Binary safe — যেকোনো data রাখা যায়
- Append এ বারবার memory allocate করতে হয় না (free space আগে থেকেই রাখা)

### String Internal Encodings

| Encoding | কখন | Memory |
|----------|-----|--------|
| **INT** | value টা integer এবং LONG range এ (±9 quintillion) | সবচেয়ে কম — pointer এর জায়গায় directly integer রাখে |
| **EMBSTR** | value ≤ 44 bytes | কম — robj আর SDS একটাই memory allocation এ, পাশাপাশি থাকায় CPU cache efficient |
| **RAW** | value > 44 bytes | বেশি — robj আর SDS আলাদা allocation এ |

**"Pointer" কী?** Memory address। যেখানে actual data আছে সেই জায়গার ঠিকানা।

**"CPU cache" কী?** CPU এর নিজস্ব ultra-fast memory। RAM থেকেও দ্রুত। কাছাকাছি memory একসাথে load হয় cache এ। তাই robj আর SDS পাশাপাশি থাকলে (EMBSTR) দুটোই একসাথে cache এ আসে — fast।

```bash
SET counter 100
OBJECT ENCODING counter      # "int"

SET name "Rahim"
OBJECT ENCODING name         # "embstr"

SET essay "This is a very long string that definitely exceeds the forty-four bytes limit"
OBJECT ENCODING essay        # "raw"
```

### String Commands — সম্পূর্ণ

```bash
# Basic
SET key value
GET key
DEL key
EXISTS key          # key আছে কিনা check (1=আছে, 0=নেই)

# Expiry (TTL — Time To Live) সহ Set
# TTL = কতক্ষণ পরে key automatically delete হবে
SET key value EX 300          # 300 seconds পরে expire
SET key value PX 300000       # 300000 milliseconds পরে expire
SET key value EXAT 1735689600 # এই Unix timestamp এ expire
# Unix timestamp = 1 January 1970 থেকে seconds count
SET key value KEEPTTL         # existing TTL maintain করবে (নতুন TTL set না করে)

# Conditional Set
SET key value NX              # NX = "Not eXists" — শুধু key না থাকলে set করবে
SET key value XX              # XX = "eXists" — শুধু key থাকলে update করবে

# Atomic Counter
# "Atomic" = operation টা complete হবে অথবা হবেই না, মাঝপথে থামবে না
# Race condition থেকে safe
INCR counter                  # 1 বাড়াবে
INCRBY counter 5              # 5 বাড়াবে
DECR counter
DECRBY counter 3
INCRBYFLOAT price 1.50        # decimal number বাড়াবে

# Multiple Keys একসাথে
MSET key1 val1 key2 val2      # M = Multiple
MGET key1 key2 key3
MSETNX key1 val1 key2 val2    # সব keys না থাকলেই set করবে (atomic)

# String Manipulation
APPEND key " more text"       # existing value এর শেষে add করো
STRLEN key                    # string এর length
GETRANGE key 0 4              # 0 থেকে 4 index পর্যন্ত substring
SETRANGE key 6 "Redis"        # position 6 থেকে overwrite

# Atomic Get+Set
GETSET key newvalue           # পুরনো value return করে, নতুন set করে (atomic)
GETDEL key                    # value return করে, key delete করে
GETEX key EX 300              # value return করে, expiry set করে
```

### String Use Cases

**Rate Limiting:**
```bash
# প্রতি minute এ 100 requests limit
INCR user:1001:rate
EXPIRE user:1001:rate 60
# value > 100 হলে block করো
# প্রতি 60 seconds এ counter auto-reset হয় (key expire হয়ে যায়)
```

**Session Token:**
```bash
SET session:abc123 "user_id:1001" EX 3600  # 1 hour TTL
```

**Distributed Counter:**
```bash
INCR page:homepage:views     # atomic — multiple servers একসাথে করলেও সমস্যা নেই
```

---

## 2. Hash

একটা key এর নিচে multiple field-value pair। Object এর মতো।

### Internal Encodings

**Listpack** (Redis 7.0+ এ listpack, আগে ziplist ছিল):

**"Listpack/Ziplist" কী?**
Compact একটা format যেখানে সব data একটা continuous (একটানা) memory block এ রাখা হয়। মনে করো একটা লম্বা চাদরে সব জিনিস একসারিতে রাখা।

- Field সংখ্যা ≤ 128 এবং প্রতিটা value ≤ 64 bytes হলে use হয় (এই limits config করা যায়)
- সব data একটা continuous memory block এ — **cache friendly** (CPU cache এ একসাথে লোড হয়)
- Lookup O(n) — linear scan করতে হয় (প্রতিটা item check করতে হয়)
- Memory অনেক কম

**Hashtable:**
- Threshold পার হলে automatically convert হয়
- Lookup O(1) — directly jump করা যায়
- Memory বেশি

**এই automatic conversion কেন?**
ছোট data এর জন্য compact format efficient। বড় হলে speed এর জন্য hashtable দরকার। Redis smartly decide করে।

### Hash Commands

```bash
# Set
HSET user:1001 name "Rahim" city "Dhaka" age 28
# H = Hash, SET = set করো
HSETNX user:1001 email "rahim@example.com"   # NX = না থাকলেই set

# Get
HGET user:1001 name
HMGET user:1001 name city age                # M = Multiple fields
HGETALL user:1001                            # সব fields এবং values
HKEYS user:1001                              # শুধু field names
HVALS user:1001                              # শুধু values
HLEN user:1001                               # field এর count
HEXISTS user:1001 email                      # field আছে কিনা

# Update
HINCRBY user:1001 age 1                      # numeric field বাড়াও
HINCRBYFLOAT product:1 price 10.5

# Delete
HDEL user:1001 email
```

### Hash vs String for Objects — কোনটা ভালো

**String approach:**
```bash
SET user:1001 '{"name":"Rahim","city":"Dhaka","age":28}'
# সমস্যা: age update করতে পুরো JSON parse → update → serialize → save করতে হয়
# "JSON" = JavaScript Object Notation, data exchange format
# "parse" = text থেকে object এ convert করা
# "serialize" = object থেকে text এ convert করা
```

**Hash approach:**
```bash
HSET user:1001 name "Rahim" city "Dhaka" age 28
HINCRBY user:1001 age 1   # শুধু age field update, বাকি সব unchanged
# সুবিধা: partial update, individual field access efficient
```

---

## 3. List

Ordered (ক্রমানুসারে সাজানো) collection of strings। Doubly linked list হিসেবে implement করা।

**"Doubly Linked List" কী?**
মনে করো একটা train। প্রতিটা বগি (node) এর সামনের বগির address এবং পেছনের বগির address জানা আছে। এভাবে যেকোনো দিক থেকে traverse করা যায়। "Doubly" মানে দুই দিকেই link আছে।

### Internal Encodings

**Listpack:**
- Element সংখ্যা ≤ 128 হলে
- Compact memory

**Quicklist:**

**"Quicklist" কী?**
Linked list of listpacks। কল্পনা করো একটা train যেখানে প্রতিটা বগি নিজেই একটা ছোট সাজানো বাক্স। বাক্সের ভেতরে কিছু items। বাক্সগুলো linked list এ connected।

এটা Redis এর smart compromise:
- Pure linked list এর চেয়ে memory efficient (সব কিছু পাশাপাশি থাকে)
- Pure array এর চেয়ে insert/delete fast (মাঝখানে add করতে সব shift করতে হয় না)

### List Commands

```bash
# Push (add করো)
LPUSH mylist "first"          # L = Left, left থেকে push
RPUSH mylist "last"           # R = Right, right থেকে push
LPUSHX mylist "val"           # X = শুধু list exist করলে push
RPUSHX mylist "val"

# Pop (remove করো)
LPOP mylist                   # left থেকে pop (remove + return)
RPOP mylist                   # right থেকে pop
LPOP mylist 3                 # 3টা একসাথে pop

# Blocking Pop — Queue এর জন্য crucial
BLPOP mylist 30               # B = Blocking। element না থাকলে 30 seconds wait করবে
BRPOP mylist 30
# "Blocking" মানে thread wait করবে — CPU waste না করে OS কে বলে "জাগাও যখন data আসবে"

# Read
LRANGE mylist 0 -1            # সব elements (0 থেকে last পর্যন্ত, -1 = last)
LRANGE mylist 0 4             # প্রথম 5টা (index 0,1,2,3,4)
LINDEX mylist 0               # index 0 এর element
LLEN mylist                   # list এর length

# Modify
LSET mylist 2 "newvalue"      # index 2 এর value change করো
LINSERT mylist BEFORE "val" "newval"  # "val" এর আগে insert করো
LREM mylist 2 "value"         # "value" এর 2টা occurrence remove করো (left থেকে শুরু)
LTRIM mylist 0 99             # শুধু প্রথম 100টা রাখো, বাকি delete করো

# Move between lists
LMOVE source destination LEFT RIGHT   # atomic move — source থেকে নিয়ে destination এ দাও
```

### List Use Cases

**Message Queue:**
```bash
# Queue = লাইনে দাঁড়ানো। যে আগে এসেছে আগে সেবা পাবে (FIFO — First In First Out)
# Producer = যে message তৈরি করে
RPUSH job:queue '{"job_id": 123, "type": "send_email"}'

# Consumer = যে message নেয় এবং কাজ করে (blocking — CPU waste করে না)
BLPOP job:queue 0    # 0 মানে indefinitely (অনির্দিষ্টকাল) wait করো
```

**Activity Feed:**
```bash
LPUSH user:1001:feed "Parcel AWB123 delivered"
LTRIM user:1001:feed 0 99    # শুধু latest 100টা রাখো
LRANGE user:1001:feed 0 9    # latest 10টা দেখাও
```

---

## 4. Set

Unique (অনন্য, duplicate নেই) strings এর unordered (ক্রম নেই) collection।

### Internal Encodings

**Listpack:**
- Element সংখ্যা ≤ 128 এবং প্রতিটা element ≤ 64 bytes হলে

**Intset:**
**"Intset" কী?** সব elements যদি integer হয় তাহলে use হয় এই special compact format। Sorted integer array — binary search দিয়ে O(log n) এ খোঁজা যায়। অনেক memory efficient।

**Hashtable:**
- Threshold পার হলে

### Set Commands

```bash
# Add/Remove
SADD myset "apple" "banana" "cherry"   # S = Set, ADD = যোগ করো
SREM myset "banana"                    # REM = remove

# Check
SISMEMBER myset "apple"          # IS MEMBER? 1 = আছে, 0 = নেই
SMISMEMBER myset "apple" "grape" # M = Multiple check
SCARD myset                      # CARD = cardinality (কতটা element)

# Read
SMEMBERS myset                    # সব elements দেখাও
# Production এ সাবধান — large set হলে Redis block হয়ে যাবে
SRANDMEMBER myset 3               # RAND = random, 3টা random element (remove করে না)
SPOP myset 2                      # random 2টা pop করে remove করে

# Set Operations — এটাই Set এর superpower
SUNION set1 set2                  # UNION = দুটো set এর সব unique elements একসাথে (A∪B)
SINTER set1 set2                  # INTER = intersection — দুটোতেই আছে এমন (A∩B)
SDIFF set1 set2                   # DIFF = difference — set1 এ আছে কিন্তু set2 তে নেই (A-B)

# Store results (result নতুন key তে save করো)
SUNIONSTORE dest set1 set2
SINTERSTORE dest set1 set2
SDIFFSTORE dest set1 set2
```

### Set Use Cases

**Online Users:**
```bash
SADD online:users user:1001
SREM online:users user:1001      # logout
SCARD online:users               # কতজন online
SISMEMBER online:users user:1001 # specific user online কিনা
```

**Tagging System:**
```bash
SADD article:101:tags "redis" "database" "cache"
SADD article:102:tags "redis" "nosql"

SINTER article:101:tags article:102:tags   # common tags
```

**Unique Visitors:**
```bash
SADD page:home:visitors:2024-03-23 "user:1001" "user:1002"
SCARD page:home:visitors:2024-03-23   # unique visitor count
```

---

## 5. Sorted Set (ZSet)

Set এর মতো unique members, কিন্তু প্রতিটা member এর একটা **floating point score** (decimal number) আছে। Score অনুযায়ী sorted থাকে।

### Internal Encodings

**Listpack:**
- Member সংখ্যা ≤ 128 এবং প্রতিটা member ≤ 64 bytes হলে

**Skiplist + Hashtable (combined):**

**"Skiplist" কী?**
একটা probabilistic (random এর উপর নির্ভরশীল) data structure। কল্পনা করো একটা express train এবং local train। Express শুধু বড় stations এ থামে, local সব stations এ। কোনো station যেতে হলে প্রথমে express এ যাও, কাছের বড় station এ নেমে local এ switch করো — অনেক কম stops।

Skiplist এ multiple levels এর linked list আছে:
- Level 3 (top): খুব কম nodes, বড় jumps
- Level 2: কিছু nodes
- Level 1 (bottom): সব nodes

উপর থেকে শুরু করে বড় bড় jump করে target এর কাছে যাই, তারপর নিচে নামি। **O(log n)** time।

Binary search tree এর মতো efficient কিন্তু implement করা অনেক সহজ।

**কেন Skiplist + Hashtable দুটো একসাথে?**
- **Skiplist** দিয়ে O(log N) এ range query (score 80-100 এর সব members)
- **Hashtable** দিয়ে O(1) এ individual member এর score পাওয়া
- দুটো একসাথে থাকায় উভয় operation efficient

### Sorted Set Commands

```bash
# Add
ZADD leaderboard 95.5 "player1"          # Z = Sorted set
ZADD leaderboard 87.0 "player2" 92.3 "player3"   # multiple
ZADD leaderboard NX 100 "player4"        # NX = শুধু না থাকলে add করো
ZADD leaderboard XX 98 "player1"         # XX = শুধু থাকলে update করো
ZADD leaderboard GT 99 "player1"         # GT = Greater Than — score বড় হলেই update
ZADD leaderboard LT 50 "player1"         # LT = Less Than — score ছোট হলেই update

# Score
ZSCORE leaderboard "player1"
ZINCRBY leaderboard 5.0 "player1"        # score বাড়াও (atomic)

# Range (score অনুযায়ী sorted)
ZRANGE leaderboard 0 -1                  # সব members (low to high score)
ZRANGE leaderboard 0 -1 WITHSCORES       # scores সহ দেখাও
ZRANGE leaderboard 0 -1 REV              # REV = reverse, high to low
ZRANGE leaderboard "(80" "+inf" BYSCORE  # score 80 এর বেশি ("+inf" = positive infinity)
# "(" মানে exclusive (80 টা include না), "[" হলে inclusive

# Rank (position)
ZRANK leaderboard "player1"             # rank (0-indexed, low score = low rank/high number)
ZREVRANK leaderboard "player1"          # REV rank (high score = rank 0 = top)

# Count
ZCARD leaderboard                        # CARD = total members
ZCOUNT leaderboard 80 100               # score range এ কতজন

# Remove
ZREM leaderboard "player1"
ZREMRANGEBYRANK leaderboard 0 9         # rank range এ remove
ZREMRANGEBYSCORE leaderboard "-inf" 50  # score range এ remove ("-inf" = negative infinity)

# Set Operations
ZUNIONSTORE dest 2 zset1 zset2          # 2 = কতটা source set
ZINTERSTORE dest 2 zset1 zset2
ZDIFFSTORE dest 2 zset1 zset2
```

### Sorted Set Use Cases

**Real-time Leaderboard:**
```bash
ZADD game:leaderboard 1500 "user:1001"
ZINCRBY game:leaderboard 50 "user:1001"      # score add করো
ZREVRANGE game:leaderboard 0 9 WITHSCORES    # top 10
ZREVRANK game:leaderboard "user:1001"        # আমার rank কত
```

**Priority Queue:**
```bash
# Queue = লাইন। Priority queue = VIP লাইন, গুরুত্ব অনুযায়ী
# Lower score = higher priority
ZADD job:queue 1 "critical_job"
ZADD job:queue 5 "normal_job"
ZADD job:queue 10 "low_priority_job"
ZPOPMIN job:queue    # সবচেয়ে কম score (highest priority) job নাও
```

**Time-series Events (Sliding Window):**
```bash
# Score হিসেবে timestamp use করো
ZADD user:1001:events 1711123200 "login"
# Last 1 hour এর events (now = current Unix timestamp)
ZRANGEBYSCORE user:1001:events (now-3600) now
```

---

## 6. Bitmap

String type এর উপর built করা। প্রতিটা bit (0 বা 1) individually manipulate করা যায়।

**"Bit" কী?** Binary digit। সবচেয়ে ছোট data unit। 0 বা 1। 8 bits = 1 byte।

**কেন Bitmap?** Large boolean (true/false) data store করতে অনেক memory efficient।

```bash
SETBIT user:1001:login 0 1     # bit position 0 = 1 (আজ login করেছে)
GETBIT user:1001:login 0       # 1 বা 0
BITCOUNT user:1001:login       # কতটা bit 1 আছে (কতদিন login করেছে)

# Bitwise operations (bit-level logical operations)
BITOP AND dest key1 key2  # AND: উভয়েই 1 হলে 1
BITOP OR dest key1 key2   # OR: যেকোনো একটায় 1 হলে 1
```

**Use case:** 1 million user এর daily login track করতে মাত্র 125KB লাগে। 1 million individual boolean String এ হলে লাগতো অনেক বেশি।

---

## 7. HyperLogLog

Unique item count estimate করার জন্য probabilistic data structure।

**"Probabilistic" কী?** Exact result না দিয়ে statistical estimation দেয়। কিছুটা error থাকে কিন্তু memory অনেক কম লাগে।

Memory সবসময় মাত্র **12KB** — যতই unique items হোক।

Trade-off: Exact count না, ~0.81% error।

```bash
PFADD visitors "user:1001" "user:1002" "user:1003"
# PF = Philippe Flajolet (inventor এর নাম)
PFCOUNT visitors              # approximate unique count
PFMERGE total visitors1 visitors2   # দুটো HyperLogLog merge করো
```

**Use case:** Billions of unique visitors count করতে। 12KB দিয়ে billion unique values track — impossible with regular Set।

---

## 8. Stream

Redis 5.0 তে add হয়েছে। Append-only log এর মতো।

**"Append-only log" কী?** একটা খাতা যেখানে শুধু নতুন লেখা যায়, পুরনো কিছু মুছে যায় না। অনেকটা bank transaction এর history।

Kafka এর মতো event streaming Redis এ করা যায়।

**"Event streaming" কী?** Events (কোনো কিছু ঘটার record) continuously produce হয় এবং consumers সেটা process করে। যেমন real-time order tracking।

```bash
# Add event
XADD mystream * user_id 1001 action "login"
# X = stream, * মানে auto-generate ID (timestamp-sequence format)

# Read
XREAD COUNT 10 STREAMS mystream 0   # সব events পড়ো (0 = beginning থেকে)
XRANGE mystream - +                  # - থেকে + মানে সব entries

# Consumer Groups (multiple consumers এর মধ্যে work distribute করো)
XGROUP CREATE mystream mygroup $ MKSTREAM
# $ মানে এখন থেকে নতুন messages শুনবে
XREADGROUP GROUP mygroup consumer1 COUNT 10 STREAMS mystream >
# > মানে শুধু এই consumer এ assign হয়নি এমন messages
XACK mystream mygroup <message-id>   # processing complete confirm করো
```

**Use case:** Event sourcing (সব events record করা), audit logs (কে কী করেছে), real-time analytics।

---

## Key Naming Convention

Redis এ key naming এর কোনো enforcement নেই, কিন্তু convention (রীতি) follow করা critical:

```
object-type:id:field
```

উদাহরণ:
```
user:1001:profile
user:1001:sessions
order:AWB123:status
courier:dhaka:active
```

**সুবিধা:**
- `SCAN` দিয়ে pattern match করে key খোঁজা যায় (`user:*`)
- Namespace collision (দুটো system একই key নাম use) এড়ানো যায়
- Human readable এবং maintainable

**Key size:** Maximum 512MB, কিন্তু key যত বড় তত memory বেশি। Short meaningful keys best।

---

<a name="layer-3"></a>
# Layer 3 — Persistence

Redis in-memory — কিন্তু server restart হলে data হারিয়ে যাওয়া থেকে বাঁচাতে disk এ save করার mechanism আছে।

**"Persistence" কী?** Data টিকিয়ে রাখা। RAM volatile (বিদ্যুৎ গেলে data চলে যায়), disk non-volatile (বিদ্যুৎ গেলেও data থাকে)।

দুটো approach: **RDB** এবং **AOF।** দুটো একসাথেও use করা যায়।

---

## RDB — Redis Database Backup

RDB হলো একটা point-in-time **snapshot** of the entire dataset।

**"Snapshot" কী?** ঠিক একটা নির্দিষ্ট মুহূর্তে সব data এর exact copy। যেমন একটা ছবি তোলা — তোলার মুহূর্তের সব কিছু capture হয়।

নির্দিষ্ট interval এ পুরো memory এর data একটা binary file এ dump করে।

### RDB কীভাবে কাজ করে

```
Redis main process
      ↓
BGSAVE command (বা auto trigger)
      ↓
fork() — child process তৈরি হয়
      ↓
Main process: normal operation continue করে (client serve করে)
Child process: memory snapshot নেয়
      ↓
Child process: data compress করে dump.rdb তে write করে
      ↓
Child process exits (শেষ হয়)
      ↓
dump.rdb complete হলে atomically replace হয়
```

**"fork()" কী?**
Unix/Linux এ একটা process নিজের exact copy তৈরি করতে পারে — এটাকে fork বলে। Parent process চলতে থাকে, child process আলাদাভাবে কাজ করে।

**"Copy-on-Write (CoW)" কী?**
fork() করলে child process parent এর exact memory copy পায়। কিন্তু actually সাথে সাথে copy হয় না — শুধু page table shared থাকে।

**"Page table" কী?** OS যে table রাখে যেটা virtual memory address কে physical RAM address এ map করে।

CoW এ: যখন parent বা child কোনো memory page modify করে, **তখনই** সেই page এর actual copy হয়। বাকি pages shared থাকে। এতে fork() fast হয় এবং memory কম লাগে।

এর ফলে:
- Main process block হয় না (client serve করতে পারে)
- Child process consistent (একটা নির্দিষ্ট সময়ের) snapshot দেখে

### RDB Configuration

```conf
# redis.conf

# Automatic save triggers (conditions)
save 3600 1       # 3600 seconds এ কমপক্ষে 1টা change হলে save করো
save 300 100      # 300 seconds এ কমপক্ষে 100টা change হলে save করো
save 60 10000     # 60 seconds এ কমপক্ষে 10000টা change হলে save করো
# Multiple conditions — যেকোনো একটা match করলেই save হবে

# File settings
dbfilename dump.rdb        # file এর নাম
dir /var/lib/redis         # কোন directory তে save হবে

# Compression
rdbcompression yes         # LZF compression use করবে — file size কমে

# Checksum
rdbchecksum yes            # CRC64 checksum add করবে — file corrupt হলে detect করা যাবে
# "Checksum" = file এর content থেকে calculate করা একটা number। File change হলে number বদলায়।

# Save disable করতে
save ""
```

### Manual RDB Commands

```bash
BGSAVE              # BG = Background — non-blocking save
SAVE                # Foreground — BLOCKING (সব client request আটকে থাকবে) — production এ avoid করো
LASTSAVE            # last successful save এর Unix timestamp
BGSAVE SCHEDULE     # BGSAVE already running থাকলে queue করে রাখে
```

### RDB এর Pros এবং Cons

**Pros:**
- Compact single file — backup এবং transfer সহজ
- Restore করা fast — পুরো dataset একটা file থেকে load
- Performance impact কম — child process এ হয়, main process unaffected
- Point-in-time snapshot — disaster recovery এর জন্য ভালো

**Cons:**
- Data loss possible — last snapshot এর পর থেকে crash হলে সেই সময়ের সব data যাবে
- Large dataset এ fork() অনেক time নিতে পারে (momentary pause)
- CoW এর কারণে memory usage temporarily spike করে (worst case প্রায় 2x)

**RDB কখন use করবে:** Data loss কিছুটা acceptable (cache use case), fast restart দরকার, backup/archival purpose।

---

## AOF — Append Only File

AOF প্রতিটা write command কে log file এ append (শেষে যোগ) করে।

**"Append-only" কী?** শুধু শেষে লেখা যায়, মাঝখানে কিছু modify বা delete করা যায় না।

Crash হলে সেই log replay (আবার execute) করে data recover করা যায়।

### AOF কীভাবে কাজ করে

```
Client → SET name Rahim
              ↓
Redis RAM এ write করে
              ↓
Command টা AOF buffer এ যায়
("buffer" = temporary memory — disk এ লেখার আগে এখানে জমে)
              ↓
fsync policy অনুযায়ী disk এ write হয়
              ↓
appendonly.aof file এ append হয়
```

**"fsync" কী?**
OS সাধারণত disk এ লেখার সময় নিজে buffer রাখে (OS page cache) এবং পরে লেখে। fsync বলে "এখনই disk এ লেখো, buffer এ রাখো না।" Crash protection এর জন্য দরকার।

File এর ভেতরে RESP format এ commands:
```
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$5\r\nRahim\r\n
```

### AOF fsync Policy — সবচেয়ে important config

```conf
appendfsync always      # প্রতিটা write এ fsync — safest (কোনো data loss নেই), slowest
appendfsync everysec    # প্রতি second এ একবার fsync — good balance (DEFAULT, recommended)
appendfsync no          # OS কে decide করতে দাও — fastest, least safe (OS crash এ data loss)
```

**everysec** মানে maximum 1 second এর data loss possible crash এ।

### AOF Rewrite — File Size কমানো

AOF file সময়ের সাথে অনেক বড় হয়। কারণ একই key অনেকবার update হলে সব commands log হয়:
```
SET counter 1
SET counter 2
SET counter 3
... (1000 বার)
SET counter 1000
```

Rewrite করলে current state এ পৌঁছানোর minimum commands দিয়ে নতুন AOF তৈরি হয়:
```
SET counter 1000   (শুধু এটাই দরকার)
```

```bash
BGREWRITEAOF    # BG = Background, non-blocking AOF rewrite
```

Config:
```conf
auto-aof-rewrite-percentage 100   # AOF file size double হলে rewrite trigger করো
auto-aof-rewrite-min-size 64mb    # minimum 64mb হলে তবেই rewrite
```

**Rewrite process:**
```
fork() — child process তৈরি হয়
Child: current memory state থেকে minimum commands লিখে নতুন AOF তৈরি করে
Main: নতুন write commands আসলে old AOF তে + buffer এ রাখে
Child শেষ হলে: নতুন AOF + buffer এর commands merge হয়
Atomically পুরনো AOF replace হয় (একধাপে replace — মাঝপথে crash হলেও ভাঙে না)
```

### AOF এর Pros এবং Cons

**Pros:**
- Data loss অনেক কম (everysec এ maximum 1 second)
- Human readable log file — manual recovery possible (ভুলে কিছু delete করলে AOF edit করে recover করা যায়)
- Corrupted হলে `redis-check-aof` দিয়ে repair করা যায়

**Cons:**
- File size অনেক বড় হয়
- Recovery সময় বেশি লাগে (সব commands replay করতে হয়)
- fsync এর কারণে performance কিছুটা কম

---

## RDB + AOF একসাথে (Recommended for Production)

Redis 2.4+ থেকে দুটো একসাথে use করা যায়। Restart এ AOF use হয় (বেশি complete data আছে বলে)।

```conf
save 3600 1
save 300 100
save 60 10000

appendonly yes
appendfsync everysec
```

### Redis 7.0 — AOF with Multiple Files

Redis 7.0 তে নতুন AOF format:
- **Base file** — RDB format এ current snapshot (fast load)
- **Incremental files** — তারপর থেকে নতুন changes

Restart time কমায় এবং rewrite আরো efficient করে।

---

## No Persistence

Cache only use case এ persistence বন্ধ রাখো:
```conf
save ""
appendonly no
```

---

## Persistence এর সময় Data Integrity Check

```bash
redis-check-rdb dump.rdb         # RDB file validate করো
redis-check-aof appendonly.aof   # AOF file validate করো
redis-check-aof --fix appendonly.aof  # error থাকলে fix করার চেষ্টা করো
```

---

<a name="layer-4"></a>
# Layer 4 — Replication

Replication মানে একটা Redis server এর data অন্য Redis server এ automatically copy হওয়া।

**কেন Replication দরকার?**
- **High Availability (HA):** Primary server crash হলে backup সার্ভার থেকে serve করা যায়
- **Read Scaling:** Multiple servers থেকে read করা যায়, load distribute হয়
- **Data Safety:** Multiple copies থাকলে data হারানোর risk কমে

Primary server কে বলে **Master (বা Primary)**, copy গুলো কে বলে **Replica (বা Slave)।**

---

## Replication Architecture

```
Master (Read + Write করা যায়)
    ├── Replica 1 (শুধু Read)
    ├── Replica 2 (শুধু Read)
    └── Replica 3 (শুধু Read)
             └── Sub-Replica (Cascading replication — Replica এর Replica)
```

**Asynchronous replication:**
"Asynchronous" মানে Master Replica কে acknowledge (confirm) এর জন্য wait করে না। Master লিখে এগিয়ে যায়, Replica পরে sync করে।

ফলে:
- Master fast থাকে
- কিন্তু Replica কিছুটা behind থাকতে পারে (replication lag — কতটা পিছিয়ে)
- Master crash এ কিছু data Replica তে না পৌঁছাতে পারে

---

## Replication কীভাবে কাজ করে

### Initial Sync (Full Synchronization)

```
Replica → "PSYNC replicationid offset" → Master পাঠায়
# "replicationid" = Master এর unique ID
# "offset" = Replica এখন পর্যন্ত কতটা data পেয়েছে

Master: offset match করে কিনা check করে
Match না করলে (নতুন Replica বা অনেকক্ষণ disconnect ছিল):
      ↓
Master → BGSAVE (RDB snapshot তৈরি করে)
Master: নতুন commands আসলে replication buffer এ জমা রাখে
RDB ready হলে:
      ↓
Master → RDB file পাঠায় → Replica
Replica: existing data flush (মুছে) করে RDB load করে
      ↓
Master → buffer এর commands পাঠায় → Replica
      ↓
Steady state: এরপর থেকে প্রতিটা write command realtime Replica তে যায়
```

### Partial Resync

Connection briefly drop হলে full resync দরকার নেই। Master একটা **replication backlog buffer** রাখে (default 1MB circular buffer — নতুন data পুরনোটাকে overwrite করে)। Reconnect এ শুধু missed commands পাঠায়।

```conf
repl-backlog-size 1mb      # বড় করলে partial resync success rate বাড়ে
repl-backlog-ttl 3600      # কতক্ষণ backlog রাখবে যদি কোনো Replica না থাকে
```

---

## Replication Configuration

### Master Configuration

```conf
# redis.conf (master)
bind 0.0.0.0             # সব IP এ listen করো
requirepass masterpassword
```

### Replica Configuration

```conf
# redis.conf (replica)
replicaof 192.168.1.10 6379          # master এর IP এবং port
masterauth masterpassword             # master এ connect করার password

replica-read-only yes                 # replica write করতে পারবে না (recommended)
replica-priority 100                  # Sentinel এ failover priority
# কম মানে বেশি priority (100 = default, 0 = কখনো master হবে না)
```

### Runtime Commands

```bash
# Replica তে
REPLICAOF 192.168.1.10 6379          # master set করো
REPLICAOF NO ONE                      # master থেকে detach করো (standalone হও বা নতুন master হও)

# Master তে
INFO replication                      # replication status দেখো
```

---

## Replication Monitoring

```bash
INFO replication
```

Output এ important fields:
```
role:master
connected_slaves:2                    # কতটা replica connected
slave0:ip=192.168.1.11,port=6379,state=online,offset=12345,lag=0
slave1:ip=192.168.1.12,port=6379,state=online,offset=12340,lag=1
master_replid:abc123...              # master এর unique replication ID
master_repl_offset:12345             # master এখন পর্যন্ত কতটা data পাঠিয়েছে
repl_backlog_size:1048576            # backlog buffer size
```

**lag** — Replica কতটা behind। 0 বা 1 হওয়া উচিত। বেশি হলে network বা performance সমস্যা।

---

## Read Scaling with Replicas

Replica গুলো read traffic handle করতে পারে। Application level এ read queries Replica তে পাঠাও, write queries Master এ।

```bash
# Replica read-only verify করো
redis-cli -h replica1 -p 6379 SET test value
# Error: READONLY You can't write against a read only replica.
```

**সতর্কতা:** Replication lag এর কারণে Replica এ stale (পুরনো) data পড়তে পারো। Consistency (সব জায়গায় same data) critical হলে Master থেকেই read করো।

---

## min-replicas-to-write — Data Safety

Master এ configure করা যায় যে minimum কতটা Replica acknowledge করলে write accept করবে:

```conf
min-replicas-to-write 1             # কমপক্ষে 1 Replica কে write পৌঁছাতে হবে
min-replicas-max-lag 10             # Replica maximum 10 seconds পিছিয়ে থাকতে পারবে
```

এই condition পূরণ না হলে Master write reject করবে। Data loss কমায় কিন্তু availability (সার্ভিস চলমান থাকা) কমায়।

---

<a name="layer-5"></a>
# Layer 5 — Redis Sentinel

Replication setup এ একটা বড় সমস্যা: **Master crash হলে manually failover করতে হয়।**

**"Failover" কী?** Primary system fail করলে backup system এ switch করা। Manually করতে হলে downtime হয়।

Sentinel এই সমস্যা solve করে — automatic failover।

---

## Sentinel কী করে

Sentinel এর তিনটা মূল কাজ:
1. **Monitoring** — Master এবং Replica continuously monitor করে (heartbeat পাঠিয়ে দেখে কে জীবিত)
2. **Notification** — কিছু wrong হলে admin কে notify করে (email, webhook, etc.)
3. **Automatic Failover** — Master down হলে automatically একটা Replica কে promote (উন্নীত) করে নতুন Master বানায়

---

## Sentinel Architecture

```
          Sentinel 1
         /     |     \
        /      |      \
Sentinel 2   Master   Sentinel 3
        \      |      /
         \     |     /
          Replica 1
          Replica 2
```

Sentinel নিজেরাও একটা distributed system (অনেক servers মিলে কাজ করে)। **Quorum** এর concept আছে।

**"Quorum" কী?**
Latin শব্দ, মানে "minimum সংখ্যা।" ভোটিং এর মতো — একটা decision নিতে minimum কতজন agree করতে হবে। এটা নিশ্চিত করে যে একটা ভুল Sentinel পুরো system ভেঙে দিতে পারবে না।

**Minimum 3টা Sentinel** recommend করা হয়:
- 1টা Sentinel: Single point of failure (1টা fail করলে system কাজ করে না)
- 2টা: Split-brain possible — দুটো আলাদা decision নিতে পারে, কেউ majority পায় না
- 3টা: Quorum (2/3 agree) কাজ করে, 1টা fail হলেও system চলে

**"Split-brain" কী?** Network partition এ দুটো part আলাদা হয়ে যায় এবং দুটোই নিজেকে "সঠিক" মনে করে। যেমন দুটো Sentinel আলাদাভাবে দুটো Replica কে Master promote করে দিলে দুটো Master হয়ে যাবে — data inconsistency।

---

## Sentinel Configuration

```conf
# sentinel.conf

port 26379
daemonize yes
logfile /var/log/redis/sentinel.log

# Sentinel টা কোন Master monitor করবে
sentinel monitor mymaster 192.168.1.10 6379 2
# mymaster = এই master এর নাম (আমরা দিই)
# 192.168.1.10 = master IP
# 6379 = port
# 2 = quorum (failover এর জন্য কতটা Sentinel agree করতে হবে)

sentinel auth-pass mymaster masterpassword

# Master কতক্ষণ unreachable থাকলে down বলবে (milliseconds)
sentinel down-after-milliseconds mymaster 5000

# Failover কতক্ষণের মধ্যে complete করতে হবে (milliseconds)
sentinel failover-timeout mymaster 60000

# Failover এর পর একসাথে কতটা Replica নতুন Master থেকে sync করবে
# 1 মানে একটা একটা করে — নতুন Master এর load কম থাকে
sentinel parallel-syncs mymaster 1
```

---

## Failover Process — Step by Step

```
1. Sentinel গুলো প্রতি second এ Master কে PING করে
2. Master respond না করলে প্রতিটা Sentinel নিজে নিজে
   "SDOWN" (Subjective Down = আমার কাছ থেকে down দেখাচ্ছে) mark করে
3. Quorum সংখ্যক Sentinel SDOWN করলে "ODOWN" (Objective Down = সত্যিই down) হয়
4. Sentinel গুলো নিজেদের মধ্যে vote করে একজন Leader Sentinel select করে
5. Leader Sentinel failover শুরু করে:
   a. Best Replica select করে (replication lag কম, priority বেশি এমন)
   b. সেই Replica তে REPLICAOF NO ONE পাঠায় (সে এখন Master)
   c. অন্য Replica গুলোকে নতুন Master এ point করে
   d. পুরনো Master (যদি ফিরে আসে) কে Replica হিসেবে configure করে
6. Clients কে নতুন Master এর address জানায়
```

---

## Client Connection with Sentinel

Client directly Master এ connect করে না। Sentinel কে জিজ্ঞেস করে Master কোথায়:

```bash
# Current master কে খোঁজো
redis-cli -p 26379 SENTINEL get-master-addr-by-name mymaster
# Output: 192.168.1.10, 6379
```

Application code এ Sentinel-aware client library use করতে হয়। এটা automatically নতুন Master খুঁজে নেয় failover এর পরে।

---

## Sentinel Commands

```bash
# Sentinel info
SENTINEL masters                              # সব monitored masters এর info
SENTINEL master mymaster                      # specific master এর info
SENTINEL replicas mymaster                    # master এর replicas এর info
SENTINEL sentinels mymaster                   # অন্য Sentinels এর info

# Manual failover (planned maintenance এর জন্য)
SENTINEL failover mymaster

# Runtime config change (restart ছাড়াই)
SENTINEL monitor newmaster 192.168.1.20 6379 2
SENTINEL remove mymaster
SENTINEL set mymaster down-after-milliseconds 3000
```

---

## Sentinel এর Limitations

- **Split-brain possibility** — network partition এ দুটো Master হতে পারে
- **Client complexity** — Sentinel-aware client দরকার
- **Vertical scaling only** — একটাই Master, write scaling নেই (বেশি write করতে পারো না)
- **Large dataset** — পুরো dataset একটাই server এ থাকতে হয়

এই limitations এর জন্য Redis Cluster আসে।

---

<a name="layer-6"></a>
# Layer 6 — Redis Cluster

Redis Cluster হলো Redis এর **horizontal scaling solution।**

**"Horizontal scaling" কী?**
**Vertical scaling** = একটা server কে বড় করা (বেশি RAM, CPU)। এর limit আছে।
**Horizontal scaling** = বেশি servers যোগ করা। Theoretically infinite।

Cluster এ data automatically multiple nodes এ distribute হয়। কোনো node fail হলে automatically handle করে।

---

## Cluster Architecture

```
Node 1 (Master)          Node 2 (Master)          Node 3 (Master)
Slots: 0-5460            Slots: 5461-10922         Slots: 10923-16383
    |                        |                          |
Replica 1A               Replica 2A                Replica 3A
(Node 1 এর backup)      (Node 2 এর backup)        (Node 3 এর backup)
```

### Hash Slots — Data Distribution এর Core Concept

Redis Cluster এ **16384টা hash slot** আছে (16384 = 2^14, convenient number)।

প্রতিটা key একটা slot এ map হয়:
```
slot = CRC16(key) % 16384
```

**"CRC16" কী?** Cyclic Redundancy Check — একটা hashing algorithm। key এর উপর এই function চালালে একটা number বের হয়।

**"%" কী?** Modulo operation — ভাগ করে remainder নাও। 17 % 5 = 2।

এই slots গুলো nodes এর মধ্যে distribute হয়। 3 master node এ:
- Node 1: slots 0-5460
- Node 2: slots 5461-10922
- Node 3: slots 10923-16383

কোনো key SET করলে Redis সেটার slot calculate করে, সেই slot কোন node এ আছে সেখানে redirect করে।

### Hash Tags — একই Node এ রাখা

কখনো কখনো কিছু keys একই node এ রাখা দরকার (multi-key operations এর জন্য — একসাথে কাজ করে এমন)। Key name এ `{}` দিয়ে hash tag specify করা যায়। শুধু `{}` এর ভেতরের অংশ দিয়ে slot calculate হয়।

```bash
SET {user:1001}.name "Rahim"
SET {user:1001}.age 28
# দুটোই একই slot এ যাবে কারণ "user:1001" same → same CRC16 → same slot
```

---

## Cluster Setup

Minimum **6 nodes** দরকার production এ — 3 Master + 3 Replica।

**কেন minimum 3 Master?** যেকোনো 1টা Master fail হলেও cluster চলবে (2টা majority)। 2 Masters এ 1টা fail করলে quorum পাওয়া যায় না।

### Configuration (প্রতিটা node এ)

```conf
port 7001
cluster-enabled yes
cluster-config-file nodes.conf        # cluster state file — Redis নিজেই manage করে, edit করো না
cluster-node-timeout 5000             # node কতক্ষণ unreachable থাকলে down বলবে (ms)
appendonly yes
```

### Cluster Create

```bash
# 6টা instance চালু করার পর
redis-cli --cluster create \
  192.168.1.10:7001 \
  192.168.1.10:7002 \
  192.168.1.10:7003 \
  192.168.1.11:7001 \
  192.168.1.11:7002 \
  192.168.1.11:7003 \
  --cluster-replicas 1
# --cluster-replicas 1 মানে প্রতি Master এ 1টা Replica
# Redis নিজেই slots assign করে এবং Master-Replica pair করে
```

---

## Cluster Operation

```bash
# Cluster info দেখো
redis-cli -c -h 192.168.1.10 -p 7001 CLUSTER INFO
redis-cli -c -h 192.168.1.10 -p 7001 CLUSTER NODES

# -c flag — cluster mode এ connect (auto redirect enable করে)
redis-cli -c -h 192.168.1.10 -p 7001
> SET name Rahim
# Automatically correct node এ redirect হবে
```

### MOVED এবং ASK Redirect

Client যদি wrong node এ request করে, Redis redirect করে:

```
Client → Node 1 তে: SET foo bar পাঠায়
Node 1 → "MOVED 7638 192.168.1.11:7001" বলে
(7638 = foo এর slot, ওই slot Node 2 এ)
Client → Node 2 তে: SET foo bar পাঠায়
```

**MOVED** — permanent redirect। Smart client এটা cache করে পরেরবার সরাসরি সঠিক node এ যায়।
**ASK** — temporary redirect। Slot migration (slot এক node থেকে আরেকটায় move) চলছে, এই request এর জন্য শুধু ওখানে যাও।

---

## Cluster Scaling — Resharding

**"Resharding" কী?** Slots এক node থেকে আরেকটায় move করা। নতুন node add হলে এটা দরকার।

```bash
# নতুন node add করো
redis-cli --cluster add-node 192.168.1.12:7001 192.168.1.10:7001

# Slots migrate করো (interactive mode)
redis-cli --cluster reshard 192.168.1.10:7001
# কতটা slots migrate করবে, কোথা থেকে, কোথায় জিজ্ঞেস করবে
```

**Zero downtime resharding:** Slot migration চলাকালীন data live copy হয়। ASK redirect দিয়ে requests serve হয় — users কোনো interruption দেখে না।

---

## Cluster Failover

Master fail হলে Replica automatically promote হয়। Sentinel ছাড়াই cluster নিজেই এটা handle করে।

```bash
# Manual failover (planned maintenance)
redis-cli -c -h replica_host -p 7001 CLUSTER FAILOVER
# Replica নিজেই promote হওয়ার request করে Master এর কাছে (graceful)
```

### Cluster Node Management

```bash
# Node remove করো (আগে সব slots অন্যত্র migrate করতে হবে)
redis-cli --cluster del-node 192.168.1.10:7001 <node-id>

# Replica add করো
redis-cli --cluster add-node 192.168.1.13:7001 192.168.1.10:7001 \
  --cluster-slave --cluster-master-id <master-node-id>

# Cluster inconsistency fix করো
redis-cli --cluster fix 192.168.1.10:7001

# Cluster health check করো
redis-cli --cluster check 192.168.1.10:7001
```

---

## Cluster Limitations

- **Multi-key operations সীমিত** — MSET, MGET, pipeline শুধু same slot এর keys এ কাজ করে
- **Database 0 only** — multiple logical databases নেই
- **Lua scripts** — সব keys same slot এ থাকতে হবে
- **Transactions** — MULTI/EXEC same slot এর keys এ সীমিত

---

## Cluster vs Sentinel — কোনটা কখন

| | Sentinel | Cluster |
|--|---------|---------|
| **উদ্দেশ্য** | High Availability (HA) | HA + Horizontal Scaling |
| **Write scaling** | না (single Master) | হ্যাঁ (multiple Masters) |
| **Data size** | Single server এর মধ্যে | Multiple servers এ distribute |
| **Complexity** | কম (simple setup) | বেশি (16384 slots, routing) |
| **Multi-key ops** | সব করা যায় | Same slot এ সীমিত |
| **কখন use করবে** | Data ছোট, HA দরকার | Data বড় বা write heavy |

---

<a name="layer-7"></a>
# Layer 7 — Performance & Tuning

Redis by default fast, কিন্তু misconfiguration এ performance খারাপ হতে পারে।

---

## Memory Management

### maxmemory এবং Eviction Policies

**"Eviction" কী?**
Memory full হলে নতুন data রাখার জায়গা নেই। তখন পুরনো data কে বের করে (evict করে) জায়গা করতে হয়।

```conf
maxmemory 4gb                      # maximum memory limit
maxmemory-policy allkeys-lru       # limit পূরণ হলে কী করবে
```

**Eviction policies:**

| Policy | কাজ |
|--------|-----|
| `noeviction` | নতুন write reject করবে, error দেবে (default) |
| `allkeys-lru` | সব keys থেকে LRU evict করবে |
| `volatile-lru` | শুধু TTL আছে এমন keys থেকে LRU evict |
| `allkeys-lfu` | সব keys থেকে LFU evict |
| `volatile-lfu` | TTL আছে এমন keys থেকে LFU evict |
| `allkeys-random` | Random evict (যেকোনো key) |
| `volatile-random` | TTL আছে এমন keys থেকে random evict |
| `volatile-ttl` | সবচেয়ে কম TTL আছে এমন key আগে যাবে |

**"LRU" কী? (Least Recently Used)**
সবচেয়ে কম recently (সম্প্রতি) use হওয়া key evict করে। মানে যেটা সবচেয়ে বেশিদিন ধরে touch করা হয়নি সেটা বের করে দাও।

**"LFU" কী? (Least Frequently Used)**
সবচেয়ে কম frequency (কতবার) তে access হওয়া key evict করে। মানে যেটা সবচেয়ে কমবার use হয়েছে সেটা বের করে দাও।

**LRU vs LFU:**
- LRU: একবার access করলেও সেটা "recent" — কিন্তু পরে আর কেউ চাইবে না এমন data থেকে যেতে পারে
- LFU: কতবার access হয়েছে count করে — বেশি popular data থাকে, কম popular যায়
- LFU generally better কারণ truly hot data retain করে

**Cache use case এ:** `allkeys-lru` বা `allkeys-lfu`
**Mixed use case এ (cache + persistent data):** `volatile-lru`
**Primary database এ:** `noeviction`

### Memory Optimization Techniques

**1. Appropriate data structures use করো:**
Hash ব্যবহার করো individual String এর বদলে। 1000টা user field আলাদা String এ রাখার চেয়ে Hash এ রাখলে memory কম লাগে।

**2. Encoding thresholds tune করো:**
```conf
hash-max-listpack-entries 128     # এর নিচে থাকলে listpack (compact format) use হবে
hash-max-listpack-value 64        # প্রতিটা value এর maximum size
list-max-listpack-size -2         # -2 = 8kb per quicklist node
set-max-intset-entries 512        # integer set এর জন্য compact format threshold
zset-max-listpack-entries 128
zset-max-listpack-value 64
```

ছোট data compact encoding এ থাকে — memory কম। এই thresholds বাড়ালে বেশি data compact থাকবে — memory কমবে কিন্তু CPU বাড়বে (linear scan করতে হয়)।

**3. Key expiry set করো:**
অপ্রয়োজনীয় data জমা না হওয়ার জন্য TTL ব্যবহার করো।

**4. Active defragmentation:**
```conf
activedefrag yes
active-defrag-ignore-bytes 100mb    # 100mb fragmentation হলে শুরু করবে
active-defrag-enabled yes
```

---

## Slow Log — Slow Queries ধরা

**"Slow log" কী?** যেসব commands নির্দিষ্ট সময়ের বেশি নিচ্ছে সেগুলো automatically log হয়।

```conf
slowlog-log-slower-than 10000    # 10ms (10000 microseconds) এর বেশি সময় নেওয়া commands log করবে
slowlog-max-len 128              # maximum 128টা log entry রাখবে
```

```bash
SLOWLOG GET 10           # last 10টা slow queries দেখো
SLOWLOG LEN              # কতটা log আছে
SLOWLOG RESET            # log clear করো
```

Output এ:
- Execution time (microseconds)
- Command
- Timestamp
- Client info

---

## Dangerous Commands — Production এ Block করো

```conf
rename-command KEYS ""              # KEYS disable করো (empty string = disabled)
rename-command FLUSHALL ""          # FLUSHALL disable করো (সব data delete)
rename-command FLUSHDB ""           # FLUSHDB disable করো (current DB delete)
rename-command DEBUG ""             # DEBUG disable করো
rename-command CONFIG "ADMIN-CONFIG" # rename করো — শুধু যারা নাম জানে তারা use করতে পারবে
```

**KEYS কেন dangerous:** O(n) operation — সব keys scan করে। 1 million keys থাকলে 1 million step। এই সময় Redis block থাকে, কোনো request handle করতে পারে না।

### SCAN — Safe Alternative to KEYS

```bash
# KEYS * এর বদলে SCAN use করো
SCAN 0 MATCH "user:*" COUNT 100
# 0 = cursor শুরুর position
# COUNT 100 = প্রতি iteration এ approximate 100টা keys scan করো
# Return: [new_cursor, [keys...]]
# new_cursor = 0 হলে scan complete
```

**"Cursor" কী?** একটা pointer যেটা বলে "এখন পর্যন্ত কোথায় পৌঁছেছি।" SCAN এ cursor দিয়ে বলো কোথা থেকে continue করবে।

SCAN non-blocking কারণ incremental করে কাজ করে। প্রতি call এ কিছু keys return করে, cursor দিয়ে পরের batch এ যায়।

---

## Connection Pool

**"Connection pool" কী?**
TCP connection establish করতে time লাগে (handshake, authentication)। প্রতি request এ নতুন connection মানে এই overhead বারবার। Connection pool এ কিছু connections তৈরি রাখা হয়। Request এলে pool থেকে একটা নেওয়া হয়, কাজ শেষে return করা হয়। পুনরায় ব্যবহারযোগ্য।

```conf
maxclients 10000         # maximum concurrent connections
tcp-keepalive 300        # idle connection 300 seconds পর close করো
timeout 0                # idle client disconnect timeout (0 = never disconnect)
```

---

## Pipeline — Batch Commands

**"Pipeline" কী?**
Network এ request-response round trip অনেক সময় নেয়। Pipeline এ multiple commands একসাথে পাঠানো যায়। Server সব execute করে সব responses একসাথে পাঠায়।

```bash
# Without pipeline — 3টা round trips (3x network delay)
SET key1 val1
SET key2 val2
SET key3 val3

# With pipeline — 1টা round trip
(pipeline start)
SET key1 val1
SET key2 val2
SET key3 val3
(execute — সব একসাথে পাঠাও)
```

Large bulk operations এ 10x+ performance gain।

---

## Lua Scripting — Atomic Operations

**"Lua" কী?** একটা lightweight scripting language। Redis এ Lua scripts চালানো যায়।

Multiple commands কে atomic করতে Lua script use করো:

```bash
EVAL "
  local current = redis.call('GET', KEYS[1])
  if current == false then
    redis.call('SET', KEYS[1], ARGV[1])
    return 1
  end
  return 0
" 1 mykey myvalue
# 1 = কতটা KEYS argument
# mykey = KEYS[1]
# myvalue = ARGV[1]
```

**EVALSHA** — script hash দিয়ে call করো:
```bash
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# SHA1 hash return করে
# "SHA1" = একটা hashing algorithm, fixed length output দেয়
EVALSHA <sha1> 1 mykey
# Script বারবার পাঠাতে হয় না — শুধু hash পাঠাও
```

---

## Latency Monitoring

**"Latency" কী?** Request পাঠানো থেকে response পাওয়া পর্যন্ত সময়। কম latency = fast।

```bash
redis-cli --latency -h 127.0.0.1 -p 6379    # real-time latency দেখো
redis-cli --latency-history                   # historical latency data
redis-cli --latency-dist                      # latency distribution

# Latency monitoring
CONFIG SET latency-monitor-threshold 100      # 100ms এর বেশি হলে log করো
LATENCY LATEST                                 # latest latency events
LATENCY HISTORY event                         # specific event এর history
LATENCY RESET                                 # clear করো
```

---

## OS Level Tuning

Redis এর performance শুধু Redis config এ না, OS level এও tuning দরকার।

```bash
# Transparent Huge Pages (THP) disable করো
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

**"Transparent Huge Pages (THP)" কী?**
সাধারণত OS memory কে 4KB page এ ভাগ করে। THP এ OS automatically 2MB বা 1GB এর বড় pages use করার চেষ্টা করে।

**কেন disable করতে হয়?**
Redis fork() করে snapshot নেয়। CoW এর সময় 4KB এর ছোট page modify হলে শুধু সেই 4KB copy হয়। কিন্তু 2MB এর huge page modify হলে পুরো 2MB copy হয় — অনেক বেশি memory এবং latency spike।

```bash
# Overcommit memory enable করো
echo 1 > /proc/sys/vm/overcommit_memory
# অথবা /etc/sysctl.conf এ:
vm.overcommit_memory = 1
```

**"Overcommit memory" কী?**
Linux এ process memory request করলে OS actually সব memory দিতে না পারলেও "হ্যাঁ" বলে। Actual use এর সময় দেয়। Redis fork() করার সময় theoretically double memory দরকার (CoW আগে)। Overcommit ছাড়া Linux এটা deny করতে পারে। `vm.overcommit_memory = 1` মানে সবসময় allocate করে দাও।

```bash
# Somaxconn — TCP connection queue size
echo 511 > /proc/sys/net/core/somaxconn
net.core.somaxconn = 511
```

**"somaxconn" কী?** TCP connection queue — server কতটা pending connection একসাথে রাখতে পারবে।

```conf
# Redis config এ
tcp-backlog 511
```

---

## Redis INFO — Health Monitoring

```bash
INFO                      # সব sections একসাথে
INFO server               # Redis version, OS, config file info
INFO clients              # connected clients, blocked clients count
INFO memory               # memory usage, fragmentation ratio
INFO stats                # ops/sec (প্রতি second এ কতটা operations), hits, misses
INFO replication          # replication status
INFO cpu                  # CPU usage
INFO keyspace             # database এর key count, expiry info
```

Important metrics:
```
used_memory_human          # actual data রাখতে কতটা memory
used_memory_rss_human      # OS এর কাছ থেকে allocated total memory
# "RSS" = Resident Set Size — OS এর দৃষ্টিতে process এর memory usage
mem_fragmentation_ratio    # 1.0-1.5 healthy
connected_clients          # current connection count
instantaneous_ops_per_sec  # এই মুহূর্তে ops/sec
keyspace_hits              # cache hit — Redis এ data পাওয়া গেছে
keyspace_misses            # cache miss — Redis এ data ছিল না
# hit ratio = hits / (hits + misses) — বেশি ভালো
```

---

## Benchmark

```bash
# Built-in benchmark tool
redis-benchmark -h 127.0.0.1 -p 6379 -n 100000 -c 50
# -n: total requests count
# -c: concurrent clients count

# Specific commands test
redis-benchmark -t set,get -n 100000

# Pipeline test
redis-benchmark -t set -n 100000 -P 16
# -P: pipeline size (16 commands একসাথে)
```

---

<a name="layer-8"></a>
# Layer 8 — Real Patterns

Redis এর theory জানার পর এখন দেখবো production এ কোথায় কীভাবে use হয়।

---

## 1. Caching Patterns

### Cache-Aside (Lazy Loading) — সবচেয়ে common

"Lazy" মানে — আগে থেকে সব cache করি না, দরকার হলে তখন cache এ রাখি।

```
Application: Redis এ data আছে? (cache check)
  হ্যাঁ (cache hit) → Redis থেকে return করো (fast)
  না (cache miss) → Database থেকে নিয়ে Redis এ set করো → return করো
```

**Pros:** শুধু read হওয়া data cache হয়। Redis restart এ gradual warmup (ধীরে ধীরে cache ভরে)।
**Cons:** প্রথম request এ cache miss — database hit।

### Write-Through

Write এর সময়ই cache update করো।

```
Application: data write করবে
     ↓
প্রথমে Redis এ লেখো
     ↓
তারপর Database এ লেখো
     ↓
দুটোই consistent থাকে
```

**Pros:** Cache always fresh (stale data নেই)।
**Cons:** Write slow (দুই জায়গায় লিখতে হয়), rarely read হওয়া data ও cache এ জমে।

### Write-Behind (Write-Back)

Cache এ লিখে immediately return করো, database এ পরে (batch এ) লেখো।

```
Application → Redis এ লেখো → return করো (fast)
Background process → Redis থেকে batch এ Database এ লেখো
```

**Pros:** Write অনেক fast।
**Cons:** Redis crash হলে database এ না যাওয়া data হারিয়ে যাবে।

### Cache Invalidation

Data update হলে cache invalid (পুরনো) করতে হয়:
```bash
# Option 1: Delete করো (next read এ fresh data আসবে)
DEL user:1001:profile

# Option 2: Direct update করো
HSET user:1001:profile name "New Name"

# Option 3: TTL based (expire হলে নিজেই fresh হবে)
EXPIRE user:1001:profile 300
```

**"Cache invalidation" কেন কঠিন?** কোন কোন cache update করতে হবে জানা কঠিন। Complex dependency থাকলে আরো কঠিন। CS এর famous quote: "There are only two hard things in Computer Science: cache invalidation and naming things."

---

## 2. Session Management

**"Session" কী?** Web এ HTTP stateless — প্রতিটা request আলাদা। Login করার পরে তুমি কে সেটা server কে মনে রাখতে হয় — এই "মনে রাখা" টাই session।

```bash
# Login হলে session তৈরি করো
SET session:TOKEN_HERE '{"user_id":1001,"role":"admin"}' EX 3600
# TOKEN_HERE = random unique string (token)

# Request validation — token দিয়ে user কে identify করো
GET session:TOKEN_HERE
# null হলে expired বা invalid

# Session refresh (user active থাকলে TTL extend করো)
EXPIRE session:TOKEN_HERE 3600

# Logout — session delete করো
DEL session:TOKEN_HERE
```

---

## 3. Rate Limiting

**"Rate limiting" কী?** কোনো user বা IP কে নির্দিষ্ট সময়ে নির্দিষ্টের বেশি request করতে না দেওয়া। DDoS attack, API abuse থেকে protect করে।

### Fixed Window

```bash
INCR rate:user:1001:minute:1711123200  # key তে timestamp আছে (প্রতি minute এ নতুন key)
EXPIRE rate:user:1001:minute:1711123200 60  # 60 seconds পর delete
# Value > limit হলে block করো
```

**সমস্যা: Window boundary তে burst।** Window শেষে 100 request + পরের window শুরুতে 100 request = 200 requests in 2 seconds।

### Sliding Window (Sorted Set দিয়ে)

Fixed window এর burst সমস্যা নেই।

```bash
# Current timestamp
now = 1711123500
window = 60  # 60 seconds window

# এই user এর request log এ current timestamp add করো
ZADD rate:user:1001 now now   # score=timestamp, member=timestamp

# 60 seconds এর বেশি পুরনো entries remove করো (expired)
ZREMRANGEBYSCORE rate:user:1001 0 (now - window)

# Last 60 seconds এ কতটা request হয়েছে count করো
count = ZCARD rate:user:1001
# count > limit হলে block করো

# Auto-expire করো
EXPIRE rate:user:1001 window
```

---

## 4. Distributed Lock

**"Distributed lock" কী?**
Multiple servers (distributed system) এ একটা shared resource এ একসাথে access করলে race condition হয়। Lock দিয়ে বলা হয় "আমি এটা use করছি, অন্যরা wait করো।"

Database এর row lock এর মতো, কিন্তু multiple servers এর জন্য।

### Simple Lock

```bash
# Lock acquire (নেওয়া)
SET lock:resource1 "server1_unique_id" NX EX 30
# NX — না থাকলেই set (atomic check-and-set)
# EX 30 — 30 seconds এ auto-expire (deadlock prevent করতে)
# "server1_unique_id" = যে server lock নিয়েছে তার unique identifier

# Lock release (ছাড়া) — Lua script দিয়ে atomic check-and-delete
EVAL "
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  else
    return 0
  end
" 1 lock:resource1 "server1_unique_id"
```

**কেন unique_id দরকার?**
Server A lock নেয়। Server A slow হয়, lock expire হয়। Server B lock নেয়। Server A আবার চালু হয়ে নিজের lock ছেড়ে দিতে চায় — কিন্তু এখন lock Server B এর। unique_id check করলে Server A বুঝবে এটা তার lock না, release করবে না।

**কেন Lua দিয়ে release?**
GET করে check, তারপর DEL করা atomic না। GET এর পরে DEL এর আগে অন্য server lock নিতে পারে। Lua script Redis এ atomically execute হয়।

**"Deadlock" কী?** Server A lock নিলো এবং crash করলো — কখনো release করবে না। অন্য servers forever wait করবে। EX (expiry) দিলে auto-release হয়।

### Redlock — Multi-Node Distributed Lock

Single node lock reliable না (node fail হলে lock হারায়)। **Redlock algorithm** N nodes এ majority (N/2 + 1) lock acquire করে:

```
N = 5 nodes
Lock acquire করার চেষ্টা করো সব 5টায়
3টায় (majority) success হলে lock acquired
1-2টায় fail হলেও চলে
```

Production এ Redlock library use করো।

---

## 5. Pub/Sub — Messaging

**"Pub/Sub" কী?** Publisher-Subscriber pattern।
- **Publisher** = message পাঠায় (broadcast)
- **Subscriber** = message receive করে
- **Channel** = topic বা category

```bash
# Subscriber (আগে subscribe করতে হবে)
SUBSCRIBE news:tech news:sports
# এখন block হয়ে wait করবে

# Publisher (message পাঠাও)
PUBLISH news:tech "Redis 8.0 released!"
# যেকোনো subscriber যারা news:tech subscribe করেছে তারা পাবে

# Pattern subscribe (wildcard)
PSUBSCRIBE news:*    # "news:" দিয়ে শুরু সব channel এর message পাবে
```

**Pub/Sub এর limitations:**
- **No persistence** — subscriber offline থাকলে message হারিয়ে যায়
- **No acknowledgement** — delivery guarantee নেই
- Persistent messaging দরকার হলে **Stream** use করো

---

## 6. Leaderboard

Sorted Set দিয়ে real-time leaderboard:

```bash
# Score update করো (add বা update)
ZADD game:leaderboard 1500 "user:1001"
ZINCRBY game:leaderboard 50 "user:1001"      # 50 points add করো

# Top 10 দেখাও
ZREVRANGE game:leaderboard 0 9 WITHSCORES    # highest score থেকে

# User এর rank দেখাও (0-indexed, তাই +1 করলে 1-indexed হয়)
ZREVRANK game:leaderboard "user:1001"

# User এর score দেখাও
ZSCORE game:leaderboard "user:1001"

# User এর আশেপাশে কারা আছে (context এর জন্য)
rank = ZREVRANK leaderboard "user:1001"
ZREVRANGE leaderboard (rank-2) (rank+2) WITHSCORES
```

---

## 7. Autocomplete

Search এ user type করতে থাকলে suggestions দেওয়া:

```bash
# Terms index এ add করো (score 0, alphabetical order এ sort হবে)
ZADD autocomplete 0 "redis"
ZADD autocomplete 0 "redis cluster"
ZADD autocomplete 0 "redis sentinel"
ZADD autocomplete 0 "replication"

# "red" দিয়ে শুরু terms খোঁজো
ZRANGEBYLEX autocomplete "[red" "[red\xff"
# "[red" = "red" থেকে শুরু (inclusive)
# "[red\xff" = "red" + highest possible character — মানে "red" দিয়ে শুরু সব terms
# "\xff" = hex FF = 255 = ASCII এর highest value
```

---

## 8. Geospatial

**"Geospatial" কী?** Geographic location (latitude/longitude) based data।

Redis 3.2 থেকে geospatial support (internally Sorted Set use করে, score = geohash):

```bash
# Location add করো
GEOADD courier:locations 90.4125 23.8103 "courier:1001"
# longitude latitude name (longitude আগে — Redis এর convention)

# Distance calculate করো
GEODIST courier:locations courier:1001 courier:1002 km

# নির্দিষ্ট point এর কাছের couriers খোঁজো
GEOSEARCH courier:locations FROMLONLAT 90.4125 23.8103 BYRADIUS 5 km ASC COUNT 10
# ASC = nearest first, COUNT 10 = maximum 10 results

# Coordinates পাওয়া
GEOPOS courier:locations courier:1001
```

---

## 9. Redis Streams — Production Event Sourcing

**"Event sourcing" কী?** System এর state change গুলো events হিসেবে store করা। "Order created", "Payment received", "Order shipped" — এই events থেকে current state বের করা যায়।

**"Consumer Group" কী?** Multiple consumers মিলে একটা stream process করে। প্রতিটা message শুধু একটা consumer পায় — work distribute হয়।

```bash
# Event produce করো (* = auto ID)
XADD orders * order_id AWB123 status "created" amount 500
# ID format: timestamp-sequence (e.g., 1711123200000-0)

# Consumer Group তৈরি করো
XGROUP CREATE orders order-processor $ MKSTREAM
# $ = এখন থেকে নতুন messages শুনবো
# MKSTREAM = stream না থাকলে তৈরি করো

# Consumer হিসেবে messages নাও
XREADGROUP GROUP order-processor worker1 COUNT 10 BLOCK 2000 STREAMS orders >
# > = শুধু undelivered (নতুন) messages দাও
# BLOCK 2000 = 2 seconds wait করো নতুন message এর জন্য

# Processing complete হলে acknowledge করো
XACK orders order-processor <message-id>

# Pending (acknowledge হয়নি) messages দেখো
XPENDING orders order-processor - + 10

# অনেকক্ষণ pending থাকা messages অন্য consumer এ দাও
XAUTOCLAIM orders order-processor recovery-worker 3600000 0 COUNT 10
# 3600000ms (1 hour) এর বেশি pending এমন messages claim করো
```

---

# Quick Reference Summary

## Data Structure Selection Guide

| Use case | Data Structure | কেন |
|----------|---------------|-----|
| Simple value, counter, flag | String | Simplest, fast |
| Object/Entity attributes | Hash | Partial update, field-level access |
| Ordered list, queue, stack | List | Push/pop both ends |
| Unique items, tags, membership | Set | Automatic dedup, set operations |
| Ranking, score-based, sorted | Sorted Set | Score + range query |
| Large boolean arrays | Bitmap | Extremely memory efficient |
| Unique count (approximate) | HyperLogLog | Fixed 12KB memory |
| Event log, message queue | Stream | Persistence, consumer groups |
| Location-based | Geo | Built-in radius search |

## Architecture Selection Guide

| Need | Solution | কেন |
|------|---------|-----|
| Single server, no HA | Standalone | Simple |
| HA, small-medium data | Sentinel (min 3 Sentinels + 1 Master + 2 Replicas) | Auto failover |
| Large data, horizontal scale | Cluster (min 6 nodes) | Sharding |

## Common Configuration Checklist

```conf
# Security
requirepass strongpassword
rename-command KEYS ""          # Block dangerous commands
rename-command FLUSHALL ""
bind 127.0.0.1                  # Specific IP only

# Memory
maxmemory 4gb
maxmemory-policy allkeys-lru    # Cache use case

# Persistence (balanced approach)
save 3600 1
save 300 100
appendonly yes
appendfsync everysec            # Max 1 second data loss

# Performance
tcp-keepalive 300
tcp-backlog 511

# Monitoring
slowlog-log-slower-than 10000   # 10ms threshold
slowlog-max-len 128
```

## OS Tuning Checklist

```bash
# /etc/sysctl.conf
vm.overcommit_memory = 1        # Fork এর জন্য
net.core.somaxconn = 511        # TCP queue size

# Transparent Huge Pages — MUST disable
echo never > /sys/kernel/mm/transparent_hugepage/enabled
# /etc/rc.local এ add করো যাতে reboot এও থাকে
```

---

*Redis Complete Study Guide (Revised) | সব technical terms in-place explained*
*E-MultiSkills PostgreSQL Administration Course Companion*
