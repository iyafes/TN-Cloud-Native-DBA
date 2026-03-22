# Redis Complete Study Guide
### Layer 1 → Layer 8 | Architecture · Internals · Cluster · Tuning · Patterns

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

Redis হলো একটা **in-memory data structure store**। এটা তিনটা role এ কাজ করতে পারে:
- **Database** — primary data store হিসেবে
- **Cache** — frequently accessed data দ্রুত serve করতে
- **Message Broker** — systems এর মধ্যে message pass করতে

PostgreSQL বা MySQL এ data থাকে disk এ। Read করতে হলে disk থেকে আনতে হয়। Disk I/O এর কারণে latency milliseconds এ। Redis এ data থাকে RAM এ। RAM এর speed disk এর চেয়ে **100-1000x বেশি**। তাই Redis এর response time microseconds এ।

**Tradeoff:** RAM expensive এবং limited। Server restart হলে data হারিয়ে যেতে পারে (এটা configure করা যায়, Layer 3 এ দেখবো)।

---

## Single-Threaded Event Loop

Redis **single-threaded**। একটাই thread সব command execute করে।

এটা শুনে অনেকে ভাবে slow হওয়ার কথা। কিন্তু Redis প্রতি second এ **millions of operations** handle করে। কারণটা হলো Redis এর কাজই RAM এ read/write। RAM operation এতটাই fast যে CPU কখনো bottleneck হয় না। Bottleneck হয় network বা disk এ।

Single thread এর সুবিধা:
- কোনো locking লাগে না
- কোনো context switching নেই
- কোনো race condition নেই
- এই overhead গুলো না থাকায় Redis অনেক বেশি efficient

**Redis 6.0 থেকে পরিবর্তন:** I/O threads add হয়েছে। Network read/write এর জন্য multiple thread use করে, কিন্তু actual command execution এখনো single thread এ হয়। এই architecture কে বলে **I/O threading with single-threaded command processing।**

---

## Event Loop — I/O Multiplexing

Redis এর heart হলো এর **event loop**। এটা Linux এর `epoll` mechanism use করে (macOS এ `kqueue`, older systems এ `select`)।

ধরো 10,000 client connected আছে Redis এ। প্রতিটার জন্য আলাদা thread রাখলে 10,000 thread লাগতো — nightmare। Redis বরং একটাই thread দিয়ে সব connection monitor করে।

Flow:
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

এই cycle এতটাই দ্রুত যে practically সব client simultaneously serve হচ্ছে মনে হয়।

---

## Redis Object (robj) — Memory তে Data কীভাবে থাকে

Redis এ সব কিছু internally **Redis Object (robj)** হিসেবে represent হয়:

```c
typedef struct redisObject {
    unsigned type;      // STRING, LIST, SET, HASH, ZSET
    unsigned encoding;  // actual internal representation
    unsigned lru;       // last access time (LRU eviction এর জন্য)
    int refcount;       // reference counting for memory management
    void *ptr;          // pointer to actual data
} robj;
```

সবচেয়ে important হলো `encoding` field। একই data type এর জন্য Redis **পরিস্থিতি অনুযায়ী** আলাদা internal representation use করে memory বাঁচাতে। এটাকে বলে **encoding optimization।**

উদাহরণ: String value যদি integer এবং ছোট হয় (0-9999 range), Redis সেটাকে actual string হিসেবে store না করে directly integer হিসেবে রাখে। Memory অনেক কম লাগে।

---

## Memory Management — jemalloc

Redis নিজে memory manage করে `jemalloc` দিয়ে (default on Linux)। System এর default `malloc` এর চেয়ে বেশি efficient এবং memory fragmentation কম।

**Memory fragmentation কী?**
ধরো Redis কে 1GB memory দিলে, কিন্তু `INFO memory` বলছে সে 1.3GB use করছে। এই extra 300MB হলো fragmentation — allocate হয়েছে কিন্তু actually use হচ্ছে না।

`mem_fragmentation_ratio` দিয়ে এটা measure হয়:
- **1.0 - 1.5** → Healthy
- **1.5 এর বেশি** → Fragmentation সমস্যা, restart বা active defrag দরকার
- **1.0 এর কম** → Redis swap use করছে, মারাত্মক সমস্যা

Redis 4.0 থেকে **active defragmentation** আছে যেটা runtime এ fragmentation কমাতে পারে।

---

## RESP Protocol — Client-Server Communication

Redis client আর server এর মধ্যে communication হয় **RESP (REdis Serialization Protocol)** দিয়ে। Text-based এবং human readable।

`SET name Rahim` wire এ যায় এভাবে:
```
*3\r\n          ← 3টা element আছে এই command এ
$3\r\n          ← প্রথম element 3 bytes
SET\r\n
$4\r\n          ← দ্বিতীয় element 4 bytes
name\r\n
$5\r\n          ← তৃতীয় element 5 bytes
Rahim\r\n
```

Server response:
```
+OK\r\n         ← Simple string response
```

Error response:
```
-ERR wrong number of arguments\r\n
```

Integer response:
```
:42\r\n
```

**RESP3** (Redis 6.0+) — নতুন version। আরো rich data types support করে, better client-side caching enable করে।

---

## Command Execution Flow — শুরু থেকে শেষ পর্যন্ত

```
Client → TCP connection establish
      ↓
Redis এ connection register হয় (epoll এ)
      ↓
Client "SET name Rahim" পাঠায়
      ↓
Network buffer এ bytes আসে
      ↓
RESP parser bytes কে command এ convert করে
      ↓
Command lookup table এ "SET" খোঁজে
      ↓
SET handler call হয়
      ↓
Key-value RAM এ write হয়
      ↓
AOF/RDB এ write (persistence configure থাকলে)
      ↓
Replication buffer এ command যায় (replica থাকলে)
      ↓
"+OK\r\n" client কে পাঠায়
```

পুরো এই flow microseconds এ শেষ হয়।

---

## Redis Database (keyspace)

একটা Redis instance এ by default **16টা logical database** থাকে (index 0-15)। এগুলো আলাদা keyspace। এক database এর key অন্য database এ দেখা যায় না।

```bash
SELECT 0    # database 0 এ যাও (default)
SELECT 1    # database 1 এ যাও
```

**Production এ এটা avoid করা উচিত।** আলাদা database দরকার হলে আলাদা Redis instance ব্যবহার করো। কারণ:
- Cluster mode এ শুধু database 0 support করে
- KEYS, FLUSHDB এরকম commands একটা database এ সীমাবদ্ধ, confusing হয়
- আলাদা instance এ আলাদা memory limit, eviction policy set করা যায়

---

## Redis Configuration File — redis.conf

Redis start করার সময় config file দেওয়া যায়:
```bash
redis-server /etc/redis/redis.conf
```

Runtime এ config change করা যায়:
```bash
CONFIG SET maxmemory 2gb
CONFIG GET maxmemory
CONFIG REWRITE    # running config কে redis.conf এ save করে
```

Important initial config:
```conf
bind 127.0.0.1              # কোন IP এ listen করবে
port 6379                   # default port
daemonize yes               # background এ run করবে
loglevel notice             # debug, verbose, notice, warning
logfile /var/log/redis/redis.log
databases 16                # logical database এর সংখ্যা
requirepass yourpassword    # authentication password
maxmemory 2gb               # maximum memory limit
```

---

<a name="layer-2"></a>
# Layer 2 — Data Structures: Deep Internals

Redis এর data structures শুধু "কী কী আছে" জানলে হবে না। প্রতিটা structure ভেতরে কীভাবে implement হয়েছে, কোন situation এ কোনটা use করবে, performance implications — এটা বুঝতে হবে।

---

## 1. String

Redis এর সবচেয়ে basic type। ভেতরে C এর সাধারণ string না — Redis নিজস্ব implementation use করে যার নাম **SDS (Simple Dynamic String)।**

### SDS কেন?

C string এ সমস্যা:
- Length জানতে O(n) — পুরো string traverse করতে হয়
- Binary safe না — null byte এর মধ্যে data রাখা যায় না

SDS structure:
```c
struct sdshdr {
    int len;      // বর্তমান length — O(1) access
    int free;     // allocated কিন্তু unused space
    char buf[];   // actual data
};
```

SDS এর সুবিধা:
- Length O(1) এ জানা যায়
- Binary safe — যেকোনো data রাখা যায়
- Append এ বারবার memory allocate করতে হয় না

### String Internal Encodings

| Encoding | কখন | Memory |
|----------|-----|--------|
| **INT** | value টা integer এবং LONG range এ | সবচেয়ে কম — pointer এর জায়গায় directly integer |
| **EMBSTR** | value ≤ 44 bytes | কম — robj আর SDS একটাই allocation এ |
| **RAW** | value > 44 bytes | বেশি — robj আর SDS আলাদা allocation এ |

```bash
SET counter 100
OBJECT ENCODING counter      # "int"

SET name "Rahim"
OBJECT ENCODING name         # "embstr"

SET essay "This is a very long string that definitely exceeds the forty-four bytes limit in Redis"
OBJECT ENCODING essay        # "raw"
```

### String Commands — সম্পূর্ণ

```bash
# Basic
SET key value
GET key
DEL key
EXISTS key

# Expiry সহ Set
SET key value EX 300          # seconds
SET key value PX 300000       # milliseconds
SET key value EXAT 1735689600 # Unix timestamp
SET key value KEEPTTL         # existing TTL maintain করবে

# Conditional Set
SET key value NX              # শুধু key না থাকলে
SET key value XX              # শুধু key থাকলে

# Atomic Counter
INCR counter                  # 1 বাড়াবে, atomic
INCRBY counter 5
DECR counter
DECRBY counter 3
INCRBYFLOAT price 1.50

# Multiple Keys
MSET key1 val1 key2 val2
MGET key1 key2 key3
MSETNX key1 val1 key2 val2    # সব keys না থাকলেই set করবে (atomic)

# String Manipulation
APPEND key " more text"
STRLEN key
GETRANGE key 0 4              # 0 থেকে 4 index পর্যন্ত
SETRANGE key 6 "Redis"        # position 6 থেকে overwrite

# Atomic Get+Set
GETSET key newvalue           # পুরনো value return করে, নতুন set করে
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
```

**Session Token:**
```bash
SET session:abc123 "user_id:1001" EX 3600  # 1 hour TTL
```

**Distributed Counter:**
```bash
INCR page:homepage:views     # atomic, race condition নেই
```

---

## 2. Hash

একটা key এর নিচে multiple field-value pair। একটা object বা database row represent করার জন্য perfect।

### Internal Encodings

**Listpack** (আগে ziplist ছিল, Redis 7.0+ এ listpack):
- Field সংখ্যা ≤ 128 (config: `hash-max-listpack-entries`) এবং প্রতিটা value ≤ 64 bytes (config: `hash-max-listpack-value`) হলে use হয়
- সব data একটা continuous memory block এ — cache friendly
- Lookup O(n) কারণ linear scan
- Memory অনেক কম

**Hashtable:**
- Threshold পার হলে automatically convert হয়
- Lookup O(1)
- Memory বেশি

এই automatic conversion টা বোঝা important। Small hash গুলোকে Redis সবসময় compact রাখার চেষ্টা করে।

### Hash Commands

```bash
# Set
HSET user:1001 name "Rahim" city "Dhaka" age 28
HSETNX user:1001 email "rahim@example.com"   # field না থাকলেই set

# Get
HGET user:1001 name
HMGET user:1001 name city age                # multiple fields
HGETALL user:1001                            # সব fields এবং values
HKEYS user:1001                              # শুধু fields
HVALS user:1001                              # শুধু values
HLEN user:1001                               # field count
HEXISTS user:1001 email                      # field আছে কিনা

# Update
HINCRBY user:1001 age 1
HINCRBYFLOAT product:1 price 10.5

# Delete
HDEL user:1001 email
```

### Hash vs String for Objects

**String approach:**
```bash
SET user:1001 '{"name":"Rahim","city":"Dhaka","age":28}'
# সমস্যা: age update করতে পুরো JSON parse → update → serialize → save করতে হয়
```

**Hash approach:**
```bash
HSET user:1001 name "Rahim" city "Dhaka" age 28
HINCRBY user:1001 age 1   # শুধু age field update, বাকি সব unchanged
# সুবিধা: partial update, individual field access efficient
```

---

## 3. List

Ordered collection of strings। Doubly linked list হিসেবে implement করা।

### Internal Encodings

**Listpack:**
- Element সংখ্যা ≤ 128 (config: `list-max-listpack-size`) হলে
- Compact memory

**Quicklist:**
- Threshold পার হলে — linked list of listpacks
- প্রতিটা node একটা listpack, listpack গুলো linked list এ connected
- এটা Redis এর একটা smart compromise — pure linked list এর বেশি memory efficient, pure array এর চেয়ে insert/delete fast

### List Commands

```bash
# Push
LPUSH mylist "first"          # left থেকে push
RPUSH mylist "last"           # right থেকে push
LPUSHX mylist "val"           # শুধু list exist করলে push
RPUSHX mylist "val"

# Pop
LPOP mylist                   # left থেকে pop
RPOP mylist                   # right থেকে pop
LPOP mylist 3                 # 3টা একসাথে pop

# Blocking Pop (Queue এর জন্য crucial)
BLPOP mylist 30               # element না থাকলে 30 seconds wait করবে
BRPOP mylist 30

# Read
LRANGE mylist 0 -1            # সব elements (0 থেকে last পর্যন্ত)
LRANGE mylist 0 4             # প্রথম 5টা
LINDEX mylist 0               # index 0 এর element
LLEN mylist                   # length

# Modify
LSET mylist 2 "newvalue"      # index 2 এর value change
LINSERT mylist BEFORE "val" "newval"
LREM mylist 2 "value"         # "value" এর 2টা occurrence remove (left থেকে)
LTRIM mylist 0 99             # শুধু প্রথম 100টা রাখো, বাকি delete

# Move between lists
LMOVE source destination LEFT RIGHT   # atomic move
```

### List Use Cases

**Message Queue:**
```bash
# Producer
RPUSH job:queue '{"job_id": 123, "type": "send_email"}'

# Consumer (blocking — CPU waste করে না)
BLPOP job:queue 0    # 0 মানে indefinitely wait
```

**Activity Feed (Latest N items):**
```bash
LPUSH user:1001:feed "Parcel AWB123 delivered"
LTRIM user:1001:feed 0 99    # শুধু latest 100টা রাখো
LRANGE user:1001:feed 0 9    # latest 10টা দেখাও
```

---

## 4. Set

Unique strings এর unordered collection। Duplicate allow করে না।

### Internal Encodings

**Listpack:**
- Element সংখ্যা ≤ 128 এবং প্রতিটা element ≤ 64 bytes হলে

**Intset:**
- সব elements যদি integer হয় — অনেক memory efficient special encoding

**Hashtable:**
- Threshold পার হলে

### Set Commands

```bash
# Add/Remove
SADD myset "apple" "banana" "cherry"
SREM myset "banana"

# Check
SISMEMBER myset "apple"          # 1 বা 0
SMISMEMBER myset "apple" "grape" # multiple check
SCARD myset                       # element count

# Read
SMEMBERS myset                    # সব elements (production এ সাবধান — large set হলে block করবে)
SRANDMEMBER myset 3               # random 3টা (destructive না)
SPOP myset 2                      # random 2টা pop করে remove করে

# Set Operations — এটাই Set এর superpower
SUNION set1 set2                  # union
SINTER set1 set2                  # intersection
SDIFF set1 set2                   # difference (set1 তে আছে, set2 তে নেই)

# Store results
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

# redis tag আছে এমন articles খোঁজো
SINTER article:101:tags article:102:tags   # common tags
```

**Unique Visitors:**
```bash
SADD page:home:visitors:2024-03-23 "user:1001" "user:1002"
SCARD page:home:visitors:2024-03-23   # unique visitor count
```

---

## 5. Sorted Set (ZSet)

Set এর মতো unique members, কিন্তু প্রতিটা member এর একটা **floating point score** আছে। Score অনুযায়ী sorted থাকে।

### Internal Encodings

**Listpack:**
- Member সংখ্যা ≤ 128 এবং প্রতিটা member ≤ 64 bytes হলে

**Skiplist + Hashtable (combined):**
- Threshold পার হলে
- **Skiplist** — O(log N) এ range query করা যায়
- **Hashtable** — O(1) এ individual member এর score পাওয়া যায়
- দুটো একসাথে থাকায় উভয় operation efficient

**Skiplist কী?**
Skiplist হলো একটা probabilistic data structure। Multiple levels এর linked list। উপরের level এ কম nodes থাকে, নিচের level এ বেশি। Search এ উপর থেকে শুরু করে দ্রুত target এ পৌঁছানো যায়। Binary search tree এর মতো O(log N) কিন্তু implement করা অনেক সহজ।

### Sorted Set Commands

```bash
# Add
ZADD leaderboard 95.5 "player1"
ZADD leaderboard 87.0 "player2" 92.3 "player3"   # multiple
ZADD leaderboard NX 100 "player4"                  # শুধু না থাকলে add
ZADD leaderboard XX 98 "player1"                   # শুধু থাকলে update
ZADD leaderboard GT 99 "player1"                   # score বড় হলেই update
ZADD leaderboard LT 50 "player1"                   # score ছোট হলেই update

# Score
ZSCORE leaderboard "player1"
ZINCRBY leaderboard 5.0 "player1"                  # score বাড়াও

# Range (score অনুযায়ী sorted)
ZRANGE leaderboard 0 -1                            # সব members (low to high)
ZRANGE leaderboard 0 -1 WITHSCORES                 # scores সহ
ZRANGE leaderboard 0 -1 REV                        # high to low
ZRANGE leaderboard "(80" "+inf" BYSCORE            # score 80 এর বেশি
ZRANGE leaderboard 0 2 BYSCORE LIMIT 0 10          # pagination

# Rank
ZRANK leaderboard "player1"                        # rank (0-indexed, low score = low rank)
ZREVRANK leaderboard "player1"                     # rank (high score = low rank)

# Count
ZCARD leaderboard                                  # total members
ZCOUNT leaderboard 80 100                          # score range এ কতজন

# Remove
ZREM leaderboard "player1"
ZREMRANGEBYRANK leaderboard 0 9                    # rank range এ remove
ZREMRANGEBYSCORE leaderboard "-inf" 50             # score range এ remove

# Set Operations
ZUNIONSTORE dest 2 zset1 zset2
ZINTERSTORE dest 2 zset1 zset2
ZDIFFSTORE dest 2 zset1 zset2
```

### Sorted Set Use Cases

**Real-time Leaderboard:**
```bash
ZADD game:leaderboard 1500 "user:1001"
ZINCRBY game:leaderboard 50 "user:1001"     # score add করো
ZREVRANGE game:leaderboard 0 9 WITHSCORES   # top 10
ZREVRANK game:leaderboard "user:1001"       # আমার rank
```

**Priority Queue:**
```bash
# Lower score = higher priority
ZADD job:queue 1 "critical_job"
ZADD job:queue 5 "normal_job"
ZADD job:queue 10 "low_priority_job"
ZPOPMIN job:queue    # সবচেয়ে high priority job নাও
```

**Time-series Events (Sliding Window):**
```bash
# Score হিসেবে timestamp use করো
ZADD user:1001:events 1711123200 "login"
ZADD user:1001:events 1711123500 "purchase"
# Last 1 hour এর events
ZRANGEBYSCORE user:1001:events (now-3600) now
```

---

## 6. Bitmap

String type এর উপর built করা। প্রতিটা bit individually manipulate করা যায়। Large boolean data store করতে extremely memory efficient।

```bash
SETBIT user:1001:login 0 1     # bit position 0 = 1 (logged in today)
GETBIT user:1001:login 0
BITCOUNT user:1001:login       # কতটা bit 1 আছে

# Bitwise operations
BITOP AND dest key1 key2
BITOP OR dest key1 key2
```

**Use case:** 1 million user এর daily login track করতে মাত্র 125KB লাগে।

---

## 7. HyperLogLog

Unique item count estimate করার জন্য probabilistic data structure। Memory constant — সব সময় মাত্র **12KB।**

Trade-off: Exact count না, ~0.81% error।

```bash
PFADD visitors "user:1001" "user:1002" "user:1003"
PFCOUNT visitors              # approximate unique count
PFMERGE total visitors1 visitors2   # merge করো
```

**Use case:** Billions of unique visitors count করতে। Exact না হলেও চলে।

---

## 8. Stream

Redis 5.0 তে add হয়েছে। Append-only log এর মতো। Kafka এর মতো event streaming Redis এ করা যায়।

```bash
# Add event
XADD mystream * user_id 1001 action "login" ip "192.168.1.1"
# * মানে auto-generate ID (timestamp-sequence format)

# Read
XREAD COUNT 10 STREAMS mystream 0   # সব events পড়ো
XRANGE mystream - +                  # সব entries

# Consumer Groups
XGROUP CREATE mystream mygroup $ MKSTREAM
XREADGROUP GROUP mygroup consumer1 COUNT 10 STREAMS mystream >
XACK mystream mygroup <message-id>   # processing confirm
```

**Use case:** Event sourcing, audit logs, real-time analytics।

---

## Key Naming Convention

Redis এ key naming এর কোনো enforcement নেই, কিন্তু convention follow করা critical:

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
- `KEYS user:*` দিয়ে সব user key খোঁজা যায়
- Namespace collision এড়ানো যায়
- Readable এবং maintainable

**Key size:** Maximum 512MB, কিন্তু key যত বড় তত memory বেশি। Short meaningful keys best।

---

<a name="layer-3"></a>
# Layer 3 — Persistence

Redis in-memory, কিন্তু data হারিয়ে না যাওয়ার জন্য disk এ save করার mechanism আছে। দুটো approach: **RDB** এবং **AOF।** দুটো একসাথেও use করা যায়।

---

## RDB — Redis Database Backup

RDB হলো একটা point-in-time **snapshot** of the entire dataset। নির্দিষ্ট interval এ পুরো memory এর data একটা binary file এ dump করে।

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
Child process exits
      ↓
dump.rdb complete হলে atomically replace হয়
```

**fork() এর ব্যাপারটা বোঝো:** Unix এ fork করলে child process parent এর exact memory copy পায়। কিন্তু এটা **Copy-on-Write (CoW)** mechanism এ কাজ করে। মানে actually copy হয় না, শুধু page table shared থাকে। যখন parent বা child কোনো page modify করে, তখনই সেই page টার actual copy হয়।

এর ফলে main process block হয় না, child process consistent snapshot দেখে।

### RDB Configuration

```conf
# redis.conf

# Automatic save triggers
save 3600 1       # 3600 seconds এ কমপক্ষে 1টা change হলে save
save 300 100      # 300 seconds এ কমপক্ষে 100টা change হলে save
save 60 10000     # 60 seconds এ কমপক্ষে 10000টা change হলে save

# Multiple conditions — যেকোনো একটা match করলেই save হবে

# File settings
dbfilename dump.rdb
dir /var/lib/redis

# Compression (LZF compression)
rdbcompression yes

# Checksum (CRC64)
rdbchecksum yes

# Save disable করতে
save ""
```

### Manual RDB Commands

```bash
BGSAVE              # background এ save (non-blocking)
SAVE                # foreground এ save (BLOCKING — production এ avoid করো)
LASTSAVE            # last successful save এর Unix timestamp
BGSAVE SCHEDULE     # BGSAVE already running থাকলে queue করে রাখে
```

### RDB এর Pros এবং Cons

**Pros:**
- Compact single file — backup এবং transfer সহজ
- Restore করা fast — পুরো dataset একটা file থেকে load
- Performance impact কম — child process এ হয়, main process unaffected
- Data এর point-in-time snapshot — disaster recovery এর জন্য ভালো

**Cons:**
- Data loss possible — last snapshot এর পর থেকে crash হলে সব যাবে
- Large dataset এ fork() অনেক time নিতে পারে
- CoW এর কারণে memory usage temporarily spike করে (worst case 2x)

**RDB কখন use করবে:** Data loss কিছুটা acceptable (cache use case), fast restart দরকার, backup/archival purpose।

---

## AOF — Append Only File

AOF প্রতিটা write command কে log file এ append করে। Crash হলে সেই log replay করে data recover করা যায়।

### AOF কীভাবে কাজ করে

```
Client → SET name Rahim
              ↓
Redis RAM এ write করে
              ↓
Command টা AOF buffer এ যায়
              ↓
fsync policy অনুযায়ী disk এ write হয়
              ↓
appendonly.aof file এ append হয়
```

File এর ভেতরে:
```
*3\r\n$3\r\nSET\r\n$4\r\nname\r\n$5\r\nRahim\r\n
*3\r\n$6\r\nEXPIRE\r\n$4\r\nname\r\n$4\r\n3600\r\n
```

### AOF fsync Policy — সবচেয়ে important config

```conf
appendfsync always      # প্রতিটা write এ fsync — safest, slowest
appendfsync everysec    # প্রতি second এ fsync — good balance (DEFAULT, recommended)
appendfsync no          # OS কে decide করতে দাও — fastest, least safe
```

**everysec** মানে maximum 1 second এর data loss possible crash এ।

### AOF Rewrite — File Size কমানো

AOF file সময়ের সাথে অনেক বড় হয়ে যায়। কারণ একই key অনেকবার update হলে সব commands log হয়। Rewrite করলে current state এ পৌঁছানোর minimum commands দিয়ে নতুন AOF তৈরি হয়।

```bash
BGREWRITEAOF    # background এ AOF rewrite
```

Config:
```conf
auto-aof-rewrite-percentage 100   # AOF size double হলে rewrite
auto-aof-rewrite-min-size 64mb    # minimum 64mb হলে rewrite
```

Rewrite process:
```
fork() — child process
Child: current memory state থেকে minimum commands দিয়ে নতুন AOF লেখে
Main: নতুন commands আসলে AOF buffer এ রাখে
Child শেষ হলে: নতুন AOF + buffer এর commands merge হয়
Atomically পুরনো AOF replace হয়
```

### AOF এর Pros এবং Cons

**Pros:**
- Data loss অনেক কম (everysec এ maximum 1 second)
- Human readable log file — manual recovery possible
- Corrupted হলে `redis-check-aof` দিয়ে repair করা যায়

**Cons:**
- File size অনেক বড় হয়
- Recovery সময় বেশি লাগে (সব commands replay করতে হয়)
- fsync এর কারণে performance কিছুটা কম

---

## RDB + AOF একসাথে (Recommended for Production)

Redis 2.4+ থেকে দুটো একসাথে use করা যায়। Restart এ AOF use হয় (বেশি complete)।

```conf
save 3600 1
save 300 100
save 60 10000

appendonly yes
appendfsync everysec
```

### Redis 7.0 — AOF with Multiple Files

Redis 7.0 তে নতুন AOF format এসেছে:
- **Base file** — RDB format এ current snapshot
- **Incremental files** — তারপর থেকে changes

এটা restart time কমায় এবং rewrite আরো efficient করে।

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
redis-check-aof appendonly.aof   # AOF file validate করো, error থাকলে fix করো
redis-check-aof --fix appendonly.aof
```

---

<a name="layer-4"></a>
# Layer 4 — Replication

Replication মানে একটা Redis server এর data অন্য Redis server এ automatically copy হওয়া। Primary server কে বলে **Master (বা Primary)**, copy গুলো কে বলে **Replica (বা Slave)।**

---

## Replication Architecture

```
Master (Read + Write)
    ├── Replica 1 (Read only)
    ├── Replica 2 (Read only)
    └── Replica 3 (Read only)
             └── Sub-Replica (Cascading replication)
```

**Asynchronous replication** — Master Replica কে acknowledge এর জন্য wait করে না। তাই:
- Master fast থাকে
- কিন্তু Replica কিছুটা behind থাকতে পারে (replication lag)
- Master crash এ কিছু data Replica তে না পৌঁছাতে পারে

---

## Replication কীভাবে কাজ করে

### Initial Sync (Full Synchronization)

```
Replica → PSYNC replicationid offset → Master
Master: offset match করে কিনা check করে
Match না করলে:
      ↓
Master → BGSAVE (RDB snapshot তৈরি করে)
Master: নতুন commands replication buffer এ জমা রাখে
RDB ready হলে:
      ↓
Master → RDB file পাঠায় → Replica
Replica: existing data flush করে RDB load করে
      ↓
Master → buffer এর commands পাঠায় → Replica
      ↓
Steady state: প্রতিটা write command realtime Replica তে যায়
```

### Partial Resync

Connection briefly drop হলে full resync দরকার নেই। Master একটা **replication backlog buffer** রাখে (default 1MB)। Reconnect এ শুধু missed commands পাঠায়।

```conf
repl-backlog-size 1mb      # বড় করলে partial resync success rate বাড়ে
repl-backlog-ttl 3600      # কতক্ষণ backlog রাখবে কোনো Replica না থাকলে
```

---

## Replication Configuration

### Master Configuration

```conf
# redis.conf (master)
bind 0.0.0.0
requirepass masterpassword
```

### Replica Configuration

```conf
# redis.conf (replica)
replicaof 192.168.1.10 6379          # master এর IP এবং port
masterauth masterpassword             # master এর password

replica-read-only yes                 # replica শুধু read করবে
replica-lazy-flush no
replica-priority 100                  # Sentinel এ failover priority (কম মানে বেশি priority)
```

### Runtime Commands

```bash
# Replica তে
REPLICAOF 192.168.1.10 6379          # master set করো
REPLICAOF NO ONE                      # master থেকে detach করো (promote করতে)

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
connected_slaves:2
slave0:ip=192.168.1.11,port=6379,state=online,offset=12345,lag=0
slave1:ip=192.168.1.12,port=6379,state=online,offset=12340,lag=1
master_replid:abc123...
master_repl_offset:12345
repl_backlog_size:1048576
```

**lag** — Replica কতটা behind। 0 বা 1 হওয়া উচিত। বেশি হলে network বা performance সমস্যা।

---

## Replication Lag কমানোর উপায়

```conf
# Master এ
repl-diskless-sync yes              # RDB disk এ না লিখে directly network এ পাঠায়
repl-diskless-sync-delay 5          # multiple replicas কে sync করার জন্য wait
repl-timeout 60                     # replication timeout

# Replica তে
replica-lazy-flush yes              # full sync এ old data lazily flush করে
```

---

## Read Scaling with Replicas

Replica গুলো read traffic handle করতে পারে। Application level এ read queries Replica তে পাঠাও, write queries Master এ।

```bash
# Replica read-only verify
redis-cli -h replica1 -p 6379 SET test value
# Error: READONLY You can't write against a read only replica.
```

**সতর্কতা:** Replication lag এর কারণে Replica এ stale data পড়তে পারো। Consistency critical হলে Master থেকেই read করো।

---

## min-replicas-to-write — Data Safety

Master এ configure করা যায় যে minimum কতটা Replica acknowledge করলে write accept করবে:

```conf
min-replicas-to-write 1             # কমপক্ষে 1 Replica কে write পৌঁছাতে হবে
min-replicas-max-lag 10             # Replica maximum 10 seconds পিছিয়ে থাকতে পারবে
```

এই condition পূরণ না হলে Master write reject করবে। এটা data loss কমায় কিন্তু availability কমায়।

---

<a name="layer-5"></a>
# Layer 5 — Redis Sentinel

Replication setup এ একটা বড় সমস্যা: **Master crash হলে manually failover করতে হয়।** Sentinel এই সমস্যা solve করে।

---

## Sentinel কী করে

Sentinel এর তিনটা মূল কাজ:
1. **Monitoring** — Master এবং Replica continuously monitor করে
2. **Notification** — কিছু wrong হলে admin কে notify করে
3. **Automatic Failover** — Master down হলে automatically একটা Replica কে promote করে নতুন Master বানায়

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

Sentinel নিজেরাও একটা distributed system। **Quorum** এর concept আছে — একটা নির্দিষ্ট সংখ্যক Sentinel agree করলেই failover হবে।

**Minimum 3টা Sentinel** recommend করা হয়। কারণ:
- 1টা Sentinel: Single point of failure
- 2টা: Split-brain possible (দুটো disagree করলে কাজ হয় না)
- 3টা: Quorum (2/3 agree) কাজ করে, 1টা fail হলেও system চলে

---

## Sentinel Configuration

```conf
# sentinel.conf

port 26379
daemonize yes
logfile /var/log/redis/sentinel.log

# Sentinel টা কোন Master monitor করবে
sentinel monitor mymaster 192.168.1.10 6379 2
# mymaster = নাম, 192.168.1.10 = master IP, 6379 = port
# 2 = quorum (failover এর জন্য কতটা Sentinel agree করতে হবে)

sentinel auth-pass mymaster masterpassword

# Master কতক্ষণ unreachable থাকলে down বলবে
sentinel down-after-milliseconds mymaster 5000

# Failover কতক্ষণের মধ্যে complete করতে হবে
sentinel failover-timeout mymaster 60000

# Failover এর পর একসাথে কতটা Replica নতুন Master থেকে sync করবে
sentinel parallel-syncs mymaster 1
```

---

## Failover Process — Step by Step

```
1. Sentinel গুলো Master কে ping করে
2. Master respond না করলে প্রতিটা Sentinel নিজে নিজে "SDOWN" (Subjective Down) mark করে
3. Quorum সংখ্যক Sentinel SDOWN করলে "ODOWN" (Objective Down) হয়
4. Sentinel গুলো নিজেদের মধ্যে vote করে একজন Leader Sentinel select করে
5. Leader Sentinel failover শুরু করে:
   a. Best Replica select করে (replication lag, priority দেখে)
   b. সেই Replica তে REPLICAOF NO ONE পাঠায় (promote করে)
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

Application code এ Sentinel-aware client library use করতে হয়। যেকোনো language এর Redis client এ Sentinel support আছে।

---

## Sentinel Commands

```bash
# Sentinel info
SENTINEL masters                              # সব monitored masters
SENTINEL master mymaster                      # specific master info
SENTINEL replicas mymaster                    # master এর replicas
SENTINEL sentinels mymaster                   # অন্য Sentinels

# Manual failover
SENTINEL failover mymaster                    # force failover

# Runtime config change
SENTINEL monitor newmaster 192.168.1.20 6379 2
SENTINEL remove mymaster
SENTINEL set mymaster down-after-milliseconds 3000
```

---

## Sentinel এর Limitations

- **Split-brain possibility** — network partition এ দুটো Master হতে পারে
- **Client complexity** — Sentinel-aware client দরকার
- **Vertical scaling only** — একটাই Master, write scaling নেই
- **Large dataset** — পুরো dataset একটা server এ থাকতে হয়

এই limitations এর জন্য Redis Cluster আসে।

---

<a name="layer-6"></a>
# Layer 6 — Redis Cluster

Redis Cluster হলো Redis এর **horizontal scaling solution।** Data automatically multiple nodes এ distribute হয়। কোনো node fail হলে automatically handle করে।

---

## Cluster Architecture

```
Node 1 (Master)          Node 2 (Master)          Node 3 (Master)
Slots: 0-5460            Slots: 5461-10922         Slots: 10923-16383
    |                        |                          |
Replica 1A               Replica 2A                Replica 3A
```

### Hash Slots — Data Distribution এর Core Concept

Redis Cluster এ **16384টা hash slot** আছে। প্রতিটা key একটা slot এ map হয়:

```
slot = CRC16(key) % 16384
```

এই slots গুলো nodes এর মধ্যে distribute হয়। 3 master node এ সমান ভাগে:
- Node 1: slots 0-5460
- Node 2: slots 5461-10922
- Node 3: slots 10923-16383

কোনো key SET করলে Redis সেটার slot calculate করে, সেই slot কোন node এ আছে সেখানে redirect করে।

### Hash Tags — একই Node এ রাখা

কখনো কখনো কিছু keys একই node এ রাখা দরকার (multi-key operations এর জন্য)। Key name এ `{}` দিয়ে hash tag specify করা যায়। শুধু `{}` এর ভেতরের অংশ দিয়ে slot calculate হয়।

```bash
SET {user:1001}.name "Rahim"
SET {user:1001}.age 28
# দুটোই একই slot এ যাবে কারণ "user:1001" same
```

---

## Cluster Setup

Minimum **6 nodes** দরকার production এ — 3 Master + 3 Replica।

### Configuration (প্রতিটা node এ)

```conf
port 7001
cluster-enabled yes
cluster-config-file nodes.conf        # cluster state file (auto-managed)
cluster-node-timeout 5000             # node unreachable timeout (ms)
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
```

Redis নিজেই slots assign করে এবং Master-Replica relationship set করে।

---

## Cluster Operation

```bash
# Cluster info
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
Client → Node 1: SET foo bar
Node 1 → MOVED 7638 192.168.1.11:7001
Client → Node 2 (192.168.1.11:7001): SET foo bar
```

**MOVED** — permanent redirect, smart client এর cache update করে।
**ASK** — temporary redirect, slot migration চলছে।

---

## Cluster Scaling — Resharding

নতুন node add করলে কিছু slots তাকে দিতে হবে:

```bash
# নতুন node add করো
redis-cli --cluster add-node 192.168.1.12:7001 192.168.1.10:7001

# Slots migrate করো
redis-cli --cluster reshard 192.168.1.10:7001
# Interactive mode — কতটা slots, কোথা থেকে, কোথায় জিজ্ঞেস করবে
```

**Zero downtime resharding:** Slot migration এ data live copy হয়। Migration চলাকালীন ASK redirect দিয়ে requests serve হয়।

---

## Cluster Failover

Master fail হলে Replica automatically promote হয়। Sentinel ছাড়াই cluster নিজেই এটা handle করে।

```bash
# Manual failover (planned maintenance)
redis-cli -c -h replica_host -p 7001 CLUSTER FAILOVER
# Replica নিজেই promote হওয়ার request করে
```

### Cluster Node Management

```bash
# Node remove করো (আগে slots migrate করতে হবে)
redis-cli --cluster del-node 192.168.1.10:7001 <node-id>

# Replica add করো
redis-cli --cluster add-node 192.168.1.13:7001 192.168.1.10:7001 --cluster-slave --cluster-master-id <master-node-id>

# Cluster fix করো (কিছু inconsistency হলে)
redis-cli --cluster fix 192.168.1.10:7001

# Cluster check করো
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
| **উদ্দেশ্য** | High Availability | HA + Horizontal Scaling |
| **Write scaling** | না | হ্যাঁ |
| **Data size** | Single server এর মধ্যে | Multiple servers এ distribute |
| **Complexity** | কম | বেশি |
| **Multi-key ops** | সব | Same slot এ সীমিত |
| **কখন use করবে** | Data ছোট, HA দরকার | Data বড় বা write heavy |

---

<a name="layer-7"></a>
# Layer 7 — Performance & Tuning

Redis by default fast, কিন্তু misconfiguration এ performance খারাপ হতে পারে। এই layer এ দেখবো কীভাবে Redis কে tune করতে হয়।

---

## Memory Management

### maxmemory এবং Eviction Policies

```conf
maxmemory 4gb                      # maximum memory limit
maxmemory-policy allkeys-lru       # limit পূরণ হলে কী করবে
```

**Eviction policies:**

| Policy | কাজ |
|--------|-----|
| `noeviction` | নতুন write reject করবে (default) |
| `allkeys-lru` | সব keys থেকে LRU evict করবে |
| `volatile-lru` | শুধু TTL আছে এমন keys থেকে LRU evict |
| `allkeys-lfu` | সব keys থেকে LFU evict (কম use হওয়া আগে যাবে) |
| `volatile-lfu` | TTL আছে এমন keys থেকে LFU evict |
| `allkeys-random` | Random evict |
| `volatile-random` | TTL আছে এমন keys থেকে random evict |
| `volatile-ttl` | সবচেয়ে কম TTL আগে যাবে |

**Cache use case এ:** `allkeys-lru` বা `allkeys-lfu`
**Mixed use case এ (cache + persistent):** `volatile-lru`
**Primary database এ:** `noeviction`

**LRU vs LFU:**
- LRU (Least Recently Used) — সবচেয়ে কম recently access হওয়া evict
- LFU (Least Frequently Used) — সবচেয়ে কম frequency তে access হওয়া evict
- LFU generally better কারণ একবার access করা কিন্তু old key LRU তে রয়ে যেতে পারে

### Memory Optimization Techniques

**1. Use appropriate data structures:**
Hash ব্যবহার করো individual String এর বদলে। 1000টা user field আলাদা String এ রাখার চেয়ে Hash এ রাখলে memory কম লাগে।

**2. Encoding thresholds tune করো:**
```conf
hash-max-listpack-entries 128     # এর নিচে listpack use হবে
hash-max-listpack-value 64
list-max-listpack-size -2         # 8kb per node
set-max-intset-entries 512        # integer set এর জন্য
zset-max-listpack-entries 128
zset-max-listpack-value 64
```

ছোট data এর জন্য Redis compact encoding use করে। এই threshold গুলো বাড়ালে বেশি data compact থাকবে — memory কমবে কিন্তু CPU বাড়বে।

**3. Key expiry set করো:**
অপ্রয়োজনীয় data জমা না হয়।

**4. Active defragmentation:**
```conf
activedefrag yes
active-defrag-ignore-bytes 100mb    # 100mb fragmentation হলে শুরু করবে
active-defrag-enabled yes
```

---

## Slow Log — Slow Queries ধরা

```conf
slowlog-log-slower-than 10000    # 10ms এর বেশি সময় নেওয়া commands log করবে (microseconds এ)
slowlog-max-len 128              # maximum 128টা log রাখবে
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
rename-command KEYS ""              # KEYS disable করো
rename-command FLUSHALL ""          # FLUSHALL disable করো
rename-command FLUSHDB ""           # FLUSHDB disable করো
rename-command DEBUG ""             # DEBUG disable করো
rename-command CONFIG "ADMIN-CONFIG" # rename করো
```

**KEYS কেন dangerous:** O(n) operation। Large keyspace এ Redis block হয়ে যায়। `SCAN` ব্যবহার করো।

### SCAN — Safe Alternative to KEYS

```bash
# KEYS * এর বদলে SCAN use করো
SCAN 0 MATCH "user:*" COUNT 100
# cursor, match pattern, approximate count per iteration
# 0 return হলে scan complete
```

SCAN non-blocking কারণ incremental করে কাজ করে। প্রতি call এ কিছু keys return করে, cursor দিয়ে পরের batch এ যায়।

---

## Connection Pool

Redis TCP connection establish করতে কিছুটা time লাগে। প্রতি request এ নতুন connection expensive।

**Connection pool use করো:** Application একটা pool of connections maintain করে। Request এ pool থেকে connection নেয়, শেষে return করে।

```conf
maxclients 10000         # maximum concurrent connections
tcp-keepalive 300        # idle connection 300 seconds পর close
timeout 0                # idle client disconnect timeout (0 = never)
```

---

## Pipeline — Batch Commands

```bash
# Without pipeline — 3টা round trips
SET key1 val1
SET key2 val2
SET key3 val3

# With pipeline — 1টা round trip
(pipeline)
SET key1 val1
SET key2 val2
SET key3 val3
(execute)
```

Pipeline এ multiple commands একসাথে পাঠানো যায়। Network latency এর impact কমে। Large bulk operations এ 10x+ performance gain।

---

## Lua Scripting — Atomic Operations

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
```

**EVALSHA** — script hash দিয়ে call করো, script বারবার পাঠাতে হয় না:
```bash
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# sha1 hash return করে
EVALSHA <sha1> 1 mykey
```

---

## Latency Monitoring

```bash
redis-cli --latency -h 127.0.0.1 -p 6379    # real-time latency
redis-cli --latency-history                   # historical data
redis-cli --latency-dist                      # distribution

# Latency monitoring
CONFIG SET latency-monitor-threshold 100      # 100ms এর বেশি হলে log
LATENCY LATEST                                 # latest latency events
LATENCY HISTORY event                         # specific event history
LATENCY RESET                                 # clear
```

---

## OS Level Tuning

Redis এর performance শুধু Redis config এ না, OS level এও tuning দরকার।

```bash
# Transparent Huge Pages disable করো (Redis performance এ negative impact)
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Overcommit memory enable করো (BGSAVE এর জন্য)
echo 1 > /proc/sys/vm/overcommit_memory
# অথবা /etc/sysctl.conf এ:
vm.overcommit_memory = 1

# Somaxconn — TCP backlog
echo 511 > /proc/sys/net/core/somaxconn
net.core.somaxconn = 511

# Redis config এ
tcp-backlog 511
```

**Transparent Huge Pages কেন disable করতে হয়:** THP enabled থাকলে fork() এ CoW অনেক বেশি memory নেয় এবং latency spike করে।

---

## Redis INFO — Health Monitoring

```bash
INFO                      # সব sections
INFO server               # Redis version, OS, config file
INFO clients              # connected clients, blocked clients
INFO memory               # memory usage, fragmentation
INFO stats                # ops/sec, hits, misses
INFO replication          # replication status
INFO cpu                  # CPU usage
INFO keyspace             # database এর key count, expiry info
```

Important metrics:
```
used_memory_human          # actual data memory
used_memory_rss_human      # OS এর কাছ থেকে allocated memory
mem_fragmentation_ratio    # 1.0-1.5 healthy
connected_clients          # current connections
instantaneous_ops_per_sec  # ops/sec
keyspace_hits              # cache hit count
keyspace_misses            # cache miss count
# hit ratio = hits / (hits + misses)
```

---

## Benchmark

```bash
# Built-in benchmark tool
redis-benchmark -h 127.0.0.1 -p 6379 -n 100000 -c 50
# -n: total requests, -c: concurrent clients

# Specific commands
redis-benchmark -t set,get -n 100000

# Pipeline test
redis-benchmark -t set -n 100000 -P 16
```

---

<a name="layer-8"></a>
# Layer 8 — Real Patterns

Redis এর theory জানার পর এখন দেখবো production এ কোথায় কীভাবে use হয়।

---

## 1. Caching Patterns

### Cache-Aside (Lazy Loading) — সবচেয়ে common

```
Application: Redis এ data আছে?
  হ্যাঁ → Redis থেকে return করো
  না  → Database থেকে নিয়ে Redis এ set করো → return করো
```

**Pros:** শুধু read হওয়া data cache হয়। Redis restart এ gradual warmup।
**Cons:** প্রথম request এ cache miss — database hit।

### Write-Through

```
Application: data write করবে
     ↓
প্রথমে Redis এ লেখো
     ↓
তারপর Database এ লেখো
```

**Pros:** Cache always fresh।
**Cons:** Write slow, rarely read হওয়া data ও cache এ জমে।

### Write-Behind (Write-Back)

```
Application → Redis এ লেখো → return করো (fast)
Background process → Redis থেকে batch এ Database এ লেখো
```

**Pros:** Write অনেক fast।
**Cons:** Redis crash এ data loss possible।

### Cache Invalidation

Data update হলে cache invalid করতে হয়:
```bash
# Option 1: Delete করো (next read এ fresh data আসবে)
DEL user:1001:profile

# Option 2: Update করো
HSET user:1001:profile name "New Name"

# Option 3: TTL based (expire হলে নিজেই fresh হবে)
EXPIRE user:1001:profile 300
```

---

## 2. Session Management

```bash
# Login
SET session:TOKEN_HERE '{"user_id":1001,"role":"admin"}' EX 3600

# Request validation
GET session:TOKEN_HERE
# null হলে expired বা invalid

# Session refresh (activity এ TTL extend)
EXPIRE session:TOKEN_HERE 3600

# Logout
DEL session:TOKEN_HERE
```

---

## 3. Rate Limiting

### Fixed Window

```bash
INCR rate:user:1001:minute:1711123200
EXPIRE rate:user:1001:minute:1711123200 60
# Value > limit হলে block
```

**সমস্যা:** Window boundary তে burst possible (window শেষে 100 + পরের window শুরুতে 100 = 200 requests)।

### Sliding Window (Sorted Set দিয়ে)

```bash
# Current timestamp
now = 1711123500
window = 60  # 60 seconds

# এই user এর requests এর log
ZADD rate:user:1001 now now   # score=timestamp, member=timestamp

# Old entries remove করো
ZREMRANGEBYSCORE rate:user:1001 0 (now - window)

# Count check করো
count = ZCARD rate:user:1001
# count > limit হলে block

# TTL set করো
EXPIRE rate:user:1001 window
```

---

## 4. Distributed Lock

Multiple servers যখন একটা shared resource access করে race condition হয়। Redis দিয়ে distributed lock implement করা যায়।

### Simple Lock

```bash
# Lock acquire
SET lock:resource1 "server1_unique_id" NX EX 30
# NX — না থাকলেই set (atomic check-and-set)
# EX 30 — 30 seconds এ auto-expire (deadlock prevent)

# Lock release (Lua দিয়ে atomic check-and-delete)
EVAL "
  if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
  else
    return 0
  end
" 1 lock:resource1 "server1_unique_id"
```

**কেন unique_id দরকার:** নিজের lock ছাড়া অন্যের lock delete না করার জন্য।
**কেন Lua দিয়ে release:** GET + DEL atomic না হলে race condition হতে পারে।

### Redlock — Multi-Node Distributed Lock

Single node lock reliable না (node fail হলে)। **Redlock algorithm** N nodes এ majority (N/2 + 1) lock acquire করে:

```
N = 5 nodes
Lock acquire করো সব 5টায়
3টায় (majority) success হলে lock acquired
```

Production এ Redlock library use করো (Redisson, node-redlock, etc.)।

---

## 5. Pub/Sub — Messaging

```bash
# Subscriber (channel subscribe করো)
SUBSCRIBE news:tech news:sports

# Publisher (message publish করো)
PUBLISH news:tech "Redis 8.0 released!"

# Pattern subscribe
PSUBSCRIBE news:*    # news: দিয়ে শুরু সব channel
```

**Pub/Sub এর limitations:**
- Message persistence নেই — subscriber offline থাকলে message হারায়
- No acknowledgement — delivery guarantee নেই
- Persistent messaging দরকার হলে **Stream** use করো

---

## 6. Leaderboard

```bash
# Score update
ZADD game:leaderboard 1500 "user:1001"
ZINCRBY game:leaderboard 50 "user:1001"   # score add

# Top 10
ZREVRANGE game:leaderboard 0 9 WITHSCORES

# User এর rank
ZREVRANK game:leaderboard "user:1001"

# User এর score
ZSCORE game:leaderboard "user:1001"

# Rank এর কাছাকাছি users (user এর আশেপাশে কে আছে)
rank = ZREVRANK leaderboard "user:1001"
ZREVRANGE leaderboard (rank-2) (rank+2) WITHSCORES
```

---

## 7. Autocomplete

```bash
# Terms index এ add করো (score 0, alphabetical order এ থাকবে)
ZADD autocomplete 0 "redis"
ZADD autocomplete 0 "redis cluster"
ZADD autocomplete 0 "redis sentinel"
ZADD autocomplete 0 "replication"

# "red" দিয়ে শুরু terms খোঁজো
ZRANGEBYLEX autocomplete "[red" "[red\xff"
# \xff — highest possible character, এর আগের সব match করবে
```

---

## 8. Geospatial

Redis 3.2 থেকে geospatial support:

```bash
# Location add করো
GEOADD courier:locations 90.4125 23.8103 "courier:1001"
GEOADD courier:locations 90.3938 23.7272 "courier:1002"

# Distance
GEODIST courier:locations courier:1001 courier:1002 km

# নির্দিষ্ট point এর কাছের couriers
GEOSEARCH courier:locations FROMMEMBER courier:1001 BYRADIUS 5 km ASC COUNT 10
# বা coordinates দিয়ে
GEOSEARCH courier:locations FROMLONLAT 90.4125 23.8103 BYRADIUS 5 km ASC

# Coordinates পাওয়া
GEOPOS courier:locations courier:1001
```

---

## 9. Redis as Message Queue vs Kafka

| | Redis List/Stream | Kafka |
|--|---------|---------|
| **Throughput** | High | Very High |
| **Persistence** | Optional | Always |
| **Consumer Groups** | Streams তে আছে | আছে |
| **Replay** | Streams তে আছে | আছে |
| **Complexity** | Simple | Complex |
| **Use when** | Simple queuing, lower volume | High volume, replay critical |

---

## 10. Redis Streams — Production Event Sourcing

```bash
# Event produce করো
XADD orders * order_id AWB123 status "created" amount 500

# Consumer Group তৈরি
XGROUP CREATE orders order-processor $ MKSTREAM

# Consume করো
XREADGROUP GROUP order-processor worker1 COUNT 10 BLOCK 2000 STREAMS orders >
# > মানে শুধু নতুন (undelivered) messages

# Acknowledge করো
XACK orders order-processor <message-id>

# Pending messages দেখো (acknowledged হয়নি)
XPENDING orders order-processor - + 10

# Dead letter handling — অনেকক্ষণ pending থাকা messages
XAUTOCLAIM orders order-processor recovery-worker 3600000 0 COUNT 10
```

---

# Quick Reference Summary

## Data Structure Selection Guide

| Use case | Data Structure |
|----------|---------------|
| Simple value, counter, flag | String |
| Object/Entity attributes | Hash |
| Ordered list, queue, stack | List |
| Unique items, tags, membership | Set |
| Ranking, score-based, sorted | Sorted Set |
| Large boolean arrays | Bitmap |
| Unique count (approximate) | HyperLogLog |
| Event log, message queue | Stream |
| Location-based | Geo |

## Architecture Selection Guide

| Need | Solution |
|------|---------|
| Single server, no HA | Standalone |
| HA, small-medium data | Sentinel (3 Sentinels + 1 Master + 2 Replicas minimum) |
| Large data, horizontal scale | Cluster (6 nodes minimum) |

## Common Configuration Checklist

```conf
# Security
requirepass strongpassword
rename-command KEYS ""
rename-command FLUSHALL ""
bind 127.0.0.1  # বা specific IP

# Memory
maxmemory 4gb
maxmemory-policy allkeys-lru

# Persistence (balanced)
save 3600 1
save 300 100
appendonly yes
appendfsync everysec

# Performance
tcp-keepalive 300
tcp-backlog 511

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 128
```

## OS Tuning Checklist

```bash
# /etc/sysctl.conf
vm.overcommit_memory = 1
net.core.somaxconn = 511

# Transparent Huge Pages
echo never > /sys/kernel/mm/transparent_hugepage/enabled
# /etc/rc.local এ add করো permanent করতে
```

---

*Redis Complete Study Guide | E-MultiSkills PostgreSQL Administration Course Companion*
