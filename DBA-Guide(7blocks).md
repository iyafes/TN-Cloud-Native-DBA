# Senior DBA Master Guide
**PostgreSQL-primary | MySQL secondary reference | Concept-first | Maximum Depth**

> **ভাষা নীতি:** বাংলায় ব্যাখ্যা, technical term English-এ।
> **লক্ষ্য:** "কী" + "কেন" + "কীভাবে ভেতরে কাজ করে" + "না জানলে কী হয়" + "big picture-এ কোথায় fit করে"

---

# BLOCK 1 — HOW A DATABASE ACTUALLY WORKS

---

## Topic 1 — Database Architecture: Storage Engine, Memory, Process Model

### Big Picture: কেন এটা দিয়ে শুরু?

তুমি যখন `SELECT * FROM orders WHERE user_id = 5` লেখো, এটা দেখতে সহজ। কিন্তু এই এক লাইনের পেছনে একটা পুরো শহর কাজ করে — OS, disk, memory, multiple processes, locking system। এই "শহর"-এর map না জানলে:

- Performance problem দেখলে কোথায় খুঁজবে জানবে না
- Configuration parameter কেন set করছো বুঝবে না
- Index কেন কাজ করছে না ধরতে পারবে না
- Crash হলে কী হয় — কিছুই বুঝবে না

এই topic হলো সেই map। বাকি ৩০টা topic এই map-এর উপর দাঁড়িয়ে।

---

### তিনটা Layer — এবং কেন এই division?

```
┌─────────────────────────────────────────────────────────┐
│                  CLIENT / APPLICATION                   │
└──────────────────────────┬──────────────────────────────┘
                           │  SQL (text)
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   PROCESS MODEL                         │
│   "কে কাজটা করবে এবং কীভাবে coordinate করবে?"         │
│                                                         │
│  Postmaster ──fork()──► Backend Process (per connection)│
│       │                                                 │
│       └──► Background Workers (bgwriter, autovacuum...) │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   MEMORY LAYER                          │
│   "কাজটা কোথায় হবে? Disk-এ না গিয়ে কী করা যায়?"      │
│                                                         │
│  Shared Memory:                                         │
│    Shared Buffer Pool ← সবচেয়ে গুরুত্বপূর্ণ            │
│    WAL Buffer                                           │
│    Lock Table                                           │
│    CLOG (Commit Log)                                    │
│                                                         │
│  Local Memory (per backend):                            │
│    work_mem | maintenance_work_mem | temp_buffers       │
└──────────────────────────┬──────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                   STORAGE ENGINE                        │
│   "Data আসলে কোথায় থাকে? কীভাবে persist হয়?"          │
│                                                         │
│  Heap Files (tables) | Index Files | WAL Files          │
│  FSM | VM | TOAST tables                                │
└─────────────────────────────────────────────────────────┘
```

এই তিনটা layer আলাদা করে design করার কারণ হলো **separation of concerns** — প্রতিটা layer নিজের কাজ করে, একটার change অন্যটাকে সরাসরি impact না করে। এই design-এর কারণেই PostgreSQL crash-safe, concurrent এবং performant।

---

### Layer 1 — Storage Engine: Data Disk-এ কীভাবে থাকে?

Storage Engine-এর কাজ হলো disk-এ data persist করা এবং দরকারমতো পড়া। এটা সরাসরি OS-এর file system-এর সাথে কথা বলে।

**PostgreSQL-এর physical directory structure:**

```
$PGDATA/  (usually /var/lib/postgresql/17/main/)
│
├── base/                    ← প্রতিটা database-এর data এখানে
│   ├── 1/                   ← template1 database (OID=1)
│   ├── 16384/               ← তোমার database (OID দিয়ে নাম)
│   │   ├── 16385            ← একটা table-এর heap file
│   │   ├── 16385_fsm        ← Free Space Map
│   │   ├── 16385_vm         ← Visibility Map
│   │   ├── 16386            ← একটা index file
│   │   └── ...
│   └── ...
│
├── global/                  ← cluster-wide system tables (pg_database, pg_roles)
├── pg_wal/                  ← Write-Ahead Log files (16MB segments)
├── pg_stat_tmp/             ← runtime statistics
├── pg_tblspc/               ← tablespace symlinks
├── postgresql.conf          ← configuration
├── pg_hba.conf              ← authentication rules
└── postmaster.pid           ← Postmaster-এর PID (running instance-এর proof)
```

**কেন OID দিয়ে file নাম?**
PostgreSQL প্রতিটা object-কে (table, index, database) একটা unique integer দেয় — Object Identifier (OID)। Human-readable নাম (`pg_class.relname`) শুধু catalog-এ থাকে। Disk-এ file-এর নাম OID। এতে rename করলে file move করতে হয় না — catalog-এ শুধু নাম বদলে।

```sql
-- একটা table-এর OID এবং physical file path দেখো
SELECT oid, relname, relfilenode, relpages
FROM pg_class
WHERE relname = 'orders';

-- Physical path:
SELECT pg_relation_filepath('orders');
-- Output: base/16384/16385
```

**Storage Engine-এর core responsibility:**
1. **Read path:** "Page X দরকার" → file থেকে 8KB read → memory-তে দাও
2. **Write path:** "এই page dirty হয়েছে" → eventually disk-এ লেখো
3. **Durability:** Crash-এর পরেও data intact থাকে (WAL-এর সাহায্যে)

**PostgreSQL (Heap) vs MySQL/InnoDB Storage — মূল পার্থক্য:**

PostgreSQL heap-based storage মানে rows কোনো inherent order ছাড়া যেকোনো page-এ থাকতে পারে। MySQL InnoDB-তে table মানেই clustered B-Tree — rows physically PRIMARY KEY order-এ থাকে।

এই একটা design choice-এর cascading effects:

| Effect | PostgreSQL (Heap) | MySQL InnoDB (Clustered) |
|--------|-------------------|--------------------------|
| PK range scan | Random page access (slower) | Sequential pages (faster) |
| Secondary index lookup | ctid → directly heap page | PK value → clustered index lookup (double lookup) |
| INSERT order | যেকোনো free page-এ | PK order maintain করতে হয় (page split) |
| UUID PK problem | কম severe | Severe (random insert = constant page split) |
| MVCC old versions | Same heap-এ dead tuple | Undo log-এ (আলাদা system) |

---

### Layer 2 — Memory Architecture: RAM-এর সঠিক ব্যবহার

Disk access কতটা slow সেটা একটু visualize করি:

```
Memory hierarchy (latency):
L1 Cache:     ~1 ns       ████
L2 Cache:     ~4 ns       ████
L3 Cache:     ~40 ns      ████
RAM:          ~100 ns     ████████
SSD:          ~100 µs     ████████████████████████████████ (1,000x slower than RAM)
HDD:          ~10 ms      ██████████... (100,000x slower than RAM)
```

Database-এর job হলো RAM-কে যতটা সম্ভব কার্যকরভাবে ব্যবহার করে disk hit কমানো।

#### Shared Memory — সব Process একসাথে ব্যবহার করে

**Shared Buffer Pool — এটাই সবকিছুর কেন্দ্র:**

```
Disk-এর data pages এখানে cache হয়।

Structure:
┌──────────────────────────────────────────────────────────┐
│                   SHARED BUFFER POOL                     │
│                   (default: 128MB)                       │
│                                                          │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │
│  │ Page    │ │ Page    │ │ Page    │ │ Page    │  ...    │
│  │(8KB)    │ │(8KB)    │ │(8KB)    │ │(8KB)    │         │
│  │table A  │ │index B  │ │table A  │ │table C  │         │
│  │pg 0     │ │pg 5     │ │pg 1     │ │pg 100   │         │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘        │
│                                                          │
│  Hash Table: (relation_id, page_number) → buffer slot   │
│  LRU-like eviction (Clock Sweep algorithm)               │
└──────────────────────────────────────────────────────────┘
```

**Buffer Pool-এর internal mechanism — Clock Sweep:**

PostgreSQL Buffer Pool LRU (Least Recently Used) ব্যবহার করে না exactly — কারণ true LRU-তে প্রতিটা access-এ linked list update করতে হয় যেটা concurrent environment-এ lock-heavy। PostgreSQL ব্যবহার করে **Clock Sweep** algorithm:

```
প্রতিটা buffer-এ একটা "usage count" (0-5) থাকে।
Page access করলে usage count বাড়ে।
নতুন page দরকার হলে "clock hand" ঘোরে:
  - usage count > 0? → decrement করো, skip করো
  - usage count = 0? → এই buffer evict করো, নতুন page রাখো

এতে frequently used page-গুলো বেশি usage count থাকায় evict হয় না।
One-time sequential scan (যেমন COPY বড় data) buffer pool দখল করে না।
```

**Cache Hit vs Miss — production-এর সবচেয়ে গুরুত্বপূর্ণ metric:**

```sql
-- Database-level cache hit ratio
SELECT
  datname,
  blks_hit,
  blks_read,
  round(
    blks_hit::numeric / nullif(blks_hit + blks_read, 0) * 100,
    2
  ) AS cache_hit_pct
FROM pg_stat_database
WHERE datname = current_database();

-- Table-level (কোন table সবচেয়ে বেশি disk read করছে?)
SELECT
  relname,
  heap_blks_hit,
  heap_blks_read,
  round(
    heap_blks_hit::numeric / nullif(heap_blks_hit + heap_blks_read, 0) * 100,
    2
  ) AS cache_hit_pct,
  idx_blks_hit,
  idx_blks_read
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 20;
```

**Target:** production-এ ≥ 99%। ৯৮% মানে ২% miss — যদি ১০০ queries/sec হয়, তাহলে ২ disk read/sec। ৯০% মানে ১০ disk read/sec — ১০০x slow operation।

**shared_buffers sizing:**

```
Rule of thumb: RAM-এর ২৫%

কেন ২৫%? কেন ৫০% না?
কারণ OS-এরও file system cache আছে (page cache)।
PostgreSQL disk read করলে OS সেটা নিজেও cache করে।
তাই effective cache = shared_buffers + OS page cache।
effective_cache_size = ৭৫% of RAM (planner-কে hint দাও)

২৫% এর বেশি দিলে OS-এর জন্য কম থাকে → 
OS cache miss বাড়ে → disk I/O বাড়ে → paradoxically slow হতে পারে।

তবে dedicated DB server-এ 40% পর্যন্ত দেওয়া যায়।
```

**WAL Buffer:**

```
Transaction-এর changes disk-এ যাওয়ার আগে এখানে জমা হয়।
COMMIT হলে WAL Buffer → WAL file (disk) → তারপর client "OK" পায়।

Size: wal_buffers = 64MB (default, auto-tuned from shared_buffers)
High-write workload-এ 256MB পর্যন্ত দিতে পারো।
```

**Lock Table:**

```
কোন transaction কোন relation/page/row lock করেছে সেটা shared memory-তে থাকে।
সব backend process এটা দেখতে পায়।
Lock acquire করতে হলে এই table-এ entry করতে হয়।
```

**CLOG (Commit Log / pg_xact):**

```
প্রতিটা transaction-এর status: in-progress / committed / aborted / sub-committed
MVCC-তে "কোন row visible?" determine করতে এটা দরকার।
(বিস্তারিত Topic 4-এ)
```

#### Local Memory — প্রতিটা Backend-এর নিজস্ব

**work_mem — সবচেয়ে বেশি misunderstood parameter:**

```
Sort operation, Hash Join build phase — এই memory ব্যবহার করে।
এটা per-operation, per-backend নয়।

একটা complex query-তে:
  - ৩টা Sort node থাকতে পারে → ৩ × work_mem
  - ২টা Hash Join থাকতে পারে → ২ × work_mem
  - Parallel worker-ও আলাদা work_mem নেয়

Worst case calculation:
work_mem × max_parallel_workers_per_query × max_connections
= 64MB × 4 × 200 = 51,200MB = 50GB!

বাস্তবে সবাই একসাথে complex query করে না।
কিন্তু এটা না বুঝলে work_mem = 256MB set করলে OOM হতে পারে।

Safe approach:
  - Global: work_mem = 4-16MB (conservative)
  - Session-level: SET LOCAL work_mem = '256MB'; (specific heavy query-এ)
  - Role-level: ALTER ROLE analyst SET work_mem = '128MB';
```

**work_mem যথেষ্ট না হলে কী হয়:**

```sql
-- Sort disk-এ গেছে কিনা দেখো
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders ORDER BY amount DESC;

-- Output-এ দেখবে:
-- Sort Method: external merge  Disk: 45678kB   ← BAD: disk sort
-- অথবা:
-- Sort Method: quicksort  Memory: 8192kB       ← GOOD: in-memory sort
```

**maintenance_work_mem:**

```
VACUUM, CREATE INDEX, ALTER TABLE ADD FOREIGN KEY — এরা ব্যবহার করে।
Default 64MB। Production-এ 1-4GB দাও।

কেন বড় দরকার?
CREATE INDEX on 100M row table:
  - maintenance_work_mem = 64MB → sort করতে পারছে না, বারবার pass করতে হয়
  - maintenance_work_mem = 2GB → একবারে sort, অনেক দ্রুত index তৈরি
```

---

### Layer 3 — Process Model: কে কাজটা করে?

PostgreSQL **multi-process** architecture। এটা MySQL-এর **multi-thread** থেকে মৌলিকভাবে আলাদা।

**কেন multi-process? Thread নয় কেন?**

```
Thread model (MySQL):
├── একটা process-এর মধ্যে সব threads
├── Shared heap memory → একটা bug সব thread-এর memory corrupt করতে পারে
├── একটা thread crash → পুরো process (সব connections) down
└── Context switch: lightweight

Process model (PostgreSQL):
├── প্রতিটা connection = আলাদা OS process
├── আলাদা virtual address space → একটা crash বাকিদের affect করে না
├── OS-এর process isolation → security boundary
└── Context switch: heavier, কিন্তু stability বেশি

PostgreSQL-এর decision (1990s-এ) ছিল stability-first।
আজকে OS threading অনেক ভালো, কিন্তু PostgreSQL-এর codebase এভাবেই built।
PostgreSQL 17-এ experimental thread support আসছে।
```

**Process hierarchy:**

```
init / systemd
    │
    └── Postmaster (postgres: master process)
            │
            ├── Backend Process (for Client 1)  ← fork() on connection
            ├── Backend Process (for Client 2)
            ├── Backend Process (for Client 3)
            │
            ├── Background Writer (bgwriter)
            ├── WAL Writer
            ├── Checkpointer
            ├── Autovacuum Launcher
            │       └── Autovacuum Worker (spawned as needed)
            ├── Statistics Collector
            ├── WAL Archiver (if archiving enabled)
            ├── WAL Receiver (if standby)
            └── Logical Replication Workers (if configured)
```

**Postmaster-এর role — Gate Keeper:**

```
1. PostgreSQL start হলে Postmaster প্রথমে চলে।
2. Port 5432-এ listen করে।
3. Client connect করতে চাইলে:
   a. TCP handshake
   b. Authentication (pg_hba.conf check)
   c. fork() → নতুন Backend Process
   d. Backend Process শুরু হয়, Postmaster নতুন connection-এর জন্য wait করে
4. কোনো Backend crash করলে:
   a. Postmaster SIGCHLD signal পায়
   b. অন্য backends-কে "restart হচ্ছে" signal পাঠায়
   c. Shared memory clean করে
   d. নতুন connections accept করতে থাকে
5. Postmaster নিজে crash করলে → সব backends die করে, restart দরকার।
```

**Background Workers — প্রতিটার কারণ বোঝা জরুরি:**

**bgwriter (Background Writer):**
```
কাজ: Shared Buffer-এর dirty pages আগে থেকে disk-এ লিখে রাখে।
কেন দরকার?
  Checkpoint-এর সময় সব dirty pages একসাথে disk-এ লিখতে হয়।
  bgwriter না থাকলে Checkpoint = হঠাৎ বিশাল I/O spike।
  bgwriter আগে থেকে কিছু লিখে রাখলে Checkpoint-এর load কমে।

bgwriter কখন লেখে?
  bgwriter_lru_maxpages: প্রতি round-এ সর্বোচ্চ কত page লিখবে (default 100)
  bgwriter_delay: কত ms পর পর scan করবে (default 200ms)
  bgwriter_lru_multiplier: কতটা anticipate করবে (default 2.0)

Monitoring:
SELECT buffers_clean, maxwritten_clean, buffers_backend
FROM pg_stat_bgwriter;
-- buffers_backend বেশি হলে bgwriter যথেষ্ট লিখছে না
```

**WAL Writer:**
```
কাজ: WAL Buffer থেকে WAL files-এ data লেখে।
কখন লেখে?
  - wal_writer_delay interval-এ (default 200ms)
  - Buffer ভরে গেলে
  - COMMIT-এ (synchronous_commit=on হলে commit করতেই লেখে)

কেন আলাদা process?
  Backend process নিজেও WAL লিখতে পারে, কিন্তু
  dedicated writer I/O বেশি efficient (group commits possible)।
```

**Checkpointer:**
```
কাজ: Checkpoint নেয় — সব dirty pages flush করে WAL-এ marker লেখে।
কখন?
  - checkpoint_timeout (default 5min) পর পর
  - WAL size max_wal_size (default 1GB) ছাড়িয়ে গেলে (forced checkpoint)
  - Manual: CHECKPOINT; command

Checkpoint = recovery point।
Crash হলে শেষ checkpoint থেকে WAL replay শুরু হয়।
Checkpoint interval বড় → recovery time বেশি, কিন্তু I/O কম।
```

**Autovacuum Launcher + Workers:**
```
Launcher: কোন table-এ vacuum দরকার সেটা monitor করে।
Worker: আসলে vacuum করে।

কেন background-এ?
  MVCC-র কারণে dead tuples জমে।
  Dead tuples না সরালে:
    - Table bloat → disk বেশি, I/O বেশি
    - Sequential scan slow
    - XID wraparound (catastrophic failure)
  
  Autovacuum না থাকলে manual vacuum করতে হতো → অসম্ভব।
```

---

### একটা Query-র সম্পূর্ণ জীবন — End to End

```
তুমি লিখলে: SELECT name, email FROM users WHERE id = 42;

═══════════════════════════════════════════════════
STEP 1: Network → Postmaster
═══════════════════════════════════════════════════
Client TCP connection → port 5432
Postmaster: pg_hba.conf check (IP allowed? Authentication method?)
            → scram-sha-256 handshake
            → fork() → Backend Process P1 তৈরি

═══════════════════════════════════════════════════
STEP 2: Backend P1 — Parse
═══════════════════════════════════════════════════
SQL string receive করলো: "SELECT name, email FROM users WHERE id = 42"
Parser: tokenize → parse tree (AST)
  SelectStmt {
    targetList: [name, email],
    fromClause: [users],
    whereClause: (id = 42)
  }
Syntax error? এখানেই ধরা পড়বে।

═══════════════════════════════════════════════════
STEP 3: Backend P1 — Analyze
═══════════════════════════════════════════════════
pg_catalog query:
  - "users" table আছে? pg_class check
  - "name", "email", "id" column আছে? pg_attribute check
  - current user-এর SELECT permission আছে? pg_class.relacl check
  - Type check: id = 42 → id column bigint, 42 integer → implicit cast ok?

Query Tree তৈরি হলো (semantic-aware)।

═══════════════════════════════════════════════════
STEP 4: Backend P1 — Plan
═══════════════════════════════════════════════════
Planner: "id = 42" — এটা কীভাবে fetch করবো?

Option 1: Sequential Scan
  Cost estimate: seq_page_cost × relpages + cpu_tuple_cost × reltuples
  = 1.0 × 500 + 0.01 × 100000 = 1500 (estimated)

Option 2: Index Scan (users_pkey on id)
  Cost estimate: random_page_cost × (tree height) + cpu_index_tuple_cost × ...
  = 4.0 × 3 + ... ≈ 8.5 (estimated)

Index Scan wins! Plan = Index Scan using users_pkey.

═══════════════════════════════════════════════════
STEP 5: Backend P1 — Execute
═══════════════════════════════════════════════════
Executor: "Index Scan on users_pkey WHERE id=42"

5a. Index lookup:
    Buffer Pool check: users_pkey root page আছে? 
    → HIT: buffer থেকে পড়ো
    → MISS: disk read → buffer-এ load → পড়ো
    
    B-Tree traverse: root → intermediate → leaf
    Leaf node-এ: id=42 → ctid = (page_3, slot_7)

5b. Heap fetch:
    Buffer Pool check: users page_3 আছে?
    → HIT: buffer slot থেকে পড়ো
    → MISS: disk read → buffer-এ load (8KB পুরো page)
    
    Page_3, slot_7: tuple header check
    t_xmin: committed? t_xmax: 0 (not deleted)?
    → Visible! name, email extract করো।

5c. MVCC visibility check (বিস্তারিত Topic 4-এ)

═══════════════════════════════════════════════════
STEP 6: Result → Client
═══════════════════════════════════════════════════
Row serialize করো → network protocol → client

Total time breakdown (typical):
  Network: ~1ms
  Parse+Analyze+Plan: ~0.1ms
  Execute (cache hit): ~0.1ms
  Execute (cache miss): ~10ms (disk I/O)
  
এই পুরো process একটা Backend Process একাই করে।
```

---

### না জানলে কী হয় — Real Failure Scenarios

**Scenario 1: shared_buffers ছোট → কী হয়?**
```
shared_buffers = 128MB (default), database data = 50GB

Cache hit ratio: 60% (terrible)
→ প্রতি ১০০ query-তে ৪০টা disk read
→ Disk IOPS limit hit → query latency 100x বাড়ে
→ User দেখে: "Database slow হয়ে গেছে"
→ DBA দেখে CPU ok, memory ok... কিন্তু disk I/O maxed

Fix: shared_buffers = 12GB (RAM-এর ২৫%)
Result: cache hit → 99%, disk I/O → 90% কমে
```

**Scenario 2: work_mem বেশি globally → কী হয়?**
```
work_mem = 256MB (seemingly reasonable)
max_connections = 300
peak: 200 concurrent complex queries, প্রতিটায় ৩টা Sort

RAM usage: 256MB × 3 × 200 = 153,600MB = 150GB
Available RAM: 64GB

→ Linux OOM Killer আসে
→ PostgreSQL process kill হয়
→ Database crash, সব connection হারায়
→ Production outage

Fix: work_mem = 16MB globally
     Specific analytical queries-এ: SET LOCAL work_mem = '512MB';
```

**Scenario 3: Autovacuum বন্ধ করা → কী হয়?**
```
কেউ ভাবলো: "Autovacuum server load বাড়াচ্ছে, বন্ধ করি"
autovacuum = off

৬ মাস পরে:
→ Table bloat: 100GB table এখন 800GB (dead tuples জমেছে)
→ Sequential scan: 8x slow (বেশি page পড়তে হচ্ছে)
→ XID age বাড়ছে... 1.5B... 1.8B...
→ PostgreSQL warning: "database is not accepting commands to avoid 
   wraparound data loss in database"
→ Database read-only হয়ে গেছে!
→ Emergency VACUUM FREEZE লাগবে (hours of downtime)

এটা production-এ হয়েছে বহু জায়গায়।
```

**Scenario 4: max_connections বেশি, Pooler নেই → কী হয়?**
```
max_connections = 1000
Application: connection pool নেই, প্রতি request-এ নতুন connection

1000 concurrent connections:
→ 1000 OS processes
→ ~10GB RAM শুধু process overhead
→ OS scheduler: 1000 process context switch করতে করতে busy
→ Shared memory lock contention বাড়ে
→ Throughput কমে (more connections = less performance, paradoxically)

Actual benchmark: PostgreSQL optimal at ~100-200 connections
Beyond that: throughput drops, latency rises

Fix: PgBouncer (Topic 30) → application 1000 connections দেখে,
     PostgreSQL দেখে মাত্র 50-100।
```

---

### Big Picture — এই Topic অন্যগুলোর সাথে কীভাবে Connected

```
Topic 1: Architecture (এটা)
    │
    ├──[Storage Engine]──► Topic 2: Physical Storage
    │                          Page structure, heap file, FSM, VM
    │                          "Storage Engine physically কীভাবে কাজ করে"
    │
    ├──[Memory/WAL Buffer]──► Topic 5: WAL
    │                             WAL Buffer → WAL Writer → disk
    │                             "Durability কীভাবে implement হয়"
    │
    ├──[Process Model]──► Topic 30: Connection Pooling
    │                         "কেন Pooler দরকার" এখানে বোঝা যায়
    │
    ├──[Shared Buffers]──► Topic 11: Index
    │                          Index pages-ও Buffer Pool-এ cache হয়
    │                          Buffer-এর size index effectiveness-এ impact করে
    │
    ├──[Autovacuum Process]──► Topic 15: Vacuum & Statistics
    │                              Background worker-এর কাজের detail
    │
    ├──[Transaction/MVCC]──► Topic 3: Transaction ACID
    │                             Topic 4: Concurrency & Locking
    │                             "CLOG কীভাবে ব্যবহার হয়"
    │
    └──[Checkpointer]──► Topic 24: Backup & Recovery
                             "Checkpoint কেন recovery-র জন্য critical"
```

---

## Topic 2 — How Data is Physically Stored: Pages, Blocks, Heap Files

### Big Picture: কেন Physical Storage বুঝতে হবে?

অনেক DBA জীবনে কখনো `heap_page_items()` call করেনি। তাহলে এটা কেন জানতে হবে?

কারণ যখন তুমি দেখবে:
- "Table 10GB কিন্তু মাত্র 2M rows — কেন?"
- "VACUUM করলাম কিন্তু disk space ফেরত পেলাম না"
- "Index আছে কিন্তু Index-Only Scan হচ্ছে না কেন?"
- "UPDATE করলে কেন table size বাড়ে?"

এই প্রশ্নগুলোর উত্তর Physical Storage না বুঝলে দেওয়া যায় না।

---

### Heap File — Table-এর Physical রূপ

"Heap" নামটা এসেছে data structure-এর heap concept থেকে — অর্ধেক sorted না, যেখানে জায়গা পাওয়া যায় সেখানে রাখা হয়।

```
Table: orders (physical file: base/16384/16385)

┌──────────────────────────────────────────────────────────────┐
│  Page 0 (8192 bytes)  │  Page 1 (8192 bytes)  │  Page 2 ... │
└──────────────────────────────────────────────────────────────┘

File size = page_count × 8192 bytes
1GB হলে নতুন file: 16385.1
2GB: 16385.2
(segment size: 1GB by default, compile-time configurable)
```

**কেন 8KB page size?**
```
OS virtual memory page: 4KB (x86-64)
Database page: 8KB (= 2 OS pages)

8KB কেন?
- OS I/O minimum unit = OS page = 4KB
- 4KB page: ছোট table-এর জন্য ভালো, large row-এ অনেক overflow
- 16KB+: বড় row ভালো রাখে, কিন্তু cache miss বেশি costly
- 8KB: sweet spot for OLTP workloads

PostgreSQL compile-time: --with-blocksize=8 (1, 2, 4, 8, 16, 32 KB)
MySQL InnoDB: 16KB default (larger rows, analytical workload-এ better)
```

---

### Page Structure — 8KB-এর ভেতরে কী আছে?

```
┌─────────────────────────────────────────────────────────┐  ← offset 0
│                   PAGE HEADER (24 bytes)                │
│                                                         │
│  pd_lsn        (8 bytes): Last WAL record LSN           │
│                এই page-এ শেষ পরিবর্তন কোন WAL record-এ │
│                Crash recovery-তে ব্যবহার হয়             │
│                                                         │
│  pd_checksum   (2 bytes): page checksum (if enabled)    │
│                Silent data corruption detect করে        │
│                                                         │
│  pd_flags      (2 bytes): page state flags              │
│                PD_HAS_FREE_LINES: hole আছে?             │
│                PD_ALL_VISIBLE: সব tuple visible?        │
│                                                         │
│  pd_lower      (2 bytes): free space শুরুর offset       │
│                (item pointer array-এর শেষ)              │
│                                                         │
│  pd_upper      (2 bytes): free space শেষের offset       │
│                (tuple data-এর শুরু)                     │
│                                                         │
│  pd_special    (2 bytes): special space offset          │
│                (index page-এ extra info, heap-এ = 8192) │
│                                                         │
│  pd_pagesize_version (2 bytes): page size + version     │
│  pd_prune_xid  (4 bytes): oldest dead tuple XID         │
├─────────────────────────────────────────────────────────┤  ← offset 24
│              ITEM POINTER ARRAY                         │
│              (Line Pointer Array)                       │
│                                                         │
│  [LP 1][LP 2][LP 3][LP 4][LP 5]...                     │
│                                                         │
│  প্রতিটা Line Pointer = 4 bytes:                        │
│    lp_off (15 bits): tuple-এর page-এর মধ্যে offset      │
│    lp_flags (2 bits): UNUSED/NORMAL/REDIRECT/DEAD       │
│    lp_len (15 bits): tuple-এর byte length               │
│                                                         │
│  নতুন row insert → নতুন LP add (নিচে বাড়ে ↓)           │
│                                                         │
│  pd_lower এখানে point করে                              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│                    FREE SPACE                           │
│             (pd_lower থেকে pd_upper পর্যন্ত)            │
│                                                         │
│   নতুন INSERT: tuple উপর থেকে নামে ↑                   │
│                LP নিচ থেকে উঠে ↓                       │
│   pd_lower >= pd_upper হলে page full                    │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                   TUPLE DATA                            │
│              (উপর থেকে নিচে বাড়ে ↑)                    │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │              TUPLE N (newest)                     │  │
│  │  Header (23+ bytes) | NULL bitmap | Row data      │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │              TUPLE 2                              │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │              TUPLE 1 (oldest)                     │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  pd_upper এখানে point করে                              │
├─────────────────────────────────────────────────────────┤  ← offset 8192
│         SPECIAL SPACE (index page-এর জন্য)             │
│         Heap page-এ: empty (pd_special = 8192)         │
└─────────────────────────────────────────────────────────┘
```

**কেন LP array নিচে বাড়ে এবং Tuple data উপরে বাড়ে?**
```
মাঝখানে free space রাখতে হবে।
যদি দুটোই একদিক থেকে বাড়তো, অর্ধেক page সবসময় waste হতো।
এই bidirectional growth-এ free space সবসময় একটা contiguous block।
```

---

### Tuple Header — Row-এর Metadata

```
┌──────────────────────────────────────────────────────────┐
│                   TUPLE HEADER (23 bytes minimum)        │
│                                                          │
│  t_xmin    (4 bytes): এই tuple INSERT করা transaction-এর│
│                        Transaction ID (XID)              │
│                        "কে এই row তৈরি করেছে?"          │
│                                                          │
│  t_xmax    (4 bytes): এই tuple DELETE/UPDATE করা XID    │
│                        0 = এখনো live (delete হয়নি)      │
│                        "কে এই row মেরেছে?"              │
│                                                          │
│  t_cid     (4 bytes): command ID (transaction-এর মধ্যে  │
│                        কত নম্বর command)                 │
│                                                          │
│  t_ctid    (6 bytes): এই tuple-এর physical location      │
│                        (page_number, item_number)        │
│                        UPDATE হলে নতুন tuple-কে point করে│
│                        "আমার latest version কোথায়?"    │
│                                                          │
│  t_infomask  (2 bytes): various flags                    │
│    HEAP_HASNULL: NULL values আছে?                        │
│    HEAP_HASVARWIDTH: variable-length column আছে?         │
│    HEAP_XMIN_COMMITTED: t_xmin committed?               │
│    HEAP_XMAX_COMMITTED: t_xmax committed?               │
│    HEAP_XMIN_INVALID: aborted transaction                │
│    (MVCC visibility shortcut — CLOG না দেখেই বলতে পারে) │
│                                                          │
│  t_infomask2 (2 bytes): attribute count + HOT flags      │
│  t_hoff     (1 byte): header end offset (NULL bitmap এর │
│                        জন্য variable)                    │
│                                                          │
├──────────────────────────────────────────────────────────┤
│  NULL BITMAP (variable): প্রতিটা nullable column-এ 1 bit│
│  Column i null হলে bit i = 0                            │
│  (শুধু nullable column থাকলে এই bitmap থাকে)           │
├──────────────────────────────────────────────────────────┤
│  OBJECT ID (4 bytes): শুধু WITH OIDS table-এ (obsolete)  │
├──────────────────────────────────────────────────────────┤
│  ACTUAL ROW DATA                                         │
│  Fixed-length columns: alignment rule মেনে              │
│  Variable-length: varlena header + data                  │
└──────────────────────────────────────────────────────────┘
```

**t_xmin এবং t_xmax দিয়ে MVCC কীভাবে কাজ করে (preview):**

```
Row: t_xmin=100, t_xmax=0 → Transaction 100 insert করেছে, কেউ delete করেনি

Transaction 200 এই row দেখবে?
  - t_xmin=100: XID 100 কি আমার আগে committed? হ্যাঁ → row visible
  - t_xmax=0: delete হয়নি → visible!

Transaction 200 UPDATE করলো:
  Old row: t_xmin=100, t_xmax=200  (XID 200 delete mark করলো)
  New row: t_xmin=200, t_xmax=0    (XID 200 insert করলো)

Transaction 300 (started before 200 committed):
  Old row: t_xmin=100 (committed), t_xmax=200 (NOT yet committed) → VISIBLE!
  New row: t_xmin=200 (NOT yet committed) → NOT VISIBLE

Transaction 300 (started after 200 committed):
  Old row: t_xmin=100 (committed), t_xmax=200 (committed) → NOT VISIBLE
  New row: t_xmin=200 (committed), t_xmax=0 → VISIBLE!

এভাবেই একসাথে অনেক version একই page-এ থাকতে পারে।
(বিস্তারিত Topic 4-এ)
```

---

### ctid — Row-এর Physical Address

```sql
-- ctid দেখো
SELECT ctid, id, name FROM users ORDER BY ctid LIMIT 10;

  ctid  | id |   name
--------+----+----------
 (0,1)  |  1 | Alice      ← Page 0, Item Pointer 1
 (0,2)  |  2 | Bob        ← Page 0, Item Pointer 2
 (0,3)  |  3 | Charlie
 (1,1)  |  4 | David      ← Page 1, Item Pointer 1
 (1,2)  |  5 | Eve
(2,17)  | 42 | Frank      ← Page 2, Item Pointer 17 (gap = deleted rows ছিল)
```

**ctid-এর critical role:**
```
B-Tree index leaf node-এ কী store থাকে?
  (indexed_value, ctid)

Index scan process:
  1. B-Tree-তে value খোঁজো → ctid = (page_3, slot_7) পাওয়া গেলো
  2. Heap page_3 load করো
  3. slot_7-এর tuple পড়ো
  4. Visibility check (t_xmin, t_xmax দিয়ে)
  5. Visible হলে return

ctid stable না — UPDATE করলে new ctid হয়।
তাই index-এ ctid store করা risky?
  → Index scan-এর পরে visibility check করে।
  → Stale ctid হলে → "this tuple has been updated" → t_ctid follow করে new tuple খোঁজে।
```

```sql
-- pageinspect দিয়ে actual page দেখো
CREATE EXTENSION IF NOT EXISTS pageinspect;

-- Page header দেখো
SELECT * FROM page_header(get_raw_page('users', 0));
-- lsn, checksum, flags, lower, upper, special, pagesize, version, prune_xid

-- Tuple details দেখো
SELECT
  lp,           -- line pointer number
  lp_off,       -- offset in page
  lp_flags,     -- 0=unused, 1=normal, 2=redirect, 3=dead
  lp_len,       -- tuple length
  t_xmin,       -- insert XID
  t_xmax,       -- delete XID (0 = live)
  t_ctid,       -- current ctid (points to newer version if updated)
  t_infomask,   -- flags
  t_infomask2,
  t_hoff,       -- header offset
  t_bits,       -- null bitmap
  t_oid,
  t_data        -- raw row data (hex)
FROM heap_page_items(get_raw_page('users', 0));
```

---

### UPDATE-এর Physical Reality

```
BEFORE UPDATE:
Page 0:
  LP1 → [t_xmin=100, t_xmax=0, ctid=(0,1)] | "Alice" | age=29

UPDATE users SET age=30 WHERE id=1; (Transaction XID=200)

AFTER UPDATE (same page-এ জায়গা থাকলে):
Page 0:
  LP1 → [t_xmin=100, t_xmax=200, ctid=(0,2)] | "Alice" | age=29  ← old (dead)
  LP2 → [t_xmin=200, t_xmax=0,   ctid=(0,2)] | "Alice" | age=30  ← new (live)

UPDATE-এ কী হয়:
1. Old tuple-এর t_xmax = current XID (200) set হয়
2. Old tuple-এর t_ctid → new tuple-এর location (0,2) point করে
3. New tuple insert হয় (t_xmin=200, t_xmax=0)
4. Index update: যদি indexed column বদলায় → index-এও new entry

এই কারণে:
  - UPDATE = effective DELETE + INSERT
  - Dead tuple জমতে থাকে (VACUUM দরকার)
  - Table size বাড়তে থাকে UPDATE-এও
```

**HOT Update — একটা গুরুত্বপূর্ণ Optimization:**

```
HOT = Heap-Only Tuple

শর্ত:
  1. Indexed column-এর value পরিবর্তন হয়নি (non-indexed column update)
  2. Same page-এ free space আছে (fillfactor দিয়ে ensure করা যায়)

HOT Update হলে:
  - Index update লাগে না!
  - Old LP → new tuple chain (redirect)
  - Index এখনো old LP point করে, কিন্তু LP সেটাকে নতুনটায় redirect করে

Normal Update vs HOT Update:
  Normal: table write + ALL index writes (n indexes = n+1 writes)
  HOT: শুধু table write (index write ZERO)

High UPDATE workload-এ HOT এর impact:
  - Write amplification কমে
  - WAL কম লেখা লাগে
  - Index bloat কমে

fillfactor = 70:
  Page-এর ৩০% খালি রাখে → UPDATE-এ same page-এ জায়গা থাকে → HOT possible

fillfactor = 100 (default):
  Page পুরো ভরে → UPDATE-এ same page-এ জায়গা নেই → নতুন page → NOT HOT
```

```sql
-- HOT update statistics দেখো
SELECT relname, n_tup_upd, n_tup_hot_upd,
  round(n_tup_hot_upd::numeric / nullif(n_tup_upd, 0) * 100, 2) AS hot_update_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 0
ORDER BY n_tup_upd DESC;

-- High UPDATE table-এ HOT rate কম হলে fillfactor কমিয়ে দেখো
ALTER TABLE orders SET (fillfactor = 70);
VACUUM orders;  -- existing pages-এ space তৈরি করো
```

---

### Free Space Map (FSM) — INSERT কোথায় যাবে?

```
FSM structure:
  Binary tree of free space information
  প্রতিটা leaf = একটা page-এর available free space (1 byte = 32 byte granularity)
  Parent = max of children
  Root = page with most free space

INSERT হলে:
  1. FSM root থেকে শুরু করে largest free page খোঁজো
  2. সেই page-এ space আছে কিনা confirm করো
  3. Tuple insert করো
  4. FSM update করো

VACUUM FSM update করে:
  Dead tuple সরানো → free space বাড়ে → FSM-এ update

FSM ছাড়া কী হতো?
  প্রতিটা INSERT-এ random page check করতে হতো → slow
  বা সবসময় file-এর শেষে যেতে হতো → file অনুচিত বড় হতো
```

```sql
-- FSM statistics
SELECT relpages, reltuples,
  pg_size_pretty(pg_relation_size('orders')) AS heap_size,
  pg_size_pretty(pg_relation_size('orders', 'fsm')) AS fsm_size,
  pg_size_pretty(pg_relation_size('orders', 'vm')) AS vm_size
FROM pg_class WHERE relname = 'orders';
```

---

### Visibility Map (VM) — Vacuum এবং Index-Only Scan-এর জন্য

```
VM: প্রতিটা page-এর জন্য 2 bits

Bit 1 — all-visible:
  এই page-এর সব live tuple কি সব current এবং future transaction-এর কাছে visible?
  মানে: কোনো dead tuple নেই, কোনো uncommitted transaction নেই

Bit 2 — all-frozen:
  সব tuple কি frozen? (t_xmin = FrozenXID)
  মানে: XID wraparound থেকে safe

VM-এর দুটো ব্যবহার:

1. Autovacuum optimization:
   VACUUM এ page scan করার সময় VM check করে।
   all-visible = true হলে → এই page skip (dead tuple নেই)
   শুধু all-visible = false page-গুলো process করে।
   Large stable table-এ VACUUM অনেক দ্রুত হয়।

2. Index-Only Scan:
   Index-এ সব দরকারি column আছে (covering index)।
   Heap-এ যাওয়া লাগবে না — কিন্তু!
   Tuple visible কিনা জানতে হবে (t_xmin/t_xmax check)।
   VM.all-visible = true হলে → heap visit SKIP! সরাসরি index data return।
   VM.all-visible = false হলে → heap visit করতে হবে।
```

```sql
-- Index-Only Scan হচ্ছে কিনা
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name FROM users WHERE id BETWEEN 1 AND 1000;
-- "Index Only Scan" দেখলে OK
-- "Heap Fetches: 0" → VM all-visible page skip হয়েছে (perfect)
-- "Heap Fetches: 500" → কিছু page all-visible না, heap visit করতে হয়েছে

-- VM update করতে VACUUM করো
VACUUM users;
-- এরপর Heap Fetches কমবে
```

---

### TOAST — 8KB-এর বড় Data কোথায় যায়?

```
Page size: 8KB
একটা row ≈ 8KB এর বেশি হলে?
  → TOAST: The Oversized-Attribute Storage Technique

Rule: যদি row > ~2KB হয়, variable-length column-গুলো TOAST করার চেষ্টা করে।

TOAST process:
1. Compression চেষ্টা করো (pglz বা lz4 algorithm)
   Compressed < page threshold → inline রাখো (compressed)
2. এখনো বড়? → TOAST table-এ out-of-line store করো
   Main table-এ শুধু 18-byte pointer থাকে

TOAST table structure:
  pg_toast_<OID>:
    chunk_id   int4  → কোন large value
    chunk_seq  int4  → chunk নম্বর (0, 1, 2, ...)
    chunk_data bytea → actual data chunk (default: 2KB chunks)

TOAST table-এ B-Tree index: (chunk_id, chunk_seq)
```

**TOAST Storage Strategies:**

```sql
SELECT attname, attstorage,
  CASE attstorage
    WHEN 'p' THEN 'PLAIN - never TOAST (fixed types: int, bool, etc.)'
    WHEN 'e' THEN 'EXTENDED - compress first, then out-of-line (default for text)'
    WHEN 'm' THEN 'MAIN - compress inline if possible, avoid out-of-line'
    WHEN 'x' THEN 'EXTERNAL - out-of-line without compression'
  END AS storage_desc
FROM pg_attribute
WHERE attrelid = 'products'::regclass
  AND attnum > 0
  AND NOT attisdropped;

-- Strategy change করো
ALTER TABLE products ALTER COLUMN description SET STORAGE EXTERNAL;
-- Uncompressed TOAST: compression CPU cost নেই, কিন্তু space বেশি
-- Pattern match, substring operations-এ faster
```

**TOAST Gotchas:**

```
Gotcha 1: SELECT * করলে সব TOAST column fetch হয়
  → Network overhead
  → TOAST table I/O
  Fix: SELECT only needed columns

Gotcha 2: TOAST column-এ WHERE clause
  WHERE description LIKE '%keyword%'
  → Full TOAST fetch for every row, then LIKE check
  → Full-text search (GIN index) ব্যবহার করো

Gotcha 3: TOAST chunk size এবং UPDATE
  JSONB column update করলে → পুরো JSONB TOAST-এ rewrite হয়
  Large JSONB frequent update → TOAST table bloat
  Fix: JSONB-এর মধ্যে ছোট field update করো
       অথবা separate columns

Gotcha 4: pg_dump এবং TOAST
  pg_dump TOAST data-ও dump করে।
  Large TOAST → pg_dump slow
  Large text/bytea column-এ pg_dump time estimate করো
```

```sql
-- TOAST table size দেখো
SELECT
  c.relname AS table_name,
  t.relname AS toast_table,
  pg_size_pretty(pg_total_relation_size(c.oid)) AS table_total,
  pg_size_pretty(pg_total_relation_size(t.oid)) AS toast_total
FROM pg_class c
JOIN pg_class t ON t.oid = c.reltoastrelid
WHERE c.relname = 'products';
```

---

### Column Alignment এবং Storage Size — অদৃশ্য Waste

```
CPU data read করে aligned memory address থেকে।
Unaligned read: CPU error বা multiple read cycle (slower)।
PostgreSQL column data align করে → padding bytes waste হতে পারে।

Alignment rules:
  1 byte  (bool, char): কোনো padding নেই
  2 byte  (smallint): 2-byte boundary
  4 byte  (int, float4, date): 4-byte boundary
  8 byte  (bigint, float8, timestamp): 8-byte boundary
  Variable (text, bytea): 4-byte boundary (length header)

Example — BAD column order:
CREATE TABLE bad_layout (
  a bool,        -- 1 byte  + 7 bytes PADDING (next is bigint, needs 8-byte align)
  b bigint,      -- 8 bytes
  c bool,        -- 1 byte  + 7 bytes PADDING
  d bigint,      -- 8 bytes
  e bool,        -- 1 byte  + 7 bytes PADDING (end-of-row alignment)
  f int          -- 4 bytes + 4 bytes PADDING
);
-- Per row: 1+7+8+1+7+8+1+7+4+4 = 48 bytes

Example — GOOD column order (larger types first):
CREATE TABLE good_layout (
  b bigint,      -- 8 bytes (8-byte aligned)
  d bigint,      -- 8 bytes
  f int,         -- 4 bytes
  a bool,        -- 1 byte
  c bool,        -- 1 byte
  e bool         -- 1 byte  + 1 byte trailing padding
);
-- Per row: 8+8+4+1+1+1+1 = 24 bytes
-- 50% কম! 100M rows = 2.4GB vs 4.8GB
```

```sql
-- Column alignment দেখো
SELECT attname, atttypid::regtype AS type, attalign, attlen,
  CASE attalign
    WHEN 'c' THEN '1-byte (char)'
    WHEN 's' THEN '2-byte (short)'
    WHEN 'i' THEN '4-byte (int)'
    WHEN 'd' THEN '8-byte (double)'
  END AS alignment
FROM pg_attribute
WHERE attrelid = 'orders'::regclass
  AND attnum > 0
ORDER BY attnum;
```

---

### না জানলে কী হয় — Real Scenarios

**Scenario 1: Dead tuple bloat বোঝে না**
```
Problem: orders table 500GB, কিন্তু মাত্র 50M live rows
         প্রতিটা order status বারবার update হয়।

কারণ: প্রতিটা UPDATE = old dead tuple + new tuple
       Dead tuple VACUUM না হলে জমে থাকে।
       500GB এর 450GB হয়তো dead tuples!

কীভাবে ধরলাম?
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname='orders';
-- n_dead_tup = 450M, n_live_tup = 50M

Fix: VACUUM ANALYZE orders;
     autovacuum tuning (scale_factor কমাও)
     fillfactor = 70 দিলে HOT update বাড়বে → কম dead tuple
```

**Scenario 2: Index-Only Scan কাজ করছে না**
```
Problem: Covering index আছে, কিন্তু EXPLAIN দেখাচ্ছে Heap Fetches: high

কারণ: VACUUM বহুদিন হয়নি → VM সব page all-visible mark করেনি
       → Index-Only Scan-এও heap visit করতে হচ্ছে

Fix: VACUUM ANALYZE table_name;
     Heap Fetches drop করবে।
```

**Scenario 3: TOAST এবং query performance**
```
Problem: "SELECT * FROM documents" extremely slow
          documents table মাত্র 1000 rows

কারণ: প্রতিটা row-এ large PDF content (bytea) → TOAST
       SELECT * → প্রতিটা row-এর TOAST fetch → 1000 TOAST lookups

Fix: SELECT id, title, created_at FROM documents; (content বাদ দাও)
     Content শুধু যখন দরকার তখন fetch করো।
```

---

### Big Picture Connection

```
Topic 2 (Physical Storage)
    │
    ├──[t_xmin, t_xmax]──► Topic 3 (Transaction) + Topic 4 (Concurrency/MVCC)
    │                          Dead tuple কেন তৈরি হয়, কীভাবে visible হয়
    │
    ├──[pd_lsn, page dirty]──► Topic 5 (WAL)
    │                              LSN কীভাবে WAL এবং page-কে link করে
    │
    ├──[ctid, page structure]──► Topic 11 (Index)
    │                                Index leaf-এ ctid store, heap fetch process
    │
    ├──[dead tuple, FSM, VM]──► Topic 15 (Vacuum & Statistics)
    │                               VACUUM কী পরিষ্কার করে, FSM/VM কীভাবে update করে
    │
    ├──[HOT update, fillfactor]──► Topic 8 (Table Design)
    │                                  fillfactor কীভাবে HOT rate affect করে
    │
    └──[TOAST]──► Topic 7 (Data Types)
                    কোন type TOAST হয়, storage strategy কীভাবে choose করবে
```

---

## Topic 3 — Transaction: ACID Properties

### Big Picture: Transaction ছাড়া Database কী হতো?

Transaction হলো database-এর সবচেয়ে fundamental guarantee। এটা ছাড়া:

```
Scenario: Bank transfer ₹10,000 from Account A to Account B

Step 1: Account A থেকে ১০,০০০ debit
         → Server crash হলো!
Step 2: Account B-তে ১০,০০০ credit → NEVER EXECUTED

Result: ১০,০০০ টাকা বাতাসে গেলো।
        Database-এ: A তে ১০,০০০ কম, B-তে কোনো পরিবর্তন নেই।
        Inconsistent state!

Transaction-এ:
  BEGIN;
  UPDATE accounts SET balance -= 10000 WHERE id = 'A';
  UPDATE accounts SET balance += 10000 WHERE id = 'B';
  COMMIT;

Crash হলে — দুটোই undo। Consistent state।
```

ACID চারটা guarantee দেয় যেন কখনো এই ধরনের inconsistency না হয়।

---

### A — Atomicity: সব হবে, নয়তো কিছুই না

**কীভাবে implement হয়:**

```
Atomicity implement হয় WAL দিয়ে।

COMMIT-এর flow:
  1. সব changes WAL-এ লেখা হয়
  2. WAL-এ COMMIT record লেখা হয়
  3. WAL disk-এ fsync হয় (OS buffer না, actual disk)
  4. Client "OK" পায়

Crash হলে (COMMIT record WAL-এ আছে):
  → Restart-এ WAL replay → transaction apply হয় → committed

Crash হলে (COMMIT record WAL-এ নেই):
  → Restart-এ WAL replay → transaction skip হয় → as if it never happened

ROLLBACK-এর flow:
  1. WAL-এ ABORT record লেখা হয়
  2. Heap-এর dirty pages-এ old XID-এর changes: t_xmax set → dead tuple
  3. CLOG-এ XID mark: ABORTED
  4. কোনো undo operation নেই! (MVCC-র কারণে)
     Dead tuple হলো — VACUUM পরিষ্কার করবে।
```

**PostgreSQL vs MySQL Atomicity:**

```
PostgreSQL:
  Rollback = CLOG-এ ABORTED mark
  Old data alive (as dead tuples), নতুন data visible হয় না
  Fast rollback (no undo operation)

MySQL InnoDB:
  Rollback = Undo log থেকে actual undo operation
  Physical undo: changed data-কে old value-এ ফিরিয়ে দেয়
  Large transaction rollback = large undo work = slow

এই পার্থক্যের কারণ:
  PostgreSQL MVCC: old version heap-এ থাকে, visibility XID দিয়ে control
  MySQL MVCC: undo log দিয়ে old version reconstruct করে
```

---

### C — Consistency: Database সবসময় Valid State-এ থাকে

**Consistency দুই জায়গা থেকে আসে:**

```
1. Database-enforced consistency:
   - Constraints (PK, FK, Unique, Check, Not Null)
   - Triggers
   - Rules

2. Application-enforced consistency:
   - Business logic
   - Application-level validation

Database শুধু নিজের constraints guarantee করে।
Business rule-এর consistency application-এর দায়িত্ব।
```

```sql
-- Consistency violation example:
CREATE TABLE accounts (
  id bigint PRIMARY KEY,
  balance numeric NOT NULL CHECK (balance >= 0)  -- Balance negative হবে না
);

BEGIN;
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
-- Balance 0-এর নিচে গেলে:
-- ERROR: new row for relation "accounts" violates check constraint "accounts_balance_check"
-- Transaction automatically rolls back (or you must ROLLBACK)
```

**Consistency এবং Deferred Constraints:**

```sql
-- FK constraint normally immediate:
CREATE TABLE orders (
  user_id bigint REFERENCES users(id)
  -- DEFERRABLE INITIALLY IMMEDIATE (default)
);

-- Deferred করলে transaction শেষে check হয়:
CREATE TABLE orders (
  user_id bigint REFERENCES users(id)
  DEFERRABLE INITIALLY DEFERRED
);

-- Use case: circular reference বা batch insert
BEGIN;
INSERT INTO orders (id, user_id) VALUES (1, 100);  -- user 100 নেই এখনো
INSERT INTO users (id, name) VALUES (100, 'Alice'); -- এখন তৈরি
COMMIT;  -- এখন FK check → pass!
```

---

### I — Isolation: Concurrent Transactions একে অপরকে Interfere করবে না

**Isolation SQL Standard-এ 4 level define করে। তিনটা problem prevent করে:**

```
Problem 1: Dirty Read
  Transaction A uncommitted data লিখেছে।
  Transaction B সেটা পড়লো।
  Transaction A rollback করলো।
  → B ভুল data দিয়ে কাজ করেছে!

Problem 2: Non-Repeatable Read
  Transaction A: SELECT age → 29
  Transaction B: UPDATE age = 30; COMMIT;
  Transaction A: SELECT age → 30  (same transaction-এ different result!)
  → A-এর within-transaction consistency নষ্ট

Problem 3: Phantom Read
  Transaction A: SELECT COUNT(*) WHERE age > 25 → 10
  Transaction B: INSERT new row WHERE age = 30; COMMIT;
  Transaction A: SELECT COUNT(*) WHERE age > 25 → 11  (phantom row!)
  → New row appeared

Problem 4 (SQL standard-এ নেই): Serialization Anomaly
  Concurrent transactions যদি serial order-এ চলতো তাহলে যে result হতো
  সেটার সাথে concurrent result মিলছে না।
```

**PostgreSQL Isolation Levels:**

```
READ UNCOMMITTED → PostgreSQL-এ READ COMMITTED-এ উন্নীত হয়
READ COMMITTED   → Default। Dirty read prevent। Statement-level snapshot।
REPEATABLE READ  → Non-repeatable read prevent। Transaction-level snapshot।
SERIALIZABLE     → Phantom read prevent। SSI algorithm।

Level              Dirty Read  Non-Repeatable  Phantom  Serialization
READ UNCOMMITTED   Possible*   Possible        Possible Possible
READ COMMITTED     Prevented   Possible        Possible Possible
REPEATABLE READ    Prevented   Prevented       Possible* Possible
SERIALIZABLE       Prevented   Prevented       Prevented Prevented

* PostgreSQL-এ REPEATABLE READ-এও phantom read prevent হয়
  (snapshot isolation-এর কারণে)
  MySQL REPEATABLE READ-এ next-key lock দিয়ে prevent করে (different approach)
```

**READ COMMITTED এর Detail:**

```sql
-- READ COMMITTED: প্রতিটা statement নতুন snapshot নেয়

Session A:                        Session B:
BEGIN;                            BEGIN;
SELECT count(*) FROM orders;      -- 1000
                                  INSERT INTO orders ...; -- 1001 rows
                                  COMMIT;
SELECT count(*) FROM orders;      -- 1001 (sees B's commit!)
-- Same transaction, different result!
-- কারণ: প্রতিটা SELECT নতুন snapshot = committed data দেখে
COMMIT;

এই behavior কখন problem?
  - Report generation: পুরো report-এ consistent data দরকার
  - Aggregation: SUM, COUNT-এ inconsistent result
  Fix: REPEATABLE READ ব্যবহার করো
```

**REPEATABLE READ এর Detail:**

```sql
-- REPEATABLE READ: transaction শুরুতে একটা snapshot, সব statement same snapshot

Session A (REPEATABLE READ):     Session B:
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM orders;     -- 1000
                                 INSERT INTO orders ...; COMMIT;
SELECT count(*) FROM orders;     -- এখনো 1000! B-এর insert দেখছে না

-- কিন্তু নিজের changes দেখতে পাবে:
UPDATE orders SET status='done' WHERE id=1;
SELECT status FROM orders WHERE id=1;  -- 'done' দেখবে
COMMIT;
```

**SERIALIZABLE — SSI (Serializable Snapshot Isolation):**

```
PostgreSQL 9.1+ এ Serializable Snapshot Isolation।
True serializability, but without lock-based blocking।

SSI tracks:
  - কে কোন data পড়েছে (read set)
  - কে কোন data লিখেছে (write set)
  - Read-write dependency cycles detect করে

Cycle detect হলে → একটা transaction abort করে দেয়।
Application retry করে।

Performance cost: more bookkeeping।
Use when: financial, inventory — correctness > performance।
```

---

### D — Durability: COMMIT হলে Data থাকবে, Crash হলেও

**কীভাবে implement হয়:**

```
Durability = WAL + fsync

COMMIT flow:
  1. WAL Buffer → WAL file
  2. fsync() call: OS kernel buffer → physical disk
  3. তারপরই client "COMMIT" পায়

fsync() ছাড়া:
  Data OS memory-তে আছে, disk-এ নেই।
  Power outage → data lost (even though COMMIT returned OK)

PostgreSQL fsync settings:
  fsync = on         → true durability (default, recommended)
  fsync = off        → NEVER in production (data corruption risk)
  
  synchronous_commit = on     → WAL disk-এ যাওয়ার পরে COMMIT return
  synchronous_commit = off    → WAL disk-এ যাওয়ার আগেই COMMIT return
                                 crash হলে শেষ কয়েকটা commit হারাতে পারো
                                 কিন্তু corruption হবে না (DB consistent থাকে)
```

**`synchronous_commit = off` — কখন safe?**

```
Safe:
  - Logging table (access log, audit event)
  - Real-time analytics event
  - Session/cache data
  - যেখানে কয়েকটা row হারানো acceptable

Never safe:
  - Financial transaction
  - Inventory/order data
  - User account data
  - যেকোনো business-critical write

Performance gain: ~30% throughput increase (no fsync per commit)
Risk: শেষ ~200ms-এর commits হারাতে পারো (wal_writer_delay)

Per-transaction basis-এ set করা যায়:
  SET LOCAL synchronous_commit = off;  -- শুধু এই transaction-এ
  INSERT INTO access_logs ...;         -- এই insert async হবে
  COMMIT;
```

---

### Transaction State Machine

```
                          BEGIN
                            │
                            ▼
                     ┌─────────────┐
              ┌──────│    ACTIVE   │──────┐
              │      └──────┬──────┘      │
              │             │ SQL stmt    │
              │             ▼             │
              │      ┌─────────────┐      │
              │      │   ACTIVE    │      │
              │      │  (in stmt)  │      │
              │      └──────┬──────┘      │
              │             │             │
         ERROR│             │COMMIT       │ROLLBACK
              │             │             │
              ▼             ▼             ▼
       ┌────────────┐ ┌──────────┐ ┌──────────┐
       │   ERROR    │ │COMMITTED │ │ ABORTED  │
       │  (aborted) │ │          │ │          │
       └────────────┘ └──────────┘ └──────────┘
              │             │             │
              └─────────────┴─────────────┘
                            │
                         END (connection continues)
```

**Error state-এর subtlety:**

```sql
BEGIN;
INSERT INTO users VALUES (1, 'Alice');
INSERT INTO users VALUES (1, 'Bob');  -- Duplicate key error!
-- এখন transaction "aborted" state-এ
-- যেকোনো SQL → "ERROR: current transaction is aborted, 
--               commands ignored until end of transaction block"

-- Option 1: ROLLBACK করো, নতুন transaction শুরু করো
ROLLBACK;

-- Option 2: Savepoint ব্যবহার করো (partial rollback)
BEGIN;
  INSERT INTO users VALUES (1, 'Alice');
  SAVEPOINT before_bob;
  INSERT INTO users VALUES (1, 'Bob');  -- Error!
  ROLLBACK TO SAVEPOINT before_bob;     -- শুধু এই পর্যন্ত rollback
  INSERT INTO users VALUES (2, 'Bob');  -- এটা হবে
COMMIT;
-- users তে: (1, Alice), (2, Bob) — দুটোই
```

---

### DDL Transactions — PostgreSQL vs MySQL-এর বড় পার্থক্য

```
PostgreSQL-এ DDL transactional:

BEGIN;
  CREATE TABLE backup_orders AS SELECT * FROM orders;  -- DDL
  DELETE FROM orders WHERE created_at < '2023-01-01'; -- DML
  -- কিছু problem হলে:
ROLLBACK;
-- backup_orders তৈরিই হয়নি, orders delete হয়নি

This is HUGE for zero-downtime migrations:
  BEGIN;
    ALTER TABLE orders ADD COLUMN notes text;
    UPDATE orders SET notes = 'migrated';
    -- validation করো...
  COMMIT; -- বা ROLLBACK যদি কিছু ভুল হয়


MySQL/InnoDB-এ DDL implicit COMMIT করে:

START TRANSACTION;
INSERT INTO orders VALUES (...);   -- ১ম insert
CREATE TABLE test (id int);         -- Implicit COMMIT! INSERT-ও committed
ROLLBACK;                           -- কাজ করবে না, already committed

Migration script-এ এই behavior catastrophic হতে পারে।
PostgreSQL-এ migration tool (Flyway, Liquibase) এই transactional DDL ব্যবহার করে।
```

---

### Implicit Transaction vs Explicit Transaction

```sql
-- Auto-commit mode (default psql):
UPDATE users SET name = 'Bob' WHERE id = 1;
-- Internally:
-- BEGIN (implicit)
-- UPDATE users SET name = 'Bob' WHERE id = 1;
-- COMMIT (implicit)

-- Explicit:
BEGIN;
UPDATE users SET name = 'Bob' WHERE id = 1;
-- ... validation ...
COMMIT; -- or ROLLBACK

-- psql-এ auto-commit বন্ধ করো (development-এ useful):
\set AUTOCOMMIT off
-- এখন প্রতিটা command-এর পরে explicit COMMIT দিতে হবে
-- ভুলে গেলে? Session-end-এ implicit ROLLBACK
```

**Long transaction — production killer:**

```sql
-- কেউ BEGIN করে COMMIT ভুলে গেলে:
BEGIN;
SELECT * FROM large_table;  -- Snapshot ধরে রাখলো
-- ... ঘণ্টার পর ঘণ্টা কিছু করছে না ...

এই transaction চলাকালীন:
  - Snapshot ধরে আছে
  - Dead tuple VACUUM করতে পারছে না (transaction দেখছে পুরানো snapshot)
  - Table bloat বাড়ছে
  - VACUUM ineffective

Monitor করো:
SELECT pid, now() - xact_start AS duration, state, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY duration DESC;

-- 5 minute-এর বেশি idle-in-transaction:
SELECT pid, xact_start, state, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '5 minutes';

-- Kill করো:
SELECT pg_terminate_backend(pid);

-- Prevention:
idle_in_transaction_session_timeout = '10min'  -- postgresql.conf
-- 10 minute idle-in-transaction হলে auto-kill
```

---

### না জানলে কী হয়

**Scenario 1: fsync=off in production**
```
কেউ "performance বাড়াতে" fsync=off করলো।
Benchmark: 5x faster!
Production-এ: power outage হলো।
Restart: data file এবং WAL-এর মধ্যে inconsistency।
PostgreSQL corrupt database detect করে start refuse করতে পারে।
অথবা: silent data corruption — কিছু page wrong data।
Fix: pg_resetwal দিয়ে WAL reset (data loss!) অথবা backup থেকে restore।

fsync=off শুধু: throwaway test environment, benchmark, CI/CD (disposable data)।
```

**Scenario 2: MySQL-background থেকে আসা developer DDL transaction ভুলে গেলো**
```
Python script:
  conn.autocommit = False
  conn.execute("BEGIN")
  conn.execute("INSERT INTO orders ...")
  conn.execute("CREATE TABLE backup ...")  # PostgreSQL-এ OK, MySQL-এ implicit COMMIT
  conn.execute("DELETE FROM orders WHERE ...")
  conn.execute("ROLLBACK")  # MySQL-এ: backup table still exists!
  
MySQL background থেকে PostgreSQL-এ migrate করার সময় এই ধরনের bug ধরতে হয়।
```

---

### Big Picture Connection

```
Topic 3 (Transaction/ACID)
    │
    ├──[Atomicity, WAL]──► Topic 5 (WAL)
    │                          WAL কীভাবে Atomicity implement করে
    │
    ├──[Isolation, MVCC]──► Topic 4 (Concurrency & Locking)
    │                            Isolation level কীভাবে implement হয়
    │
    ├──[Consistency, Constraints]──► Topic 9 (Constraints)
    │                                    Check, FK, Unique কীভাবে consistency enforce করে
    │
    ├──[Durability, WAL]──► Topic 24 (Backup) + Topic 25 (PITR)
    │                           WAL = durability এবং recovery-র backbone
    │
    └──[DDL Transaction]──► Topic 6 (Schema Design)
                                Migration strategy-তে transactional DDL কেন গুরুত্বপূর্ণ
```

---

## Topic 4 — Concurrency & Locking

### Big Picture: Concurrent Access এর চ্যালেঞ্জ

Database-এ হাজারো connection একসাথে কাজ করছে। কেউ পড়ছে, কেউ লিখছে, কেউ update করছে। কোনো control না থাকলে:

```
Scenario: দুটো ticket booking system একই seat book করছে

T1: SELECT remaining_seats → 1
T2: SELECT remaining_seats → 1
T1: UPDATE SET remaining_seats = 0; COMMIT;  -- seat booked
T2: UPDATE SET remaining_seats = 0; COMMIT;  -- ALSO booked!
Result: একটা seat দুইজনকে দেওয়া হলো → oversold!
```

Concurrency control এই সমস্যা solve করে।

PostgreSQL দুটো approach combine করে:
1. **MVCC** — Read এবং Write একে অপরকে block করে না
2. **Locking** — Write conflicts handle করে

---

### MVCC — Multi-Version Concurrency Control

**Core idea:** Data-এর একাধিক version একসাথে exist করে। প্রতিটা transaction নিজের "snapshot" দেখে।

```
Physical reality (heap-এ):
┌──────────────────────────────────────────────────────────────┐
│ Page 5:                                                      │
│   Tuple 1: t_xmin=100, t_xmax=200, data="price=100"         │
│            (XID 100 insert, XID 200 delete করেছে)           │
│   Tuple 2: t_xmin=200, t_xmax=0,   data="price=150"         │
│            (XID 200 insert করেছে, এখনো live)                │
└──────────────────────────────────────────────────────────────┘

Transaction A (snapshot at XID 150, before XID 200 committed):
  Query: SELECT price → Tuple 1 visible (xmin=100 committed, xmax=200 NOT committed)
  Result: price = 100

Transaction C (snapshot at XID 250, after XID 200 committed):
  Query: SELECT price → Tuple 2 visible (xmin=200 committed, xmax=0)
  Result: price = 150

Same time, same table, different results — by design!
```

**MVCC Snapshot কীভাবে কাজ করে:**

```
Snapshot = (xmin, xmax, active_xids)

xmin: snapshot নেওয়ার সময় committed সর্বোচ্চ XID
xmax: snapshot নেওয়ার সময় চলমান সর্বোচ্চ XID
active_xids: snapshot নেওয়ার সময় চলমান সব XID

Visibility rule:
  Tuple visible যদি:
  1. t_xmin < snapshot.xmin AND t_xmin NOT IN active_xids (committed before snapshot)
     AND (t_xmax = 0 OR t_xmax >= snapshot.xmax OR t_xmax IN active_xids)
  
  Simplified:
  - Insert XID committed এবং snapshot-এর আগে
  - Delete XID হয় 0 (live), অথবা uncommitted (at snapshot time), অথবা snapshot-এর পরে
```

**MVCC-র কারণে Reader কখনো Writer-কে block করে না:**

```
MySQL (without MVCC, MyISAM):
  Writer holds table lock → all readers wait

PostgreSQL (MVCC):
  Writer: নতুন tuple insert (তার XID-এ)
  Reader: পুরানো committed version দেখে
  No blocking!

MySQL InnoDB-তেও MVCC আছে (undo log দিয়ে)।
কিন্তু implementation আলাদা:
  PostgreSQL: old version heap-এ (dead tuple as overhead)
  MySQL: undo log-এ (cleanup পরে হয়, কিন্তু undo log size বাড়তে পারে)
```

---

### Transaction ID (XID) — MVCC-র Heart

```
XID: 32-bit unsigned integer (PostgreSQL)
  Range: 0 to 4,294,967,295 (~4.3 billion)

Special XIDs:
  0: Invalid XID
  1: Bootstrap XID
  2: Frozen XID (FrozenTransactionId) — "so old, always visible"
  3: First normal XID

XID assignment:
  প্রতিটা transaction (write transaction) একটা unique XID পায়।
  Read-only transaction: XID নাও পেতে পারে (optimization)।
  
Current XID দেখো:
SELECT txid_current();         -- current transaction XID
SELECT txid_current_snapshot(); -- current snapshot info
```

**XID Wraparound — সবচেয়ে ভয়ের scenario:**

```
XID space: 0 → 4.3 billion → 0 (circular)

কিন্তু PostgreSQL comparison করে modular arithmetic-এ:
  XID-এর "পাশে" ২.১ বিলিয়ন XID "future" (invisible)
  অন্য ২.১ বিলিয়ন "past" (visible)

XID wraparound problem:
  ধরো current XID = 3,000,000,000
  একটা tuple-এর t_xmin = 500,000,000
  
  Age = 3B - 500M = 2.5B > 2.1B limit!
  PostgreSQL এই tuple-কে "future transaction" মনে করে → INVISIBLE!
  Existing data disappear করতে পারে!

Protection: VACUUM FREEZE
  Dead tuple-কে frozen করে (t_xmin = FrozenXID = 2)
  Frozen tuple: সবার কাছে visible, forever
  Frozen হলে আর XID age check দরকার নেই

Monitor:
SELECT datname, age(datfrozenxid) AS db_age
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

SELECT relname, age(relfrozenxid) AS table_age
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 20;

Alert thresholds:
  age > 1,500,000,000: Warning — vacuum ASAP
  age > 1,800,000,000: Critical — emergency vacuum
  age > 2,000,000,000: Database forced to read-only mode!
```

---

### Lock Types — PostgreSQL-এর Lock Hierarchy

**Table-Level Locks (8 levels):**

```
Weakest ─────────────────────────────────────────► Strongest

ACCESS SHARE          (SELECT)
ROW SHARE             (SELECT FOR UPDATE/SHARE)
ROW EXCLUSIVE         (INSERT, UPDATE, DELETE)
SHARE UPDATE EXCLUSIVE (VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY)
SHARE                 (CREATE INDEX)
SHARE ROW EXCLUSIVE   (CREATE TRIGGER, some ALTER TABLE)
EXCLUSIVE             (rare, mostly internal)
ACCESS EXCLUSIVE      (ALTER TABLE, DROP TABLE, VACUUM FULL, LOCK TABLE)

Conflict matrix: (✗ = conflict, ✓ = compatible)
           AS   RS   RE  SUE   S  SRE  EX  AE
AS          ✓    ✓    ✓    ✓   ✓    ✓   ✓   ✗
RS          ✓    ✓    ✓    ✓   ✓    ✓   ✗   ✗
RE          ✓    ✓    ✓    ✓   ✗    ✗   ✗   ✗
SUE         ✓    ✓    ✓    ✗   ✗    ✗   ✗   ✗
S           ✓    ✓    ✗    ✗   ✓    ✗   ✗   ✗
SRE         ✓    ✓    ✗    ✗   ✗    ✗   ✗   ✗
EX          ✓    ✗    ✗    ✗   ✗    ✗   ✗   ✗
AE          ✗    ✗    ✗    ✗   ✗    ✗   ✗   ✗
```

**Critical: ALTER TABLE = ACCESS EXCLUSIVE lock**

```sql
-- Production-এ ALTER TABLE করার আগে:

-- Step 1: Long-running queries আছে কিনা দেখো
SELECT pid, now() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Step 2: Lock timeout set করো
SET lock_timeout = '5s';  -- ৫ সেকেন্ড-এর বেশি অপেক্ষা না করে fail করো

-- Step 3: statement timeout
SET statement_timeout = '30s';

-- এখন ALTER করো
ALTER TABLE orders ADD COLUMN notes text;

-- যদি lock না পাওয়া যায়: ERROR: canceling statement due to lock timeout
-- → retry করো, off-peak time-এ করো
```

**Lock Queue এবং Starvation:**

```
┌──────────────────────────────────────────────┐
│ T1 (SELECT): ACCESS SHARE lock ✓ acquired    │
│                                              │
│ T2 (ALTER TABLE): ACCESS EXCLUSIVE → WAITING │
│ (T1 এর lock release-এর জন্য)                 │
│                                              │
│ T3 (SELECT): ACCESS SHARE → WAITING          │
│ (T2 এর জন্য wait করছে!)                     │
│                                              │
│ T4 (SELECT): ACCESS SHARE → WAITING          │
└──────────────────────────────────────────────┘

T2 (ALTER TABLE) পুরো system block করে দিয়েছে!
T1 চলছে, T3, T4 এমনকি simple SELECT করতে পারছে না।

কারণ: Lock queue-এ T2 আছে, T3/T4 T2-এর পরে wait করছে।
       নতুন ACCESS SHARE lock T2-এর আগে যেতে পারবে না।

Real-world scenario:
  - বড় long-running query চলছে (T1)
  - কেউ ALTER TABLE করতে গেলো (T2) → lock নিতে পারছে না, waiting
  - এখন সব নতুন SELECT-ও block (T3, T4...)
  - Application unresponsive মনে হচ্ছে!

Solution:
  1. lock_timeout: দ্রুত fail করো
  2. ALTER TABLE-এর আগে long-running query kill করো
  3. Zero-downtime migration tools (pg_repack, online ALTER alternatives)
```

---

### Row-Level Locks

```sql
-- Explicit row lock
SELECT * FROM orders WHERE id = 5 FOR UPDATE;
-- row 5 lock করলো, অন্য transaction UPDATE করতে পারবে না

-- Shared row lock
SELECT * FROM orders WHERE id = 5 FOR SHARE;
-- অন্যরাও FOR SHARE নিতে পারবে, কিন্তু UPDATE/DELETE পারবে না

-- NOWAIT: lock না পেলে immediately fail করো
SELECT * FROM orders WHERE id = 5 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "orders"
-- Application retry logic দিয়ে handle করো

-- SKIP LOCKED: lock করা rows skip করো (queue/job processing pattern)
SELECT * FROM jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;
-- Lock থাকা jobs বাদ দিয়ে available 10টা নাও

-- Multiple workers একসাথে চালানো যায়!
-- Worker 1: rows 1-10 নিলো
-- Worker 2: rows 11-20 নিলো (rows 1-10 skip)
-- কোনো overlap নেই, কোনো deadlock নেই
```

---

### Deadlock

```
Deadlock:
T1 locks row A, then wants row B
T2 locks row B, then wants row A
→ দুজনই infinite wait

PostgreSQL deadlock detection:
  deadlock_timeout (default 1s) পর পর cycle detect algorithm চলে
  Cycle found → একটা transaction abort করা হয় (younger one usually)
  Aborted transaction: ERROR: deadlock detected
                       DETAIL: Process 12345 waits for ShareLock on transaction 67890
  
Application: retry on deadlock error।
```

**Deadlock Prevention Pattern:**

```sql
-- Pattern: সবসময় consistent order-এ lock নাও

-- ভুল (deadlock possible):
-- T1: lock account 1, then lock account 2
-- T2: lock account 2, then lock account 1

-- ঠিক (consistent order):
SELECT * FROM accounts
WHERE id = ANY(ARRAY[1, 2])
ORDER BY id  -- ascending order
FOR UPDATE;
-- T1 এবং T2 দুজনেই প্রথমে account 1, তারপর 2 lock করবে
-- কোনো deadlock possible না

-- Another pattern: SELECT FOR UPDATE with specific ordering
WITH locked_accounts AS (
  SELECT id, balance FROM accounts
  WHERE id IN (source_id, dest_id)
  ORDER BY id  -- consistent order!
  FOR UPDATE
)
UPDATE accounts SET balance = ...;
```

---

### Advisory Locks — Application-Level Coordination

```sql
-- Distributed lock (e.g., prevent duplicate cron job execution)

-- Session-level (manually release করতে হয়)
SELECT pg_advisory_lock(12345);
-- ... critical section ...
SELECT pg_advisory_unlock(12345);

-- Transaction-level (auto-release on commit/rollback)
SELECT pg_advisory_xact_lock(12345);
-- ... critical section ...
COMMIT; -- auto-unlock

-- Try (non-blocking)
SELECT pg_try_advisory_lock(12345);
-- true: got the lock
-- false: someone else has it

-- Named lock (text-based, hashed)
SELECT pg_advisory_lock(hashtext('my_job_name'));

-- Use case: Batch job deduplication
DO $$
BEGIN
  IF pg_try_advisory_lock(1001) THEN
    -- Got lock, proceed with job
    PERFORM process_pending_orders();
    PERFORM pg_advisory_unlock(1001);
  ELSE
    RAISE NOTICE 'Job already running, skipping';
  END IF;
END;
$$;
```

---

### Monitoring Locks

```sql
-- Current locks
SELECT
  locktype,
  relation::regclass,
  mode,
  granted,
  pid,
  pg_stat_activity.query,
  now() - pg_stat_activity.query_start AS duration
FROM pg_locks
JOIN pg_stat_activity USING (pid)
WHERE NOT granted OR mode LIKE '%Exclusive%'
ORDER BY duration DESC;

-- Lock waits (who is blocking whom)
SELECT
  blocked.pid AS blocked_pid,
  blocked.query AS blocked_query,
  blocking.pid AS blocking_pid,
  blocking.query AS blocking_query,
  now() - blocked.query_start AS wait_duration
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Lock wait graph (PostgreSQL 14+)
SELECT * FROM pg_stat_activity
WHERE wait_event_type = 'Lock'
ORDER BY query_start;
```

---

### না জানলে কী হয়

**Scenario 1: REPEATABLE READ না বুঝে inconsistent report**
```
Financial report script (READ COMMITTED):
  Query 1: Total revenue = SELECT SUM(amount) FROM orders → 1,000,000
  -- এর মধ্যে নতুন order আসে: 50,000
  Query 2: Order count = SELECT COUNT(*) FROM orders → 1001
  
Report: Revenue 1,000,000 কিন্তু orders 1001
        (১০০১ নম্বর order revenue-এ নেই!)
        
Fix: BEGIN ISOLATION LEVEL REPEATABLE READ;
     ... সব queries ...
     COMMIT;
```

**Scenario 2: FOR UPDATE না করায় race condition**
```
Ticket booking:
  T1: SELECT seats_available → 1
  T2: SELECT seats_available → 1
  T1: UPDATE SET seats_available = 0; COMMIT;  -- OK
  T2: UPDATE SET seats_available = 0; COMMIT;  -- ALSO "OK"!
  Result: Oversold!

Fix:
  SELECT seats_available FROM events WHERE id=1 FOR UPDATE;
  -- T2 এখন wait করবে T1-এর commit-এর জন্য
  -- T1 commit করলে T2 দেখবে seats_available = 0
  -- T2: IF seats_available > 0 THEN book ELSE error
```

---

## Topic 5 — WAL (Write-Ahead Logging)

### Big Picture: WAL কেন এত Critical?

WAL database-এর সবচেয়ে important reliability mechanism। এটা ছাড়া:
- Crash recovery সম্ভব না
- Replication সম্ভব না
- PITR সম্ভব না

WAL-এর fundamental principle: **"Write the log before the data"**

```
Without WAL:
  UPDATE users SET name='Bob' WHERE id=1;
  1. Shared buffer-এ page load করো
  2. Page modify করো (name='Bob')
  3. Disk-এ লেখো
  
  Step 2 এবং 3-এর মাঝে crash হলে?
  → Page আধা-লেখা (torn page)
  → Data corrupt!

With WAL:
  UPDATE users SET name='Bob' WHERE id=1;
  1. Shared buffer-এ page load করো
  2. WAL Buffer-এ change record করো
     "XID 200, relation users, page 5, old='Alice', new='Bob'"
  3. Page modify করো (buffer-এ, dirty)
  4. [Later] WAL → disk (WAL file)
  5. [Even later] Dirty page → disk (data file)
  
  কোনো step-এ crash হলেও:
  WAL disk-এ আছে → restart-এ replay → consistent state
```

---

### WAL-এর Physical Structure

```
$PGDATA/pg_wal/ directory:

000000010000000000000001  (16MB WAL segment file)
000000010000000000000002
000000010000000000000003
...

Naming convention: TimelineID + High 32 bits of LSN + Low 32 bits of LSN

File name থেকে LSN বের করা:
  000000010000000000000003 → file offset 0 = LSN 0/3000000

প্রতিটা file:
  16MB = 16 × 1024 × 1024 bytes
  (compile-time: --with-wal-segsize=16, 1-1024 MB possible)
```

**WAL Record Structure:**

```
WAL Record:
┌─────────────────────────────────────────────────────────┐
│                    RECORD HEADER                        │
│  xl_tot_len  (4 bytes): total record length             │
│  xl_xid      (4 bytes): transaction XID                 │
│  xl_prev     (8 bytes): previous WAL record LSN         │
│              (linked list backward chaining)            │
│  xl_info     (1 byte): record type flags                │
│  xl_rmid     (1 byte): Resource Manager ID              │
│              (HEAP, INDEX, XACT, SMGR, etc.)            │
│  xl_crc      (4 bytes): CRC32 of record                │
├─────────────────────────────────────────────────────────┤
│                    RECORD BODY                          │
│  Variable data depending on record type:                │
│                                                         │
│  HEAP_INSERT:                                           │
│    relation OID, page number, offset                    │
│    new tuple data (full or diff)                        │
│                                                         │
│  HEAP_UPDATE:                                           │
│    old ctid, new ctid                                   │
│    old tuple (for rollback) OR just new tuple           │
│    (wal_log_hints বা full_page_writes এর উপর নির্ভর)   │
│                                                         │
│  XACT_COMMIT:                                           │
│    XID, commit timestamp, subtransaction XIDs           │
│                                                         │
│  CHECKPOINT:                                            │
│    checkpoint location, redo point, etc.               │
└─────────────────────────────────────────────────────────┘
```

**LSN (Log Sequence Number):**

```
LSN = 64-bit integer, byte offset in WAL stream
Format: A/B where A = high 32 bits, B = low 32 bits (hex)
Example: 0/3FA2B80 = byte 66,707,328 in WAL stream

LSN monotonically increases।
প্রতিটা WAL write LSN advance করে।

Page header-এ pd_lsn = এই page-এর শেষ WAL write-এর LSN।
Crash recovery-তে: page-এর pd_lsn < WAL record LSN → apply করো।
                    page-এর pd_lsn >= WAL record LSN → skip (already applied)।
```

```sql
-- LSN operations
SELECT pg_current_wal_lsn();          -- current WAL position
SELECT pg_wal_lsn_diff(
  pg_current_wal_lsn(),
  '0/3FA2B80'
) AS bytes_since;                      -- কত bytes ago

-- Replication lag (LSN difference):
SELECT pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- WAL files list করো
SELECT * FROM pg_ls_waldir() ORDER BY modification DESC LIMIT 10;
```

---

### WAL-এর পুরো Flow — Step by Step

```
Transaction: UPDATE orders SET status='shipped' WHERE id=42;

═══════════════════════════════════════════════════════════
Phase 1: Transaction Active
═══════════════════════════════════════════════════════════

1.1 Heap page load:
    orders table, page containing id=42 → Shared Buffer
    (cache miss হলে disk read)

1.2 Page modify (in buffer):
    Old tuple: t_xmax = current XID (marking delete)
    New tuple: insert with t_xmin = current XID

1.3 WAL Buffer-এ write:
    WAL record: "XID=500, Heap Update, page (5,7), 
                 old: status='pending', new: status='shipped'"
    
    full_page_write = on হলে (default):
    প্রথম modification-এ পুরো page WAL-এ লেখে।
    কেন? Torn page protection: checkpoint-এর পরে প্রথম write-এ
    crash হলে page আধা লেখা থাকতে পারে।
    Full page image WAL-এ থাকলে → replay-এ full page restore।

═══════════════════════════════════════════════════════════
Phase 2: COMMIT
═══════════════════════════════════════════════════════════

2.1 WAL Buffer-এ COMMIT record write:
    "XID=500, COMMIT, timestamp=..."

2.2 WAL Buffer → WAL file (disk):
    wal_writer_delay বা synchronous_commit=on হলে immediate fsync
    
    fsync() = OS-কে বলছি: "তুমি যা buffer করেছো, এখনই disk-এ লেখো"
    এটা না হলে power cut-এ OS buffer হারায়।

2.3 Client: "COMMIT" response পায়।
    Data এখনো Shared Buffer-এ (dirty page)।
    Disk-এর data file-এ এখনো পুরানো value!
    But WAL-এ সব আছে। Crash হলে WAL থেকে recover হবে।

═══════════════════════════════════════════════════════════
Phase 3: Background Flush (async)
═══════════════════════════════════════════════════════════

3.1 bgwriter বা Checkpointer:
    Dirty pages → data files (disk)
    এটা background-এ হয়, client wait করে না।

3.2 Checkpoint:
    "LSN X পর্যন্ত সব dirty page disk-এ লেখা হয়েছে"
    WAL record: CHECKPOINT at LSN X

3.3 Archiver (if enabled):
    WAL segments → archive location (S3, NFS, etc.)
    PITR-এর জন্য।
```

---

### Checkpoint — WAL Management-এর Key

```
Checkpoint = sync point
"এই LSN পর্যন্ত সব পরিবর্তন data file-এ আছে"

Checkpoint-এর কাজ:
1. সব dirty shared buffer pages → disk (data files)
2. pg_wal/-এ CHECKPOINT record লেখো
3. এই LSN-এর আগের WAL → আর দরকার নেই (delete বা archive)

Checkpoint trigger:
  - checkpoint_timeout (default 5 min): নির্দিষ্ট সময় পর পর
  - max_wal_size (default 1GB): WAL এই size ছাড়ালে forced checkpoint
  - Manual: CHECKPOINT; command
  - pg_basebackup শুরুতে

Checkpoint I/O spike সমস্যা:
  Checkpoint = অনেক dirty page একসাথে disk-এ লেখে।
  → Sudden I/O spike → query latency বাড়ে।

Solution: checkpoint_completion_target = 0.9 (default 0.5→0.9 in PG 14+)
  Checkpoint interval-এর ৯০% সময়ে ছড়িয়ে লেখো।
  I/O spread out → spike কমে।

Checkpoint monitoring:
SELECT checkpoints_timed, checkpoints_req,
  checkpoint_write_time, checkpoint_sync_time,
  buffers_checkpoint, buffers_clean, buffers_backend,
  maxwritten_clean
FROM pg_stat_bgwriter;

-- checkpoints_req বেশি → max_wal_size বাড়াও
-- buffers_backend বেশি → bgwriter tuning দরকার
```

---

### full_page_writes — Torn Page Protection

```
full_page_writes = on (default, recommended)

কেন দরকার?
OS write-এর minimum unit: 512 bytes বা 4096 bytes (sector size)
Database page: 8192 bytes = 2-16 OS sectors

Write করার সময় crash হলে:
  প্রথম 4096 bytes নতুন, বাকি 4096 bytes পুরানো → TORN PAGE!
  WAL record শুধু diff থাকলে → corrupt page-এ apply করলে → wrong result!

full_page_writes = on solution:
  Checkpoint-এর পরে একটা page-এর প্রথম modification-এ:
  পুরো page (8KB) WAL-এ লেখো।
  এরপর crash হলে → WAL থেকে পুরো page restore → তারপর diff apply।

Performance cost:
  WAL size বাড়ে (full pages store করে)।
  Write I/O বাড়ে।

full_page_writes = off:
  শুধু যদি hardware/filesystem নিজে atomic write guarantee করে।
  (ZFS, certain RAID controllers with battery-backed cache)
  এমন না হলে NEVER turn off।

wal_init_zero = on: নতুন WAL file-এ zeros লেখে (filesystem fragmentation কমায়)
wal_recycle = on: পুরানো WAL file recycle করে (নতুন file create overhead কমায়)
```

---

### WAL Levels এবং ব্যবহার

```
wal_level = minimal
  Crash recovery-র জন্য minimum।
  Replication impossible।
  শুধু isolated server, no HA needed।

wal_level = replica (default, recommended)
  Crash recovery।
  Physical streaming replication।
  pg_basebackup।

wal_level = logical
  সব উপরের +
  Logical decoding।
  Logical replication।
  CDC (Change Data Capture)।
  
  Extra WAL: column-level change tracking।
  বেশি WAL size, বেশি I/O।
  শুধু logical replication বা CDC দরকার হলে।
```

---

### WAL Archiving এবং PITR

```
archive_mode = on
archive_command = 'cp %p /wal_archive/%f'
  %p: source file path
  %f: file name

WAL segment complete হলে archive_command চলে।
Archive করা WAL = PITR-এর ingredient।

Base backup + WAL archive = যেকোনো point-এ restore।
(বিস্তারিত Topic 25-এ)

S3 archiving:
archive_command = 'aws s3 cp %p s3://mybackup/wal/%f'

wal-g tool (recommended):
archive_command = 'wal-g wal-push %p'
restore_command = 'wal-g wal-fetch %f %p'
```

---

### WAL Performance Tuning

```sql
-- WAL-related settings দেখো
SHOW wal_level;
SHOW wal_buffers;
SHOW synchronous_commit;
SHOW checkpoint_timeout;
SHOW max_wal_size;
SHOW checkpoint_completion_target;
SHOW full_page_writes;
SHOW wal_compression;

-- WAL statistics
SELECT * FROM pg_stat_wal;  -- PostgreSQL 14+
-- wal_records, wal_fpi (full page images), wal_bytes, wal_buffers_full
-- wal_write (WAL writes), wal_sync (fsyncs), wal_write_time, wal_sync_time
```

**WAL Compression:**
```
wal_compression = off (default) | pglz | lz4 | zstd (PG 15+)

lz4: fast compression, moderate ratio
zstd: better ratio, slightly more CPU
pglz: old, slower, less ratio

When to use:
  - Disk I/O bound (WAL write bottleneck)
  - Replication bandwidth limited
  - Storage cost concern
  
CPU tradeoff: compression CPU ↑, disk I/O ↓
```

---

### না জানলে কী হয়

**Scenario 1: max_wal_size ছোট → forced checkpoint storm**
```
max_wal_size = 1GB (default)
High-write workload: 500MB/min WAL

WAL 1GB-এ পৌঁছায় → forced checkpoint!
checkpoint_completion_target এর কোনো মানে নেই,
full checkpoint এখনই হতে হবে।

10 second-এ পরের 1GB পূর্ণ → আবার checkpoint!
→ Continuous I/O spike → query latency spikes

Monitor: checkpoints_req in pg_stat_bgwriter
         (checkpoints_req >> checkpoints_timed হলে problem)

Fix: max_wal_size = 4GB (বা আরো বেশি, disk space থাকলে)
     checkpoint_timeout = 15min
```

**Scenario 2: WAL-এর উপর disk ভরে গেলে**
```
WAL archive slow (S3 slow network) বা archive_command fail হলে:
→ WAL segment archive হচ্ছে না
→ pg_wal/ directory বাড়তে থাকে
→ Disk full!
→ PostgreSQL PANIC: could not write to file "pg_wal/..."
→ Database crash!

Monitor: pg_wal/ directory size
         archive_status/: ready (to archive), done (archived)
         
SELECT * FROM pg_stat_archiver;
-- archived_count, failed_count, last_failed_time, last_failed_wal
```

---

### Big Picture Connection

```
Topic 5 (WAL)
    │
    ├──[WAL Buffer, fsync]──► Topic 3 (Transaction - Durability)
    │                             WAL কীভাবে Durability implement করে
    │
    ├──[WAL replay, Checkpoint]──► Topic 25 (PITR)
    │                                  Base backup + WAL = point-in-time restore
    │
    ├──[WAL streaming]──► Topic 26 (Replication)
    │                         Primary WAL → Standby continuous replay
    │
    ├──[full_page_writes, pd_lsn]──► Topic 2 (Physical Storage)
    │                                    Page-এর pd_lsn WAL-এর সাথে link
    │
    └──[WAL archiver process]──► Topic 1 (Architecture)
                                     Background worker-এর role
```
# Senior DBA Master Guide — Block 2
## Database & Object Design

---

## Topic 6 — Database & Schema: Isolation, Namespace, Permission Boundary + Multi-tenancy

### Big Picture: কেন এই distinction জানা দরকার?

"Database বানাবো না schema বানাবো?" — এই প্রশ্নের উত্তর না জানলে SaaS architecture ভুল হবে। আর ভুল architecture পরে fix করা মানে migration, downtime, data risk।

PostgreSQL-এ এই তিনটা concept clearly আলাদা:
```
Cluster → Database → Schema → Object (Table, Index, View...)
```
প্রতিটা level-এ isolation, permission, এবং management আলাদাভাবে কাজ করে।

---

### PostgreSQL Cluster — সবকিছুর উপরে

```
PostgreSQL Cluster = একটা PostgreSQL instance (একটা port-এ চলছে)

$PGDATA/ এর সব content = একটা cluster

Cluster-wide shared:
  pg_roles     → সব user/role (সব database-এ)
  pg_database  → সব database-এর list
  pg_tablespace → disk location mapping
  pg_authid    → authentication info
  WAL          → সব database-এর WAL একসাথে

Cluster-wide NOT shared:
  Data files   → প্রতিটা database আলাদা directory
  Catalogs     → pg_class, pg_attribute — প্রতি database-এ আলাদা
  Connections  → একটা connection একটা database-এ
```

---

### Database — Hardest Isolation Boundary

Database হলো PostgreSQL-এর সবচেয়ে শক্ত isolation boundary।

**Database-এর properties:**

```
1. Cross-database query impossible (by default):
   ecommerce_db-এ connected থেকে:
   SELECT * FROM analytics_db.public.events;
   → ERROR: cross-database references are not implemented

   শুধু dblink বা Foreign Data Wrapper (FDW) দিয়ে possible।
   কিন্তু: performance overhead, connection overhead, transaction isolation নেই।

2. আলাদা pg_catalog:
   প্রতিটা database-এর নিজস্ব system tables।
   pg_class, pg_attribute, pg_index — প্রতি database-এ independent।
   একটা database-এর table অন্য database-এর catalog-এ নেই।

3. আলাদা connection:
   psql -d ecommerce_db  → এই connection শুধু ecommerce_db দেখে।
   \c analytics_db       → নতুন connection (port, user same হলেও)।

4. Shared: Roles/Users
   CREATE ROLE alice; → cluster-wide।
   alice সব database-এ exist করে (permission আলাদা হলেও)।

5. আলাদা Encoding এবং Collation:
   CREATE DATABASE fr_app ENCODING 'UTF8' LC_COLLATE 'fr_FR.UTF-8';
   প্রতিটা database-এ আলাদা character sorting।
```

```sql
-- Database তৈরি
CREATE DATABASE ecommerce
  ENCODING 'UTF8'
  LC_COLLATE 'en_US.UTF-8'
  LC_CTYPE 'en_US.UTF-8'
  TEMPLATE template0  -- clean template (template1 থেকে extra objects নেই)
  OWNER app_admin;

-- Database-level config (এই database-এ connect করলে এই setting)
ALTER DATABASE ecommerce SET timezone = 'Asia/Dhaka';
ALTER DATABASE ecommerce SET search_path = myapp, public;
ALTER DATABASE ecommerce SET work_mem = '32MB';

-- Database list দেখো
SELECT datname, datdba::regrole, pg_encoding_to_char(encoding),
  datcollate, datctype,
  pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
WHERE datistemplate = false;
```

---

### Schema — Soft Namespace

Schema হলো database-এর ভেতরে logical grouping। এটা namespace — same name-এর object দুটো schema-তে থাকতে পারে।

```
ecommerce database:
├── Schema: public        (default)
│     ├── Table: users
│     └── Table: orders
├── Schema: analytics
│     ├── Table: events   (public.users থেকে আলাদা namespace)
│     └── MView: daily_summary
├── Schema: audit
│     └── Table: audit_log
└── Schema: internal
      └── Function: recalculate_stats()
```

**Schema-র properties:**

```
1. Cross-schema query possible (same database-এ):
   SELECT u.name, e.event_type
   FROM public.users u
   JOIN analytics.events e ON e.user_id = u.id;
   → Works! Same transaction, same connection।

2. search_path — schema lookup order:
   SHOW search_path;  → "$user", public
   "$user": current user-এর নামে schema থাকলে সেটা আগে
   public: default fallback

   SET search_path = analytics, public;
   SELECT * FROM events;  → analytics.events (analytics আগে)

3. Permission granularity:
   Schema-এ USAGE grant না করলে object দেখা যাবে না।
   Schema-এ USAGE আছে কিন্তু table-এ SELECT নেই → table দেখা যাবে না।
   উভয় দরকার।

4. Object name isolation:
   public.orders এবং archive.orders — দুটো আলাদা table।
   Conflict নেই।
```

```sql
-- Schema তৈরি এবং permission
CREATE SCHEMA analytics AUTHORIZATION analytics_owner;
CREATE SCHEMA audit;
CREATE SCHEMA internal;

-- Schema-level permission
GRANT USAGE ON SCHEMA analytics TO reporting_role;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO reporting_role;

-- ভবিষ্যতের objects-এও auto-grant
ALTER DEFAULT PRIVILEGES IN SCHEMA analytics
  GRANT SELECT ON TABLES TO reporting_role;

-- Schema isolation: internal schema hide করো
REVOKE ALL ON SCHEMA internal FROM PUBLIC;
GRANT USAGE ON SCHEMA internal TO internal_app_role;

-- search_path দিয়ে schema collide করো না
-- ভুল: public schema-তে সব রাখা
-- ভালো: application-specific schema, public শুধু extensions-এর জন্য
REVOKE CREATE ON SCHEMA public FROM PUBLIC;  -- public schema-তে কেউ object বানাতে পারবে না
```

---

### pg_catalog এবং information_schema

```
প্রতিটা database-এ দুটো special schema:

pg_catalog:
  PostgreSQL-specific system tables।
  pg_class, pg_attribute, pg_index, pg_constraint, etc.
  সবচেয়ে fast, সবচেয়ে complete।
  Production monitoring-এ এটাই ব্যবহার করো।

information_schema:
  SQL standard-এর views।
  Cross-database portable।
  কিন্তু slow (pg_catalog-এর উপর views)।
  PostgreSQL-specific feature দেখা যায় না।

search_path-এ pg_catalog সবসময় implicit আগে থাকে।
```

---

### Multi-tenancy — তিনটা Pattern গভীরে

#### Pattern 1: Database per Tenant

```
cluster:
├── db_tenant_acme      (Acme Corp-এর পুরো database)
├── db_tenant_globex    (Globex Corp)
├── db_tenant_initech   (Initech)
└── db_tenant_umbrella  (Umbrella Corp)
```

**ভেতরে কী হয়:**
```
প্রতিটা tenant-এর:
  - আলাদা pg_catalog (system tables)
  - আলাদা disk files
  - আলাদা connection pool (PgBouncer config)
  - আলাদা backup schedule সম্ভব
  - আলাদা PostgreSQL config সম্ভব (ALTER DATABASE)
  - আলাদা encoding/collation সম্ভব

Cross-tenant query: impossible without FDW
Cross-tenant analytics: FDW বা ETL pipeline দরকার
```

**Pros — কেন এটা choose করবে:**
```
✅ Maximum isolation — একটা tenant-এর data অন্যটায় যাওয়া physically impossible
✅ Regulatory compliance — GDPR "right to erasure" = DROP DATABASE (instant)
✅ Per-tenant backup/restore — একটা tenant corrupt হলে শুধু সেটা restore
✅ Per-tenant migration — Acme-কে v2 schema দাও, Globex এখনো v1-এ
✅ Performance isolation — Acme-র heavy query Globex-কে affect করে না
     (shared_buffers এবং connections share হয়, কিন্তু data pages আলাদা)
✅ Offboarding simple — DROP DATABASE tenant_acme (সব data gone)
```

**Cons — কেন avoid করবে:**
```
❌ Connection overhead: প্রতিটা tenant-এর জন্য আলাদা connection pool
   ১০০ tenant × ১০ connections = ১০০০ PostgreSQL connections
   max_connections hit করে

❌ Schema migration nightmare:
   ALTER TABLE users ADD COLUMN phone text;
   → ১০০ database-এ ১০০ বার run করতে হবে
   → Script লিখতে হবে, error handling, rollback logic
   → একটা fail করলে tenants-এ inconsistent schema

❌ pg_catalog bloat:
   ১০০০ database-এ ১০০০ pg_catalog instance
   Cluster-level operations slow হয়

❌ Monitoring complexity:
   প্রতিটা database আলাদা monitor করতে হয়

❌ Disk space overhead:
   প্রতিটা database-এর minimum overhead (system tables, WAL tracking)
```

**কখন Database per Tenant:**
```
Tenant count: < 500 (ideally < 100)
Compliance: HIPAA, GDPR, PCI-DSS strict isolation দরকার
Contract: Enterprise B2B, tenant isolation legally required
Scale: Predictable, slow-growing tenant count
Team: Dedicated DBA team যে এই complexity handle করতে পারবে
```

---

#### Pattern 2: Schema per Tenant

```
ecommerce_saas (একটা database):
├── tenant_acme     (schema)
│     ├── users
│     ├── orders
│     └── products
├── tenant_globex   (schema)
│     ├── users     (acme থেকে আলাদা namespace)
│     ├── orders
│     └── products
└── tenant_initech  (schema)
      ├── users
      ├── orders
      └── products
```

**ভেতরে কী হয়:**
```
সব tenant একই database connection ব্যবহার করে।
PgBouncer: একটা pool → সব tenant।

Application প্রতিটা request-এ search_path set করে:
  SET search_path TO tenant_acme, public;
  → এই connection-এ "users" মানে tenant_acme.users

Data isolation: search_path দিয়ে enforce।
Bug: ভুল search_path set হলে wrong tenant data দেখবে।

pg_catalog: একটাই (এই database-এর জন্য)
  কিন্তু pg_class-এ সব tenant-এর সব tables listed।
  ১০০ tenant × ১০০ tables = ১০,০০০ entries pg_class-এ।
  ১০০০ tenant-এ pg_catalog slow হতে পারে।
```

```sql
-- Tenant provisioning
CREATE SCHEMA tenant_acme;
SET search_path TO tenant_acme;

CREATE TABLE users (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text NOT NULL,
  email text NOT NULL UNIQUE,
  created_at timestamptz DEFAULT now()
);

CREATE TABLE orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id bigint NOT NULL REFERENCES users(id),
  amount numeric(12,2) NOT NULL,
  status text NOT NULL DEFAULT 'pending',
  created_at timestamptz DEFAULT now()
);

-- Tenant-specific role
CREATE ROLE tenant_acme_role;
GRANT USAGE ON SCHEMA tenant_acme TO tenant_acme_role;
GRANT ALL ON ALL TABLES IN SCHEMA tenant_acme TO tenant_acme_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA tenant_acme
  GRANT ALL ON TABLES TO tenant_acme_role;

-- Application connection string:
-- SET search_path = tenant_acme, public
-- এটা connection-এ SET করো, PgBouncer transaction mode-এ:
-- SET LOCAL search_path = tenant_acme, public;
```

**Schema Migration — সব tenant-এ একসাথে:**
```sql
-- Migration script loop
DO $$
DECLARE
  tenant_schema text;
BEGIN
  FOR tenant_schema IN
    SELECT schema_name FROM information_schema.schemata
    WHERE schema_name LIKE 'tenant_%'
    ORDER BY schema_name
  LOOP
    EXECUTE format(
      'ALTER TABLE %I.users ADD COLUMN IF NOT EXISTS phone text',
      tenant_schema
    );
    RAISE NOTICE 'Migrated schema: %', tenant_schema;
  END LOOP;
END;
$$;
```

**Pros:**
```
✅ একটা database connection pool (efficient)
✅ Same PostgreSQL config সব tenant-এ
✅ Cross-tenant analytics সহজ (same database, different schema)
✅ Moderate isolation (search_path + schema permissions)
✅ Per-tenant backup: pg_dump -n tenant_acme (schema-only dump)
✅ Offboarding: DROP SCHEMA tenant_acme CASCADE
```

**Cons:**
```
❌ search_path bug = tenant data leak (severe security risk)
❌ pg_catalog size বাড়ে (large tenant count-এ)
❌ Migration এখনো loop দরকার (কিন্তু একটা database-এ, সহজ)
❌ Performance isolation নেই — Acme-র heavy query Globex-কে affect করতে পারে
❌ Transaction isolation: একটা migration transaction-এ সব tenant affect করে
```

**search_path bug-এর বিপদ:**
```python
# Application code (ভুল):
def handle_request(tenant_id, user_id):
    conn.execute(f"SET search_path TO tenant_{tenant_id}")
    # ... কিছু error হলো, exception catch করলো ...
    # search_path reset হলো না!
    # পরের request ভুল tenant-এর schema দেখছে!
    
# ঠিক:
def handle_request(tenant_id, user_id):
    conn.execute("SET LOCAL search_path TO tenant_" + tenant_id)
    # SET LOCAL: transaction-scoped, commit/rollback-এ auto-reset
    # PgBouncer transaction mode-এ এটাই safe
```

---

#### Pattern 3: Shared Schema (Row-Level Tenant Isolation)

```
ecommerce_saas (একটা database, একটা schema):
├── users    (tenant_id column সহ)
├── orders   (tenant_id column সহ)
└── products (tenant_id column সহ)

users:
  id | tenant_id | name  | email
   1 |    101    | Alice | alice@acme.com
   2 |    102    | Bob   | bob@globex.com
   3 |    101    | Carol | carol@acme.com
```

**ভেতরে কী হয়:**
```
Isolation: Row-Level Security (RLS) দিয়ে enforce।
Application: session variable set করে tenant_id।

  SET app.current_tenant = '101';
  SELECT * FROM users;  -- RLS policy: WHERE tenant_id = 101
  → শুধু tenant 101-এর rows দেখবে

Data physically mixed: সব tenant-এর data একই page-এ থাকতে পারে।
Performance: PARTITION BY tenant_id দিয়ে isolate করা যায়।
```

```sql
-- Shared schema setup
CREATE TABLE users (
  id bigint GENERATED ALWAYS AS IDENTITY,
  tenant_id bigint NOT NULL,
  name text NOT NULL,
  email text NOT NULL,
  created_at timestamptz DEFAULT now(),
  PRIMARY KEY (id, tenant_id)  -- tenant_id PK-তে থাকলে partition possible
);

-- Composite index: প্রতিটা query-তে tenant_id filter করতে হবে
CREATE INDEX ON users (tenant_id, id);
CREATE INDEX ON users (tenant_id, email);

-- Row-Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE users FORCE ROW LEVEL SECURITY;  -- owner-ও bypass করতে পারবে না

CREATE POLICY tenant_isolation ON users
  USING (tenant_id = current_setting('app.current_tenant')::bigint)
  WITH CHECK (tenant_id = current_setting('app.current_tenant')::bigint);

-- Application-এ:
SET app.current_tenant = '101';
SELECT * FROM users;  -- automatically filtered
INSERT INTO users (name, email) VALUES ('Dave', 'dave@acme.com');
-- RLS: tenant_id automatically 101 হওয়া উচিত — কিন্তু না!
-- WITH CHECK verify করে INSERT কিন্তু value set করে না
-- Application-কে tenant_id explicitly set করতে হবে:
INSERT INTO users (tenant_id, name, email) VALUES (101, 'Dave', 'dave@acme.com');
```

**RLS-এর Gotcha:**
```sql
-- RLS bypass করে superuser:
-- Superuser হলে RLS ignore হয়
-- এজন্য application connection superuser দিয়ে করো না!

-- Table owner-ও RLS bypass করে (FORCE ROW LEVEL SECURITY না থাকলে):
ALTER TABLE users FORCE ROW LEVEL SECURITY;
-- এখন owner-ও RLS follow করবে

-- current_setting() null হলে error:
SET app.current_tenant = '';  -- empty string → ::bigint cast error!
-- Safe version:
CREATE POLICY tenant_isolation ON users
  USING (
    tenant_id = nullif(current_setting('app.current_tenant', true), '')::bigint
  );
-- current_setting('key', true): missing_ok=true, return NULL instead of error
```

**Pros:**
```
✅ Simplest management — একটাই schema
✅ সবচেয়ে efficient connection pooling
✅ Migration: একবারেই সব tenant-এ
✅ Cross-tenant analytics: সহজতম (GROUP BY tenant_id)
✅ Lakh+ tenant সম্ভব
✅ Storage efficient (shared pages, shared indexes)
```

**Cons:**
```
❌ Isolation সবচেয়ে দুর্বল
   RLS bug → সব tenant-এর data expose
❌ "Noisy neighbor": একটা tenant-এর heavy query → সব tenant slow
❌ Data residency compliance কঠিন
   "Acme-র data শুধু EU-তে থাকবে" — impossible with shared schema
❌ Per-tenant backup/restore very complex
❌ Tenant offboarding: DELETE FROM all tables WHERE tenant_id = 101
   Large table-এ slow, VACUUM trigger করে
```

---

### তিনটা Pattern-এর সিদ্ধান্ত Framework:

```
প্রশ্নগুলো:

1. Tenant count কত?
   < 100: Database per tenant viable
   100-10,000: Schema per tenant
   10,000+: Shared schema (with partitioning)

2. Compliance requirement?
   HIPAA/GDPR strict isolation: Database per tenant
   General data privacy: Schema per tenant + RLS
   B2C, low sensitivity: Shared schema

3. Performance isolation দরকার?
   Enterprise SLA (Acme-র slowness Globex-কে affect করবে না): Database per tenant
   Best-effort isolation: Schema per tenant
   No isolation: Shared schema

4. Operational capacity?
   Small team, many tenants: Shared schema (simplest ops)
   Medium team: Schema per tenant
   Large DBA team: Database per tenant

5. Migration complexity tolerance?
   Low tolerance: Shared schema (one migration)
   Medium: Schema per tenant (scripted loop)
   High: Database per tenant (distributed migration)

6. Cross-tenant analytics?
   Frequent, real-time: Shared schema
   Occasional: Schema per tenant (FDW বা direct query)
   Never বা ETL: যেকোনো pattern
```

---

### Hybrid Pattern — Real World

```
Production reality-তে often hybrid:

Tier 1 (Enterprise): Database per tenant
  Acme Corp pays $50k/month → dedicated database
  Full isolation, dedicated backup, custom config

Tier 2 (Business): Schema per tenant
  Medium companies → shared cluster, dedicated schema
  Good isolation, reasonable cost

Tier 3 (Starter/Free): Shared schema
  Small users → all in one schema with RLS
  Lowest cost, lowest isolation

Application routing:
  if tenant.tier == 'enterprise':
    connect_to_database(f"tenant_{tenant_id}_db")
  elif tenant.tier == 'business':
    connect_with_schema(f"tenant_{tenant_id}")
  else:
    connect_shared_schema(tenant_id)
```

---

### না জানলে কী হয়

**Scenario 1: Cross-database query-র misconception**
```
ভুল assumption: "সব PostgreSQL database একসাথে query করা যায়"

reality check:
  SELECT * FROM db1.schema1.table1, db2.schema2.table2;
  → ERROR: cross-database references are not implemented

FDW দিয়ে করা যায়:
  CREATE EXTENSION postgres_fdw;
  CREATE SERVER db2_server FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'localhost', dbname 'db2');
  CREATE FOREIGN TABLE remote_table (...) SERVER db2_server;
  SELECT * FROM remote_table;

কিন্তু: আলাদা transaction, আলাদা connection, 
         JOIN optimization ঠিকমতো হয় না।
         Analytics-এ: ETL pipeline better।
```

**Scenario 2: search_path injection**
```
Multi-tenant schema pattern-এ:
  tenant_name = "'; DROP TABLE users; --"
  SET search_path TO '; DROP TABLE users; --', public;
  → SQL injection!

Safe approach:
  -- search_path-এ user input never directly interpolate করো
  -- Validate tenant_name against allowlist:
  IF tenant_name !~ '^[a-z][a-z0-9_]*$' THEN
    RAISE EXCEPTION 'Invalid tenant name';
  END IF;
  EXECUTE format('SET LOCAL search_path TO %I, public', tenant_name);
  -- %I: identifier quoting → injection prevent
```

**Scenario 3: pg_catalog bloat with many schemas**
```
৫,০০০ tenant, প্রতিটায় ৫০ tables:
  pg_class-এ: 5000 × 50 = 250,000 entries
  pg_attribute-এ: 250,000 × avg 10 columns = 2,500,000 entries
  
DDL operations slow হয়:
  CREATE TABLE → pg_class scan
  VACUUM → pg_class scan
  pg_dump → catalog scan

Fix: Shared schema (বা database per tenant) consider করো।
Schema per tenant শুধু ~1000 tenant পর্যন্ত practically comfortable।
```

---

## Topic 7 — Data Types

### Big Picture: সঠিক Type কেন এত Critical?

```
Wrong type-এর cascading effects:

1. Disk space waste:
   varchar(255) for country_code → text best, char(2) ideal
   bigint for age → smallint যথেষ্ট
   100M rows × 6 bytes extra = 600MB extra storage

2. Index inefficiency:
   Index = B-Tree of column values।
   Bigger values = bigger B-Tree nodes = fewer values per page।
   = More I/O for index scan।

3. Type mismatch = index miss:
   column: bigint, query: WHERE user_id = '42' (text)
   → Implicit cast → index unusable → Seq Scan!

4. Precision loss:
   float-এ money store → rounding errors → financial inconsistency

5. Application bugs:
   date vs timestamptz → timezone conversion bugs
   json vs jsonb → cannot index json

এই section-এ প্রতিটা type-এর internal representation বুঝবো।
```

---

### Numeric Types — Internal Representation

**Integer family:**

```
smallint  → 2 bytes, range: -32,768 to 32,767
integer   → 4 bytes, range: -2,147,483,648 to 2,147,483,647
bigint    → 8 bytes, range: -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807

Internal: Two's complement binary।

Selection guide:
  age, rating, small counts: smallint
  most IDs, counts, quantities: integer
  financial amounts (cents), large IDs, timestamps (epoch): bigint
  
Common mistake:
  serial (integer) for ID এবং বছরের পরে overflow!
  integer max = 2.1 billion।
  Active social media app: 1M new users/day × 365 = 365M/year।
  5 years = 1.8B → approaching limit!
  Always use bigint/bigserial for any ID that might grow।
```

**Floating Point — IEEE 754:**

```
real          → 4 bytes, float32, 6 decimal digits precision
double precision → 8 bytes, float64, 15 decimal digits precision

Internal: Sign (1 bit) + Exponent + Mantissa (binary fraction)
Problem: Most decimal fractions can't be represented exactly in binary।

0.1 in binary: 0.0001100110011... (infinite repeating)
→ 0.1 stored as: 0.10000000000000000555...

Demonstration:
SELECT 0.1::float8 + 0.2::float8 = 0.3::float8;
→ FALSE!

SELECT 0.1::float8 + 0.2::float8;
→ 0.30000000000000004

Financial application-এ float use → audit হলে দেখবে:
  Expected: 100.00 × 0.15 = 15.00
  Actual:   100.00 × 0.15 = 14.999999999999998 ← diff!
  Rounding: OK এবার, কিন্তু accumulated error?

Float কখন OK:
  Scientific calculations (approximate is fine)
  Geographic coordinates (GPS precision সীমিত)
  Machine learning features
  Statistics (আনুমানিক)

Float কখন NEVER:
  Money, financial calculations
  Inventory counts যেখানে exact চাই
  Any data যেখানে = comparison করতে হবে
```

**Numeric — Exact Arbitrary Precision:**

```
NUMERIC(precision, scale)
  precision: total significant digits
  scale: decimal places
  
NUMERIC(10, 2): 12345678.99 (max)
NUMERIC: unlimited precision (any size!)

Internal representation:
  Stored as base-10000 digits (not binary)
  প্রতি 4 decimal digits → 2 bytes
  Plus: header (sign, weight, dscale)
  
  1234.56 stored as:
    digits: [1234, 5600]  (base-10000 groups)
    weight: 0 (= 1 group before decimal)
    dscale: 2 (2 decimal places)

Performance:
  numeric arithmetic: slower than float8 (software arithmetic, not hardware FPU)
  কিন্তু: correctness > speed for money

Money storage strategies:
  Option 1: NUMERIC(12, 2) for BDT with paisa → 9999999999.99
  Option 2: BIGINT storing cents (integer math, fastest)
    amount_cents = 10000 → "100.00 BDT"
    No decimal point, no rounding, pure integer math
    Application layer convert করে display-এর সময়
  
  Option 3: PostgreSQL money type — AVOID
    locale-dependent, not portable, cast restrictions
```

---

### Character Types — কোনটা কখন?

**Internal representation:**

```
text / varchar(n): varlena (variable-length) storage
  Header: 1 or 4 bytes (length encoding)
    1 byte: length < 127 bytes
    4 bytes: length ≥ 127 bytes
  Data: UTF-8 encoded bytes (PostgreSQL always UTF-8 internally if encoding=UTF8)

char(n): fixed length, space-padded
  Internal: text-এর মতোই, কিন্তু trailing spaces preserved
  Comparison: trailing spaces ignored in = but not in LIKE!
```

```sql
-- char(n) comparison trap
SELECT 'abc'::char(5) = 'abc';   -- TRUE (trailing spaces ignored)
SELECT 'abc'::char(5) LIKE 'abc'; -- FALSE! 'abc  ' LIKE 'abc' = false
SELECT 'abc'::char(5) LIKE 'abc%'; -- TRUE

-- text vs varchar(n): কোনো performance difference নেই!
-- PostgreSQL internally identical।
-- varchar(n): application-level length validation দরকার হলে
-- text: unlimited, prefer করো

-- varchar(n) limit-এর purpose:
CREATE TABLE products (
  sku varchar(50),     -- SKU সর্বোচ্চ ৫০ char
  description text     -- যতটা দরকার
);
-- DB-level constraint: ৫০ char-এর বেশি insert হবে না।
-- এই constraint useful যদি business rule থাকে।
```

**Collation — Sorting এবং Comparison:**

```
Collation = character comparison এবং sorting rules।

'a' < 'B' কি? Depends on collation!
  C collation: ASCII order → 'B' (66) < 'a' (97)
  en_US.UTF-8: locale-aware → 'a' < 'B' (human intuition)

Performance impact:
  C collation: byte comparison (fastest)
  ICU collation (PG 10+): locale-aware (slower but correct)
  
Index collation:
  Index-এর collation query-র collation match করতে হবে।
  Mismatch = index unusable।

  CREATE INDEX ON users (lower(name) COLLATE "C");
  SELECT * FROM users WHERE lower(name) COLLATE "C" = 'alice'; -- index used
  SELECT * FROM users WHERE lower(name) = 'alice'; -- different collation, index MISSED!
```

---

### Date/Time Types — Timezone এর Trap

**Internal representation:**

```
date: 4 bytes, Julian day number (days since 2000-01-01)
time: 8 bytes, microseconds since midnight
timestamp: 8 bytes, microseconds since 2000-01-01 00:00:00
timestamptz: 8 bytes, microseconds since 2000-01-01 00:00:00 UTC
interval: 16 bytes (months, days, microseconds আলাদা — কারণ DST!)
```

**timestamp vs timestamptz — সবচেয়ে common bug:**

```
timestamp (WITHOUT TIME ZONE):
  Stores: 2024-01-15 10:30:00
  No timezone info।
  PostgreSQL এটাকে server timezone-এ interpret করে।
  Server timezone বদলালে → data "বদলায়"!

timestamptz (WITH TIME ZONE):
  Stores: UTC-তে convert করে (internally)
  Display: session timezone-এ convert করে দেখায়
  Server timezone বদলালে → display বদলায়, stored value same।

Real bug:
  Server timezone: UTC (standard setup)
  User input: "2024-01-15 16:00:00" (meant to be Dhaka time, UTC+6)
  
  timestamp: stores "2024-01-15 16:00:00" as-is
             displayed as "2024-01-15 16:00:00" regardless of timezone
             actual UTC time assumed: 16:00 UTC (WRONG!)
  
  timestamptz: 
    SET timezone = 'Asia/Dhaka';
    INSERT: "2024-01-15 16:00:00" → stored as "2024-01-15 10:00:00 UTC"
    SET timezone = 'UTC';
    SELECT: shows "2024-01-15 10:00:00+00"
    SET timezone = 'Asia/Dhaka';
    SELECT: shows "2024-01-15 16:00:00+06" ← CORRECT!

Rule: ALWAYS use timestamptz। No exceptions।
      timestamp শুধু: local calendar data যেখানে timezone concept নেই
      (যেমন: "birthday: 1990-01-15" — এটা timezone-relative না)
```

```sql
-- Timezone operations
SET timezone = 'Asia/Dhaka';
SELECT now();  -- Dhaka time-এ দেখাবে

-- Convert করো
SELECT now() AT TIME ZONE 'UTC';         -- UTC-তে দেখাও
SELECT now() AT TIME ZONE 'US/Eastern';  -- Eastern time-এ

-- Timestamp arithmetic
SELECT now() + interval '7 days';         -- 7 days later
SELECT now() - interval '1 month';        -- 1 month ago
SELECT date_trunc('month', now());         -- month শুরু
SELECT extract(hour FROM now());           -- hour component

-- Date math (days দিয়ে)
SELECT '2024-12-31'::date - '2024-01-01'::date;  -- 365 (integer)
SELECT '2024-01-01'::date + 30;                   -- 2024-01-31

-- Interval arithmetic-এর subtlety:
SELECT '2024-01-31'::date + interval '1 month';
-- Result: 2024-02-29 (leap year!) or 2024-03-02?
-- PostgreSQL: 2024-02-29 (clamps to last valid day)

SELECT '2024-01-31'::date + interval '1 month' + interval '1 month';
-- Result: 2024-03-31 (not 2024-03-29!)
-- Interval math: each operation separately, not cumulative
```

---

### Boolean — Simple but Tricky

```sql
-- PostgreSQL accepts many boolean representations:
SELECT 'true'::boolean;   -- TRUE
SELECT 'yes'::boolean;    -- TRUE
SELECT '1'::boolean;      -- TRUE
SELECT 'on'::boolean;     -- TRUE
SELECT 't'::boolean;      -- TRUE
SELECT 'false'::boolean;  -- FALSE
SELECT 'no'::boolean;     -- FALSE
SELECT '0'::boolean;      -- FALSE

-- NULL boolean:
SELECT NULL::boolean = true;  -- NULL (not false!)
SELECT NULL::boolean IS TRUE;  -- FALSE
SELECT NULL::boolean IS NOT TRUE;  -- TRUE

-- Gotcha: boolean comparison in WHERE
SELECT * FROM users WHERE is_active;        -- OK (is_active = true)
SELECT * FROM users WHERE is_active = true; -- Also OK but verbose
SELECT * FROM users WHERE NOT is_active;    -- is_active = false OR NULL
-- Careful: NULL values pass NOT filter!
SELECT * FROM users WHERE is_active IS FALSE;  -- only false, not NULL
SELECT * FROM users WHERE is_active IS NOT TRUE;  -- false AND null
```

**MySQL tinyint(1) vs PostgreSQL boolean:**
```
MySQL-এ native boolean নেই।
tinyint(1) ব্যবহার করে: 1=true, 0=false।
কিন্তু 2, 3, -1 ও store করা যায়!
Application bug-এ boolean column-এ random integer যেতে পারে।

PostgreSQL boolean: strict, শুধু true/false/null।
Migration: MySQL tinyint(1) → PostgreSQL boolean careful cast করো।
  WHERE myapp_tinyint != 0 → where pg_bool = true
```

---

### JSON vs JSONB — Internal Difference

```
json:
  Text হিসেবে store করে।
  Insert করলে validate করে (valid JSON?)।
  Read: text parse করে প্রতিবার।
  Key order preserved।
  Duplicate keys preserved (last value wins in operations)।
  Cannot be indexed directly।

jsonb:
  Binary decomposed format-এ store করে।
  Insert: parse করে binary-তে convert।
  Read: binary থেকে directly। Fast।
  Key order NOT preserved (alphabetical internally)।
  Duplicate keys removed (last value kept)।
  GIN index possible।
  Operators: @>, <@, ?, ?|, ?&
```

**jsonb internal binary format:**
```
jsonb binary format হলো a tree structure:
  Header: node count, has key flags
  Nodes: type (object/array/string/number/bool/null) + offset
  String data: deduplicated (same string একবারই stored!)
  Numbers: numeric type stored (integer vs float distinguish করে)
  
Performance:
  jsonb read: binary parse, pointer arithmetic
  json read: full text scan and parse every time
  
  For a 10KB JSON document:
    jsonb: ~microseconds
    json: text scan → microsecond-to-millisecond (depending on complexity)
```

```sql
-- jsonb operations in depth
CREATE TABLE events (
  id bigint PRIMARY KEY,
  data jsonb NOT NULL,
  created_at timestamptz DEFAULT now()
);

INSERT INTO events (id, data) VALUES
(1, '{"user_id": 42, "action": "login", "metadata": {"ip": "1.2.3.4", "ua": "Chrome"}}'),
(2, '{"user_id": 42, "action": "purchase", "amount": 1500, "items": [{"id": 1}, {"id": 2}]}');

-- Key access
SELECT data->>'action' FROM events;  -- text output
SELECT data->'metadata' FROM events;  -- jsonb output
SELECT data->'metadata'->>'ip' FROM events WHERE id = 1;

-- Containment (@>): "does this jsonb contain this?"
SELECT * FROM events WHERE data @> '{"user_id": 42}';  -- GIN index used!
SELECT * FROM events WHERE data @> '{"action": "login"}';

-- Key existence (?)
SELECT * FROM events WHERE data ? 'amount';  -- has 'amount' key?
SELECT * FROM events WHERE data ?| ARRAY['amount', 'discount'];  -- either key?
SELECT * FROM events WHERE data ?& ARRAY['user_id', 'action'];   -- both keys?

-- Array element access
SELECT data->'items'->0->>'id' FROM events WHERE id = 2;  -- first item's id

-- Modify jsonb
UPDATE events
SET data = data || '{"processed": true}'  -- merge
WHERE id = 1;

UPDATE events
SET data = data - 'metadata'  -- remove key
WHERE id = 1;

UPDATE events
SET data = jsonb_set(data, '{metadata,ip}', '"2.3.4.5"')  -- nested set
WHERE id = 1;

-- Aggregate jsonb
SELECT jsonb_agg(data) FROM events WHERE data->>'user_id' = '42';
SELECT jsonb_object_agg(data->>'action', data->>'amount') FROM events;

-- Index options
CREATE INDEX ON events USING GIN (data);  -- full document GIN
CREATE INDEX ON events USING GIN (data jsonb_path_ops);  -- smaller, only @> operator
CREATE INDEX ON events ((data->>'user_id')::bigint);  -- specific key B-Tree
CREATE INDEX ON events USING GIN ((data->'items'));  -- array element index
```

**jsonb Gotchas:**
```sql
-- Gotcha 1: Type in JSON vs PostgreSQL type
SELECT (data->>'amount')::numeric FROM events;  -- string to numeric cast required
-- jsonb-এ সব value string হিসেবে extract হয় ->>'key'
-- Numeric: data->'amount' returns jsonb number, but ->>' returns text

-- Gotcha 2: NULL vs JSON null
SELECT data->>'nonexistent' FROM events;  -- SQL NULL (key doesn't exist)
SELECT data->'null_field' FROM events WHERE data = '{"null_field": null}'::jsonb;
-- returns jsonb 'null' (not SQL NULL!)
SELECT data->>'null_field' FROM events WHERE data = '{"null_field": null}'::jsonb;
-- returns SQL NULL

-- Gotcha 3: Index-এ function wrap করলে index miss
SELECT * FROM events WHERE (data->>'user_id')::int = 42;
-- এই query GIN index ব্যবহার করতে পারে না!
-- GIN: containment এবং existence operators শুধু
-- B-Tree index on (data->>'user_id') দরকার এই query-র জন্য

-- Gotcha 4: Large JSONB এবং UPDATE
-- ১MB jsonb update করলে → পুরো column rewrite
-- TOAST table-এ পুরো নতুন chunk → dead TOAST data
-- Frequent large jsonb update → TOAST table bloat
```

---

### Array Types

```sql
-- Array internal:
-- varlena header + dimension info + null bitmap + elements
-- Elements: fixed-size type-এর জন্য packed, variable-size-এর জন্য pointers

CREATE TABLE products (
  id bigint PRIMARY KEY,
  tags text[],
  prices numeric[],
  matrix int[][]  -- multi-dimensional array
);

-- Array construction
SELECT ARRAY['a', 'b', 'c'];
SELECT ARRAY(SELECT name FROM categories WHERE active);

-- Element access (1-indexed!)
SELECT tags[1] FROM products;  -- first element (not 0!)
SELECT tags[2:4] FROM products;  -- slice (elements 2 to 4)

-- Array operators
SELECT * FROM products WHERE 'sale' = ANY(tags);      -- element exists
SELECT * FROM products WHERE tags @> ARRAY['sale'];   -- contains
SELECT * FROM products WHERE tags <@ ARRAY['sale', 'new', 'hot'];  -- contained by
SELECT * FROM products WHERE tags && ARRAY['sale', 'discount'];    -- overlap

-- Array functions
SELECT array_length(tags, 1) FROM products;  -- first dimension length
SELECT cardinality(tags) FROM products;       -- total elements (any dimension)
SELECT unnest(tags) FROM products;            -- expand to rows
SELECT array_append(tags, 'clearance') FROM products;
SELECT array_remove(tags, 'old') FROM products;
SELECT array_cat(tags, ARRAY['new1', 'new2']) FROM products;

-- GIN index on array
CREATE INDEX ON products USING GIN (tags);
-- @>, <@, && operators use this index
```

**Array Gotcha:**
```sql
-- Gotcha: = operator checks exact match, not containment
SELECT * FROM products WHERE tags = ARRAY['sale', 'new'];
-- শুধু exactly ['sale', 'new'] match করবে, order important!

-- Gotcha: NULL in array
SELECT ARRAY[1, NULL, 3];  -- [1, NULL, 3] — allowed
SELECT 1 = ANY(ARRAY[1, NULL, 3]);  -- TRUE
SELECT 5 = ANY(ARRAY[1, NULL, 3]);  -- NULL (not FALSE! because NULL=5 is NULL)
SELECT NOT (5 = ANY(ARRAY[1, NULL, 3]));  -- NULL (not TRUE!)

-- Safe NULL handling:
SELECT * FROM products WHERE 'sale' = ANY(tags) OR tags IS NULL;
```

---

### UUID — Internal এবং Performance

```sql
-- UUID: 128-bit (16 bytes) value
-- Text representation: 8-4-4-4-12 hex digits with hyphens
-- e.g., 550e8400-e29b-41d4-a716-446655440000

-- Versions:
-- v1: timestamp + MAC address (sequential, MAC leaks)
-- v4: random (most common, but random insert = index fragmentation)
-- v7: timestamp + random (ordered by time, PostgreSQL 17+)

-- PostgreSQL built-in (13+):
SELECT gen_random_uuid();  -- v4 random UUID

-- uuid-ossp extension (older):
CREATE EXTENSION "uuid-ossp";
SELECT uuid_generate_v1();  -- v1
SELECT uuid_generate_v4();  -- v4

-- Storage: 16 bytes (binary internally)
-- Text: 36 bytes (with hyphens) — PostgreSQL stores binary, displays as text

-- UUID vs bigserial performance:
-- bigserial: sequential, 8 bytes, simple
-- UUID v4: random, 16 bytes, index fragmentation

-- UUID v4 index fragmentation:
-- New UUID always goes to a random position in B-Tree
-- B-Tree leaf pages constantly splitting and rebalancing
-- Page fill factor drops → more pages → larger index
-- Random I/O → cache miss বেশি

-- UUIDv7 (ordered):
-- Format: 48-bit millisecond timestamp + 74-bit random
-- Monotonically increasing (within same millisecond)
-- B-Tree insert mostly at end → no random splits
-- Best of both: globally unique + sequential insert

-- Using UUIDv7 (PostgreSQL 17+ or extension):
-- pg_uuidv7 extension:
CREATE EXTENSION pg_uuidv7;
SELECT uuid_generate_v7();
-- তুলনা:
-- v4: 550e8400-e29b-41d4-a716-446655440000 (random)
-- v7: 018e8e5a-5b87-74e4-b8b6-3a2e8a5c1234 (time-ordered)
```

---

### Range Types — PostgreSQL-Unique Powerful Type

```sql
-- Range types:
int4range, int8range, numrange
daterange, tsrange, tstzrange

-- Range literals:
'[1, 10)'::int4range   -- inclusive lower, exclusive upper: 1,2,...,9
'(1, 10]'::int4range   -- exclusive lower, inclusive upper: 2,...,10
'[1, 10]'::int4range   -- both inclusive
'(1, 10)'::int4range   -- both exclusive: 2,...,9
'[2024-01-01, 2024-12-31]'::daterange

-- Range operators:
SELECT '[1,10)'::int4range @> 5;   -- contains element: TRUE
SELECT '[1,5)'::int4range @> '[2,4)'::int4range;  -- contains range: TRUE
SELECT '[1,5)'::int4range && '[4,10)'::int4range;  -- overlaps: TRUE
SELECT '[1,5)'::int4range + '[4,10)'::int4range;   -- union: [1,10)
SELECT '[1,10)'::int4range * '[5,15)'::int4range;  -- intersection: [5,10)

-- Real use case: Hotel/room booking
CREATE TABLE room_bookings (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  room_id int NOT NULL,
  guest_name text NOT NULL,
  during daterange NOT NULL,

  -- EXCLUDE constraint: same room-এ overlapping booking prevent করো
  CONSTRAINT no_overlap EXCLUDE USING GIST (
    room_id WITH =,       -- same room_id
    during WITH &&        -- overlapping dates
  )
);

-- এখন:
INSERT INTO room_bookings (room_id, guest_name, during)
VALUES (101, 'Alice', '[2024-01-15, 2024-01-20)');  -- OK

INSERT INTO room_bookings (room_id, guest_name, during)
VALUES (101, 'Bob', '[2024-01-18, 2024-01-25)');    -- ERROR!
-- conflict with Alice's booking (overlap)

INSERT INTO room_bookings (room_id, guest_name, during)
VALUES (102, 'Bob', '[2024-01-18, 2024-01-25)');    -- OK (different room)

-- Query: কোন rooms available?
SELECT room_id FROM rooms
WHERE room_id NOT IN (
  SELECT room_id FROM room_bookings
  WHERE during && '[2024-01-15, 2024-01-20)'::daterange
);

-- GiST index for range queries
CREATE INDEX ON room_bookings USING GIST (room_id, during);
```

---

### Domain — Custom Type with Constraint

```sql
-- Domain = existing type + constraint
CREATE DOMAIN email_address AS text
  CHECK (VALUE ~ '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$');

CREATE DOMAIN positive_amount AS numeric(12,2)
  CHECK (VALUE > 0);

CREATE DOMAIN country_code AS char(2)
  CHECK (VALUE ~ '^[A-Z]{2}$');

-- Use:
CREATE TABLE customers (
  id bigint PRIMARY KEY,
  email email_address NOT NULL,  -- domain type
  country country_code,
  credit_limit positive_amount
);

-- Benefit: constraint logic centralizes
-- সব table-এ email validation একই domain দিয়ে
-- Constraint update: ALTER DOMAIN email_address ADD CONSTRAINT ...
```

---

### Type Selection Quick Reference

```
Auto-increment ID (small table):    serial / integer GENERATED ALWAYS AS IDENTITY
Auto-increment ID (large table):    bigserial / bigint GENERATED ALWAYS AS IDENTITY
Distributed/global unique ID:       uuid (v7 for ordered insert)
Boolean flag:                       boolean
Whole numbers (age, count < 32K):   smallint
Whole numbers (general):            integer
Whole numbers (large):              bigint
Money (exact):                      numeric(12,2) বা bigint (cents)
Scientific/approximate:             double precision
Fixed short text (codes):           char(2) বা varchar(n)
Variable text (general):            text
Long text (no limit):               text
Date only:                          date
Time only:                          time
Date + time:                        timestamptz (ALWAYS timezone-aware)
Duration:                           interval
Flexible schema:                    jsonb
Tags, lists:                        text[] বা jsonb
IP address:                         inet
Network:                            cidr
Date range, time range:             daterange, tstzrange
Custom validated type:              domain
Binary data (files, images):        bytea (বা external storage better)
```

---

### না জানলে কী হয়

**Scenario 1: float money calculation**
```
E-commerce order totals:
  price = 19.99, tax_rate = 0.08
  tax = 19.99 * 0.08 = 1.5992 → rounded 1.60
  
  float: 19.99 * 0.08 = 1.5992000000000002 → rounded 1.60 ← OK here
  But: sum of 1M such calculations → accumulated error = $0.02 total
  
  Audit: "Your reported revenue is $1,234,567.89 but actual bank = $1,234,567.87"
  → Two cents off. Accountants not happy.
  
  numeric: 19.99 * 0.08 = 1.5992 (exact) → round = 1.60 (exact)
  Sum of 1M: exact. No accumulated error.
```

**Scenario 2: timestamp without timezone on server move**
```
Old server: Asia/Dhaka timezone
  "2024-01-15 10:00:00" stored as timestamp → means 10:00 Dhaka time

Migrate to new server: UTC timezone
  Same value "2024-01-15 10:00:00" → now interpreted as 10:00 UTC!
  = 16:00 Dhaka time!
  
All historical timestamps 6 hours off!
Fix: mass update with timezone conversion → huge operation, potential data risk

If timestamptz was used:
  Old server stores: 2024-01-15 04:00:00 UTC (internally)
  New server reads: 2024-01-15 04:00:00 UTC → converts to local = correct!
  No problem!
```

**Scenario 3: varchar(255) everywhere**
```
MySQL background থেকে আসা developer:
  MySQL: varchar(255) everywhere (historical default)
  
  PostgreSQL-এ varchar(255) and text: same performance!
  কিন্তু varchar(255):
    - country_code varchar(255) → should be char(2) or varchar(2)
    - 255 byte limit meaningless for description field (should be text)
    - Index size unnecessarily large (B-Tree reserves max width)
    
  Proper design:
    code varchar(2) or char(2)  -- short fixed codes
    name text                    -- names (no practical limit)
    description text             -- long text
```

---

## Topic 8 — Table: Heap, Structure, Physical Storage-এর সাথে Connection

### Big Picture: Table Design = Physical Reality Design

Table CREATE করা মানে শুধু columns define করা না। এটা মানে:
- কীভাবে disk-এ pages ভরবে
- UPDATE করলে HOT হবে কিনা
- VACUUM কতটা effective হবে
- Index কতটা bloat পাবে

Topic 2-এ physical storage দেখেছিলাম। এখন সেই knowledge দিয়ে table design করতে শিখবো।

---

### CREATE TABLE — Physical Reality

```sql
CREATE TABLE orders (
  id          bigint GENERATED ALWAYS AS IDENTITY,
  user_id     bigint NOT NULL,
  amount      numeric(12, 2) NOT NULL,
  status      text NOT NULL DEFAULT 'pending',
  created_at  timestamptz NOT NULL DEFAULT now(),
  updated_at  timestamptz,
  notes       text,
  metadata    jsonb,
  PRIMARY KEY (id)
);
```

এই statement physically কী তৈরি করে?

```
1. Heap file: $PGDATA/base/<db_oid>/<table_oid>
2. FSM file:  ...<table_oid>_fsm
3. VM file:   ...<table_oid>_vm
4. Primary Key index (B-Tree): ...<index_oid>
5. Sequence for id: আলাদা object (pg_class-এ entry)
6. pg_class entry: table metadata
7. pg_attribute entries: প্রতিটা column-এর metadata
8. pg_constraint entry: PK constraint

Total disk objects: ৩ (heap + fsm + vm) + ১ (index) + ১ (sequence) = ৫+ files
```

---

### Column Order এবং Alignment — Real Impact

Topic 2-এ বলেছিলাম alignment waste হতে পারে। এখন design-level দেখি।

**Orders table-এর alignment analysis:**

```
Column:         Type:          Size:  Alignment:
id              bigint         8      8-byte (perfect)
user_id         bigint         8      8-byte (perfect)
amount          numeric(12,2)  variable 4-byte header
status          text           variable 4-byte header
created_at      timestamptz    8      8-byte
updated_at      timestamptz    8      8-byte
notes           text           variable 4-byte header
metadata        jsonb          variable 4-byte header

Fixed-size columns first (bigint, timestamptz) → variable-size last।
এই order already reasonable।
```

**Counter-example: Padding waste:**

```sql
-- এই table-এ প্রতিটা row extra padding waste করে:
CREATE TABLE wasteful (
  flag1   boolean,        -- 1 byte
                          -- 7 bytes padding (bigint needs 8-byte align)
  big1    bigint,         -- 8 bytes
  flag2   boolean,        -- 1 byte
                          -- 3 bytes padding (int needs 4-byte align)
  num1    integer,        -- 4 bytes
  flag3   boolean,        -- 1 byte
                          -- 7 bytes padding
  big2    bigint          -- 8 bytes
);
-- Per row fixed overhead: 1+7+8+1+3+4+1+7+8 = 40 bytes

-- Better:
CREATE TABLE efficient (
  big1    bigint,         -- 8 bytes
  big2    bigint,         -- 8 bytes
  num1    integer,        -- 4 bytes
  flag1   boolean,        -- 1 byte
  flag2   boolean,        -- 1 byte
  flag3   boolean         -- 1 byte + 1 byte padding (end alignment)
);
-- Per row fixed overhead: 8+8+4+1+1+1+1 = 24 bytes
-- 40% কম! 10M rows × 16 bytes = 160MB saved

-- দেখার tool:
SELECT attname, atttypid::regtype, attlen, attalign,
  CASE attalign
    WHEN 'c' THEN 1
    WHEN 's' THEN 2
    WHEN 'i' THEN 4
    WHEN 'd' THEN 8
  END AS align_bytes
FROM pg_attribute
WHERE attrelid = 'wasteful'::regclass AND attnum > 0
ORDER BY attnum;
```

---

### Table Storage Options — Physical Behavior Control

**fillfactor:**

```sql
-- Default fillfactor = 100 (page পুরো ভরো)
-- UPDATE-heavy table-এ কমাও:

CREATE TABLE order_status_log (
  id bigint PRIMARY KEY,
  order_id bigint NOT NULL,
  old_status text,
  new_status text,
  changed_at timestamptz DEFAULT now()
) WITH (fillfactor = 100);  -- INSERT-only log: 100 OK

CREATE TABLE inventory (
  product_id bigint PRIMARY KEY,
  quantity int NOT NULL,
  reserved int NOT NULL DEFAULT 0,
  updated_at timestamptz DEFAULT now()
) WITH (fillfactor = 70);
-- inventory frequently updated
-- 30% খালি রাখে → HOT update possible → index update avoid

-- fillfactor কত দেবে?
-- Insert-only (log, event): 100
-- Low update (< 10% rows/day): 90
-- Medium update (10-50%): 80
-- High update (50%+): 70
-- Very high (realtime counter): 60-70
```

**UNLOGGED table:**

```sql
-- WAL লেখে না → crash হলে empty হয়, কিন্তু অনেক fast
CREATE UNLOGGED TABLE session_cache (
  session_id text PRIMARY KEY,
  user_id bigint,
  data jsonb,
  expires_at timestamptz
);

-- Performance: INSERT/UPDATE/DELETE: 2-5x faster than logged table
-- কারণ: WAL write skip

-- Replication: standby-তে replicate হয় না!
-- Crash: PostgreSQL restart-এ table truncate হয়ে যায়

-- Use cases:
--   Session storage (Redis-এর alternative, acceptable to lose on crash)
--   Temporary computation results
--   Cache tables
--   ETL staging area
--   Rate limiting counters (lose on crash? maybe OK)

-- NOT for: any data you can't afford to lose
```

**TEMPORARY table:**

```sql
-- Session-এ exist করে, session end-এ automatically drop
CREATE TEMPORARY TABLE temp_order_ids (
  order_id bigint PRIMARY KEY
);

-- ON COMMIT options (transaction table):
CREATE TEMPORARY TABLE temp_ids (id bigint)
  ON COMMIT DELETE ROWS;  -- COMMIT-এ rows delete, table exists
  -- ON COMMIT DROP: COMMIT-এ table drop
  -- ON COMMIT PRESERVE ROWS: default, rows remain

-- Use cases: complex multi-step queries, intermediate results

-- Performance: UNLOGGED-এর মতো fast (WAL নেই)
-- Index on temp table:
CREATE INDEX ON temp_order_ids (order_id);  -- possible, useful for large temp tables

-- Visibility: শুধু current session দেখতে পায়
-- Two sessions: দুটো আলাদা temp table (same name, different backing)
```

---

### Tablespace — Data কোন Disk-এ থাকবে?

```sql
-- Tablespace = data directory mapping
-- Different disk/mount point-এ data রাখো

-- Fast NVMe SSD-এ hot table রাখো:
CREATE TABLESPACE fast_nvme LOCATION '/mnt/nvme0/pgdata';
-- Default (slow HDD)-এ cold/large table:
CREATE TABLESPACE slow_hdd LOCATION '/mnt/hdd0/pgdata';

-- Table তৈরি specific tablespace-এ:
CREATE TABLE hot_sessions (...) TABLESPACE fast_nvme;
CREATE TABLE archive_orders (...) TABLESPACE slow_hdd;

-- Move existing table:
ALTER TABLE orders SET TABLESPACE fast_nvme;
-- সাবধান: ACCESS EXCLUSIVE lock, table rewrite!
-- Large table-এ: hours of downtime

-- Index separate tablespace-এ:
CREATE INDEX idx_orders_user ON orders(user_id) TABLESPACE fast_nvme;

-- Database default tablespace:
ALTER DATABASE mydb SET TABLESPACE fast_nvme;

-- pg_default: $PGDATA/base/ (default)
-- pg_global: $PGDATA/global/ (system catalogs)

-- Tablespace info:
SELECT spcname, pg_tablespace_location(oid),
  pg_size_pretty(pg_tablespace_size(oid))
FROM pg_tablespace;
```

---

### Inheritance — এবং কেন এখন Partition Use করো

```sql
-- Table inheritance (legacy partitioning approach):
CREATE TABLE measurements (
  id bigint,
  sensor_id int,
  value numeric,
  recorded_at timestamptz NOT NULL
);

CREATE TABLE measurements_2023 () INHERITS (measurements);
CREATE TABLE measurements_2024 () INHERITS (measurements);

-- Parent-এ INSERT: সব child-এ যায় না automatically!
-- Trigger দিয়ে route করতে হতো।
-- Old, complex, many gotchas।

-- Modern: PARTITION BY (Topic 20-এ বিস্তারিত)
CREATE TABLE measurements (
  id bigint,
  sensor_id int,
  value numeric,
  recorded_at timestamptz NOT NULL
) PARTITION BY RANGE (recorded_at);
-- Routing automatic, partition pruning built-in, indexes inherited।
```

---

### Generated Columns — Computed, Stored

```sql
-- Generated column: অন্য column থেকে automatically computed
CREATE TABLE products (
  id bigint PRIMARY KEY,
  price_usd numeric(10,2) NOT NULL,
  tax_rate numeric(5,4) NOT NULL DEFAULT 0.15,
  price_with_tax numeric(10,2) GENERATED ALWAYS AS (price_usd * (1 + tax_rate)) STORED,
  -- STORED: disk-এ physically store করে (VIRTUAL: not yet supported in PG)
  name text NOT NULL,
  name_lower text GENERATED ALWAYS AS (lower(name)) STORED
);

-- Generated column-এ index করা যায়:
CREATE INDEX ON products (name_lower);
-- যেখানে expression index করতাম, এখন generated column-এ সহজ।

-- Can't INSERT/UPDATE generated column directly:
UPDATE products SET price_with_tax = 100;  -- ERROR

-- Useful for:
--   Denormalized computed values (query speed)
--   Pre-computed search fields (lower, tsvector)
--   Derived metrics stored for index

-- Generated column vs Expression Index:
--   Generated column: EXPLAIN-এ column দেখায়, app code-এ usable
--   Expression index: শুধু specific expression-এর জন্য index
```

---

### Row Size Limits এবং TOAST

```sql
-- Maximum row size: ~1.6GB (with TOAST)
-- Inline (no TOAST): per-row approx. ~2KB limit before TOAST kicks in
-- 8KB page: row must fit in a page if no TOAST

-- Single-row size check:
SELECT pg_column_size(row(id, user_id, amount, status, metadata)::orders)
FROM orders WHERE id = 1;

-- Table average row size:
SELECT pg_size_pretty(pg_total_relation_size('orders') / reltuples::bigint) AS avg_row_size
FROM pg_class WHERE relname = 'orders';

-- Real issue: wide rows = fewer rows per page = more I/O
-- 100 byte rows: 80 rows per page
-- 4000 byte rows: 2 rows per page
-- Seq scan: 40x more pages for same row count!
```

---

### System Columns — Hidden Columns

```sql
-- প্রতিটা table-এ hidden system columns:
SELECT xmin, xmax, cmin, cmax, ctid, tableoid, id, name FROM users LIMIT 5;

-- xmin: row insert-করা transaction XID
-- xmax: row delete-করা transaction XID (0 if live)
-- cmin: command ID (within transaction)
-- cmax: command ID of deleting command
-- ctid: physical location (page, offset)
-- tableoid: table's OID (partition-এ কোন partition থেকে এসেছে)

-- tableoid useful for partitioned table:
SELECT tableoid::regclass AS partition_name, count(*)
FROM orders  -- partitioned table
GROUP BY tableoid::regclass;
-- orders_2023: 500000
-- orders_2024: 750000

-- xmin দিয়ে "কখন এই row insert হয়েছিল" জানা:
-- (transaction timestamp থেকে approximate)
SELECT age(xmin) AS xid_age, * FROM users WHERE id = 1;
```

---

### Table Bloat Detection এবং Prevention

```sql
-- Bloat check (approximate, no lock):
SELECT
  relname AS table_name,
  n_live_tup,
  n_dead_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
  pg_size_pretty(pg_relation_size(relid)) AS table_size,
  last_vacuum,
  last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Exact bloat (pgstattuple, brief lock):
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT
  table_len,
  tuple_count,
  tuple_len,
  dead_tuple_count,
  dead_tuple_percent,
  free_space,
  free_percent
FROM pgstattuple('orders');

-- Bloat prevention strategy:
-- 1. Autovacuum tuning (scale_factor কমাও large tables-এ)
-- 2. fillfactor = 70-80 for UPDATE-heavy tables (HOT updates)
-- 3. Avoid large long-running transactions (prevent vacuum from cleaning)
-- 4. Regular VACUUM ANALYZE (especially after bulk DELETE)
-- 5. pg_repack for online bloat removal (production-safe)
```

---

## Topic 9 — Constraints: PK, FK, Unique, Check, Not Null

### Big Picture: Constraints = Database-এর Truth Guarantor

Application code bug হয়। Developer ভুল করে। Network timeout-এ partial state হয়। Constraint হলো database-এর last line of defense।

কিন্তু constraint শুধু validation না। প্রতিটা constraint-এর physical implications আছে — index তৈরি হয়, lock নেয়, performance affect করে।

---

### Primary Key — Physical Reality

```sql
-- PK declaration:
CREATE TABLE users (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);

-- Physical কী হয়:
-- 1. NOT NULL constraint enforce হয়
-- 2. UNIQUE constraint enforce হয়
-- 3. B-Tree index automatically তৈরি হয়:
--    CREATE UNIQUE INDEX users_pkey ON users (id);
-- 4. pg_constraint-এ entry
-- 5. pg_index-এ entry (indisprimary = true)

-- দেখো:
SELECT indexname, indexdef, indisprimary
FROM pg_indexes
JOIN pg_index ON indexrelid = (indexname::regclass)::oid
WHERE tablename = 'users';
```

**Natural Key vs Surrogate Key — কোনটা বেছে নেবে?**

```sql
-- Natural Key: real-world attribute that uniquely identifies
CREATE TABLE countries (
  iso_code char(2) PRIMARY KEY,  -- 'BD', 'US' — never changes
  name text NOT NULL,
  population bigint
);
-- Pros: meaningful, no join for the identifier
-- Cons: changes possible (rare, but ISO code history exists)
--       text comparison slightly slower than integer
--       FK references text — wider index entries

-- Surrogate Key: system-generated, meaningless
CREATE TABLE users (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email text UNIQUE NOT NULL,  -- natural key as unique, not PK
  name text NOT NULL
);
-- Pros: stable (never changes), narrow (8 bytes), sequential insert
-- Cons: extra join sometimes (need to join to get "real" identifier)

-- Decision rule:
-- Surrogate key: always, unless natural key is immutable AND narrow AND simple
-- ISO codes, currency codes: natural key OK
-- User email, product name: surrogate key (can change)
-- Composite natural keys: usually surrogate better (FK references complex)
```

**Composite Primary Key:**

```sql
-- Many-to-many junction table:
CREATE TABLE user_roles (
  user_id bigint NOT NULL REFERENCES users(id),
  role_id bigint NOT NULL REFERENCES roles(id),
  granted_at timestamptz DEFAULT now(),
  granted_by bigint REFERENCES users(id),
  PRIMARY KEY (user_id, role_id)
);

-- PK index: B-Tree on (user_id, role_id)
-- এই index (user_id) query-তেও কাজ করে (leftmost prefix)
-- কিন্তু শুধু (role_id) query-তে কাজ করে না

-- Additional index for role_id queries:
CREATE INDEX ON user_roles (role_id);
-- "কোন users এই role-এ আছে?" → role_id index

-- Partition key + PK:
-- Partitioned table-এ PK-তে partition key থাকতে হয়:
CREATE TABLE orders (
  id bigint GENERATED ALWAYS AS IDENTITY,
  tenant_id bigint NOT NULL,
  amount numeric(12,2),
  created_at timestamptz DEFAULT now(),
  PRIMARY KEY (id, tenant_id)  -- tenant_id partition key include
) PARTITION BY HASH (tenant_id);
```

---

### Foreign Key — Lock এবং Performance Impact

```sql
-- FK declaration:
CREATE TABLE orders (
  id bigint PRIMARY KEY,
  user_id bigint NOT NULL REFERENCES users(id)
    ON DELETE RESTRICT
    ON UPDATE CASCADE
);

-- FK physical behavior:
-- 1. Referential integrity check on every INSERT/UPDATE to child
-- 2. Referential integrity check on every DELETE/UPDATE to parent
-- 3. NO automatic index on FK column (in PostgreSQL!)
```

**FK Lock mechanics — critical production knowledge:**

```sql
-- INSERT into orders (child table):
-- 1. users table-এ user_id row-এ SHARE ROW EXCLUSIVE lock নেয়
-- 2. Row exists? OK, release lock
-- 3. Row doesn't exist? FK violation error

-- DELETE from users (parent table):
-- 1. orders table full scan করে user_id = deleted_user_id আছে কিনা
--    যদি index না থাকে on orders(user_id)!
-- 2. Index থাকলে: index scan

-- FK column-এ index MUST:
CREATE INDEX ON orders (user_id);
-- এটা না থাকলে:
--   DELETE FROM users → orders table full scan
--   1M orders? 1M row scan প্রতিটা user delete-এ!

-- Verify FK indexes:
-- FK আছে কিন্তু index নেই — এই query বের করো:
SELECT conrelid::regclass AS table_name,
  conname AS constraint_name,
  a.attname AS column_name
FROM pg_constraint c
JOIN pg_attribute a ON a.attrelid = c.conrelid
  AND a.attnum = ANY(c.conkey)
WHERE c.contype = 'f'  -- foreign key
  AND NOT EXISTS (
    SELECT 1 FROM pg_index i
    WHERE i.indrelid = c.conrelid
      AND a.attnum = ANY(i.indkey)
  )
ORDER BY table_name, constraint_name;
```

**ON DELETE / ON UPDATE options — কখন কোনটা:**

```sql
-- RESTRICT (default): parent delete করা যাবে না যদি child আছে
-- Parent-এর row protect করে।
-- Use: financial records, audit trails

-- CASCADE: parent delete হলে child-ও delete
-- Use: comments on posts (post delete → all comments delete)
-- WARNING: cascading delete অনেক row affect করতে পারে silently!

-- SET NULL: parent delete হলে FK column NULL হয়
-- Use: optional relationship (order.assigned_to nullable)

-- SET DEFAULT: parent delete হলে FK column default value নেয়
-- Use: rare, when there's a meaningful default reference

-- NO ACTION: RESTRICT-এর মতো, কিন্তু deferred possible

-- CASCADE example with unintended consequence:
-- users → orders (cascade) → order_items (cascade) → ...
-- DELETE FROM users WHERE id = 1;
-- → deletes 100 orders → deletes 500 order_items → ...
-- One DELETE triggers hundreds of deletes!
-- Audit log-এ কিছু নেই (no explicit deletes)
-- Careful with CASCADE chains।
```

**Deferrable FK — Circular Reference সমাধান:**

```sql
-- Circular reference problem:
-- employees.manager_id → employees.id
-- Bootstrap: insert employee with manager_id = some other employee
-- But that employee doesn't exist yet!

CREATE TABLE employees (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text NOT NULL,
  manager_id bigint REFERENCES employees(id) DEFERRABLE INITIALLY DEFERRED
);

BEGIN;
-- Insert CEO (no manager):
INSERT INTO employees (id, name, manager_id) OVERRIDING SYSTEM VALUE
VALUES (1, 'CEO', NULL);

-- Insert VP who reports to CEO:
INSERT INTO employees (id, name, manager_id) OVERRIDING SYSTEM VALUE
VALUES (2, 'VP', 1);

COMMIT;  -- FK check happens here, both rows exist now

-- Another use: mutual references
-- users.default_address_id → addresses.id
-- addresses.user_id → users.id
-- Both DEFERRABLE INITIALLY DEFERRED allows bootstrapping
```

---

### Unique Constraint — NULL Behavior এবং Partial Unique

```sql
-- Unique constraint:
CREATE TABLE users (
  id bigint PRIMARY KEY,
  email text UNIQUE NOT NULL,
  username text UNIQUE  -- nullable unique
);

-- Behind the scenes:
-- CREATE UNIQUE INDEX users_email_key ON users (email);
-- CREATE UNIQUE INDEX users_username_key ON users (username);

-- NULL-এর unique behavior:
INSERT INTO users (id, email, username) VALUES (1, 'a@a.com', NULL);
INSERT INTO users (id, email, username) VALUES (2, 'b@b.com', NULL);
-- Both NULL usernames: OK! NULL ≠ NULL in unique constraint.
-- Multiple NULL values allowed।
-- SQL Standard: NULL is distinct from NULL for uniqueness purposes।
-- (MySQL: same behavior)

-- NULLS NOT DISTINCT (PostgreSQL 15+):
CREATE UNIQUE INDEX ON users (username) NULLS NOT DISTINCT;
-- Now: two NULL usernames = UNIQUE violation!
-- Use: if you want truly unique including NULL (like "only one unassigned")

-- Composite unique:
CREATE TABLE team_members (
  team_id bigint,
  user_id bigint,
  role text,
  UNIQUE (team_id, user_id)  -- same user can't be in same team twice
);
-- NULL handling: if either column NULL → no unique violation
-- (NULL, 5) and (NULL, 5): both allowed!

-- Partial unique index (conditional uniqueness):
CREATE UNIQUE INDEX ON orders (user_id)
WHERE status = 'active';
-- একজন user-এর একটাই active order থাকবে।
-- কিন্তু multiple cancelled/completed orders OK।

-- This is NOT a constraint, it's an index.
-- EXPLAIN-এ "Index Scan using orders_user_id..." দেখাবে when querying with status='active'
```

---

### Check Constraint — Validation Logic

```sql
-- Simple check:
CREATE TABLE products (
  id bigint PRIMARY KEY,
  price numeric CHECK (price > 0),
  discount_pct numeric CHECK (discount_pct BETWEEN 0 AND 100),
  stock_count int CHECK (stock_count >= 0)
);

-- Named check (important for error messages):
CREATE TABLE employees (
  id bigint PRIMARY KEY,
  salary numeric CONSTRAINT salary_positive CHECK (salary > 0),
  hire_date date NOT NULL,
  end_date date,
  CONSTRAINT date_order CHECK (end_date IS NULL OR end_date > hire_date),
  CONSTRAINT valid_salary_range CHECK (salary BETWEEN 10000 AND 10000000)
);

-- Multi-column check:
CREATE TABLE flight_bookings (
  departure_airport char(3) NOT NULL,
  arrival_airport char(3) NOT NULL,
  CONSTRAINT different_airports CHECK (departure_airport != arrival_airport),
  departure_time timestamptz NOT NULL,
  arrival_time timestamptz NOT NULL,
  CONSTRAINT valid_flight_time CHECK (arrival_time > departure_time)
);

-- Check constraint limitation: no subqueries!
-- CHECK (user_id IN (SELECT id FROM users)) → ERROR
-- এই ধরনের cross-table validation: FK, Trigger, বা application logic
```

**Check constraint validation timing:**

```sql
-- Check: row insert/update-এর সময় evaluate হয়
-- Transaction-এ:

BEGIN;
INSERT INTO products (id, price) VALUES (1, -50);
-- ERROR: new row for relation "products" violates check constraint "products_price_check"
-- Transaction aborted!

-- Check constraint with complex expression:
CREATE TABLE schedules (
  id bigint PRIMARY KEY,
  day_of_week smallint CHECK (day_of_week BETWEEN 0 AND 6),
  start_hour smallint CHECK (start_hour BETWEEN 0 AND 23),
  end_hour smallint CHECK (end_hour BETWEEN 0 AND 23),
  CONSTRAINT valid_hours CHECK (
    CASE WHEN end_hour < start_hour
         THEN end_hour + 24 - start_hour <= 12  -- overnight shift max 12h
         ELSE end_hour - start_hour <= 12
    END
  )
);
```

---

### Not Null — The Silent Performance Boost

```sql
-- NOT NULL: seemingly simple
-- But physical impact:
-- 1. NULL bitmap in tuple header
--    Nullable column: 1 bit per column in null bitmap
--    NOT NULL column: no bit needed (smaller header)
-- 2. Query planner optimization:
--    NOT NULL columns → certain optimizations possible (e.g., index-only)
--    NULL possible → must check

-- Adding NOT NULL to existing table — the safe way:
-- Naive approach (DANGEROUS on large table):
ALTER TABLE orders ALTER COLUMN amount SET NOT NULL;
-- This scans every row to verify none are NULL
-- Takes ACCESS EXCLUSIVE lock!
-- Large table = long lock = production outage

-- Safe approach (PostgreSQL 12+: CHECK NOT VALID trick):
-- Step 1: Add constraint without validating existing rows
ALTER TABLE orders
  ADD CONSTRAINT amount_not_null CHECK (amount IS NOT NULL) NOT VALID;
-- Fast! No table scan. No long lock.

-- Step 2: Validate in background (SHARE UPDATE EXCLUSIVE lock — allows reads/writes!)
ALTER TABLE orders VALIDATE CONSTRAINT amount_not_null;
-- Scans table, but other operations continue

-- Step 3: Now add actual NOT NULL (planner can verify constraint exists)
ALTER TABLE orders ALTER COLUMN amount SET NOT NULL;
-- Fast! Constraint already validated, just updates pg_attribute

-- DEFAULT with NOT NULL:
ALTER TABLE orders ADD COLUMN priority int NOT NULL DEFAULT 0;
-- PostgreSQL 11+: fast ADD COLUMN with default (no table rewrite!)
-- Under the hood: stores default value in pg_attrdef, 
-- returns default for existing rows without rewriting pages
-- This is a huge improvement from PostgreSQL 10 and earlier

-- Verify (PostgreSQL 11+):
SELECT attname, attnotnull, atthasdef
FROM pg_attribute
WHERE attrelid = 'orders'::regclass AND attnum > 0;
```

---

### Constraint Deferability — Transaction Control

```sql
-- NOT DEFERRABLE (default):
--   প্রতিটা statement-এর পরে check হয়

-- DEFERRABLE INITIALLY IMMEDIATE:
--   Default: immediate
--   কিন্তু transaction-এ SET CONSTRAINTS DEFERRED করা যায়

-- DEFERRABLE INITIALLY DEFERRED:
--   Default: transaction শেষে check
--   কিন্তু SET CONSTRAINTS IMMEDIATE করতে পারো

-- Use case: data migration বা batch load
BEGIN;
SET CONSTRAINTS ALL DEFERRED;
-- এখন FK, UNIQUE সব deferred

INSERT INTO users (id, name, manager_id) VALUES (10, 'Alice', 20);  -- manager 20 নেই এখনো
INSERT INTO users (id, name, manager_id) VALUES (20, 'Bob', NULL);   -- manager created

COMMIT;  -- এখন সব constraint check হবে, OK
```

---

### Constraint Inspection

```sql
-- সব constraints দেখো
SELECT
  tc.table_name,
  tc.constraint_name,
  tc.constraint_type,
  pg_get_constraintdef(c.oid) AS definition,
  c.condeferrable,
  c.condeferred
FROM information_schema.table_constraints tc
JOIN pg_constraint c ON c.conname = tc.constraint_name
  AND c.conrelid = tc.table_name::regclass
WHERE tc.table_schema = 'public'
ORDER BY tc.table_name, tc.constraint_type;

-- FK relationships map করো:
SELECT
  src.relname AS source_table,
  string_agg(a.attname, ', ') AS source_columns,
  dst.relname AS target_table,
  c.confupdtype AS on_update,
  c.confdeltype AS on_delete
FROM pg_constraint c
JOIN pg_class src ON src.oid = c.conrelid
JOIN pg_class dst ON dst.oid = c.confrelid
JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY(c.conkey)
WHERE c.contype = 'f'
GROUP BY src.relname, dst.relname, c.confupdtype, c.confdeltype
ORDER BY src.relname;
```

---

## Topic 10 — Normalization: 1NF, 2NF, 3NF, BCNF এবং কখন Denormalize করতে হয়

### Big Picture: Normalization Theory এবং Real-World Pragmatism

Normalization একটা formal theory — mathematical rules দিয়ে "good" schema define করে। কিন্তু production database design হলো tradeoff — consistency vs performance, simplicity vs scalability।

Senior DBA বোঝে কখন normalize করবে এবং কখন জেনেশুনে violate করবে।

---

### Functional Dependency — Normalization-এর Mathematical Foundation

```
Functional Dependency (FD): A → B
মানে: A-এর value জানলে B-এর value uniquely determine হয়।

Examples:
  student_id → student_name   (student_id থেকে name determine হয়)
  (order_id, product_id) → quantity   (composite → determines quantity)
  zip_code → city   (zip থেকে city determine হয়)
  employee_id → department_id   (employee → one department)
  department_id → department_name   (dept_id → dept_name)

Transitive FD: A → B → C
  employee_id → department_id → department_name
  employee_id → department_name (transitively)

Partial FD: (A, B) → C, কিন্তু A → C alone
  (order_id, product_id) → product_name
  কিন্তু product_id → product_name (partial dependency!)
```

---

### 1NF — Atomicity এবং No Repeating Groups

**Rule:** প্রতিটা column atomic value রাখবে। কোনো multi-valued attribute নেই।

```
1NF Violation:
┌────────────────────────────────────────────────────────────┐
│ student_id │ name   │ courses                             │
│ 1          │ Alice  │ Math, Physics, Chemistry            │ ← multi-value!
│ 2          │ Bob    │ Math, Biology                       │
└────────────────────────────────────────────────────────────┘

Problems:
  - "Math" নেওয়া সব students খুঁজো → substring search, slow
  - Course add/remove করো → string manipulation
  - Course count → complex parsing

1NF conform:
┌─────────────────────────────────┐
│ student_id │ name  │ course     │
│ 1          │ Alice │ Math       │
│ 1          │ Alice │ Physics    │ ← name repeated (redundancy)
│ 1          │ Alice │ Chemistry  │
│ 2          │ Bob   │ Math       │
│ 2          │ Bob   │ Biology    │
└─────────────────────────────────┘
PK: (student_id, course)
```

**Modern nuance — PostgreSQL array এবং jsonb:**

```
Technically 1NF violation কিন্তু pragmatically acceptable:

CREATE TABLE posts (
  id bigint PRIMARY KEY,
  title text,
  tags text[]  -- ← 1NF violation, কিন্তু...
);

কেন acceptable?
  - tags-এ GIN index করা যায়: "এই tag আছে?" efficiently query করা যায়
  - tags-এর order meaningful হতে পারে
  - Junction table (post_tags) overhead বেশি: extra JOIN, extra index, extra I/O

কখন junction table better?
  - Tags-এ metadata থাকলে (tag created_at, tag color)
  - Tag-specific query বেশি (tag → all posts)
  - Tag count > ~100 প্রতি row
  - Referential integrity দরকার (tag must exist in tags table)

Array tags OK:
  - Simple string tags, no metadata
  - Post → tags query mostly
  - Controlled tag set (enum-like)
```

---

### 2NF — Partial Dependency Eliminate

**Rule:** 1NF + Composite PK-এর প্রতিটা non-key attribute পুরো PK-এর উপর depend করে।

```
2NF Violation:
PK: (order_id, product_id)

┌──────────────────────────────────────────────────────────────────┐
│ order_id │ product_id │ quantity │ product_name │ product_price  │
│ PK       │ PK         │          │ ← product_id only!           │
└──────────────────────────────────────────────────────────────────┘

Functional dependencies:
  (order_id, product_id) → quantity     ✓ full dependency
  product_id → product_name              ✗ partial dependency!
  product_id → product_price             ✗ partial dependency!

Problems:
  Update anomaly: product price বদলালে সব order rows update করতে হবে।
  Insert anomaly: product এর কোনো order না থাকলে product store করা যাবে না।
  Delete anomaly: last order delete হলে product info হারায়।

2NF conform:
orders_items: (order_id PK, product_id PK, quantity)
products:     (product_id PK, product_name, product_price)
```

---

### 3NF — Transitive Dependency Eliminate

**Rule:** 2NF + Non-key attribute অন্য non-key attribute-এর উপর depend না।

```
3NF Violation:
PK: employee_id

┌───────────────────────────────────────────────────────────────────┐
│ employee_id │ name │ department_id │ department_name │ manager_id │
│ PK          │      │               │ ← transitive!               │
└───────────────────────────────────────────────────────────────────┘

Functional dependencies:
  employee_id → department_id        ✓ direct
  department_id → department_name    ← transitive!
  department_id → manager_id         ← transitive!
  employee_id → department_name      (transitively through dept_id)

Problems:
  Department rename → সব employees-এ update।
  Department-এর কোনো employee না থাকলে department info store করা যাবে না।

3NF conform:
departments: (department_id PK, department_name, manager_id)
employees:   (employee_id PK, name, department_id FK → departments)
```

---

### BCNF — Boyce-Codd Normal Form

**Rule:** Every functional dependency X → Y-এ X must be a superkey।

3NF-এর চেয়ে slightly stricter। Rare violation, কিন্তু exists।

```
Classic example:
  Subject: Students can enroll in courses.
  Each course is taught by one teacher.
  Each teacher teaches one course (but a course may have multiple teachers).
  
  Wait, let's say: each teacher teaches only one course,
  each course can have multiple teachers,
  each student takes a course with a specific teacher.

Table: student_courses(student_id, course, teacher)

FDs:
  (student_id, course) → teacher  [a student in a course has one teacher]
  teacher → course                [a teacher teaches one course]

Candidate keys: (student_id, course) and (student_id, teacher)
teacher → course: teacher is NOT a superkey!
→ BCNF violation.

Decompose:
  teacher_courses(teacher PK, course)
  student_teachers(student_id PK, teacher FK)

Trade-off: BCNF decomposition sometimes loses information or adds join complexity.
3NF sometimes preferred over BCNF for practical reasons.
```

---

### কখন Denormalize করবে — এবং কীভাবে

**Denormalization হলো জেনেশুনে করা trade-off।** Redundancy তৈরি করো read speed-এর জন্য।

**Signal #1: JOIN-heavy slow queries:**

```sql
-- Normalized (5-table JOIN):
SELECT
  u.name, u.email,
  o.id AS order_id, o.created_at,
  p.name AS product_name, p.price,
  c.name AS category_name,
  s.name AS supplier_name
FROM users u
JOIN orders o ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
JOIN categories c ON c.id = p.category_id
JOIN suppliers s ON s.id = p.supplier_id
WHERE o.created_at > now() - interval '7 days';

-- EXPLAIN: 5 Hash Joins, large intermediate result sets
-- Execution: 500ms

-- Denormalized (pre-join in reporting table):
CREATE TABLE order_report (
  order_id bigint,
  user_name text,
  user_email text,
  product_name text,
  product_price numeric,
  category_name text,
  supplier_name text,
  created_at timestamptz
);
-- Populated by trigger বা scheduled refresh
-- Query: SELECT * FROM order_report WHERE created_at > now() - '7 days';
-- Execution: 5ms (single table scan with index)
```

**Signal #2: Counter/Aggregate frequently needed:**

```sql
-- Normalized: প্রতিবার count করো
SELECT user_id, count(*) FROM orders GROUP BY user_id;
-- Large orders table → slow, especially if no index or stale stats

-- Denormalized: pre-computed counter
ALTER TABLE users ADD COLUMN order_count int NOT NULL DEFAULT 0;
ALTER TABLE users ADD COLUMN total_spent numeric(12,2) NOT NULL DEFAULT 0;

-- Trigger to maintain:
CREATE OR REPLACE FUNCTION update_user_stats()
RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE users SET
      order_count = order_count + 1,
      total_spent = total_spent + NEW.amount
    WHERE id = NEW.user_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE users SET
      order_count = order_count - 1,
      total_spent = total_spent - OLD.amount
    WHERE id = OLD.user_id;
  ELSIF TG_OP = 'UPDATE' THEN
    UPDATE users SET
      total_spent = total_spent - OLD.amount + NEW.amount
    WHERE id = NEW.user_id;
  END IF;
  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER maintain_user_stats
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION update_user_stats();

-- Query now:
SELECT order_count, total_spent FROM users WHERE id = 42;
-- Index lookup, instant!
-- Trade-off: trigger overhead on every order INSERT/UPDATE/DELETE
```

**Signal #3: Lookup value copy:**

```sql
-- Orders-এ user name copy (reporting convenience):
ALTER TABLE orders ADD COLUMN customer_name text;

-- Populate:
UPDATE orders o SET customer_name = u.name FROM users u WHERE u.id = o.user_id;

-- Risk: user name বদলালে orders-এ stale data থাকে।
-- Acceptable if: historical order record should show name AT TIME OF ORDER
-- Not acceptable if: always show current name
```

---

### Materialized View — Best of Both Worlds

```sql
-- Normalized source, denormalized query result
CREATE MATERIALIZED VIEW sales_summary AS
SELECT
  date_trunc('day', o.created_at) AS sale_date,
  c.name AS category,
  sum(oi.quantity * p.price) AS revenue,
  count(DISTINCT o.id) AS order_count,
  count(DISTINCT o.user_id) AS unique_customers
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
JOIN categories c ON c.id = p.category_id
WHERE o.status = 'completed'
GROUP BY 1, 2
ORDER BY 1, 2;

-- Index for fast query:
CREATE UNIQUE INDEX ON sales_summary (sale_date, category);
CREATE INDEX ON sales_summary (sale_date);

-- Refresh strategies:
-- Option 1: Full refresh (simple, consistent)
REFRESH MATERIALIZED VIEW sales_summary;
-- Blocks read during refresh! Use CONCURRENTLY:

REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
-- Non-blocking (UNIQUE INDEX required)
-- Old data served during refresh
-- Refresh takes longer but no downtime

-- Option 2: Incremental (complex, custom)
-- Not built-in. Trigger-based or scheduled partial refresh.

-- Option 3: Scheduled refresh
-- pg_cron extension:
SELECT cron.schedule('refresh-sales', '0 * * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary');

-- Option 4: Event-driven (after large batch insert)
-- Application calls REFRESH after bulk load
```

---

### Denormalization Decision Matrix:

```
                    Write Frequency
                    Low          High
Read Frequency  ┌────────────┬───────────────────┐
High            │Denormalize │Carefully denorm   │
                │(read wins) │(counter pattern,  │
                │            │ trigger maintained)│
                ├────────────┼───────────────────┤
Low             │Normalize   │Normalize          │
                │(simple)    │(write perf first) │
                └────────────┴───────────────────┘

Denormalization tools:
  Pre-computed column: simple, trigger-maintained, fast read
  Materialized view: auto-computed, scheduled refresh, complex queries
  Reporting table: ETL-populated, historical snapshots, analytics
  Denormalized copy: event sourcing, audit trail, immutable records
  JSONB blob: flexible schema, varying attributes, document model

When NEVER to denormalize:
  Financial transactions: must be 100% consistent
  Inventory: race conditions with denormalized counts (use SELECT FOR UPDATE)
  Any data with strict ACID requirements
```

---

### না জানলে কী হয়

**Scenario 1: 2NF violation এবং Update Anomaly**
```
E-commerce যেখানে product price order table-এ stored:

orders: (order_id, product_id, quantity, product_price)
product_id → product_price (partial dependency)

Product price বাড়লো ৳100 থেকে ৳120:
  UPDATE orders SET product_price = 120
  WHERE product_id = 5;  -- 50,000 rows!

  Transaction লাগবে। Lock লাগবে। Slow।
  Missing one row? Inconsistency।

Fix: orders শুধু price_at_time_of_order store করে (historical snapshot):
  order_items: (order_id, product_id, quantity, unit_price_at_purchase)
  products: (product_id, current_price)
  unit_price_at_purchase = products.price at order time (snapshot)
  
এটা denormalization? না — এটা intentional historical data.
```

**Scenario 2: Over-normalization এবং Performance**
```
Dashboard query: "Last 7 days sales by category"
5-table normalized JOIN:
  EXPLAIN ANALYZE: 2.3 seconds
  Dashboard: loads slowly, users unhappy

Team decision: "Add more indexes"
  8 indexes later: 1.8 seconds (still slow)
  Write performance degraded!

Real fix: Materialized view, refreshed hourly
  Query: 12ms
  Data freshness: up to 1 hour old (acceptable for dashboard)
  Write performance: unchanged (no extra indexes on source tables)
```

---

### Block 2 Summary — Dots Connected

```
Topic 6 (Database & Schema)
    │
    ├──► Topic 21 (Users & Roles): Schema-level permissions
    ├──► Topic 22 (RLS): Row-level security = shared schema multi-tenancy
    └──► Topic 26 (Replication): Database-level replication strategy

Topic 7 (Data Types)
    │
    ├──► Topic 2 (Physical Storage): Type alignment = padding waste
    ├──► Topic 11 (Index): Type choice = index size = index efficiency
    └──► Topic 15 (Vacuum): TOAST = separate vacuum needed

Topic 8 (Table Design)
    │
    ├──► Topic 2 (Physical Storage): fillfactor = HOT update rate
    ├──► Topic 15 (Vacuum): Bloat detection and prevention
    └──► Topic 20 (Partition): Partition = multiple heap files

Topic 9 (Constraints)
    │
    ├──► Topic 4 (Concurrency): FK lock mechanics
    ├──► Topic 11 (Index): Constraint-backed indexes (PK, Unique)
    └──► Topic 15 (Vacuum): Constraint validation requires table scan

Topic 10 (Normalization)
    │
    ├──► Topic 12 (Query Execution): JOIN strategies depend on normalization level
    ├──► Topic 16 (Views): Materialized view = controlled denormalization
    └──► Topic 18 (Triggers): Counter maintenance via triggers
```
# Senior DBA Master Guide — Block 3
## Performance & Query Optimization

---

## Topic 11 — Index: B-Tree, Hash, GIN, GiST, BRIN — কখন কাজে লাগে না

### Big Picture: Index কী, এবং কেন এত Critical?

Index হলো একটা separate data structure যেটা table-এর data-র উপর built, এবং specific query pattern-কে dramatically fast করে। কিন্তু index cost-free না:

```
Index-এর cost:
  Write overhead: প্রতিটা INSERT/UPDATE/DELETE-এ index-ও update করতে হয়
  Storage: index নিজেও disk space নেয় (sometimes table-এর চেয়ে বড়!)
  Memory: Buffer Pool-এ index pages cache করতে হয়
  Maintenance: VACUUM index-ও clean করতে হয়
  Planning time: planner প্রতিটা query-তে index consider করে

Index-এর benefit:
  Selective query: লক্ষ row-এর মধ্যে কয়েকটা row খুঁজে পাওয়া
  Sort avoid: ORDER BY column-এ index থাকলে sort skip
  Index-Only Scan: heap visit ছাড়াই result দেওয়া

Senior DBA-র job: কোন index রাখবো, কোনটা রাখবো না — এই decision।
```

---

### B-Tree — সবচেয়ে Common Index

**Internal Structure:**

```
B-Tree = Balanced Tree
প্রতিটা node-এ keys + child pointers।
Leaf nodes-এ actual index entries (key + ctid)।
All leaves same depth।
Self-balancing — insert/delete-এ automatic rebalance।

B-Tree-এর "B" মানে Balanced, Bayer (inventor), বা Broad।

Structure (simplified, order-3 B-Tree):

                    [30 | 60]                    ← Root node
                   /    |    \
           [10|20]   [40|50]   [70|80]           ← Internal nodes
          /  |  \   /  |  \   /  |  \
        [5] [15] [25] [35] [45] [55] [65] [75] [85]  ← Leaf nodes
         |    |    |    |    |    |    |    |    |
       ctids ctids ...                             ← heap pointers

Leaf nodes: doubly linked list (range scan-এর জন্য)
```

**B-Tree properties PostgreSQL-এ:**

```
Default page size: 8KB (same as heap)
B-Tree page structure:
  BTreePageOpaqueData: type (leaf/internal/root/deleted), left/right sibling LSN
  Item pointers + key data

Fill factor: 90% (not 100%!) by default for indexes
  কেন? Sequential insert-এ split কম হয়
  Random insert-এ: split হলেও fragment কম

Node split (when page full):
  Page 90% full + new entry → split into two pages
  Parent-এ new separator key + new child pointer
  Root split → tree height বাড়ে (rare, amortized cost)

Deletion: "mark deleted" (not immediate reclaim)
  VACUUM later reclaims
  Heavy delete workload → index bloat
```

```sql
-- B-Tree index দেখো
CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_orders_created ON orders (created_at DESC);  -- descending
CREATE INDEX idx_orders_composite ON orders (user_id, status, created_at);

-- B-Tree internal stats (pg_stats):
SELECT
  attname,
  correlation,  -- physical vs logical order (-1 to 1)
  n_distinct,   -- distinct values (negative = fraction of rows)
  most_common_vals,
  most_common_freqs,
  histogram_bounds
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'user_id';

-- correlation = 1.0: physically sorted by this column
--   Index scan = sequential heap reads = very efficient
-- correlation ≈ 0: random heap access
--   Index scan = random I/O = expensive (might prefer seq scan!)

-- B-Tree page inspection (pageinspect):
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT * FROM bt_page_stats('idx_orders_user_id', 0);
-- type: r (root), l (leaf), i (internal)
-- live_items, dead_items, free_size, btpo_level

SELECT * FROM bt_page_items('idx_orders_user_id', 1) LIMIT 5;
-- itemoffset, ctid, itemlen, nulls, vars, data (key value in hex)
```

---

### Composite B-Tree — Column Order Matters Enormously

**Leftmost Prefix Rule:**

```
Index: (a, b, c)

Query → Uses index?
WHERE a = 1                      → ✅ uses (a, b, c) index on column a
WHERE a = 1 AND b = 2            → ✅ uses (a, b, c) on columns a, b
WHERE a = 1 AND b = 2 AND c = 3 → ✅ full index
WHERE b = 2                      → ❌ leading column a missing
WHERE b = 2 AND c = 3            → ❌ leading column a missing
WHERE a = 1 AND c = 3            → ⚠️ index on a only, c filtered separately
WHERE a BETWEEN 1 AND 10         → ✅ range on a
WHERE a BETWEEN 1 AND 10 AND b = 2 → ⚠️ range on a + filter b (b not indexed efficiently)
WHERE a = 1 AND b BETWEEN 1 AND 10 AND c = 3 → ✅ a equality + b range + c filtered

Range column should be LAST in composite index!
```

**Column Order Decision:**

```
Rule 1: Equality conditions → leftmost columns
Rule 2: Range condition → rightmost of the used columns
Rule 3: Higher selectivity column → earlier (not always, depends on query mix)
Rule 4: ORDER BY column → rightmost (can eliminate sort)

Example:
  Queries:
    WHERE user_id = ? AND status = 'active'                     (frequent)
    WHERE user_id = ? AND status = 'active' ORDER BY created_at (frequent)
    WHERE user_id = ?                                           (occasional)

  Good index: (user_id, status, created_at)
    user_id: equality, leftmost ✓
    status: equality, second ✓
    created_at: ORDER BY, rightmost ✓
    covers all three query patterns

  Bad index: (status, user_id, created_at)
    WHERE user_id = ? → ❌ status leading column missing
```

```sql
-- Composite index planning:
-- Queries-এর actual usage pattern দেখো:
SELECT query, calls, mean_exec_time, rows
FROM pg_stat_statements
WHERE query LIKE '%orders%'
ORDER BY calls DESC;

-- Index usage statistics:
SELECT
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE tablename = 'orders'
ORDER BY idx_scan DESC;
```

---

### Hash Index

**Internal Structure:**

```
Hash function: hash(key) → bucket number
Bucket: list of (key, ctid) pairs
Overflow buckets: when primary bucket full

PostgreSQL hash index structure:
  Meta page: hash function info, split point, fill factor
  Bucket pages: primary + overflow
  Bitmap pages: free space tracking

hash('alice@example.com') → bucket 42 → [(ctid1), (ctid2)]
hash('bob@example.com')   → bucket 17 → [(ctid3)]
```

```sql
-- Hash index
CREATE INDEX idx_users_email_hash ON users USING HASH (email);

-- Supports ONLY:
SELECT * FROM users WHERE email = 'alice@example.com';  -- ✅ uses hash index

-- Does NOT support:
SELECT * FROM users WHERE email LIKE 'alice%';  -- ❌ no hash for range/prefix
SELECT * FROM users ORDER BY email;             -- ❌ no ordering
SELECT * FROM users WHERE email > 'a@a.com';    -- ❌ no range

-- When hash beats B-Tree:
-- Long string keys: hash is constant size, B-Tree stores full key
-- Pure equality lookups only
-- High-cardinality text column (email, token, hash)

-- PostgreSQL 10+: WAL-safe hash index
-- Before 10: hash index not crash-safe! (had to be rebuilt after crash)
-- Now: safe to use in production

-- In practice: B-Tree is usually competitive even for equality
-- Hash index: marginal benefit, narrow use case
-- Most production systems: just use B-Tree
```

---

### GIN — Generalized Inverted Index

**Internal Structure:**

```
GIN = Inverted index (like full-text search engines)

For a column with multiple "values" per row (array, jsonb, tsvector):
  Maps: element → list of ctids containing that element

Example: tags column
  Row 1 (ctid=0,1): tags=['postgresql', 'database', 'performance']
  Row 2 (ctid=0,2): tags=['mysql', 'database']
  Row 3 (ctid=0,3): tags=['postgresql', 'index']

GIN index:
  'database'    → [ctid(0,1), ctid(0,2)]
  'index'       → [ctid(0,3)]
  'mysql'       → [ctid(0,2)]
  'performance' → [ctid(0,1)]
  'postgresql'  → [ctid(0,1), ctid(0,3)]

Query: WHERE tags @> ARRAY['postgresql']
  GIN lookup: 'postgresql' → [ctid(0,1), ctid(0,3)]
  Result: rows 1 and 3

Query: WHERE tags @> ARRAY['postgresql', 'database']
  'postgresql' → {0,1; 0,3}
  'database'   → {0,1; 0,2}
  Intersection: {0,1} → row 1

No heap visit needed for containment! (unless visibility check fails)
```

**GIN pending list (fast insert):**

```
GIN update is expensive (maintain inverted lists)।
PostgreSQL optimization: "pending list" in GIN meta page

New inserts → pending list (not immediately indexed)
Pending list: small, in memory-friendly
VACUUM বা threshold hit → pending list flush to main GIN structure

gin_pending_list_limit (default 4MB):
  Threshold পার হলে auto-cleanup

Trade-off:
  Insert: fast (pending list)
  Query: pending list + main index both searched
  After vacuum: fully indexed, fast query
```

```sql
-- GIN for different types:

-- Array:
CREATE INDEX ON posts USING GIN (tags);
SELECT * FROM posts WHERE tags @> ARRAY['postgresql'];
SELECT * FROM posts WHERE tags && ARRAY['postgresql', 'mysql'];  -- overlap

-- JSONB:
CREATE INDEX ON events USING GIN (data);
SELECT * FROM events WHERE data @> '{"action": "login"}';
SELECT * FROM events WHERE data ? 'amount';  -- key exists

-- JSONB path ops (smaller index, only @> operator):
CREATE INDEX ON events USING GIN (data jsonb_path_ops);
-- Better for containment queries, doesn't support ? and ?| ?&

-- Full-text search:
CREATE INDEX ON articles USING GIN (to_tsvector('english', content));
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'postgresql & performance');

-- Trigram (pg_trgm extension):
CREATE EXTENSION pg_trgm;
CREATE INDEX ON users USING GIN (name gin_trgm_ops);
SELECT * FROM users WHERE name ILIKE '%ali%';  -- GIN used!
SELECT * FROM users WHERE name % 'Alice';       -- similarity search

-- GIN vs GiST for full-text:
-- GIN: faster query, slower update, larger index
-- GiST: faster update, slower query, smaller index
-- Recommendation: GIN for mostly-read, GiST for frequently updated
```

---

### GiST — Generalized Search Tree

```
GiST = Framework for building custom index types
  R-Tree (geometric), interval trees, etc. — all via GiST

GiST stores bounding boxes / ranges at internal nodes
  Internal node: "any child could have values in this range"
  Leaf node: actual key + ctid

Range example:
  Row 1: during = [Jan 1, Jan 15]
  Row 2: during = [Jan 10, Jan 25]
  Row 3: during = [Feb 1, Feb 28]

GiST internal node might store: [Jan 1, Feb 28] (bounding)
Child 1: [Jan 1, Jan 25]
Leaves: Row1, Row2

Overlap query: '&&' → traverse tree, check overlap
False positives possible at internal nodes (check heap)
```

```sql
-- GiST for ranges:
CREATE INDEX ON bookings USING GIST (during);
SELECT * FROM bookings WHERE during && '[2024-01-15, 2024-01-20)'::tstzrange;
SELECT * FROM bookings WHERE during @> '2024-01-17'::timestamptz;  -- contains point

-- GiST for geometry (PostGIS):
CREATE INDEX ON locations USING GIST (geom);
SELECT * FROM locations
WHERE ST_DWithin(geom, ST_MakePoint(90.4, 23.7)::geography, 1000);  -- within 1km

-- GiST for trigram (fuzzy search, smaller than GIN):
CREATE INDEX ON users USING GIST (name gist_trgm_ops);
SELECT * FROM users WHERE name % 'Ahme';  -- similarity match
-- similarity(name, 'Ahme') > pg_trgm.similarity_threshold

-- GiST EXCLUDE constraint:
CREATE TABLE room_bookings (
  room_id int,
  during tstzrange,
  EXCLUDE USING GIST (room_id WITH =, during WITH &&)
);
-- Prevents overlapping bookings for same room
```

---

### BRIN — Block Range INdex

**Internal Structure:**

```
BRIN = Extremely compact index for naturally-ordered data

Instead of indexing each value, stores summary per block range:
  Block range: 128 pages (default, configurable)
  Summary: min and max values within those blocks

For a time-series table with naturally ordered timestamps:

  Blocks 0-127:   min=2024-01-01, max=2024-01-05
  Blocks 128-255: min=2024-01-05, max=2024-01-12
  Blocks 256-383: min=2024-01-12, max=2024-01-20
  ...

Query: WHERE created_at BETWEEN '2024-01-10' AND '2024-01-15'
  Block range 0-127: max=Jan5 < Jan10 → SKIP
  Block range 128-255: overlaps Jan10-Jan15 → CHECK (might have false positives)
  Block range 256-383: min=Jan12 ≤ Jan15 → CHECK
  Block range 384+: min > Jan15 → SKIP

Size: ~1 page per 128 block ranges = tiny!
A 1TB table → ~8MB BRIN index
vs B-Tree: might be 100GB+
```

```sql
-- BRIN index:
CREATE INDEX ON sensor_readings USING BRIN (recorded_at);
-- or with custom range size:
CREATE INDEX ON sensor_readings USING BRIN (recorded_at)
  WITH (pages_per_range = 64);

-- BRIN works well when:
-- Data physically ordered (time-series, auto-increment ID)
-- Range queries (not point lookups)
-- Very large tables (TB scale)
-- Insert-only or append-only workload

-- BRIN fails when:
-- Random insert order (correlation ≈ 0)
-- Point lookups (specific value)
-- High cardinality without physical order

-- Check correlation first:
SELECT correlation FROM pg_stats
WHERE tablename = 'sensor_readings' AND attname = 'recorded_at';
-- correlation > 0.9: BRIN excellent
-- correlation < 0.5: BRIN useless, use B-Tree

-- BRIN vs B-Tree for time-series:
-- 100M row table, 365 days of data:
-- B-Tree: ~8GB index, random access O(log n)
-- BRIN: ~1MB index, range scan O(n/pages_per_range) + false positive heap checks
-- For range queries on ordered data: BRIN wins on storage, similar query speed

-- Update BRIN after bulk insert:
SELECT brin_summarize_new_values('idx_sensor_recorded_at');
-- Summarizes new block ranges not yet in index
```

---

### Partial Index — Selective Indexing

```sql
-- Index শুধু relevant rows-এর:

-- Pattern 1: Status-based (common query pattern)
CREATE INDEX ON orders (user_id) WHERE status = 'pending';
-- শুধু pending orders index হয়
-- Much smaller than full index
-- Query: WHERE user_id = 42 AND status = 'pending' → uses partial index

-- Pattern 2: Non-null values
CREATE INDEX ON users (phone_number) WHERE phone_number IS NOT NULL;
-- 60% users have no phone → don't index them

-- Pattern 3: Recent data (rolling window)
CREATE INDEX ON events (user_id, created_at)
WHERE created_at > '2024-01-01';
-- শুধু 2024 data index
-- Historical data: Seq scan বা BRIN
-- কিন্তু: static date → index becomes useless over time!
-- Better: partition by date

-- Pattern 4: Soft delete
CREATE INDEX ON users (email) WHERE deleted_at IS NULL;
-- Active users-এর email unique lookup
-- Deleted users excluded from index

-- Pattern 5: Unique within subset
CREATE UNIQUE INDEX ON subscriptions (user_id)
WHERE status = 'active';
-- One active subscription per user
-- Multiple cancelled subscriptions allowed

-- Partial index size check:
SELECT indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  pg_size_pretty(pg_total_relation_size(indexrelid)) AS total_size,
  idx_scan
FROM pg_stat_user_indexes
WHERE tablename = 'orders';
```

---

### Expression Index (Function-Based Index)

```sql
-- Index on expression result:

-- Case-insensitive search:
CREATE INDEX ON users (lower(email));
SELECT * FROM users WHERE lower(email) = 'alice@example.com';
-- এখন index ব্যবহার করবে

-- Date truncation:
CREATE INDEX ON orders (date_trunc('day', created_at));
SELECT * FROM orders WHERE date_trunc('day', created_at) = '2024-01-15';

-- JSONB field:
CREATE INDEX ON events ((data->>'user_id')::bigint);
SELECT * FROM events WHERE (data->>'user_id')::bigint = 42;
-- Type cast include করা জরুরি! Index-এ যেভাবে আছে query-তেও সেভাবে

-- Computed hash (for long text equality):
CREATE INDEX ON documents (md5(content));
SELECT * FROM documents WHERE md5(content) = md5('some long text here');

-- Expression index + IMMUTABLE function:
-- Function must be IMMUTABLE (same input → always same output)
-- VOLATILE function: index করা যাবে না
CREATE OR REPLACE FUNCTION fiscal_year(ts timestamptz)
RETURNS int LANGUAGE sql IMMUTABLE AS $$
  SELECT CASE WHEN extract(month FROM ts) >= 7
    THEN extract(year FROM ts)::int
    ELSE extract(year FROM ts)::int - 1
  END;
$$;
CREATE INDEX ON orders (fiscal_year(created_at));
SELECT * FROM orders WHERE fiscal_year(created_at) = 2024;
```

---

### Covering Index (INCLUDE) — Index-Only Scan

```sql
-- Problem: index-এ key আছে কিন্তু query-তে extra column দরকার
-- → heap visit করতে হয়

-- Index: (user_id)
-- Query: SELECT user_id, status, amount FROM orders WHERE user_id = 42
-- → index থেকে ctid → heap থেকে status, amount

-- Solution: INCLUDE extra columns in index leaf nodes
CREATE INDEX ON orders (user_id) INCLUDE (status, amount);

-- এখন:
-- Index leaf: (user_id, ctid, status, amount)
-- Query can be answered from index alone → Index-Only Scan!
-- Heap visit শুধু visibility check-এর জন্য (VM-এ all-visible হলে heap-ও না)

-- INCLUDE columns:
-- Stored only in leaf nodes (not internal nodes)
-- NOT used for filtering or ordering
-- Just "carried along" for coverage
-- No size impact on tree traversal (internal nodes small)

-- When to use INCLUDE:
-- Frequently queried column combination
-- Query: SELECT key_cols, extra_cols FROM t WHERE key_cols = ?
-- extra_cols in INCLUDE → Index-Only Scan

-- Check Index-Only Scan in action:
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, status, amount FROM orders WHERE user_id = 42;
-- "Index Only Scan" → good
-- "Heap Fetches: 0" → perfect (all pages all-visible)
-- "Heap Fetches: N" → some pages not all-visible (VACUUM করো)

-- Composite index vs INCLUDE:
CREATE INDEX ON orders (user_id, status);        -- both used for filtering/ordering
CREATE INDEX ON orders (user_id) INCLUDE (status); -- status only for coverage
-- If you filter on status too: use (user_id, status) not INCLUDE
```

---

### কখন Index কাজ করে না — Complete Gotcha List

**Gotcha 1: Implicit Type Cast**

```sql
-- orders.user_id column type: bigint
-- Application sends string from ORM:
SELECT * FROM orders WHERE user_id = '42';
-- PostgreSQL casts '42' to bigint → index works here (safe cast)

-- But:
-- orders.code column type: varchar
SELECT * FROM orders WHERE code = 42;  -- integer
-- PostgreSQL must cast every varchar value → Seq Scan!

-- More dangerous:
-- users.created_at column type: timestamptz
SELECT * FROM users WHERE created_at = '2024-01-15';
-- '2024-01-15' → timestamptz cast works, but:
-- What timezone? Server timezone-এ interpret হবে
-- If you meant UTC midnight but server is UTC+6 → wrong results!

-- Always explicit cast:
SELECT * FROM users WHERE created_at >= '2024-01-15'::timestamptz AT TIME ZONE 'UTC';
```

**Gotcha 2: Function Wrapping the Column**

```sql
-- Index: (created_at)

-- ভুল — function wraps the indexed column:
SELECT * FROM orders WHERE date(created_at) = '2024-01-15';
SELECT * FROM orders WHERE extract(year FROM created_at) = 2024;
SELECT * FROM orders WHERE to_char(created_at, 'YYYY') = '2024';
-- All: Seq Scan! Index can't be used.

-- ঠিক — range query on the column:
SELECT * FROM orders
WHERE created_at >= '2024-01-15 00:00:00'::timestamptz
  AND created_at < '2024-01-16 00:00:00'::timestamptz;

SELECT * FROM orders
WHERE created_at >= date_trunc('year', '2024-01-01'::timestamptz)
  AND created_at < date_trunc('year', '2025-01-01'::timestamptz);

-- অথবা expression index:
CREATE INDEX ON orders (date(created_at));
SELECT * FROM orders WHERE date(created_at) = '2024-01-15';  -- now uses index
```

**Gotcha 3: Leading Wildcard**

```sql
-- Index: (name)

SELECT * FROM users WHERE name LIKE 'Ali%';   -- ✅ prefix → index used (B-Tree)
SELECT * FROM users WHERE name LIKE '%Ali%';  -- ❌ leading % → Seq Scan
SELECT * FROM users WHERE name LIKE '%ali';   -- ❌ trailing only → Seq Scan
SELECT * FROM users WHERE name ILIKE 'Ali%';  -- ❌ ILIKE → case-insensitive, different

-- Fix for LIKE '%Ali%':
CREATE EXTENSION pg_trgm;
CREATE INDEX ON users USING GIN (name gin_trgm_ops);
SELECT * FROM users WHERE name LIKE '%Ali%';  -- GIN trigram index used!

-- Fix for ILIKE:
CREATE INDEX ON users USING GIN (lower(name) gin_trgm_ops);
SELECT * FROM users WHERE lower(name) LIKE '%ali%';
-- or:
SELECT * FROM users WHERE name ILIKE '%Ali%';  -- GIN with gin_trgm_ops handles ILIKE
```

**Gotcha 4: OR Conditions**

```sql
-- Two separate indexes: idx_a on (a), idx_b on (b)
SELECT * FROM t WHERE a = 1 OR b = 2;

-- Plan options:
-- Option 1: Seq Scan (if low selectivity)
-- Option 2: Bitmap Index Scan on idx_a + Bitmap Index Scan on idx_b → BitmapOr
--   Each index returns a bitmap of matching pages
--   BitmapOr: union of bitmaps
--   Then heap scan those pages

-- Bitmap OR works but less efficient than single index scan
-- Better restructure if possible:
SELECT * FROM t WHERE a = 1
UNION ALL
SELECT * FROM t WHERE b = 2 AND a != 1;  -- avoid double counting
-- Each query uses its own index efficiently

-- OR on same column:
SELECT * FROM orders WHERE status = 'pending' OR status = 'processing';
-- Better:
SELECT * FROM orders WHERE status IN ('pending', 'processing');
-- IN: planner treats as equality → index used efficiently
```

**Gotcha 5: Low Selectivity**

```sql
-- Index: (status) on orders table
-- status values: 'pending', 'processing', 'completed', 'cancelled'
-- Distribution: 5% pending, 5% processing, 85% completed, 5% cancelled

SELECT * FROM orders WHERE status = 'completed';
-- 85% of rows! Seq Scan is FASTER than index scan!
-- Index scan: 85% × N random page reads (expensive)
-- Seq Scan: N sequential page reads (fast, OS prefetch works)

-- Planner usually makes correct decision based on statistics:
SELECT * FROM orders WHERE status = 'pending';
-- 5% → Index Scan ✓
SELECT * FROM orders WHERE status = 'completed';
-- 85% → Seq Scan ✓

-- When planner gets it wrong (bad statistics):
ANALYZE orders;  -- update statistics
-- Or increase statistics target for this column:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;

-- Partial index for low-selectivity but frequent pattern:
CREATE INDEX ON orders (created_at) WHERE status IN ('pending', 'processing');
-- Only 10% of rows, but these are the "hot" rows needing fast access
```

**Gotcha 6: Statistics Mismatch (Non-uniform Distribution)**

```sql
-- user_id distribution:
-- User 1 (admin): 500,000 orders (5% of table)
-- Other users: average 10 orders each

-- Statistics: avg 10 orders per user (misleading!)
-- Query: WHERE user_id = 1 → planner estimates 10 rows
-- Reality: 500,000 rows → Index Scan chosen but Seq Scan would be better

-- Fix: extended statistics
CREATE STATISTICS orders_user_stats (ndistinct, dependencies)
  ON user_id, status FROM orders;
ANALYZE orders;

-- Or: increase per-column statistics
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 1000;
ANALYZE orders;

-- Or: query hint via configuration
-- SET enable_seqscan = off; (testing only, not production)
```

**Gotcha 7: Volatile Function in WHERE**

```sql
-- Index: (created_at)
SELECT * FROM orders WHERE created_at > now() - interval '7 days';
-- now() is STABLE (same within query), not VOLATILE
-- This WORKS with index!

-- But:
CREATE OR REPLACE FUNCTION get_cutoff() RETURNS timestamptz
LANGUAGE sql VOLATILE AS $$ SELECT now() - interval '7 days'; $$;

SELECT * FROM orders WHERE created_at > get_cutoff();
-- VOLATILE function → planner can't use index efficiently
-- (VOLATILE means "may return different values on each call")
-- Fix: make function STABLE
CREATE OR REPLACE FUNCTION get_cutoff() RETURNS timestamptz
LANGUAGE sql STABLE AS $$ SELECT now() - interval '7 days'; $$;
```

**Gotcha 8: NULL Handling**

```sql
-- B-Tree index DOES index NULLs (unlike some other databases)
-- IS NULL and IS NOT NULL CAN use B-Tree index

CREATE INDEX ON orders (deleted_at);
SELECT * FROM orders WHERE deleted_at IS NULL;   -- ✅ index used!
SELECT * FROM orders WHERE deleted_at IS NOT NULL; -- ✅ index used!

-- But high NULL percentage = low selectivity issue:
-- 95% of rows have deleted_at IS NULL
-- Index scan for IS NULL → 95% of rows → Seq Scan better

-- Partial index solution:
CREATE INDEX ON orders (user_id) WHERE deleted_at IS NULL;
-- Index only active (non-deleted) orders
-- Smaller index, used for WHERE user_id = ? AND deleted_at IS NULL
```

**Gotcha 9: Index on Expression Type Mismatch**

```sql
-- Expression index:
CREATE INDEX ON events ((data->>'user_id')::bigint);

-- Query must match EXACTLY:
SELECT * FROM events WHERE (data->>'user_id')::bigint = 42;  -- ✅ index used
SELECT * FROM events WHERE data->>'user_id' = '42';           -- ❌ no cast, no index
SELECT * FROM events WHERE (data->>'user_id')::int = 42;      -- ❌ ::int not ::bigint!

-- The cast in query must exactly match the cast in index definition
```

**Gotcha 10: Too Many Indexes**

```sql
-- 10 indexes on orders table
-- Write workload: every INSERT updates 10 indexes
-- Maintenance: VACUUM cleans 10 indexes
-- WAL: 10x more WAL per write
-- Planning: planner must consider 10 index combinations

-- Find unused indexes (run after months of production traffic):
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan AS times_used,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%_pkey'  -- keep primary keys
ORDER BY pg_relation_size(indexrelid) DESC;

-- Redundant indexes (prefix of another index):
-- (user_id) is redundant if (user_id, status) exists
-- because (user_id) queries can use (user_id, status) index too
-- Exception: if (user_id, status) is much larger and (user_id) query is hot
```

---

### Index Maintenance

```sql
-- Index bloat check:
SELECT
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  round(
    (pg_relation_size(indexrelid)::float /
     pg_total_relation_size(indexrelid)::float) * 100
  ) AS index_pct_of_total
FROM pg_stat_user_indexes
WHERE tablename = 'orders'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild bloated index (concurrent, no downtime):
REINDEX INDEX CONCURRENTLY idx_orders_user_id;
-- PostgreSQL 12+: CONCURRENTLY keyword
-- Builds new index alongside old one
-- Swaps atomically
-- Allows reads and writes during reindex

-- Why does index get bloated?
-- B-Tree: deleted entries marked "dead" not immediately reclaimed
-- VACUUM reclaims dead entries
-- But if deleted pages are mostly empty → page not recycled (fragmented)
-- REINDEX CONCURRENTLY → fresh, compact index

-- Monitor index efficiency:
SELECT
  relname,
  indexrelname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch,
  round(idx_tup_fetch::numeric / nullif(idx_tup_read, 0) * 100, 2) AS fetch_ratio
FROM pg_stat_user_indexes
JOIN pg_stat_user_tables ON relid = pg_stat_user_indexes.relid
ORDER BY idx_scan DESC;
-- fetch_ratio < 50%: many dead tuples in index (VACUUM দরকার)
```

---

## Topic 12 — Query Execution: Parser, Planner, Executor

### Big Picture: SQL → Result — ভেতরে কী হয়?

```
SQL string আসে → ৪টা stage → Result যায়

                SQL String
                    │
              ┌─────▼──────┐
              │   Parser   │  → Parse Tree (AST)
              └─────┬──────┘    Syntax check only
                    │
              ┌─────▼──────┐
              │  Analyzer  │  → Query Tree
              └─────┬──────┘    Semantic check, view expansion
                    │
              ┌─────▼──────┐
              │  Planner   │  → Execution Plan
              └─────┬──────┘    Cost estimation, plan selection
                    │
              ┌─────▼──────┐
              │  Executor  │  → Result rows
              └────────────┘    Pull-based iterator model

প্রতিটা stage আলাদা কাজ করে। 
Senior DBA মূলত Planner এবং Executor নিয়ে কাজ করে।
```

---

### Stage 1: Parser — Syntax Police

```
Parser-এর একটাই কাজ: SQL string → Parse Tree

Input: "SELECT u.name, count(o.id) FROM users u JOIN orders o ON o.user_id = u.id GROUP BY u.name"

Output: Abstract Syntax Tree (AST)
  SelectStmt {
    targetList: [
      ColumnRef(u.name),
      FuncCall(count, ColumnRef(o.id))
    ],
    fromClause: [
      JoinExpr {
        jointype: JOIN_INNER,
        larg: RangeVar(users, alias=u),
        rarg: RangeVar(orders, alias=o),
        quals: A_Expr(=, ColumnRef(o.user_id), ColumnRef(u.id))
      }
    ],
    groupClause: [ColumnRef(u.name)]
  }

Parser-এ কী হয় না:
  - Table exist করে কিনা: না জানে
  - Column exist করে কিনা: না জানে
  - Permission আছে কিনা: না জানে
  - Type match কিনা: না জানে

Parser-এ কী ধরা পড়ে:
  - Syntax error: SELECT FORM users → ERROR
  - Missing keyword: SELECT name users → ERROR (FROM missing)
  - Unbalanced parentheses
```

---

### Stage 2: Analyzer — Semantic Validator

```
Analyzer-এর কাজ: Parse Tree → Query Tree (semantic-aware)

catalog queries (pg_class, pg_attribute, pg_proc, etc.):
  - "users" table exist করে? pg_class check
  - Alias 'u' → users resolve
  - 'u.name' → users.name, type: text
  - 'o.user_id' → orders.user_id, type: bigint
  - count() function: pg_proc check
  - Type compatibility: u.id = o.user_id (bigint = bigint ✓)
  - GROUP BY validation: all non-aggregate SELECT columns in GROUP BY?
  - Permission: current_user SELECT permission on users, orders?

Rewriter (part of Analyzer stage):
  - VIEW expansion: view-এর definition দিয়ে replace করে
  - Rule application: CREATE RULE-এ defined rules apply করে

View expansion example:
  SELECT * FROM active_users;  -- active_users is a view
  
  View definition: SELECT * FROM users WHERE deleted_at IS NULL
  
  After rewrite:
  SELECT * FROM users WHERE deleted_at IS NULL;
  -- view-এর WHERE condition inline হয়ে যায়
  -- যেন view exist করে না, planner directly users table-এ optimize করে
```

---

### Stage 3: Planner/Optimizer — সবচেয়ে Complex Stage

**Cost Model:**

```
PostgreSQL cost = weighted sum of estimated operations

Base costs (configurable parameters):
  seq_page_cost = 1.0        (sequential page read cost unit)
  random_page_cost = 4.0     (random page read, 4x slower than sequential)
  cpu_tuple_cost = 0.01      (per-row CPU cost)
  cpu_index_tuple_cost = 0.005  (per-index-entry CPU cost)
  cpu_operator_cost = 0.0025   (per-operator evaluation cost)
  parallel_tuple_cost = 0.1    (parallel query overhead)
  parallel_setup_cost = 1000.0 (parallel query startup)

Formula for Sequential Scan:
  cost = seq_page_cost × relpages + cpu_tuple_cost × reltuples
  = 1.0 × 5000 + 0.01 × 100000
  = 5000 + 1000
  = 6000

Formula for Index Scan:
  cost = random_page_cost × (tree_height + matched_pages)
         + cpu_index_tuple_cost × matched_index_entries
         + cpu_tuple_cost × matched_rows
  (simplified)

SSD optimization:
  SSD-এ random vs sequential I/O cost difference অনেক কম!
  random_page_cost = 1.1 for SSD (instead of 4.0)
  → Planner more aggressively uses indexes on SSD
```

**What Planner considers:**

```
1. Scan methods per table:
   Sequential Scan: always possible
   Index Scan: if index exists and selective
   Bitmap Index Scan: if multiple conditions or index less selective
   Index-Only Scan: if covering index and all-visible pages
   TID Scan: if ctid directly specified

2. Join methods:
   Nested Loop: for small outer, indexed inner
   Hash Join: for large tables, equality join, no index
   Merge Join: for pre-sorted data (existing sort or index)

3. Join order:
   2 tables: 1 possible order (or 2 counting which is "outer")
   3 tables: 6 possible orders
   n tables: n! possible orders
   Planner uses dynamic programming for n ≤ join_collapse_limit (default 8)
   n > join_collapse_limit: GEQO (Genetic Query Optimization) — heuristic

4. Aggregation:
   HashAggregate: build hash table of groups
   GroupAggregate: sort first, then aggregate (needs sorted input)
   Partial Aggregate: for parallel queries

5. Subplan vs Initplan:
   Correlated subquery: re-execute per outer row (expensive)
   Non-correlated subquery: execute once (initplan)
   Planner tries to convert correlated to join
```

**Statistics-based estimation:**

```sql
-- Planner statistics-এর উপর নির্ভর করে estimates করে
-- Statistics outdated হলে → wrong estimates → bad plans

-- Check statistics:
SELECT
  attname,
  n_distinct,
  correlation,
  array_length(most_common_vals, 1) AS mcv_count,
  array_length(histogram_bounds, 1) AS hist_buckets
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY attname;

-- n_distinct:
--   positive: exact count (e.g., 5 = exactly 5 distinct values)
--   negative: fraction (e.g., -0.5 = 50% of rows are distinct)
--   -1: all values unique

-- correlation:
--   1.0: physically sorted ascending by this column
--   -1.0: physically sorted descending
--   0.0: no correlation (random physical order)
--   High correlation → Index Scan efficient
--   Low correlation → Planner prefers Seq Scan or Bitmap Scan

-- MCV (Most Common Values):
--   Stored up to statistics_target values
--   Used for equality estimates: WHERE status = 'pending'
--   Found in MCV list → use frequency
--   Not in MCV → use histogram or default

-- Histogram:
--   Bucket boundaries for range estimates
--   Default: 100 buckets (statistics_target)
--   More buckets → more accurate range estimates
ALTER TABLE orders ALTER COLUMN amount SET STATISTICS 500;
ANALYZE orders;
```

---

### Planner Mechanisms — Influencing Without Hints

```sql
-- 1. Statistics target (column-level):
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
-- More MCV entries → better estimates for skewed distributions

-- 2. Extended statistics (multi-column):
CREATE STATISTICS orders_corr (dependencies, ndistinct)
ON user_id, status FROM orders;
ANALYZE orders;
-- If user_id → status correlation exists, planner can estimate better

-- 3. Cost parameters (server/session level):
SET random_page_cost = 1.1;          -- SSD server
SET effective_cache_size = '80GB';   -- hint: this much OS cache available
SET seq_page_cost = 0.1;             -- very fast I/O

-- 4. Enable/disable specific methods (debugging only!):
SET enable_seqscan = off;      -- force index scans (never in production)
SET enable_hashjoin = off;     -- disable hash joins
SET enable_nestloop = off;     -- disable nested loops
SET enable_sort = off;         -- force merge join (needs pre-sorted)
-- RESET after testing!

-- 5. join_collapse_limit / from_collapse_limit:
SET join_collapse_limit = 1;  -- disable join reordering (use query-specified order)
-- Useful when you know the best join order and planner gets it wrong

-- 6. Parallel query tuning:
SET max_parallel_workers_per_gather = 4;   -- max parallel workers per node
SET parallel_tuple_cost = 0.1;              -- parallel overhead
SET min_parallel_table_scan_size = '8MB';  -- table must be this big for parallel
SET min_parallel_index_scan_size = '512kB'; -- index scan parallel threshold
```

---

### Stage 4: Executor — Volcano/Iterator Model

```
Volcano Model (also called Iterator/Pipeline model):

প্রতিটা plan node একটা operator।
Parent calls child.GetNext() → child returns one row (or done signal)।
Pull-based: data flows upward on demand।

Example plan:
  HashAggregate (GROUP BY u.name, count)
    └── Hash Join (o.user_id = u.id)
          ├── Seq Scan: users (outer)
          └── Hash: (build phase)
                └── Seq Scan: orders (inner → build hash table)

Execution:
1. HashAggregate calls GetNext() on Hash Join
2. Hash Join first: calls GetNext() on Hash (build phase)
   Hash: calls GetNext() on orders Seq Scan repeatedly
   Hash: builds in-memory hash table from ALL orders
3. Hash Join then: calls GetNext() on users Seq Scan (probe phase)
   For each user row → look up in hash table → emit matches
4. HashAggregate: accumulates groups, emits when done

Key insight: Hash Join → build entire inner table first!
  Large inner table → large memory use
  work_mem exceeded → disk spill (batches)
```

**Executor nodes in detail:**

```
Scan nodes:
  Seq Scan: sequential heap read
  Index Scan: index lookup → ctid → heap fetch (random I/O)
  Bitmap Index Scan: index → bitmap → bitmap heap scan (sorted page order)
  Index Only Scan: index lookup → visibility check (no heap if VM ok)
  TID Scan: direct ctid access

Join nodes:
  Nested Loop: for each outer row, scan inner (loop)
  Hash Join: build hash table from inner, probe with outer
  Merge Join: both inputs must be sorted, sweep both

Aggregate nodes:
  HashAggregate: hash table per group
  GroupAggregate: sorted input, accumulate per group
  Partial/Finalize Aggregate: parallel aggregation

Sort nodes:
  Sort: quicksort in memory, external merge if work_mem exceeded
  IncrementalSort: sort partially-ordered input (PG 13+)

Other:
  Limit: stop after N rows
  Unique: remove duplicates
  Append: UNION ALL
  MergeAppend: sorted UNION ALL (partition pruning)
  Gather/GatherMerge: parallel query coordination
  Subquery Scan: correlated subquery
  ProjectSet: set-returning functions
  WindowAgg: window functions
  LockRows: SELECT FOR UPDATE
  ModifyTable: INSERT/UPDATE/DELETE
```

---

### Parallel Query

```sql
-- PostgreSQL 9.6+: parallel sequential scan
-- PostgreSQL 10+: parallel index scan, join, aggregation

-- Parallel execution:
-- Gather/GatherMerge node: coordinator
--   spawns N worker processes
--   each worker handles portion of work
--   coordinator collects and returns

EXPLAIN (ANALYZE)
SELECT status, count(*), sum(amount)
FROM orders
GROUP BY status;

-- Possible output:
-- Finalize GroupAggregate
--   -> Gather Merge
--        -> Partial GroupAggregate
--             -> Parallel Seq Scan on orders

-- Configuration:
max_parallel_workers_per_gather = 4   -- workers per gather node
max_parallel_workers = 8              -- total parallel workers (from max_worker_processes)
max_worker_processes = 16             -- total background workers

-- Disable parallel for specific query (if causing issues):
SET max_parallel_workers_per_gather = 0;

-- Parallel index scan (PG 10+):
-- Parallel bitmap heap scan (PG 10+):
-- Parallel hash join (PG 11+):
-- Parallel append (PG 11+):

-- When parallel is NOT used:
--   Table too small (< min_parallel_table_scan_size)
--   Already in parallel worker
--   Query uses non-parallel-safe functions
--   Cursor-based query
--   Query modifies data (INSERT/UPDATE/DELETE)
```

---

## Topic 13 — EXPLAIN & EXPLAIN ANALYZE

### Big Picture: Plan Reading = DBA-র Core Skill

```
EXPLAIN: "What plan will you use?" (no execution)
EXPLAIN ANALYZE: "What plan did you use, and how long did it take?" (executes!)
EXPLAIN (ANALYZE, BUFFERS): additionally shows buffer hit/miss
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON): machine-readable

DANGER: EXPLAIN ANALYZE on INSERT/UPDATE/DELETE → actually executes!
Safe pattern for DML:
BEGIN;
EXPLAIN ANALYZE DELETE FROM orders WHERE status = 'cancelled';
ROLLBACK;
```

---

### Reading the Plan — Every Number Matters

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, count(o.id) AS order_count, sum(o.amount) AS total
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.active = true
  AND o.created_at > now() - interval '30 days'
GROUP BY u.name
ORDER BY total DESC
LIMIT 10;
```

```
Limit  (cost=8234.56..8234.59 rows=10 width=48)
       (actual time=234.567..234.589 rows=10 loops=1)
  ->  Sort  (cost=8234.56..8246.78 rows=4889 width=48)
            (actual time=234.543..234.556 rows=10 loops=1)
        Sort Key: (sum(o.amount)) DESC
        Sort Method: top-N heapsort  Memory: 28kB
        ->  HashAggregate  (cost=7956.23..8004.34 rows=4889 width=48)
                           (actual time=231.234..233.456 rows=4823 loops=1)
              Group Key: u.name
              Batches: 1  Memory Usage: 1024kB
              ->  Hash Join  (cost=1234.56..7834.12 rows=24422 width=40)
                             (actual time=45.678..218.901 rows=24156 loops=1)
                    Hash Cond: (o.user_id = u.id)
                    Buffers: shared hit=3456 read=789 dirtied=0 written=0
                    ->  Index Scan using idx_orders_created on orders o
                                (cost=0.56..5678.90 rows=24422 width=24)
                                (actual time=0.089..156.789 rows=24156 loops=1)
                          Index Cond: (created_at > (now() - '30 days'::interval))
                          Buffers: shared hit=2345 read=567
                    ->  Hash  (cost=987.65..987.65 rows=19792 width=24)
                               (actual time=44.567..44.568 rows=19834 loops=1)
                          Buckets: 32768  Batches: 1  Memory Usage: 1152kB
                          ->  Seq Scan on users u  (cost=0.00..987.65 rows=19792 width=24)
                                                    (actual time=0.023..34.567 rows=19834 loops=1)
                                Filter: active
                                Rows Removed by Filter: 3421
                                Buffers: shared hit=456 read=123
Planning Time: 2.345 ms
Execution Time: 234.789 ms
```

**প্রতিটা field-এর মানে:**

```
cost=8234.56..8234.59
  8234.56: startup cost (first row আসার আগে estimated work)
  8234.59: total cost (সব rows)
  Unit: abstract cost unit (seq_page_cost = 1.0)
  Note: nested nodes-এর cost cumulative!

rows=10
  Estimated rows output from this node
  If estimated ≠ actual → statistics problem

width=48
  Estimated average row size in bytes
  Large width → memory-hungry operations

actual time=234.567..234.589
  234.567: actual startup time (ms)
  234.589: actual total time (ms)
  Per loop! If loops=5, multiply by 5 for total

rows=10
  Actual rows output
  If far from estimated → wrong statistics → bad plan possible

loops=1
  How many times this node executed
  Nested Loop inner side: loops = outer_rows

Sort Method: top-N heapsort  Memory: 28kB
  "top-N heapsort": LIMIT makes sort efficient (only top N kept)
  "quicksort": in-memory full sort
  "external merge  Disk: 12345kB": disk spill! work_mem too small

Batches: 1  Memory Usage: 1024kB
  For HashAggregate/Hash Join
  Batches: 1 → fit in memory (good)
  Batches: 4 → 4 disk passes (bad, increase work_mem)

Buffers: shared hit=3456 read=789
  hit: from shared buffer pool (fast)
  read: from disk (slow)
  dirtied: pages modified
  written: dirty pages written to disk during this query (unusual)
  hit/(hit+read) = effective cache rate for this query

Rows Removed by Filter: 3421
  Seq Scan filtered this many rows
  Only appears for filter conditions (not index conditions)
  High number = consider indexing this column
```

---

### Plan Analysis Patterns — Diagnosing Problems

**Pattern 1: Rows estimate way off**

```
Estimated rows=100, Actual rows=50000

→ Statistics problem:
  ANALYZE table_name;
  Increase statistics target: ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000;
  Extended statistics for correlated columns

→ If estimate remains bad after ANALYZE:
  Data skew (user_id 1 has 90% of orders, but avg is 10)
  Solution: pg_hint_plan, or rewrite query
```

**Pattern 2: Seq Scan where Index Scan expected**

```
Seq Scan on orders (rows=1000000)
WHERE user_id = 42

Possible reasons:
1. No index on user_id → CREATE INDEX
2. Index exists but not used:
   a. Type mismatch (user_id bigint, query uses '42'::text)
   b. Function wrapping (WHERE user_id::text = '42')
   c. Low selectivity (user_id 42 has 80% of orders)
   d. Statistics: planner estimates too many rows
   e. seq_page_cost too low relative to random_page_cost

Diagnose:
  SET enable_seqscan = off;
  EXPLAIN ANALYZE SELECT ...;
  -- See if index plan is faster or slower
  -- If faster: planner made wrong choice (fix statistics or costs)
  -- If slower: seq scan was correct (low selectivity, don't fight planner)
```

**Pattern 3: High Disk Read (Buffers: read=XXXX)**

```
Buffers: shared hit=100 read=50000

→ Cache miss
→ Either shared_buffers too small
→ Or query scans too much data
→ Or data is "cold" (infrequently accessed)

Actions:
1. Check shared_buffers: SHOW shared_buffers;
2. Check cache hit ratio: pg_stat_database.blks_hit/blks_read
3. Consider: is all this data necessary?
   SELECT * → SELECT only needed columns
   Missing index → full table scan
```

**Pattern 4: Sort Disk Spill**

```
Sort Method: external merge  Disk: 45678kB

→ work_mem too small for this sort
→ Disk I/O for sort = slow

Actions:
SET work_mem = '256MB';
EXPLAIN ANALYZE SELECT ... ORDER BY ...;
-- See if Sort Method changes to "quicksort" (in-memory)

For specific sessions/queries:
SET LOCAL work_mem = '512MB';
```

**Pattern 5: Hash Join Batches > 1**

```
Hash Join
  Batches: 8  Memory Usage: 65536kB

→ Hash table didn't fit in work_mem
→ 8 disk-based batches (8x slower than in-memory)

Actions:
SET work_mem = '256MB';
-- Retry and check Batches: 1
```

**Pattern 6: Nested Loop with many loops**

```
Nested Loop
  (actual rows=1000 loops=500)
  
  → Outer: 500 rows
  → Inner: 1000 executions of inner scan
  → 500 × each inner scan time = very slow if inner not indexed

Inner side must be indexed!
  If inner is Seq Scan with loops=500 → critical performance problem
  CREATE INDEX on the join column
```

---

### EXPLAIN Tools and Visualization

```sql
-- JSON format (for tools):
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
-- Output: machine-readable JSON

-- Tools that parse EXPLAIN output:
-- explain.dalibo.com: visual explain plan (paste JSON)
-- pgMustard: detailed analysis and recommendations
-- depesz.com/explain: classic explain visualizer

-- Auto-explain: log slow query plans automatically
-- postgresql.conf:
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 1000  -- log plans for queries > 1 second
auto_explain.log_analyze = true
auto_explain.log_buffers = true
auto_explain.log_format = 'json'
auto_explain.log_nested_statements = true  -- also log subquery plans

-- After enabling (restart required):
-- Check pg_log/ for slow query plans

-- pg_stats_statements-এ plan statistics (PG 14+):
SELECT query, plans, total_plan_time, calls, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC;
```

---

## Topic 14 — Query Optimization: Join Strategies, Subquery vs CTE vs Temp Table

### Big Picture: Query লেখা এবং Query Optimize করা দুটো আলাদা skill

```
জেনে লেখা:
  - Correct result পাওয়া
  - Readable, maintainable

Optimized লেখা:
  - Fast execution
  - Low resource usage
  - Scale করে

Many queries correct কিন্তু inefficient।
Optimizer অনেক কিছু করে, কিন্তু সীমাবদ্ধতা আছে।
DBA-র job: optimizer-কে সাহায্য করা।
```

---

### Join Strategies — কখন কোনটা?

**Nested Loop Join:**

```
Algorithm:
  FOR EACH row in outer_table:
    FOR EACH row in inner_table where join_condition(outer, inner):
      output(outer, inner)

Complexity: O(outer_rows × inner_rows) worst case
Best case with index on inner:
  O(outer_rows × log(inner_rows))

Best for:
  Small outer table (few rows drive the loop)
  Indexed inner table (each loop iteration uses index)
  Result is small (early termination possible with LIMIT)

Example plan output:
  Nested Loop
    -> Index Scan on users (outer: 5 rows)
    -> Index Scan on orders using idx_orders_user (inner: 1000 rows per user)
  
  Total: 5 × index_lookup_cost ≈ cheap!
```

```sql
-- Force/verify nested loop behavior:
SET enable_hashjoin = off;
SET enable_mergejoin = off;
EXPLAIN SELECT ...;

-- Nested Loop inefficient case:
-- Outer: 100,000 rows
-- Inner: no index, 50,000 rows
-- 100,000 × 50,000 = 5 billion comparisons → avoid!
```

**Hash Join:**

```
Algorithm:
  Phase 1 (Build):
    FOR EACH row in smaller_table:
      hash(join_key) → bucket
      insert into hash table

  Phase 2 (Probe):
    FOR EACH row in larger_table:
      hash(join_key) → bucket
      check bucket for matches
      output matches

Complexity: O(smaller + larger)
Memory: hash table size = smaller table × key+value size

If hash table > work_mem:
  Split into batches:
    Batch 1: hash inner subset, scan outer subset
    Batch 2: hash next inner subset, scan outer again
    ...
  I/O intensive! Avoid by increasing work_mem

Best for:
  Large tables without useful index
  Equality join only (hash function needs equality)
  First join in a complex query where sorted input not available
```

```sql
-- Check hash join batches:
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.name, o.amount
FROM users u JOIN orders o ON o.user_id = u.id;

-- "Batches: 1  Memory Usage: 4096kB" → in-memory, good
-- "Batches: 8  Memory Usage: 65536kB" → disk batches, increase work_mem

-- Tune:
SET work_mem = '256MB';
```

**Merge Join:**

```
Algorithm:
  Prerequisite: BOTH inputs sorted on join key

  Two pointers, advance together:
    If left_key = right_key → output, advance right (might have duplicates)
    If left_key < right_key → advance left
    If left_key > right_key → advance right

Complexity: O(n + m) after sorting
Sorting cost: O(n log n + m log m) if not pre-sorted

Best for:
  Both tables have index on join key (pre-sorted, no sort needed)
  Large result sets (streaming output, no memory spike)
  Range-based joins (inequality joins: a.date BETWEEN b.start AND b.end)

Example:
  Merge Join
    -> Index Scan on users (sorted by user_id via index)
    -> Index Scan on orders (sorted by user_id via index)
  
  No sort needed! Both indexes provide pre-sorted data.
  Pure O(n+m) streaming join.
```

---

### Join Order and Selectivity

```sql
-- Bad join order: start with large table, filter later
SELECT *
FROM large_orders lo               -- 10M rows
JOIN users u ON u.id = lo.user_id  -- 100K rows
WHERE u.country = 'BD';            -- filters to 5K users

-- Good join order: filter first, then join
SELECT *
FROM users u                       -- 100K → filter to 5K
JOIN large_orders lo ON lo.user_id = u.id  -- 5K users' orders
WHERE u.country = 'BD';

-- Planner usually figures this out!
-- But if join_collapse_limit exceeded, might not reorder

-- Force join order (when you know better):
SET join_collapse_limit = 1;  -- honor query-specified order
SELECT ... FROM small_filtered_set JOIN large_table ON ...;
```

---

### Subquery vs CTE vs Temp Table — Deep Comparison

**Subquery:**

```sql
-- Non-correlated subquery (executes ONCE):
SELECT * FROM orders
WHERE user_id IN (
  SELECT id FROM users WHERE country = 'BD'
);
-- Planner often converts to JOIN or semi-join:
-- Hash Semi Join or Nested Loop Semi Join
-- Result: similar performance to explicit JOIN

-- Correlated subquery (executes PER ROW of outer query):
SELECT
  u.id,
  u.name,
  (SELECT count(*) FROM orders o WHERE o.user_id = u.id) AS order_count
FROM users u;
-- For each user → one count() query on orders
-- 100K users → 100K separate queries!
-- Catastrophically slow for large tables

-- Convert correlated subquery to JOIN:
SELECT
  u.id,
  u.name,
  count(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
-- One pass, one hash aggregate → much faster

-- EXISTS vs IN:
-- IN: builds list of all values (memory issue for large subquery)
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE active);

-- EXISTS: stops as soon as one match found (more efficient for large lists)
SELECT * FROM orders o WHERE EXISTS (
  SELECT 1 FROM users u WHERE u.id = o.user_id AND u.active
);

-- NOT IN vs NOT EXISTS: HUGE difference with NULLs!
-- NOT IN with NULL in subquery: always returns empty!
SELECT * FROM a WHERE id NOT IN (SELECT parent_id FROM b);
-- If ANY parent_id is NULL: result is EMPTY (NULL poison!)

-- Safe: NOT EXISTS
SELECT * FROM a WHERE NOT EXISTS (
  SELECT 1 FROM b WHERE b.parent_id = a.id
);
-- Works correctly even with NULL parent_ids
```

**CTE (Common Table Expression):**

```sql
-- Basic CTE:
WITH active_users AS (
  SELECT id, name, email FROM users WHERE active = true
),
user_orders AS (
  SELECT u.id, u.name, count(o.id) AS cnt, sum(o.amount) AS total
  FROM active_users u
  LEFT JOIN orders o ON o.user_id = u.id
  GROUP BY u.id, u.name
)
SELECT * FROM user_orders WHERE cnt > 5 ORDER BY total DESC;

-- PostgreSQL 12+ CTE behavior:
-- NOT MATERIALIZED by default for simple CTEs
-- Planner inlines CTE as subquery and optimizes
-- Before PG 12: CTE was always an "optimization fence"!

-- Explicit MATERIALIZED (execute once, cache result):
WITH MATERIALIZED expensive_query AS (
  SELECT user_id, complex_calculation(...)
  FROM very_large_table
  WHERE expensive_condition
)
SELECT * FROM expensive_query e1
JOIN expensive_query e2 ON e1.related_id = e2.user_id;
-- Without MATERIALIZED: expensive_query runs TWICE
-- With MATERIALIZED: runs ONCE, result cached in memory/disk

-- NOT MATERIALIZED (let planner optimize, push predicates down):
WITH NOT MATERIALIZED recent_users AS (
  SELECT * FROM users WHERE created_at > '2024-01-01'
)
SELECT * FROM recent_users WHERE country = 'BD';
-- Planner rewrites as: SELECT * FROM users WHERE created_at > '2024-01-01' AND country = 'BD'
-- Index on (country, created_at) → efficient

-- RECURSIVE CTE:
WITH RECURSIVE category_tree AS (
  -- Base case: root categories
  SELECT id, name, parent_id, 0 AS depth, ARRAY[id] AS path
  FROM categories
  WHERE parent_id IS NULL

  UNION ALL

  -- Recursive case: children
  SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.path || c.id
  FROM categories c
  JOIN category_tree ct ON c.parent_id = ct.id
  WHERE NOT c.id = ANY(ct.path)  -- cycle protection!
)
SELECT * FROM category_tree ORDER BY depth, name;

-- Recursive CTE for hierarchical data:
-- Employee hierarchy, comment threads, filesystem paths
-- Alternative: ltree extension (faster for deep hierarchies)
```

**Temp Table:**

```sql
-- When temp table wins:
-- 1. Large intermediate result, multiple uses
-- 2. Need index on intermediate result
-- 3. Statistics dönöt matter (ANALYZE the temp table!)
-- 4. Complex multi-step transformation

-- Pattern:
CREATE TEMP TABLE active_user_stats AS
SELECT
  u.id,
  u.name,
  count(o.id) AS order_count,
  sum(o.amount) AS total_spent,
  max(o.created_at) AS last_order
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.active = true
GROUP BY u.id, u.name;

-- Add indexes!
CREATE INDEX ON active_user_stats (id);
CREATE INDEX ON active_user_stats (total_spent DESC);

-- Now ANALYZE (temp table has no statistics otherwise):
ANALYZE active_user_stats;

-- Complex follow-up queries:
SELECT * FROM active_user_stats
JOIN user_preferences up ON up.user_id = active_user_stats.id
WHERE total_spent > 10000
ORDER BY total_spent DESC;

-- Without temp table: complex CTE or subquery without index → slow
-- With temp table + index + analyze: fast indexed join

-- Temp table vs CTE MATERIALIZED:
-- MATERIALIZED CTE: cannot be indexed, no ANALYZE
-- Temp table: full table, can be indexed, can be ANALYZEd

-- Session cleanup: temp table auto-dropped at session end
-- Explicit drop:
DROP TABLE IF EXISTS active_user_stats;
```

**Decision guide:**

```
Simple transformation, used once:
  → Subquery (planner optimizes freely)

Complex logic, readable:
  → CTE NOT MATERIALIZED (PG 12+) — planner inlines

Same result used multiple times:
  → CTE MATERIALIZED (compute once)

Large intermediate + need index + multiple complex queries:
  → Temp table + index + ANALYZE

Recursive / tree traversal:
  → Recursive CTE (no alternative in standard SQL)

ETL / batch processing with multiple steps:
  → Temp table (clear, explicit, debuggable)
```

---

### N+1 Query Problem — Application-Level Issue

```sql
-- N+1 in application code (Python/Django/etc.):
users = db.query("SELECT * FROM users LIMIT 100")
for user in users:
    orders = db.query(f"SELECT * FROM orders WHERE user_id = {user.id}")
    # 100 users → 100 separate queries → 101 total!

-- Fix: JOIN or batch query
SELECT u.*, json_agg(o.*) AS orders
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.id IN (1, 2, 3, ..., 100)
GROUP BY u.id;

-- Or: two queries (batch)
users = db.query("SELECT * FROM users LIMIT 100")
user_ids = [u.id for u in users]
orders = db.query(f"SELECT * FROM orders WHERE user_id = ANY(ARRAY{user_ids})")
# Map orders to users in application

-- PostgreSQL-specific: json_agg for nested data
SELECT
  u.id,
  u.name,
  json_agg(
    json_build_object('id', o.id, 'amount', o.amount, 'status', o.status)
  ) FILTER (WHERE o.id IS NOT NULL) AS orders
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.id = ANY(ARRAY[1,2,3,4,5])
GROUP BY u.id, u.name;
-- Single query, structured result
```

---

### Query Rewrite Patterns

```sql
-- Pattern 1: EXISTS instead of COUNT for existence check
-- Slow:
SELECT CASE WHEN (SELECT count(*) FROM orders WHERE user_id = 42) > 0
            THEN 'has orders' ELSE 'no orders' END;
-- count(*) scans all matching rows

-- Fast:
SELECT CASE WHEN EXISTS(SELECT 1 FROM orders WHERE user_id = 42)
            THEN 'has orders' ELSE 'no orders' END;
-- EXISTS stops at first match

-- Pattern 2: Window function instead of self-join
-- Slow self-join:
SELECT o1.*, o2.amount AS prev_amount
FROM orders o1
JOIN orders o2 ON o2.user_id = o1.user_id
  AND o2.created_at = (
    SELECT max(created_at) FROM orders
    WHERE user_id = o1.user_id AND created_at < o1.created_at
  );
-- Correlated subquery per row = terrible!

-- Fast window function:
SELECT *,
  lag(amount) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_amount
FROM orders;
-- Single pass over sorted data

-- Pattern 3: DISTINCT ON instead of GROUP BY trick
-- Get most recent order per user:
-- Slow:
SELECT o.*
FROM orders o
JOIN (
  SELECT user_id, max(created_at) AS max_date
  FROM orders GROUP BY user_id
) latest ON latest.user_id = o.user_id AND latest.max_date = o.created_at;

-- Fast (PostgreSQL specific):
SELECT DISTINCT ON (user_id)
  user_id, id, amount, status, created_at
FROM orders
ORDER BY user_id, created_at DESC;
-- Returns exactly one row per user_id, the most recent

-- Pattern 4: LATERAL JOIN for correlated subquery
-- Top 3 orders per user:
SELECT u.name, o.*
FROM users u
CROSS JOIN LATERAL (
  SELECT * FROM orders
  WHERE user_id = u.id
  ORDER BY amount DESC
  LIMIT 3
) o;
-- LATERAL: subquery can reference outer table (u)
-- PostgreSQL executes subquery per outer row
-- But with index on orders(user_id, amount DESC): efficient!
```

---

## Topic 15 — Statistics & Vacuum

### Big Picture: Database-এর Long-term Health

Topic 4-এ MVCC দেখেছি — UPDATE/DELETE করলে dead tuple জমে। এই topic-এ:
- Statistics: Planner-কে accurate estimates দেওয়া
- Vacuum: Dead tuple পরিষ্কার, XID wraparound prevent
- এই দুটো ছাড়া database eventually unusable হয়ে যাবে।

---

### Statistics — Planner-এর Intelligence

**pg_statistic এবং pg_stats:**

```sql
-- Internal table: pg_statistic (raw, hard to read)
-- Friendly view: pg_stats

SELECT
  tablename,
  attname AS column,
  null_frac,          -- fraction of NULLs
  avg_width,          -- average bytes per value
  n_distinct,         -- distinct value count
  most_common_vals,   -- array of most common values
  most_common_freqs,  -- their frequencies
  histogram_bounds,   -- value distribution buckets
  correlation         -- physical vs logical order
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY attname;
```

**Statistics collection process:**

```
ANALYZE orders;  -- explicit
-- OR: autovacuum with analyze phase

ANALYZE steps:
1. Random sample of table pages (default: 300 pages)
2. For each sampled page, read all tuples
3. Compute statistics per column:
   - null_frac: count NULLs / total
   - n_distinct: HyperLogLog estimate (not exact scan)
   - most_common_vals: top N values by frequency
   - histogram_bounds: equal-frequency histogram
   - correlation: Spearman rank correlation
4. Store in pg_statistic

Sample size: default_statistics_target × 300 samples
  default_statistics_target = 100 → 30,000 sample rows
```

**When statistics fail:**

```sql
-- Problem 1: Non-uniform distribution (skew)
-- user_id=1 has 500K orders, others have ~10 each
-- Statistics: avg ≈ 10 (misleading for user_id=1!)
-- Query for user_id=1: planner estimates 10 rows, actual 500K
-- Plan: Index Scan chosen (wrong! Seq Scan would be better for 500K rows)

-- Fix: Extended statistics (functional dependencies)
CREATE STATISTICS order_user_skew ON user_id FROM orders;
ANALYZE orders;
-- PostgreSQL now knows this column has high skew

-- Fix: Column statistics target
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 1000;
ANALYZE orders;
-- More MCV entries → user_id=1's true frequency captured

-- Problem 2: Correlated columns
-- orders: country, state  (state depends on country)
-- Query: WHERE country='BD' AND state='Dhaka'
-- Without correlation info: assumes independence → underestimate rows
-- Planner chooses Nested Loop (wrong)

-- Fix: ndistinct statistics
CREATE STATISTICS orders_location_corr (ndistinct, dependencies)
ON country, state FROM orders;
ANALYZE orders;
-- Planner now knows state is functionally dependent on country
```

**Statistics and Timing:**

```sql
-- When is statistics stale?
-- After bulk INSERT: table much larger, statistics reflect old size
-- After bulk DELETE: many rows gone, statistics show more rows
-- After bulk UPDATE: value distribution changed

-- Check: last analyze time
SELECT relname, last_analyze, last_autoanalyze, n_mod_since_analyze
FROM pg_stat_user_tables
WHERE n_mod_since_analyze > 10000  -- significant changes since last analyze
ORDER BY n_mod_since_analyze DESC;

-- Manual analyze after bulk operations:
INSERT INTO orders SELECT * FROM imported_orders;  -- millions of rows
ANALYZE orders;  -- immediately update statistics!

-- Autovacuum triggers analyze at:
-- autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × reltuples
-- Default: 50 + 0.1 × reltuples modifications
-- 1M row table: 50 + 100000 = 100,050 changes → analyze
-- This might be too late for time-sensitive query plans
```

---

### VACUUM — Dead Tuple Lifecycle

**Dead tuple lifecycle:**

```
1. Transaction T1 (XID=100) inserts row:
   Tuple: t_xmin=100, t_xmax=0 (live)

2. Transaction T2 (XID=200) updates row:
   Old tuple: t_xmin=100, t_xmax=200 (dead after T2 commits)
   New tuple: t_xmin=200, t_xmax=0 (live)

3. T2 commits. Old tuple is now DEAD (no active transaction can see it)
   But: it's still physically on the page, taking space.

4. VACUUM runs:
   a. Scans table pages
   b. Identifies dead tuples (t_xmax committed, not needed by any snapshot)
   c. Marks dead tuple slots as "free" in line pointer array
      (LP_DEAD flag → eventually LP_UNUSED)
   d. Updates FSM with newly available free space
   e. Updates VM: mark pages as all-visible if appropriate
   f. Cleans index entries pointing to dead tuples
   g. Updates pg_class.relpages, reltuples (for statistics)

5. Page has free space again → new inserts can reuse
   NOTE: space is NOT returned to OS (file doesn't shrink!)
   VACUUM FULL: rewrites table, reclaims OS space (but takes exclusive lock)
```

**VACUUM vs VACUUM FULL:**

```
VACUUM (regular):
  Lock: SHARE UPDATE EXCLUSIVE
    → Allows SELECT, INSERT, UPDATE, DELETE to continue!
    → Blocks only DDL (ALTER TABLE, CREATE INDEX, VACUUM FULL)
  Process: marks dead tuples free, updates FSM/VM
  Space: freed within file (reused by future inserts, not returned to OS)
  Speed: fast (scans needed pages only with autovacuum skip logic)

VACUUM FULL:
  Lock: ACCESS EXCLUSIVE
    → Blocks ALL operations! No reads, no writes!
  Process: creates new heap file, copies live tuples, replaces old file
  Space: OS-level space returned (file shrinks!)
  Speed: very slow (full table rewrite)
  Use case: one-time bloat removal (then adjust autovacuum to prevent recurrence)

Alternative to VACUUM FULL: pg_repack (online, no exclusive lock)
```

```sql
-- VACUUM variants:
VACUUM orders;                    -- standard (marks free space)
VACUUM VERBOSE orders;            -- with detailed output
VACUUM ANALYZE orders;            -- vacuum + update statistics
VACUUM FREEZE orders;             -- freeze old tuples (XID wraparound)
VACUUM (ANALYZE, VERBOSE) orders; -- modern syntax with options

-- VACUUM output (VERBOSE):
-- INFO:  vacuuming "public.orders"
-- INFO:  scanned index "orders_pkey" to remove 5000 row versions
-- INFO:  "orders": removed 5000 row versions in 250 pages
-- INFO:  index scan not needed: 0 pages from table (0.00% of total) have 0 dead item identifiers
-- INFO:  "orders": found 5000 removable, 95000 nonremovable row versions in 5000 out of 10000 pages

-- VACUUM FULL:
-- Use ONLY when:
-- 1. Table grossly bloated (dead_pct > 50%) AND
-- 2. Off-peak/maintenance window available AND
-- 3. Disk space available for temporary copy
-- Always prefer pg_repack in production!
```

---

### XID Wraparound — The Most Critical Vacuum Task

**Complete story:**

```
PostgreSQL transaction IDs (XID): 32-bit integer
Range: 1 to 2,147,483,647 (2^31 - 1)
Then wraps: 2,147,483,648 → back to 1? No...

PostgreSQL uses modular arithmetic:
  "Past" = previous 2^31 XIDs = 2,147,483,648 transactions
  "Future" = next 2^31 XIDs

If current XID = 3,000,000,000:
  Past: XIDs from ~852M to 3B
  Future: XIDs from 3B to ~852M (wrapped)

Old tuple: t_xmin = 500,000,000
  current - 500M = 2.5 billion > 2.1 billion limit!
  PostgreSQL thinks this is a "future" transaction → INVISIBLE!

Solution: VACUUM FREEZE
  Changes old tuple's t_xmin → FrozenXID (special value = 2)
  FrozenXID: always visible, immune to wraparound
  Once frozen: no more XID concern for this tuple

Autovacuum freeze triggers:
  vacuum_freeze_min_age = 50,000,000 (50M)
    If tuple's XID age > 50M: candidate for freezing (but not forced)
  
  vacuum_freeze_table_age = 150,000,000 (150M)
    If table's relfrozenxid age > 150M: aggressive freeze vacuum
  
  autovacuum_freeze_max_age = 200,000,000 (200M)
    If table's relfrozenxid age > 200M: autovacuum FORCED
    Even if table too small to normally vacuum
  
  Hard limit: 2,147,483,648 (2.1B)
    If table age approaches 1.6B: PostgreSQL refuses EVERYTHING except VACUUM!
```

```sql
-- Monitor XID age (CRITICAL metric!):
SELECT
  datname,
  age(datfrozenxid) AS db_xid_age,
  pg_size_pretty(pg_database_size(datname)) AS db_size
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Per-table XID age:
SELECT
  n.nspname AS schema,
  c.relname AS table,
  age(c.relfrozenxid) AS table_xid_age,
  pg_size_pretty(pg_total_relation_size(c.oid)) AS size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;

-- Alert thresholds:
-- age > 1,000,000,000: Warning — vacuum needed soon
-- age > 1,500,000,000: Critical — vacuum immediately
-- age > 1,800,000,000: Emergency — database at risk!
-- age > 2,100,000,000: Database forced read-only!

-- Emergency freeze:
VACUUM FREEZE table_name;  -- freeze all tuples

-- After emergency: investigate why autovacuum failed
-- Long-running transactions prevent vacuum!
SELECT pid, now() - xact_start AS age, state, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

---

### Autovacuum — Tuning for Production

**Default settings and why they fail for large tables:**

```
autovacuum_vacuum_threshold = 50      (minimum dead tuples)
autovacuum_vacuum_scale_factor = 0.2  (20% of table)

Trigger: 50 + 0.2 × reltuples dead tuples

Small table (1K rows): 50 + 200 = 250 → trigger at 250 dead tuples ✓
Large table (100M rows): 50 + 20,000,000 = 20M → trigger at 20M dead tuples!

20M dead tuples × average row size before vacuum triggers!
Table could be 40% bloated before autovacuum kicks in.
```

```sql
-- Per-table autovacuum tuning:
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.01,    -- 1% (not 20%)
  autovacuum_vacuum_threshold = 1000,        -- 1000 dead tuples minimum
  autovacuum_analyze_scale_factor = 0.005,  -- 0.5%
  autovacuum_analyze_threshold = 500,
  autovacuum_vacuum_cost_delay = 2,          -- ms delay between pages (reduce throttling)
  autovacuum_vacuum_cost_limit = 400         -- pages per delay cycle (default 200)
);

-- Global settings (postgresql.conf):
autovacuum_max_workers = 5          -- more workers for high-write database
autovacuum_naptime = 30s            -- check every 30 seconds (default 1min)
autovacuum_vacuum_cost_delay = 2ms  -- less throttling (default 2ms in PG 13+)
autovacuum_vacuum_cost_limit = 400  -- more work per cycle

-- Monitor autovacuum activity:
SELECT
  schemaname,
  relname,
  n_live_tup,
  n_dead_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze,
  vacuum_count,
  autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Currently running autovacuum:
SELECT
  pid,
  now() - xact_start AS duration,
  query
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%'
ORDER BY duration DESC;

-- Autovacuum blocked by long transaction:
-- Autovacuum cannot remove tuples visible to any active transaction!
SELECT pid, now() - xact_start AS age, state, query
FROM pg_stat_activity
WHERE state IN ('active', 'idle in transaction')
  AND xact_start < now() - interval '5 minutes'
ORDER BY xact_start;
-- Kill long transactions that block vacuum:
SELECT pg_terminate_backend(pid) WHERE pid = <blocking_pid>;
```

---

### Bloat Detection and Remediation

```sql
-- Table bloat estimate (no lock, approximate):
SELECT
  schemaname,
  tablename,
  n_live_tup,
  n_dead_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
  pg_size_pretty(pg_relation_size(relid)) AS table_size,
  pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Exact bloat measurement (pgstattuple, briefly scans table):
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT
  table_len / 1024 / 1024 AS table_mb,
  tuple_count AS live_tuples,
  tuple_percent,
  dead_tuple_count,
  dead_tuple_percent,
  free_space / 1024 AS free_kb,
  free_percent
FROM pgstattuple('orders');

-- Index bloat:
SELECT * FROM pgstatindex('idx_orders_user_id');
-- leaf_pages, empty_pages, deleted_pages, avg_leaf_density
-- avg_leaf_density < 70%: consider REINDEX

-- Remediation options:
-- 1. Regular VACUUM (mark free, don't shrink file):
VACUUM ANALYZE orders;

-- 2. VACUUM FULL (shrink file, long exclusive lock):
VACUUM FULL orders;  -- production-এ avoid!

-- 3. pg_repack (online, minimal lock):
-- apt install postgresql-17-repack
pg_repack -h localhost -U postgres -d mydb -t orders

-- 4. Prevent bloat: autovacuum tuning + fillfactor

-- Index bloat remediation:
REINDEX INDEX CONCURRENTLY idx_orders_user_id;  -- online, no downtime
```

---

### Block 3 Summary — Dots Connected

```
Topic 11 (Index)
    │
    ├──► Topic 2 (Physical Storage): Index pages also 8KB, stored in heap-like files
    ├──► Topic 12 (Query Execution): Planner chooses scan method based on index stats
    ├──► Topic 13 (EXPLAIN): Index Scan vs Seq Scan in execution plans
    ├──► Topic 15 (Vacuum): Dead tuples in indexes need vacuum too
    └──► Topic 20 (Partition): Partition pruning uses index-like page range info

Topic 12 (Query Execution)
    │
    ├──► Topic 1 (Architecture): Executor runs in backend process, uses work_mem
    ├──► Topic 3 (Transaction): Executor respects isolation level / snapshots
    ├──► Topic 4 (Concurrency): Executor acquires row locks for DML
    └──► Topic 15 (Statistics): Planner uses pg_statistics for cost estimation

Topic 13 (EXPLAIN)
    │
    ├──► Topic 11 (Index): Explains why index used/not used
    ├──► Topic 14 (Optimization): Diagnosis tool for query rewrites
    └──► Topic 15 (Statistics): Estimate vs actual rows → statistics quality

Topic 14 (Query Optimization)
    │
    ├──► Topic 10 (Normalization): Denormalized tables → fewer joins needed
    ├──► Topic 16 (Views): Materialized view = pre-optimized query result
    └──► Topic 17 (Functions): Volatile/Stable affects query optimization

Topic 15 (Statistics & Vacuum)
    │
    ├──► Topic 2 (Physical Storage): Dead tuples = physical pages wasted
    ├──► Topic 4 (Concurrency): Long transactions block vacuum
    ├──► Topic 5 (WAL): VACUUM also writes WAL (but less than DML)
    └──► Topic 28 (Monitoring): pg_stat_user_tables = vacuum monitoring
```
# Senior DBA Master Guide — Block 4
## Database Objects

---

## Topic 16 — View: Simple, Materialized, Updatable

### Big Picture: View কী এবং কেন?

View হলো একটা stored query — disk-এ শুধু query definition থাকে, data না। কিন্তু Materialized View-এ data physically stored থাকে। এই পার্থক্যটা fundamental।

```
Simple View:
  CREATE VIEW = stored SELECT statement
  Query করলে → view definition inline হয় → original table-এ query হয়
  Data: কোথাও extra store হয় না
  Freshness: always current
  Speed: underlying tables-এর উপর নির্ভর করে

Materialized View:
  CREATE MATERIALIZED VIEW = stored SELECT + stored RESULT
  Query করলে → stored result পড়ে (fast!)
  Data: disk-এ physically stored (separate heap file)
  Freshness: REFRESH না করলে stale
  Speed: index করা যায়, much faster for complex queries

কোনটা কখন:
  Real-time data দরকার → View
  Complex/slow query, freshness tolerable → Materialized View
  Permission boundary দরকার → View (sensitive columns hide)
  Pre-computation → Materialized View
```

---

### Simple View — Internals

**View storage:**

```sql
-- View তৈরি করলে:
CREATE VIEW active_orders AS
SELECT o.id, o.user_id, o.amount, o.status, u.name AS user_name
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status IN ('pending', 'processing');

-- pg_class-এ entry: relkind = 'v'
SELECT relname, relkind, pg_get_viewdef(oid) AS definition
FROM pg_class
WHERE relname = 'active_orders';

-- view definition stored in pg_rewrite:
SELECT rulename, ev_type, ev_enabled, pg_get_expr(ev_action, ev_class)
FROM pg_rewrite
WHERE ev_class = 'active_orders'::regclass;
```

**View rewrite mechanism:**

```
Query: SELECT * FROM active_orders WHERE amount > 1000;

Rewriter step (before planner):
  1. active_orders → pg_rewrite lookup
  2. View definition inline করো:
     SELECT o.id, o.user_id, o.amount, o.status, u.name AS user_name
     FROM orders o
     JOIN users u ON u.id = o.user_id
     WHERE o.status IN ('pending', 'processing')
  3. Outer WHERE merge করো:
     WHERE o.status IN ('pending', 'processing')
     AND o.amount > 1000  ← outer condition pushed down!

Final query (after rewrite):
  SELECT o.id, o.user_id, o.amount, o.status, u.name AS user_name
  FROM orders o
  JOIN users u ON u.id = o.user_id
  WHERE o.status IN ('pending', 'processing')
    AND o.amount > 1000;

Planner এই rewritten query optimize করে।
View এখানে transparent — planner directly sees the tables।
Index on orders(status, amount) — planner can use it!
```

**View-এর use cases:**

```sql
-- Use case 1: Security — sensitive columns hide
CREATE VIEW safe_users AS
SELECT id, name, email, created_at, active
FROM users;
-- password_hash, ssn, credit_card — excluded!

GRANT SELECT ON safe_users TO app_readonly_role;
REVOKE SELECT ON users FROM app_readonly_role;
-- app_readonly_role শুধু safe_users দেখতে পারবে

-- Use case 2: Simplify complex query
CREATE VIEW monthly_revenue AS
SELECT
  date_trunc('month', created_at) AS month,
  sum(amount) AS revenue,
  count(*) AS order_count,
  count(DISTINCT user_id) AS unique_customers
FROM orders
WHERE status = 'completed'
GROUP BY 1;

-- Reporting queries simple হয়:
SELECT * FROM monthly_revenue WHERE month >= '2024-01-01';

-- Use case 3: API compatibility during schema change
-- Table rename করতে হবে কিন্তু old name-এ application queries আছে:
ALTER TABLE orders RENAME TO purchase_orders;
CREATE VIEW orders AS SELECT * FROM purchase_orders;
-- Old queries still work!

-- Use case 4: Virtual column
CREATE VIEW products_with_margin AS
SELECT *,
  (price - cost) AS margin,
  round((price - cost) / price * 100, 2) AS margin_pct
FROM products;
```

---

### Updatable View

**কখন view automatically updatable:**

```
PostgreSQL automatically makes a view updatable if ALL:
  1. Single base table (no JOINs)
  2. No DISTINCT
  3. No GROUP BY, HAVING
  4. No aggregate functions (count, sum, etc.)
  5. No window functions
  6. No set operations (UNION, EXCEPT, INTERSECT)
  7. No LIMIT, OFFSET in view definition
  8. All columns from base table (or can be mapped)

If all satisfied → INSERT/UPDATE/DELETE on view → works on base table
```

```sql
-- Updatable view example:
CREATE VIEW pending_orders AS
SELECT id, user_id, amount, status, created_at
FROM orders
WHERE status = 'pending';

-- These work:
UPDATE pending_orders SET amount = 150 WHERE id = 5;
-- Translates to: UPDATE orders SET amount = 150 WHERE id = 5

DELETE FROM pending_orders WHERE id = 5;
-- Translates to: DELETE FROM orders WHERE id = 5

INSERT INTO pending_orders (user_id, amount, status)
VALUES (42, 100, 'pending');
-- Translates to: INSERT INTO orders (user_id, amount, status)
-- VALUES (42, 100, 'pending')

-- Problem: status='completed' row insert করা যাবে!
INSERT INTO pending_orders (user_id, amount, status)
VALUES (42, 100, 'completed');  -- Works but row disappears from view!

-- Fix: WITH CHECK OPTION
CREATE OR REPLACE VIEW pending_orders AS
SELECT id, user_id, amount, status, created_at
FROM orders
WHERE status = 'pending'
WITH CHECK OPTION;

-- Now:
INSERT INTO pending_orders (user_id, amount, status)
VALUES (42, 100, 'completed');
-- ERROR: new row violates check option for view "pending_orders"

-- LOCAL vs CASCADED check option:
-- WITH LOCAL CHECK OPTION: শুধু এই view-এর WHERE check করে
-- WITH CASCADED CHECK OPTION: এই view + base view-এর WHERE check করে (default)
```

**INSTEAD OF Trigger — Complex View Updates:**

```sql
-- Non-updatable view (JOIN থাকায়):
CREATE VIEW order_details AS
SELECT
  o.id AS order_id,
  o.amount,
  o.status,
  u.name AS customer_name,
  u.email AS customer_email
FROM orders o
JOIN users u ON u.id = o.user_id;

-- এটা automatically updatable না। কিন্তু INSTEAD OF trigger দিয়ে করা যায়:

CREATE OR REPLACE FUNCTION order_details_update()
RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  -- Update orders table
  IF NEW.amount IS DISTINCT FROM OLD.amount OR
     NEW.status IS DISTINCT FROM OLD.status THEN
    UPDATE orders
    SET amount = NEW.amount,
        status = NEW.status
    WHERE id = NEW.order_id;
  END IF;

  -- Update users table
  IF NEW.customer_name IS DISTINCT FROM OLD.customer_name OR
     NEW.customer_email IS DISTINCT FROM OLD.customer_email THEN
    UPDATE users
    SET name = NEW.customer_name,
        email = NEW.customer_email
    WHERE id = (SELECT user_id FROM orders WHERE id = NEW.order_id);
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER order_details_instead_update
INSTEAD OF UPDATE ON order_details
FOR EACH ROW EXECUTE FUNCTION order_details_update();

-- Now this works:
UPDATE order_details
SET amount = 200, customer_name = 'Alice Smith'
WHERE order_id = 42;
```

---

### Materialized View — Physical Storage এবং Refresh

**Internal storage:**

```
Materialized View = heap file + optional indexes
  relkind = 'm' (in pg_class)
  
CREATE MATERIALIZED VIEW mv_sales_summary AS
SELECT ...;

Physical files created:
  base/<db_oid>/<matview_oid>       ← actual data (like a table)
  base/<db_oid>/<matview_oid>_fsm  ← free space map
  base/<db_oid>/<matview_oid>_vm   ← visibility map

Indexes on materialized view:
  CREATE UNIQUE INDEX ON mv_sales_summary (sale_date, category);
  CREATE INDEX ON mv_sales_summary (sale_date);
  ← These are just like table indexes

Query: SELECT * FROM mv_sales_summary WHERE sale_date = '2024-01-15';
  → Index Scan on mv_sales_summary (no JOINs, no computation!)
  → Very fast
```

**REFRESH mechanics:**

```sql
-- REFRESH MATERIALIZED VIEW (blocking):
REFRESH MATERIALIZED VIEW mv_sales_summary;

-- What happens internally:
-- 1. ACCESS EXCLUSIVE lock on materialized view (blocks ALL queries!)
-- 2. Truncate existing data
-- 3. Re-execute the defining query
-- 4. Insert results into heap file
-- 5. Release lock
-- Duration: depends on query complexity + data volume
-- Downtime: readers blocked during entire refresh!

-- REFRESH CONCURRENTLY (non-blocking):
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_summary;

-- Requirements: UNIQUE INDEX on materialized view (mandatory!)
-- What happens internally:
-- 1. Re-execute the defining query into a TEMP table
-- 2. Compare temp table vs current matview:
--    - New rows: INSERT into matview
--    - Changed rows: UPDATE in matview
--    - Removed rows: DELETE from matview
-- 3. Uses standard MVCC — readers continue with old data during refresh
-- 4. Short exclusive lock only at the final swap
-- Duration: longer than regular REFRESH (extra comparison step)
-- Downtime: NONE for readers!

-- Minimum setup for CONCURRENTLY:
CREATE MATERIALIZED VIEW mv_sales_summary AS
SELECT
  date_trunc('day', o.created_at) AS sale_date,
  c.name AS category,
  sum(oi.quantity * p.price) AS revenue,
  count(DISTINCT o.id) AS order_count
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
JOIN categories c ON c.id = p.category_id
WHERE o.status = 'completed'
GROUP BY 1, 2;

-- UNIQUE index (required for CONCURRENTLY):
CREATE UNIQUE INDEX ON mv_sales_summary (sale_date, category);

-- Other indexes for query performance:
CREATE INDEX ON mv_sales_summary (sale_date);
CREATE INDEX ON mv_sales_summary (category);

-- Automated refresh with pg_cron:
CREATE EXTENSION pg_cron;
SELECT cron.schedule(
  'refresh-sales-summary',
  '*/15 * * * *',  -- every 15 minutes
  'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_summary'
);
```

**Materialized View Gotchas:**

```sql
-- Gotcha 1: Stale data
-- Application assumes matview is current, but last refresh was 2 hours ago
-- Fix: track refresh time
CREATE TABLE matview_refresh_log (
  view_name text,
  refreshed_at timestamptz DEFAULT now()
);

-- After each refresh:
INSERT INTO matview_refresh_log (view_name) VALUES ('mv_sales_summary');

-- Application checks staleness:
SELECT now() - refreshed_at AS stale_for
FROM matview_refresh_log
WHERE view_name = 'mv_sales_summary'
ORDER BY refreshed_at DESC LIMIT 1;

-- Gotcha 2: REFRESH CONCURRENTLY is slower than regular REFRESH
-- For a large matview with many changes:
-- Regular: truncate + insert (fast bulk operation)
-- Concurrent: diff + row-by-row update (slower for big changes)
-- Use CONCURRENTLY only when you need zero downtime

-- Gotcha 3: Matview doesn't automatically update when base tables change
-- No triggers on base table → matview → auto refresh
-- Must explicitly refresh (cron, application event, trigger)
-- Logical replication can trigger refresh on replica

-- Gotcha 4: VACUUM on matview
-- Matview-এও dead tuples জমে (each REFRESH creates dead tuples)
-- VACUUM ANALYZE mv_sales_summary;
-- Autovacuum handles this automatically

-- Gotcha 5: Space usage
SELECT
  relname,
  pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
  pg_size_pretty(pg_relation_size(oid)) AS data_size,
  pg_size_pretty(pg_indexes_size(oid)) AS index_size
FROM pg_class
WHERE relname = 'mv_sales_summary';
```

**Incremental Refresh Pattern (custom):**

```sql
-- Built-in REFRESH: always full refresh (no incremental)
-- Custom incremental pattern:

-- 1. Track what's been processed:
CREATE TABLE mv_refresh_state (
  view_name text PRIMARY KEY,
  last_processed_id bigint DEFAULT 0,
  last_processed_at timestamptz DEFAULT '1970-01-01'
);

INSERT INTO mv_refresh_state (view_name) VALUES ('mv_user_stats');

-- 2. Incremental refresh function:
CREATE OR REPLACE FUNCTION refresh_mv_user_stats_incremental()
RETURNS void LANGUAGE plpgsql AS $$
DECLARE
  v_last_id bigint;
  v_last_at timestamptz;
BEGIN
  SELECT last_processed_id, last_processed_at
  INTO v_last_id, v_last_at
  FROM mv_refresh_state WHERE view_name = 'mv_user_stats';

  -- Process only new/changed orders
  INSERT INTO mv_user_stats (user_id, order_count, total_spent, last_order_at)
  SELECT
    user_id,
    count(*),
    sum(amount),
    max(created_at)
  FROM orders
  WHERE id > v_last_id  -- incremental!
  GROUP BY user_id
  ON CONFLICT (user_id) DO UPDATE SET
    order_count = mv_user_stats.order_count + EXCLUDED.order_count,
    total_spent = mv_user_stats.total_spent + EXCLUDED.total_spent,
    last_order_at = GREATEST(mv_user_stats.last_order_at, EXCLUDED.last_order_at);

  -- Update state
  UPDATE mv_refresh_state
  SET last_processed_id = (SELECT max(id) FROM orders),
      last_processed_at = now()
  WHERE view_name = 'mv_user_stats';
END;
$$;
```

---

## Topic 17 — Function & Stored Procedure

### Big Picture: Database-এর Code — কখন ভালো, কখন বিপজ্জনক?

Function/Procedure database-এর ভেতরে code রাখার সুযোগ দেয়। এটা powerful — কিন্তু misuse করলে maintainability nightmare।

```
কখন function/procedure database-এ রাখা ভালো:
  ✅ Data-intensive computation (moving computation to data)
  ✅ Trigger function (trigger-এ call হয়)
  ✅ Complex constraint validation
  ✅ Reusable business calculation (tax, discount formula)
  ✅ Stored access patterns (encapsulate complex query)

কখন application code-এ রাখা ভালো:
  ❌ Complex business logic (order workflow, payment processing)
  ❌ External service calls (email, webhook)
  ❌ Heavy computation that scales horizontally
  ❌ Logic that needs unit testing framework
  ❌ Logic tied to application state
```

---

### Function Languages

```sql
-- Language options:

-- SQL: simplest, most portable, optimizer-friendly
CREATE OR REPLACE FUNCTION get_user_email(p_id bigint)
RETURNS text
LANGUAGE sql STABLE AS $$
  SELECT email FROM users WHERE id = p_id;
$$;

-- PL/pgSQL: procedural, error handling, loops
CREATE OR REPLACE FUNCTION complex_calculation(p_value numeric)
RETURNS numeric
LANGUAGE plpgsql IMMUTABLE AS $$
DECLARE
  v_result numeric;
BEGIN
  -- procedural logic
  v_result := p_value * 2;
  RETURN v_result;
END;
$$;

-- PL/Python: (requires postgresql-plpython3 package)
CREATE EXTENSION plpython3u;
CREATE OR REPLACE FUNCTION python_function(data text)
RETURNS text
LANGUAGE plpython3u AS $$
import json
obj = json.loads(data)
return str(obj.get('key', 'default'))
$$;

-- PL/V8 (JavaScript): requires pg_v8 extension
-- PL/R: R language for statistical functions
-- C: maximum performance, compiled extension
```

---

### Volatility — Planner-এর Trust Level

**এটা সবচেয়ে underunderstood function property:**

```
IMMUTABLE:
  Same input → ALWAYS same output, regardless of anything
  Database state বদলালেও না, time বদলালেও না
  Planner: can fold (pre-compute) during planning
  Index: can be used in expression indexes
  Parallel: safe to execute in parallel
  
  Examples: lower(), md5(), sqrt(), abs()
  Wrong IMMUTABLE: now(), random(), currval()

STABLE:
  Same input → same output WITHIN a single query
  Different queries may get different results (if DB state changed between)
  Cannot be used in expression indexes
  Parallel: safe (within a query)
  
  Examples: now() [same within query], current_setting()
  Functions reading table data: STABLE (data might change between queries)

VOLATILE (default):
  May return different results on EACH CALL
  No assumptions made by planner
  Index: cannot be used
  Parallel: might not be safe
  
  Examples: random(), txid_current(), nextval()
  Any function with side effects (INSERT, UPDATE, DELETE)
```

```sql
-- IMMUTABLE wrong assignment → wrong results!
CREATE OR REPLACE FUNCTION get_current_price(p_product_id bigint)
RETURNS numeric
LANGUAGE sql IMMUTABLE AS $$  -- WRONG! reads table = not immutable
  SELECT price FROM products WHERE id = p_product_id;
$$;

-- Planner might cache the result from first call!
SELECT get_current_price(1);  -- returns 100
-- price changes to 150 in products table
SELECT get_current_price(1);  -- might still return 100! (cached!)

-- Correct:
CREATE OR REPLACE FUNCTION get_current_price(p_product_id bigint)
RETURNS numeric
LANGUAGE sql STABLE AS $$  -- reads DB, stable within query
  SELECT price FROM products WHERE id = p_product_id;
$$;

-- Performance implication:
-- IMMUTABLE function in WHERE clause:
SELECT * FROM orders WHERE fiscal_year(created_at) = 2024;
-- IMMUTABLE: planner can evaluate fiscal_year(2024-...) at plan time
-- STABLE: evaluated per-row at execution time
-- VOLATILE: evaluated per-row, no optimization
```

---

### Function Execution Model

```sql
-- SQL function: inlined by planner (transparent)
CREATE OR REPLACE FUNCTION active_users_count()
RETURNS bigint
LANGUAGE sql STABLE AS $$
  SELECT count(*) FROM users WHERE active = true;
$$;

SELECT active_users_count();
-- Planner sees: SELECT count(*) FROM users WHERE active = true;
-- Optimizes normally (uses index on active if exists)

-- PL/pgSQL function: opaque to planner (black box)
CREATE OR REPLACE FUNCTION complex_user_count()
RETURNS bigint
LANGUAGE plpgsql STABLE AS $$
DECLARE
  v_count bigint;
BEGIN
  SELECT count(*) INTO v_count FROM users WHERE active = true;
  RETURN v_count;
END;
$$;

SELECT complex_user_count();
-- Planner sees: execute this black box, returns bigint
-- Cannot optimize inside the function!
-- Function executed, internal query planned separately inside

-- Implication: prefer SQL functions over PL/pgSQL when possible
-- SQL functions: planner can inline and optimize
-- PL/pgSQL: separate planning per execution
```

---

### Return Types

```sql
-- Scalar return:
CREATE OR REPLACE FUNCTION user_order_total(p_user_id bigint)
RETURNS numeric
LANGUAGE sql STABLE AS $$
  SELECT coalesce(sum(amount), 0) FROM orders WHERE user_id = p_user_id;
$$;

-- Table return (SET RETURNING FUNCTION):
CREATE OR REPLACE FUNCTION user_recent_orders(
  p_user_id bigint,
  p_limit int DEFAULT 10
)
RETURNS TABLE(
  order_id bigint,
  amount numeric,
  status text,
  created_at timestamptz
)
LANGUAGE sql STABLE AS $$
  SELECT id, amount, status, created_at
  FROM orders
  WHERE user_id = p_user_id
  ORDER BY created_at DESC
  LIMIT p_limit;
$$;

-- Use in query:
SELECT * FROM user_recent_orders(42, 5);
SELECT * FROM user_recent_orders(42) WHERE status = 'completed';

-- LATERAL join with SRF:
SELECT u.name, o.*
FROM users u
CROSS JOIN LATERAL user_recent_orders(u.id, 3) o
WHERE u.active = true;
-- Returns 3 recent orders for each active user

-- Return RECORD type (flexible schema):
CREATE OR REPLACE FUNCTION parse_address(p_full_address text)
RETURNS record
LANGUAGE plpgsql IMMUTABLE AS $$
DECLARE
  v_result record;
BEGIN
  -- Parse logic...
  SELECT 'Street', 'City', 'ZIP' INTO v_result;
  RETURN v_result;
END;
$$;

-- Must specify columns when calling:
SELECT * FROM parse_address('123 Main St, Dhaka 1000')
  AS (street text, city text, zip text);
```

---

### PL/pgSQL In Depth

```sql
-- Complete PL/pgSQL example:
CREATE OR REPLACE FUNCTION process_order(
  p_user_id bigint,
  p_items jsonb,  -- [{"product_id": 1, "quantity": 2}, ...]
  OUT v_order_id bigint,
  OUT v_total numeric,
  OUT v_error text
)
LANGUAGE plpgsql AS $$
DECLARE
  v_item jsonb;
  v_product record;
  v_item_total numeric;
BEGIN
  -- Initialize
  v_total := 0;
  v_error := NULL;

  -- Input validation
  IF p_user_id IS NULL THEN
    v_error := 'user_id cannot be null';
    RETURN;
  END IF;

  IF jsonb_array_length(p_items) = 0 THEN
    v_error := 'items cannot be empty';
    RETURN;
  END IF;

  -- Create order
  INSERT INTO orders (user_id, status, amount)
  VALUES (p_user_id, 'pending', 0)
  RETURNING id INTO v_order_id;

  -- Process each item
  FOR v_item IN SELECT * FROM jsonb_array_elements(p_items)
  LOOP
    -- Get product (with lock to prevent concurrent modification)
    SELECT id, name, price, stock_count
    INTO v_product
    FROM products
    WHERE id = (v_item->>'product_id')::bigint
    FOR UPDATE;  -- row lock

    IF NOT FOUND THEN
      v_error := format('Product %s not found', v_item->>'product_id');
      RAISE EXCEPTION '%', v_error;  -- triggers ROLLBACK of all
    END IF;

    IF v_product.stock_count < (v_item->>'quantity')::int THEN
      v_error := format('Insufficient stock for product %s', v_product.name);
      RAISE EXCEPTION '%', v_error;
    END IF;

    v_item_total := v_product.price * (v_item->>'quantity')::int;
    v_total := v_total + v_item_total;

    -- Insert order item
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (v_order_id, v_product.id, (v_item->>'quantity')::int, v_product.price);

    -- Update stock
    UPDATE products
    SET stock_count = stock_count - (v_item->>'quantity')::int
    WHERE id = v_product.id;
  END LOOP;

  -- Update order total
  UPDATE orders SET amount = v_total WHERE id = v_order_id;

EXCEPTION
  WHEN OTHERS THEN
    -- All changes rolled back automatically
    v_error := SQLERRM;
    v_order_id := NULL;
    v_total := 0;
END;
$$;

-- Call:
SELECT * FROM process_order(
  42,
  '[{"product_id": 1, "quantity": 2}, {"product_id": 3, "quantity": 1}]'::jsonb
);
```

---

### Stored Procedure — COMMIT Inside

```sql
-- Function vs Procedure:
-- Function: RETURN value, cannot COMMIT/ROLLBACK inside
-- Procedure: no return value (or OUT params), CAN COMMIT/ROLLBACK inside

-- Use case: batch processing with intermediate commits
CREATE OR REPLACE PROCEDURE batch_process_orders(p_batch_size int DEFAULT 100)
LANGUAGE plpgsql AS $$
DECLARE
  v_processed int := 0;
  v_order record;
BEGIN
  LOOP
    -- Process one batch
    FOR v_order IN
      SELECT id, user_id, amount
      FROM orders
      WHERE status = 'pending'
      ORDER BY created_at
      LIMIT p_batch_size
      FOR UPDATE SKIP LOCKED  -- concurrent-safe
    LOOP
      -- Process the order
      UPDATE orders
      SET status = 'processing',
          processed_at = now()
      WHERE id = v_order.id;

      -- Maybe call external service, update inventory, etc.
      v_processed := v_processed + 1;
    END LOOP;

    -- Commit after each batch
    COMMIT;  -- this is only possible in PROCEDURE, not FUNCTION!

    -- Exit when no more pending orders
    EXIT WHEN NOT EXISTS (
      SELECT 1 FROM orders WHERE status = 'pending'
    );
  END LOOP;

  RAISE NOTICE 'Processed % orders', v_processed;
END;
$$;

-- Call:
CALL batch_process_orders(100);
-- Commits every 100 orders
-- If process dies mid-way: completed batches committed, rest still pending
-- No giant transaction holding locks for hours!
```

---

### Security Definer vs Invoker

```sql
-- SECURITY INVOKER (default): runs with caller's privileges
CREATE OR REPLACE FUNCTION public_function()
RETURNS text
LANGUAGE sql SECURITY INVOKER AS $$
  SELECT name FROM restricted_table;  -- caller must have SELECT on restricted_table
$$;

-- SECURITY DEFINER: runs with owner's privileges
CREATE OR REPLACE FUNCTION admin_stats()
RETURNS bigint
LANGUAGE sql SECURITY DEFINER AS $$
  SELECT count(*) FROM sensitive_admin_table;
  -- Caller doesn't need permission on sensitive_admin_table
  -- Function owner's permissions used
$$;
-- Owner must have SELECT on sensitive_admin_table

-- SECURITY DEFINER gotcha: search_path injection!
-- Malicious user creates: public.sensitive_admin_table (view)
-- SECURITY DEFINER function searches public schema first!
-- Fix: SET search_path explicitly
CREATE OR REPLACE FUNCTION admin_stats()
RETURNS bigint
LANGUAGE sql SECURITY DEFINER
SET search_path = admin_schema, pg_catalog  -- explicit, safe
AS $$
  SELECT count(*) FROM sensitive_admin_table;
$$;
```

---

## Topic 18 — Trigger: এবং কখন বিপজ্জনক

### Big Picture: Trigger = Power with Responsibility

Trigger হলো database-এর event handler। Powerful, কিন্তু misuse করলে:
- Silent data modification (debugging nightmare)
- Performance degradation (invisible overhead)
- Cascade failures (unexpected behaviors)
- Testing কঠিন (application logic হিসেবে test করা যায় না)

---

### Trigger Types — Complete Matrix

```
Timing × Event × Level = 3 × 4 × 2 = 24 possible trigger types

Timing:
  BEFORE: operation-এর আগে (NEW/OLD modify করতে পারো)
  AFTER: operation-এর পরে (committed values দেখতে পারো)
  INSTEAD OF: view-এ operation replace করো

Event:
  INSERT: নতুন row
  UPDATE: existing row পরিবর্তন
  DELETE: row মুছে ফেলা
  TRUNCATE: table truncate (statement-level only)

Level:
  FOR EACH ROW: প্রতিটা affected row-এর জন্য
  FOR EACH STATEMENT: একবার (একটা statement যত rows affect করুক)
```

**Trigger function-এর special variables:**

```
TG_OP: 'INSERT', 'UPDATE', 'DELETE', 'TRUNCATE'
TG_TABLE_NAME: trigger-এর table name
TG_TABLE_SCHEMA: schema name
TG_WHEN: 'BEFORE', 'AFTER', 'INSTEAD OF'
TG_LEVEL: 'ROW', 'STATEMENT'
TG_NARGS: trigger arguments count
TG_ARGV[]: trigger arguments array

Row-level:
  NEW: new row (INSERT, UPDATE)
  OLD: old row (UPDATE, DELETE)

BEFORE trigger return:
  RETURN NEW: proceed with NEW row
  RETURN OLD: proceed with OLD row (for DELETE)
  RETURN NULL: cancel the operation! (row silently skipped)
  Modify NEW fields: insert/update with modified values

AFTER trigger return:
  RETURN value ignored (use NULL)
  Operation already done, cannot cancel
```

---

### Common Trigger Patterns

**Pattern 1: Auto-update timestamp**

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at := now();
  RETURN NEW;
END;
$$;

CREATE TRIGGER orders_set_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW
EXECUTE FUNCTION set_updated_at();

-- Clean, minimal, safe trigger
```

**Pattern 2: Audit log**

```sql
CREATE TABLE audit_log (
  id bigserial PRIMARY KEY,
  table_name text NOT NULL,
  operation text NOT NULL,
  row_id bigint,
  old_data jsonb,
  new_data jsonb,
  changed_columns text[],
  changed_at timestamptz DEFAULT now(),
  changed_by text DEFAULT current_user,
  app_user_id bigint  -- application-level user (from session variable)
);

CREATE OR REPLACE FUNCTION audit_changes()
RETURNS trigger
LANGUAGE plpgsql SECURITY DEFINER
SET search_path = public, pg_catalog AS $$
DECLARE
  v_old jsonb;
  v_new jsonb;
  v_changed text[];
  v_row_id bigint;
BEGIN
  -- Get the primary key value
  CASE TG_OP
    WHEN 'DELETE' THEN
      v_old := row_to_json(OLD)::jsonb;
      v_new := NULL;
      v_row_id := OLD.id;
    WHEN 'INSERT' THEN
      v_old := NULL;
      v_new := row_to_json(NEW)::jsonb;
      v_row_id := NEW.id;
    WHEN 'UPDATE' THEN
      v_old := row_to_json(OLD)::jsonb;
      v_new := row_to_json(NEW)::jsonb;
      v_row_id := NEW.id;

      -- Find changed columns
      SELECT array_agg(key)
      INTO v_changed
      FROM jsonb_each(v_new) AS n(key, val)
      WHERE v_old->key IS DISTINCT FROM val;
  END CASE;

  INSERT INTO audit_log (
    table_name, operation, row_id,
    old_data, new_data, changed_columns,
    app_user_id
  ) VALUES (
    TG_TABLE_NAME, TG_OP, v_row_id,
    v_old, v_new, v_changed,
    nullif(current_setting('app.current_user_id', true), '')::bigint
  );

  IF TG_OP = 'DELETE' THEN RETURN OLD;
  ELSE RETURN NEW;
  END IF;
END;
$$;

-- Apply to sensitive tables:
CREATE TRIGGER audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_changes();

CREATE TRIGGER audit_orders
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION audit_changes();
```

**Pattern 3: Enforce business rule (CHECK-এ যা করা যায় না)**

```sql
-- Rule: একটা user সর্বোচ্চ ৫টা active subscription রাখতে পারবে
CREATE OR REPLACE FUNCTION check_subscription_limit()
RETURNS trigger
LANGUAGE plpgsql AS $$
DECLARE
  v_count int;
BEGIN
  IF NEW.status = 'active' THEN
    SELECT count(*)
    INTO v_count
    FROM subscriptions
    WHERE user_id = NEW.user_id
      AND status = 'active'
      AND id != coalesce(NEW.id, -1);  -- exclude current row for UPDATE

    IF v_count >= 5 THEN
      RAISE EXCEPTION 'User % already has 5 active subscriptions', NEW.user_id
        USING ERRCODE = 'check_violation';
    END IF;
  END IF;

  RETURN NEW;
END;
$$;

CREATE TRIGGER enforce_subscription_limit
BEFORE INSERT OR UPDATE ON subscriptions
FOR EACH ROW EXECUTE FUNCTION check_subscription_limit();
```

**Pattern 4: Partition routing (legacy approach)**

```sql
-- Old approach (before native partitioning):
CREATE OR REPLACE FUNCTION route_to_partition()
RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  IF NEW.created_at >= '2024-01-01' AND NEW.created_at < '2025-01-01' THEN
    INSERT INTO orders_2024 VALUES (NEW.*);
  ELSIF NEW.created_at >= '2023-01-01' THEN
    INSERT INTO orders_2023 VALUES (NEW.*);
  ELSE
    INSERT INTO orders_old VALUES (NEW.*);
  END IF;
  RETURN NULL;  -- prevent insert into parent table
END;
$$;

CREATE TRIGGER route_order
BEFORE INSERT ON orders
FOR EACH ROW EXECUTE FUNCTION route_to_partition();

-- Modern: use PARTITION BY RANGE (Topic 20), much better
```

---

### কখন Trigger বিপজ্জনক — Detailed Scenarios

**Danger 1: Cascading trigger chains**

```sql
-- Table A trigger → INSERT into B
-- Table B trigger → UPDATE C
-- Table C trigger → UPDATE A
-- → Infinite recursion!

-- PostgreSQL কি detect করে?
-- Statement-level: trigger_depth check করে
-- Row-level: pg_trigger_depth() function দিয়ে manually check করতে হয়

CREATE OR REPLACE FUNCTION safe_trigger()
RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  IF pg_trigger_depth() > 1 THEN
    -- Already in a nested trigger, don't recurse
    RETURN NEW;
  END IF;

  -- ... trigger logic ...
  RETURN NEW;
END;
$$;

-- Maximum depth: session_replication_role এবং trigger_depth-এ নির্ভরশীল
-- Better: design to avoid cascades
```

**Danger 2: Bulk operation performance**

```sql
-- Audit trigger on orders table
-- AFTER INSERT OR UPDATE OR DELETE FOR EACH ROW

-- Bulk insert:
INSERT INTO orders SELECT * FROM temp_import_orders;  -- 1M rows

-- This triggers 1M trigger executions!
-- Each trigger: INSERT into audit_log
-- Total: 2M INSERT operations instead of 1M

-- Benchmark:
-- Without trigger: 1M rows in 5 seconds
-- With row-level audit trigger: 1M rows in 45 seconds!

-- Fix 1: Disable trigger for bulk ops:
ALTER TABLE orders DISABLE TRIGGER audit_orders;
-- bulk operation
ALTER TABLE orders ENABLE TRIGGER audit_orders;

-- Fix 2: Statement-level trigger for bulk ops:
CREATE OR REPLACE FUNCTION bulk_audit()
RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  INSERT INTO audit_log (table_name, operation, changed_at)
  VALUES (TG_TABLE_NAME, TG_OP, now());
  -- No row data — just "something changed"
  RETURN NULL;
END;
$$;

CREATE TRIGGER orders_bulk_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH STATEMENT  -- once per statement, not per row
EXECUTE FUNCTION bulk_audit();

-- Fix 3: Deferred trigger:
CREATE TRIGGER orders_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW
DEFERRABLE INITIALLY DEFERRED  -- execute at COMMIT time, batched
EXECUTE FUNCTION audit_changes();
```

**Danger 3: Hidden business logic**

```sql
-- Developer writes:
UPDATE orders SET status = 'shipped' WHERE id = 42;
-- Simple update, right?

-- But hidden trigger chain:
-- orders AFTER UPDATE trigger (status change):
--   → sends email (via dblink to mail server)
--   → updates inventory (UPDATE products SET stock...)
--   → creates shipping record (INSERT INTO shipments...)
--   → notifies warehouse (pg_notify)
--   → logs to analytics (INSERT INTO analytics_events...)

-- Side effects:
-- Email sent even in test environment!
-- ROLLBACK doesn't unsend email (external side effect)
-- New developer doesn't know about this — debugging hell
-- Performance: single UPDATE now takes 500ms due to all side effects

-- Better architecture:
-- DB trigger: only data integrity (audit, denormalized counts)
-- Application event: order shipped → application handles downstream effects
-- Event sourcing: OrderShipped event → subscribers handle their concerns
```

**Danger 4: Transaction interaction**

```sql
-- AFTER trigger runs inside the same transaction
-- If trigger fails → whole transaction rolls back

CREATE TRIGGER orders_notify
AFTER INSERT ON orders
FOR EACH ROW EXECUTE FUNCTION notify_external_service();  -- calls HTTP endpoint

-- Flow:
-- INSERT INTO orders ...
-- Trigger: HTTP call to external service
-- External service: 500 error
-- Trigger: RAISE EXCEPTION
-- Result: ORDER ROLLBACK! Order never saved!

-- Fix: Don't make external calls in triggers!
-- Pattern: write to outbox table, separate process sends notifications
CREATE OR REPLACE FUNCTION queue_notification()
RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  INSERT INTO notification_outbox (
    event_type, payload, created_at, status
  ) VALUES (
    'order_created',
    row_to_json(NEW)::jsonb,
    now(),
    'pending'
  );
  RETURN NEW;
END;
$$;
-- Separate worker reads notification_outbox and sends HTTP/email
-- If HTTP fails: retry logic, no order rollback
```

**Danger 5: RETURN NULL silently drops rows**

```sql
-- BEFORE trigger returning NULL = row operation cancelled!
CREATE OR REPLACE FUNCTION validate_order()
RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  IF NEW.amount <= 0 THEN
    RETURN NULL;  -- silently cancels the INSERT!
  END IF;
  RETURN NEW;
END;
$$;

-- Developer expects:
INSERT INTO orders (user_id, amount) VALUES (1, -50);
-- Expects: constraint violation error
-- Gets: no error, no row inserted, no indication anything went wrong!
-- Application thinks order was created, user thinks payment processed...

-- Better:
IF NEW.amount <= 0 THEN
  RAISE EXCEPTION 'Order amount must be positive, got %', NEW.amount
    USING ERRCODE = 'check_violation';
END IF;
-- Or better: use CHECK constraint directly
```

---

### Trigger Inspection and Management

```sql
-- All triggers on a table:
SELECT
  t.tgname AS trigger_name,
  t.tgenabled AS enabled,
  CASE t.tgtype & 66
    WHEN 2 THEN 'BEFORE'
    WHEN 64 THEN 'INSTEAD OF'
    ELSE 'AFTER'
  END AS timing,
  CASE
    WHEN t.tgtype & 4 > 0 THEN 'INSERT'
    WHEN t.tgtype & 8 > 0 THEN 'DELETE'
    WHEN t.tgtype & 16 > 0 THEN 'UPDATE'
    ELSE 'TRUNCATE'
  END AS event,
  CASE t.tgtype & 1 WHEN 1 THEN 'ROW' ELSE 'STATEMENT' END AS level,
  p.proname AS function_name
FROM pg_trigger t
JOIN pg_proc p ON p.oid = t.tgfoid
WHERE t.tgrelid = 'orders'::regclass
  AND NOT t.tgisinternal;  -- system triggers-এ exclude

-- Disable/enable specific trigger:
ALTER TABLE orders DISABLE TRIGGER orders_audit;
ALTER TABLE orders ENABLE TRIGGER orders_audit;

-- Disable all triggers (bulk operations):
ALTER TABLE orders DISABLE TRIGGER ALL;
-- ... bulk operation ...
ALTER TABLE orders ENABLE TRIGGER ALL;
-- WARNING: FK triggers-ও disable হয়! Data integrity risk!

-- Trigger execution count (approximate via pg_stat_user_tables):
SELECT relname, n_tup_ins, n_tup_upd, n_tup_del
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

---

## Topic 19 — Sequence & Identity

### Big Picture: Auto-increment-এর সঠিক Implementation

```
Auto-increment ID-এর পুরো history PostgreSQL-এ:

v1 (pre-2010): serial pseudo-type
  CREATE TABLE t (id serial);
  → Implicit: CREATE SEQUENCE t_id_seq; + DEFAULT nextval()
  Problem: sequence attached to column, DROP COLUMN-এ orphan sequence

v2 (10+): IDENTITY column (SQL standard)
  CREATE TABLE t (id int GENERATED ALWAYS AS IDENTITY);
  → Standard SQL syntax
  → Sequence owned by column, no orphan
  → GENERATED ALWAYS vs BY DEFAULT control
  → Better replication behavior

Recommendation: Always use IDENTITY in new code
```

---

### Sequence Internals

```sql
-- Sequence create:
CREATE SEQUENCE order_seq
  START WITH 1000
  INCREMENT BY 1
  MINVALUE 1
  MAXVALUE 9223372036854775807  -- bigint max
  NO CYCLE      -- error when maxvalue reached (CYCLE would restart)
  CACHE 50;     -- pre-allocate 50 values

-- Sequence storage:
-- Sequence = small relation (single row in a special page)
-- Stores: last_value, log_cnt (cached values), is_called

-- How CACHE works:
-- CACHE 50: PostgreSQL pre-allocates 50 values in memory
-- nextval() returns from cache (no disk I/O!)
-- cache exhausted → allocate next 50 from disk

-- Cache and crashes:
-- Server crashes after allocating cache but before using all values
-- Those values are LOST → gaps in sequence
-- This is normal and acceptable!
-- Never rely on sequence being gapless
```

```sql
-- Sequence operations:
SELECT nextval('order_seq');    -- advance and return next value
SELECT currval('order_seq');    -- current value (session-local, after nextval)
SELECT lastval();               -- last value from ANY sequence in this session
SELECT setval('order_seq', 5000);        -- set current value (next will be 5001)
SELECT setval('order_seq', 5000, false); -- set to 5000, next call returns 5000

-- Sequence info:
SELECT * FROM pg_sequences WHERE sequencename = 'order_seq';
-- sequence_schema, sequence_name, data_type, start_value,
-- min_value, max_value, increment_by, cycle_option,
-- cache_size, last_value

-- Reset sequence to max existing value (after manual data insert):
SELECT setval('orders_id_seq', (SELECT max(id) FROM orders));
-- Or with PERFORM in function context
```

---

### SERIAL vs IDENTITY — Why Prefer IDENTITY

```sql
-- SERIAL (old, avoid in new code):
CREATE TABLE t (id serial PRIMARY KEY);
-- PostgreSQL internally does:
CREATE SEQUENCE t_id_seq;
ALTER SEQUENCE t_id_seq OWNED BY t.id;
ALTER TABLE t ADD COLUMN id integer NOT NULL DEFAULT nextval('t_id_seq');
ALTER TABLE t ADD PRIMARY KEY (id);

-- Problems:
-- 1. Confusing: 'serial' is not a real type, just shorthand
-- 2. Can insert explicit value without restriction
-- 3. Type is integer (not bigint!) by default for serial
--    Use bigserial for bigint: CREATE TABLE t (id bigserial)
-- 4. pg_dump behavior inconsistent

-- IDENTITY (correct, modern):
CREATE TABLE t (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
-- GENERATED ALWAYS: explicit value insert rejected
-- GENERATED BY DEFAULT: explicit value allowed (useful for migrations)

-- ALWAYS with override (for data migration):
INSERT INTO t (id, name) OVERRIDING SYSTEM VALUE VALUES (1000, 'migrated');
-- Explicit ID allowed with OVERRIDING SYSTEM VALUE

-- GENERATED BY DEFAULT: useful for testing and migration
-- In tests: INSERT with explicit ID for predictable data
-- In migration: INSERT old IDs exactly

-- Custom sequence options with IDENTITY:
CREATE TABLE t (
  id bigint GENERATED ALWAYS AS IDENTITY
    (START WITH 1000 INCREMENT BY 5 CACHE 100)
  PRIMARY KEY
);
```

---

### Sequence and Concurrency

```sql
-- Sequences are designed for high concurrency:
-- nextval() is atomic — each call gets unique value
-- No transaction isolation issues
-- Two concurrent transactions:
--   T1: nextval → 1001
--   T2: nextval → 1002
--   Guaranteed unique, no overlap

-- Sequence and ROLLBACK:
-- T1: nextval → 1001
-- T1: INSERT fails
-- T1: ROLLBACK
-- T2: nextval → 1002 (not 1001! sequence doesn't rollback)
-- → Gap: 1001 never used
-- This is by design — preventing sequence contention

-- High-concurrency sequence performance:
-- CACHE 1: every nextval hits sequence storage (disk write per call)
-- CACHE 100: 100 calls share one disk write (100x throughput)
-- Trade-off: crash loses up to CACHE - 1 values

-- For very high-throughput (millions of inserts/sec):
-- CACHE 1000 or even higher
-- UUID v7 (time-ordered) as alternative
```

---

### Sequence and Replication

```sql
-- Sequences in streaming replication:
-- Primary sequence state replicated to standby
-- But: sequence cache local per server!
-- If standby promoted: it might reuse values primary already used

-- Best practice: leave a gap
-- Primary: CACHE 1, sequences advance every transaction
-- After failover: primary might have used 1-1000, standby continues from 1001

-- Logical replication: sequences NOT automatically replicated!
-- Must manually sync after logical replication cutover:
-- On new primary:
SELECT setval('orders_id_seq',
  (SELECT max(id) FROM orders) + 1000);  -- safe gap
```

---

## Topic 20 — Partition

### Big Picture: কেন Partition, এবং কখন?

```
Table size এবং performance:
  1M rows: no partition needed
  100M rows: consider partition
  1B rows: partition probably necessary
  10B rows: partition essential

Without partition on large table:
  Sequential scan: reads ENTIRE table
  Index scan: large B-Tree, many levels
  VACUUM: scans all pages
  Statistics: less accurate (sample from all data)
  DROP old data: DELETE many rows, slow, leaves bloat

With partition:
  Query with partition key: scans ONLY relevant partition (Partition Pruning)
  Index: per-partition, smaller, fewer levels
  VACUUM: parallel per partition
  Statistics: per-partition, more accurate
  DROP old data: DROP TABLE partition_old → instant!
```

---

### How Partitioning Works Internally

```
Partitioned table in PostgreSQL:
  Parent table: defines schema and partition strategy
  Child tables: actual heap files with data
  Each child: independent heap file, independent indexes

pg_class:
  orders: relkind='p' (partitioned table, no actual data!)
  orders_2023: relkind='r' (regular table, actual data)
  orders_2024: relkind='r'

Query routing:
  INSERT INTO orders (created_at, ...) VALUES ('2024-05-15', ...)
  → Partition check: 2024-05-15 in [2024-01-01, 2025-01-01)?  Yes
  → Route to orders_2024
  → INSERT INTO orders_2024

SELECT with partition key:
  SELECT * FROM orders WHERE created_at >= '2024-01-01'
  → Constraint exclusion: which partitions overlap this range?
  → orders_2023: max = 2024-01-01, overlap: YES (boundary)
  → orders_2024: [2024-01-01, 2025-01-01), overlap: YES
  → orders_2025: min = 2025-01-01, overlap: NO → SKIP!
  → Scan only orders_2023, orders_2024
  = Partition Pruning!
```

---

### Range Partition — Time-series

```sql
-- Parent table definition:
CREATE TABLE orders (
  id bigint NOT NULL,
  user_id bigint NOT NULL,
  amount numeric(12,2) NOT NULL,
  status text NOT NULL DEFAULT 'pending',
  created_at timestamptz NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Partitions:
CREATE TABLE orders_2023
  PARTITION OF orders
  FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024
  PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025
  PARTITION OF orders
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Default partition (required if you want to insert outside defined ranges):
CREATE TABLE orders_future
  PARTITION OF orders DEFAULT;

-- Primary key on partitioned table: must include partition key!
ALTER TABLE orders ADD PRIMARY KEY (id, created_at);
-- Cannot be just (id) because data distributed across partitions

-- Indexes on parent = indexes on all partitions:
CREATE INDEX ON orders (user_id);           -- creates index on all partitions!
CREATE INDEX ON orders (status, created_at);

-- Check partition structure:
SELECT
  parent.relname AS parent_table,
  child.relname AS partition_name,
  pg_get_expr(child.relpartbound, child.oid) AS partition_bound,
  pg_size_pretty(pg_total_relation_size(child.oid)) AS partition_size
FROM pg_class parent
JOIN pg_inherits ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON child.oid = pg_inherits.inhrelid
WHERE parent.relname = 'orders'
ORDER BY pg_get_expr(child.relpartbound, child.oid);
```

---

### Partition Pruning — How It Works

```sql
-- Partition pruning in action:
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-06-01';

-- Output:
-- Append  (cost=...)
--   -> Seq Scan on orders_2024  (cost=...)  ← included
--         Filter: (created_at >= '2024-06-01')
--   -> Seq Scan on orders_2025  (cost=...)  ← included (might have future data)
-- Note: orders_2023, orders_future NOT shown = pruned!

-- Partition pruning requires:
-- 1. WHERE condition on partition key
-- 2. Static value (not function of another column)
-- 3. enable_partition_pruning = on (default)

-- Pruning at planning time (static values):
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-01-01';
-- Pruned at plan time: partitions excluded before execution

-- Pruning at execution time (parameters):
PREPARE order_query AS
  SELECT * FROM orders WHERE created_at >= $1;
EXPLAIN EXECUTE order_query('2024-01-01');
-- Pruned at execution time (partition_pruning determines which partitions)

-- When pruning FAILS:
-- No WHERE on partition key:
EXPLAIN SELECT * FROM orders WHERE user_id = 42;
-- Scans ALL partitions! (user_id not partition key)
-- Fix: create index on user_id (already created above)
-- But still scans all partitions — index in each, not partitions skipped

-- Solution: sub-partition by user_id range (complex, usually not worth it)
-- Or: separate table for per-user hot data
```

---

### List Partition

```sql
-- Partition by discrete values:
CREATE TABLE users (
  id bigint NOT NULL,
  country_code char(2) NOT NULL,
  name text NOT NULL,
  email text NOT NULL,
  created_at timestamptz DEFAULT now()
) PARTITION BY LIST (country_code);

CREATE TABLE users_bd PARTITION OF users
  FOR VALUES IN ('BD');

CREATE TABLE users_us PARTITION OF users
  FOR VALUES IN ('US');

CREATE TABLE users_uk PARTITION OF users
  FOR VALUES IN ('GB', 'IE');  -- multiple values in one partition

CREATE TABLE users_others PARTITION OF users DEFAULT;

-- Use case: geo-based sharding
-- query: WHERE country_code = 'BD' → only users_bd scanned
-- Regulatory: BD users' data stays in BD partition
```

---

### Hash Partition

```sql
-- Even distribution when no natural range/list:
CREATE TABLE events (
  id bigint NOT NULL,
  user_id bigint NOT NULL,
  event_type text NOT NULL,
  data jsonb,
  created_at timestamptz DEFAULT now()
) PARTITION BY HASH (user_id);

-- Modulus: total partitions, Remainder: this partition's data
CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 1);
CREATE TABLE events_2 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 2);
CREATE TABLE events_3 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 3);
CREATE TABLE events_4 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 4);
CREATE TABLE events_5 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 5);
CREATE TABLE events_6 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 6);
CREATE TABLE events_7 PARTITION OF events FOR VALUES WITH (MODULUS 8, REMAINDER 7);

-- hash(user_id) % 8 determines partition
-- Each partition: ~equal data distribution

-- Use case:
-- Large table, no natural range
-- Parallel operations across partitions
-- Even I/O distribution across disks (if partitions in different tablespaces)

-- Limitation:
-- No pruning for equality on user_id?
-- Wait — actually:
EXPLAIN SELECT * FROM events WHERE user_id = 42;
-- hash(42) % 8 = computed → specific partition selected!
-- Partition Pruning for equality on hash column!
```

---

### Partition Management — Operations

```sql
-- Add new partition (for future data):
CREATE TABLE orders_2026
  PARTITION OF orders
  FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
-- No lock on parent table! Fast operation.

-- Detach partition (archive old data):
ALTER TABLE orders DETACH PARTITION orders_2022;
-- orders_2022 is now an independent regular table
-- No lock on other partitions, minimal impact

-- Still query it independently:
SELECT count(*) FROM orders_2022;

-- Reattach:
ALTER TABLE orders ATTACH PARTITION orders_2022
  FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');
-- CONSTRAINT validation required (might scan partition)
-- Use NOT VALID for large partitions:
ALTER TABLE orders ATTACH PARTITION orders_2022
  FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');
-- Validates in background with weaker lock

-- Drop old partition (instant disk space recovery):
DROP TABLE orders_2022;
-- Instant! No VACUUM needed. Space immediately returned to OS.
-- vs DELETE FROM orders WHERE created_at < '2023-01-01':
-- Millions of rows, slow, bloat, VACUUM needed after

-- Moving data to archive:
ALTER TABLE orders DETACH PARTITION orders_2022;
-- Now orders_2022 is a regular table
-- Optionally: move to slow tablespace
ALTER TABLE orders_2022 SET TABLESPACE slow_archive_disk;
-- Or: pg_dump this table for cold storage, then DROP
```

---

### Sub-partitioning

```sql
-- Two-level partition: Range × Hash
CREATE TABLE events (
  id bigint NOT NULL,
  user_id bigint NOT NULL,
  created_at timestamptz NOT NULL
) PARTITION BY RANGE (created_at);

-- Monthly partitions with hash sub-partitions:
CREATE TABLE events_2024_01
  PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01')
  PARTITION BY HASH (user_id);

CREATE TABLE events_2024_01_0 PARTITION OF events_2024_01
  FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_2024_01_1 PARTITION OF events_2024_01
  FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ...

-- Total: 12 months × 4 hash = 48 partitions
-- Query: WHERE created_at IN '2024-01' AND user_id = 42
-- → events_2024_01 (range pruning)
-- → hash(42) % 4 = specific sub-partition (hash pruning)
-- = single partition scanned!
```

---

### Partition Gotchas

```sql
-- Gotcha 1: INSERT into wrong partition boundary
INSERT INTO orders (id, created_at) VALUES (1, '2027-06-15');
-- No 2027 partition, no DEFAULT partition:
-- ERROR: no partition of relation "orders" found for row

-- Always have DEFAULT partition, or validate before insert

-- Gotcha 2: Unique constraint across partitions
-- Global unique constraint on partitioned table requires partition key!
ALTER TABLE orders ADD CONSTRAINT orders_id_unique UNIQUE (id);
-- ERROR: unique constraint on partitioned table must include all partitioning columns

-- Must be:
ALTER TABLE orders ADD CONSTRAINT orders_pk PRIMARY KEY (id, created_at);
-- id unique within a partition, not globally!
-- For global uniqueness: use surrogate key in separate non-partitioned lookup

-- Gotcha 3: FK into partitioned table
CREATE TABLE order_items (
  order_id bigint NOT NULL,
  -- This FK works in PostgreSQL 12+:
  FOREIGN KEY (order_id) REFERENCES orders (id)  -- requires (id) to be unique across all partitions
);
-- Problem: orders-এ global unique constraint on just (id) not possible!
-- Solution: either include partition key in FK, or denormalize order_id + created_at

-- Gotcha 4: Query without partition key = full scan
EXPLAIN SELECT count(*) FROM orders;
-- Append scan over ALL partitions
-- Can be parallelized but still scans everything

-- Gotcha 5: Index size still grows
-- Each partition has its own index
-- 12 monthly partitions × index size each = 12x total index size
-- vs 1 table with 1 large index: similar total size but spread across files
-- Benefit: each index smaller individually, better cache behavior

-- Gotcha 6: Partition count too large
-- 10,000 partitions: planning time very slow
-- Planner evaluates all partitions during query planning
-- Keep partition count reasonable: < 1000 per level
-- Too granular partitioning (daily for 10 years = 3650 partitions): problematic
```

---

### Block 4 Summary — Dots Connected

```
Topic 16 (View)
    │
    ├──► Topic 12 (Query Execution): View rewrite happens at analyzer stage
    ├──► Topic 22 (RLS): View + RLS = powerful access control combination
    ├──► Topic 15 (Vacuum): Materialized view needs VACUUM (dead tuples on REFRESH)
    └──► Topic 10 (Normalization): Materialized view = controlled denormalization

Topic 17 (Function)
    │
    ├──► Topic 12 (Planner): IMMUTABLE/STABLE/VOLATILE affects plan caching
    ├──► Topic 18 (Trigger): Trigger functions are PL/pgSQL functions
    └──► Topic 22 (RLS): SECURITY DEFINER functions used in RLS policies

Topic 18 (Trigger)
    │
    ├──► Topic 3 (Transaction): Trigger runs inside same transaction
    ├──► Topic 4 (Locking): BEFORE trigger → potential lock acquisition
    └──► Topic 23 (Audit): Trigger-based audit logging pattern

Topic 19 (Sequence)
    │
    ├──► Topic 4 (Concurrency): nextval() atomic, no transaction rollback
    ├──► Topic 26 (Replication): Sequence cache causes gaps after failover
    └──► Topic 9 (Constraints): IDENTITY column backed by sequence

Topic 20 (Partition)
    │
    ├──► Topic 2 (Physical Storage): Each partition = separate heap file
    ├──► Topic 11 (Index): Partition-local indexes, pruning
    ├──► Topic 15 (Vacuum): Per-partition vacuum, parallel maintenance
    └──► Topic 24 (Backup): Partition-level backup possible (detach → backup)
```
# Senior DBA Master Guide — Block 5
## Security & Access Control

---

## Topic 21 — Users, Roles & Privileges

### Big Picture: PostgreSQL-এর Security Model

PostgreSQL-এর security model একটা layered system:

```
Layer 1: Network / Connection (pg_hba.conf)
  "এই IP থেকে এই user connect করতে পারবে কিনা?"

Layer 2: Authentication
  "এই user সত্যিই সে, এটা prove করতে পারবে কিনা?"

Layer 3: Authorization (Privileges)
  "এই user এই object-এ কী করতে পারবে?"

Layer 4: Row-Level Security (Topic 22)
  "এই user কোন rows দেখতে পারবে?"

Layer 5: Column-Level Security
  "এই user কোন columns দেখতে পারবে?"

সব layer একসাথে কাজ করে। একটা bypass হলেও পুরো security broken।
```

---

### Role = User in PostgreSQL

PostgreSQL-এ "user" এবং "role" technically একই জিনিস — `pg_roles` catalog-এ একসাথে থাকে।

```
পার্থক্য শুধু attribute-এ:
  Role with LOGIN: "user" (connect করতে পারে)
  Role without LOGIN: "group role" (permission bundle করার জন্য)

Historically:
  CREATE USER = CREATE ROLE WITH LOGIN
  CREATE GROUP = CREATE ROLE (deprecated)

Modern practice: শুধু CREATE ROLE ব্যবহার করো, attribute specify করো
```

```sql
-- Role attributes:
CREATE ROLE alice
  WITH LOGIN                    -- connect করতে পারবে
  PASSWORD 'SecurePass123!'
  SUPERUSER                     -- সব কিছু করতে পারবে (be very careful!)
  CREATEDB                      -- নতুন database তৈরি করতে পারবে
  CREATEROLE                    -- নতুন role তৈরি করতে পারবে
  INHERIT                       -- granted role-এর permissions inherit করবে
  REPLICATION                   -- replication connection করতে পারবে
  BYPASSRLS                     -- Row-Level Security bypass করতে পারবে
  CONNECTION LIMIT 10           -- সর্বোচ্চ ১০টা concurrent connection
  VALID UNTIL '2025-12-31'      -- এই তারিখের পরে login reject
  IN ROLE readonly_role         -- তৈরির সময়ই member করো
  ADMIN readonly_role;          -- member + GRANT OPTION

-- Role modification:
ALTER ROLE alice PASSWORD 'NewSecurePass456!';
ALTER ROLE alice CONNECTION LIMIT 20;
ALTER ROLE alice VALID UNTIL 'infinity';  -- no expiry
ALTER ROLE alice NOLOGIN;  -- disable login
ALTER ROLE alice LOGIN;    -- re-enable

-- Role deletion:
DROP ROLE alice;
-- Error if alice owns objects or has active connections!

-- Safe drop:
REASSIGN OWNED BY alice TO postgres;  -- transfer objects to postgres
DROP OWNED BY alice;                   -- drop all alice's permissions
DROP ROLE alice;
```

---

### Role Hierarchy — Permission Inheritance

```
Role hierarchy এর real power:

                    superuser
                        │
                  ┌─────┴──────┐
               dba_admin    readonly_base
                  │               │
           ┌──────┤          ┌────┴─────┐
        app_admin app_write  app_read  reporting
           │          │          │         │
        [alice]  [bob,carol]  [dave]  [eve,frank]

Benefits:
  Permission once → many users benefit
  Role change → all members affected immediately
  Audit: "what can this role do?" instead of "what can this user do?"
```

```sql
-- Role hierarchy setup:

-- Base permission roles:
CREATE ROLE readonly_base;
GRANT CONNECT ON DATABASE mydb TO readonly_base;
GRANT USAGE ON SCHEMA public TO readonly_base;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_base;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO readonly_base;

CREATE ROLE readwrite_base;
GRANT readonly_base TO readwrite_base;  -- inherit readonly permissions
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite_base;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO readwrite_base;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT INSERT, UPDATE, DELETE ON TABLES TO readwrite_base;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT USAGE ON SEQUENCES TO readwrite_base;

-- Application-specific roles:
CREATE ROLE app_orders_writer;
GRANT readwrite_base TO app_orders_writer;
-- Extra: specific function execution
GRANT EXECUTE ON FUNCTION process_order(bigint, jsonb) TO app_orders_writer;

-- Analytics role:
CREATE ROLE analytics_reader;
GRANT readonly_base TO analytics_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA analytics TO analytics_reader;
ALTER DEFAULT PRIVILEGES IN SCHEMA analytics
  GRANT SELECT ON TABLES TO analytics_reader;

-- Actual users:
CREATE ROLE app_service WITH LOGIN PASSWORD 'app_pass';
GRANT app_orders_writer TO app_service;

CREATE ROLE analyst1 WITH LOGIN PASSWORD 'analyst_pass';
GRANT analytics_reader TO analyst1;

-- Check effective permissions:
-- What can app_service do?
SELECT has_table_privilege('app_service', 'orders', 'SELECT');  -- true (via chain)
SELECT has_table_privilege('app_service', 'orders', 'INSERT');  -- true
SELECT has_table_privilege('app_service', 'users', 'DELETE');   -- true (via readwrite_base)
```

---

### INHERIT vs SET ROLE

```sql
-- INHERIT (default): permissions automatically available
CREATE ROLE alice WITH LOGIN INHERIT;
GRANT readonly_base TO alice;
-- alice-এর session-এ readonly_base-এর permissions immediately active

-- NOINHERIT: must explicitly SET ROLE to use granted permissions
CREATE ROLE bob WITH LOGIN NOINHERIT;
GRANT dba_admin TO bob;

-- bob connects, tries to create table:
-- ERROR: permission denied (dba_admin permissions not active!)

-- bob must explicitly switch:
SET ROLE dba_admin;
-- Now dba_admin permissions active
CREATE TABLE ... ;  -- works

RESET ROLE;  -- back to bob's own permissions

-- Why NOINHERIT?
-- Principle of least privilege by default
-- High-privilege operations require explicit role switch
-- Prevents accidental privilege use
-- Useful for "break-glass" admin accounts

-- Check current role:
SELECT current_user;   -- login role (bob)
SELECT session_user;   -- login role (always bob, even after SET ROLE)
SELECT current_role;   -- current active role (dba_admin after SET ROLE)
```

---

### Privilege System — Complete Reference

**Object-level privileges:**

```sql
-- DATABASE:
GRANT CONNECT ON DATABASE mydb TO alice;
GRANT CREATE ON DATABASE mydb TO alice;     -- create schemas
GRANT TEMP ON DATABASE mydb TO alice;       -- create temp tables
-- CONNECT: connection allow করা
-- CREATE: new schema তৈরি করতে পারবে
-- TEMP: temporary table তৈরি করতে পারবে

-- SCHEMA:
GRANT USAGE ON SCHEMA public TO alice;      -- schema-এ objects দেখতে পারবে
GRANT CREATE ON SCHEMA public TO alice;     -- schema-এ objects তৈরি করতে পারবে
-- USAGE ছাড়া: schema-এ কোনো object access করা যাবে না
-- এমনকি table-এ SELECT permission থাকলেও!

-- TABLE:
GRANT SELECT ON orders TO alice;
GRANT INSERT, UPDATE, DELETE ON orders TO alice;
GRANT TRUNCATE ON orders TO alice;
GRANT REFERENCES ON orders TO alice;        -- FK reference করার অনুমতি
GRANT TRIGGER ON orders TO alice;           -- trigger তৈরি করতে পারবে
GRANT ALL ON orders TO alice;               -- সব privileges

-- Column-level (granular):
GRANT SELECT (id, name, email) ON users TO alice;  -- শুধু এই columns
GRANT UPDATE (email, phone) ON users TO alice;     -- শুধু এই columns update

-- SEQUENCE:
GRANT USAGE ON SEQUENCE orders_id_seq TO alice;   -- nextval, currval
GRANT SELECT ON SEQUENCE orders_id_seq TO alice;  -- currval only
GRANT UPDATE ON SEQUENCE orders_id_seq TO alice;  -- setval

-- FUNCTION:
GRANT EXECUTE ON FUNCTION calculate_tax(numeric) TO alice;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public TO alice;

-- TYPE:
GRANT USAGE ON TYPE order_status TO alice;

-- FOREIGN DATA WRAPPER / SERVER:
GRANT USAGE ON FOREIGN DATA WRAPPER postgres_fdw TO alice;
GRANT USAGE ON FOREIGN SERVER remote_db TO alice;
```

**GRANT OPTION — privilege delegation:**

```sql
-- Alice can grant SELECT to others:
GRANT SELECT ON orders TO alice WITH GRANT OPTION;

-- Alice connects and:
GRANT SELECT ON orders TO bob;  -- alice can grant to bob!

-- Revoke with CASCADE:
REVOKE SELECT ON orders FROM alice CASCADE;
-- Also revokes permissions alice granted to bob!

-- GRANT OPTION FOR:
REVOKE GRANT OPTION FOR SELECT ON orders FROM alice;
-- alice keeps SELECT, but can't grant to others anymore
```

---

### DEFAULT PRIVILEGES — Future Objects

```sql
-- Problem: granting on current objects is not enough
-- New table তৈরি হলে → automatically grant দরকার

-- Default privileges for future objects:
ALTER DEFAULT PRIVILEGES
  IN SCHEMA public
  GRANT SELECT ON TABLES TO readonly_base;
-- এখন থেকে public schema-এ তৈরি যেকোনো table-এ readonly_base automatically SELECT পাবে

-- Per-role default privileges (only for objects created by this role):
ALTER DEFAULT PRIVILEGES FOR ROLE app_service
  IN SCHEMA public
  GRANT SELECT ON TABLES TO reporting_role;
-- Only tables created by app_service → reporting_role gets SELECT

-- Check current default privileges:
SELECT defaclrole::regrole, defaclnamespace::regnamespace,
  defaclobjtype, pg_catalog.array_to_string(defaclacl, ',')
FROM pg_default_acl;

-- Reset default privileges:
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  REVOKE SELECT ON TABLES FROM readonly_base;
```

---

### pg_hba.conf — Authentication Configuration

```
pg_hba.conf = PostgreSQL Host-Based Authentication
প্রতিটা connection attempt এই file-এর rules দিয়ে evaluate হয়।
Top-to-bottom, first match wins।

Format:
TYPE  DATABASE  USER    ADDRESS          METHOD        OPTIONS

Types:
  local    → Unix domain socket (no network, localhost only)
  host     → TCP/IP (SSL বা non-SSL)
  hostssl  → TCP/IP, SSL required
  hostnossl → TCP/IP, SSL rejected
  hostgssenc → TCP/IP with GSSAPI encryption
```

**Production-grade pg_hba.conf:**

```
# pg_hba.conf

# Type     Database    User            Address             Method      Options

# Local connections for maintenance:
local      all         postgres                            peer
local      all         all                                 md5

# Application server (specific IP range):
hostssl    mydb        app_service     10.0.1.0/24         scram-sha-256
hostssl    mydb        app_readonly    10.0.1.0/24         scram-sha-256

# Analytics team (VPN range):
hostssl    mydb        analyst1        10.0.2.0/24         scram-sha-256
hostssl    mydb        analyst2        10.0.2.0/24         scram-sha-256

# Replication:
hostssl    replication replicator      10.0.3.0/24         scram-sha-256

# DBA access (jump server only):
hostssl    all         dba_admin       10.0.4.100/32       cert  clientcert=verify-full

# Block everything else:
host       all         all             0.0.0.0/0           reject
host       all         all             ::/0                reject
```

**Authentication methods:**

```
trust:
  No password required.
  NEVER use in production.
  OK only: localhost development, isolated test environment.

peer:
  OS user = PostgreSQL user (Unix socket only).
  postgres OS user → postgres PG role: automatic.
  Secure for local admin access.

md5:
  Password hash (MD5). Less secure than scram.
  Client sends MD5 hash of password.
  Vulnerable to offline dictionary attacks on intercepted hash.
  Use scram-sha-256 instead.

scram-sha-256:
  Salted Challenge Response Authentication Mechanism.
  Password never sent over wire (challenge-response).
  Resistant to replay attacks, dictionary attacks.
  Requires PostgreSQL 10+ and compatible client.
  RECOMMENDED for all new deployments.

cert:
  Client SSL certificate authentication.
  Client must present valid cert signed by trusted CA.
  Most secure for automated services.
  clientcert=verify-full: hostname matches CN in cert.

gss / sspi:
  Kerberos / Active Directory integration.
  Enterprise environments with existing Kerberos infrastructure.

ldap:
  LDAP server authentication.
  Enterprise LDAP/Active Directory.
  PASSWORD still sent to PostgreSQL server (not as secure as cert).

radius:
  RADIUS server authentication.
  Enterprise MFA scenarios.

reject:
  Always deny. Used to block specific users/addresses before a broader allow.
```

---

### Password Security

```sql
-- Password hashing (PostgreSQL 10+):
-- scram-sha-256: stronger, recommended
-- md5: legacy, weaker

-- Check current password encryption:
SHOW password_encryption;
-- 'scram-sha-256' (recommended)
-- 'md5' (legacy)

-- Set password encryption:
ALTER SYSTEM SET password_encryption = 'scram-sha-256';
SELECT pg_reload_conf();

-- Change password:
ALTER ROLE alice PASSWORD 'NewSecurePassword123!';
-- Password stored as: SCRAM-SHA-256$4096:<salt>:<hash>
-- Never stored in plaintext

-- Check password hash in pg_authid (superuser only):
SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'alice';
-- SCRAM-SHA-256$4096:... (hashed, safe)

-- Password in pg_hba.conf connection string:
-- BAD: psql "postgresql://alice:password@host/db"
--   Password in shell history, process list visible!
-- GOOD: .pgpass file or PGPASSWORD environment variable
--   .pgpass: chmod 0600
--   hostname:port:database:username:password

-- Password expiry:
ALTER ROLE alice VALID UNTIL '2025-06-30 00:00:00+06';
ALTER ROLE alice VALID UNTIL 'infinity';  -- no expiry

-- Check expiry:
SELECT rolname, rolvaliduntil FROM pg_roles WHERE rolname = 'alice';
```

---

### Privilege Inspection — Audit Queries

```sql
-- Who has what on which table:
SELECT
  grantee,
  table_schema,
  table_name,
  string_agg(privilege_type, ', ' ORDER BY privilege_type) AS privileges
FROM information_schema.role_table_grants
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
  AND table_name = 'orders'
GROUP BY grantee, table_schema, table_name
ORDER BY grantee;

-- All tables a role can access:
SELECT
  table_schema,
  table_name,
  string_agg(privilege_type, ', ' ORDER BY privilege_type) AS privileges
FROM information_schema.role_table_grants
WHERE grantee = 'alice'
   OR grantee IN (
     SELECT rolname FROM pg_roles
     WHERE pg_has_role('alice', oid, 'member')
   )
GROUP BY table_schema, table_name
ORDER BY table_schema, table_name;

-- Role membership tree:
WITH RECURSIVE role_tree AS (
  SELECT
    m.roleid AS role_oid,
    m.member AS member_oid,
    r.rolname AS role_name,
    mr.rolname AS member_name,
    0 AS depth
  FROM pg_auth_members m
  JOIN pg_roles r ON r.oid = m.roleid
  JOIN pg_roles mr ON mr.oid = m.member

  UNION ALL

  SELECT
    m.roleid,
    rt.member_oid,
    r.rolname,
    rt.member_name,
    rt.depth + 1
  FROM pg_auth_members m
  JOIN pg_roles r ON r.oid = m.roleid
  JOIN role_tree rt ON rt.role_oid = m.member
)
SELECT DISTINCT role_name, member_name, depth
FROM role_tree
ORDER BY depth, role_name, member_name;

-- Objects without owner:
SELECT relname, relkind
FROM pg_class
WHERE relnamespace = 'public'::regnamespace
  AND relowner = 0;  -- should never happen

-- Superusers (potential security risk):
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolbypassrls
FROM pg_roles
WHERE rolsuper = true;
-- Should be minimal! Ideally just 'postgres'
```

---

### Least Privilege — Production Setup

```sql
-- Production role setup (complete example):

-- Step 1: Revoke dangerous defaults
REVOKE ALL ON DATABASE mydb FROM PUBLIC;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;

-- Step 2: Base roles
CREATE ROLE db_readonly;
CREATE ROLE db_readwrite;
CREATE ROLE db_admin;

-- Step 3: Grant to base roles
-- readonly:
GRANT CONNECT ON DATABASE mydb TO db_readonly;
GRANT USAGE ON SCHEMA public TO db_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO db_readonly;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO db_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO db_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON SEQUENCES TO db_readonly;

-- readwrite (inherits readonly):
GRANT db_readonly TO db_readwrite;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO db_readwrite;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO db_readwrite;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA public TO db_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT INSERT, UPDATE, DELETE ON TABLES TO db_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT USAGE ON SEQUENCES TO db_readwrite;

-- admin (DDL, no SUPERUSER):
GRANT db_readwrite TO db_admin;
GRANT CREATE ON SCHEMA public TO db_admin;
GRANT CREATE ON DATABASE mydb TO db_admin;

-- Step 4: Application users
CREATE ROLE app_backend WITH LOGIN PASSWORD 'SecureAppPass!'
  CONNECTION LIMIT 50;
GRANT db_readwrite TO app_backend;

CREATE ROLE app_reporting WITH LOGIN PASSWORD 'SecureReportPass!'
  CONNECTION LIMIT 20;
GRANT db_readonly TO app_reporting;

-- Step 5: Human users
CREATE ROLE dev_alice WITH LOGIN PASSWORD 'AlicePass!'
  VALID UNTIL '2025-12-31';
GRANT db_readwrite TO dev_alice;

-- Step 6: DBA (no superuser in daily use)
CREATE ROLE dba_bob WITH LOGIN NOINHERIT PASSWORD 'DbaPass!';
GRANT db_admin TO dba_bob;
-- bob must SET ROLE db_admin to use admin privileges
-- Prevents accidental DDL in regular sessions
```

---

## Topic 22 — Row-Level Security (RLS)

### Big Picture: Same Table, Different Views

RLS হলো একটা automatic WHERE clause যেটা PostgreSQL automatically প্রতিটা query-তে apply করে। Application code-এ filter ভুলে গেলেও database enforce করে।

```
Without RLS:
  Application: "SELECT * FROM orders WHERE user_id = 42"
  Bug: "SELECT * FROM orders" → all users' orders exposed!

With RLS:
  Policy: "user can only see their own orders"
  "SELECT * FROM orders" → automatically adds "AND user_id = current_user_id"
  Bug: "SELECT * FROM orders" → only user 42's orders, even without WHERE!
```

---

### RLS Internal Mechanism

```
Query: SELECT * FROM orders;

Without RLS:
  Parser → Analyzer → Planner → Executor
  Executor: full table scan

With RLS enabled, policy: user_id = current_user_id():
  Parser → Analyzer → Rewriter → Planner → Executor
  Rewriter: inject policy as additional WHERE clause
  Effective query: SELECT * FROM orders WHERE user_id = current_user_id();
  Executor: filtered scan (index on user_id if exists!)

Multiple policies: OR-ed together for permissive policies
                   AND-ed for restrictive policies
```

```sql
-- Enable RLS on table:
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
-- Now: if NO policies defined → NO rows visible to anyone (except superuser/owner)
-- This is the safe default: deny-all until policies added

-- Owner bypass:
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
-- Table owner still bypasses by default!
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
-- Now owner ALSO follows policies (important for security!)

-- Disable RLS:
ALTER TABLE orders DISABLE ROW LEVEL SECURITY;
-- No policies enforced, all rows visible
```

---

### Policy Types — Permissive vs Restrictive

```sql
-- PERMISSIVE (default): policies OR-ed together
-- "If ANY permissive policy passes → row visible"
CREATE POLICY user_sees_own_orders ON orders
  AS PERMISSIVE          -- default
  FOR SELECT
  TO PUBLIC              -- applies to all roles
  USING (user_id = current_setting('app.current_user_id')::bigint);

CREATE POLICY admin_sees_all ON orders
  AS PERMISSIVE
  FOR SELECT
  TO admin_role
  USING (true);          -- admin sees everything

-- Result: user sees own orders (first policy) OR admin sees all (second policy)
-- Two permissive policies = OR combination

-- RESTRICTIVE: AND-ed with other policies
-- "ALL restrictive policies must pass"
CREATE POLICY only_active_tenant ON orders
  AS RESTRICTIVE
  FOR ALL
  TO app_role
  USING (tenant_id = current_setting('app.tenant_id')::bigint);
-- Even if user is admin, they can only see current tenant's data!
-- Restrictive overrides permissive!

-- Policy naming convention:
-- {table}_{operation}_{description}
-- orders_select_own_user
-- orders_all_admin_bypass
```

---

### USING vs WITH CHECK

```sql
-- USING: filter for READ operations (SELECT, UPDATE's old rows, DELETE's target)
-- WITH CHECK: filter for WRITE operations (INSERT's new row, UPDATE's new row)

CREATE POLICY orders_user_policy ON orders
  FOR ALL
  TO app_role
  USING (user_id = current_setting('app.current_user_id')::bigint)
  WITH CHECK (user_id = current_setting('app.current_user_id')::bigint);

-- SELECT: USING applies → only own orders visible
-- INSERT: WITH CHECK applies → must insert with own user_id
-- UPDATE: USING (which rows can be updated) + WITH CHECK (new values valid?)
-- DELETE: USING applies → can only delete own orders

-- USING only (SELECT):
CREATE POLICY read_own ON orders
  FOR SELECT
  USING (user_id = current_setting('app.current_user_id')::bigint);

-- WITH CHECK only (INSERT):
CREATE POLICY write_own ON orders
  FOR INSERT
  WITH CHECK (user_id = current_setting('app.current_user_id')::bigint);

-- Separate policies per operation (granular):
CREATE POLICY orders_select ON orders
  FOR SELECT USING (user_id = current_user_id());

CREATE POLICY orders_insert ON orders
  FOR INSERT WITH CHECK (user_id = current_user_id());

CREATE POLICY orders_update ON orders
  FOR UPDATE
  USING (user_id = current_user_id())           -- which rows can be updated
  WITH CHECK (user_id = current_user_id());     -- new values must be own

CREATE POLICY orders_delete ON orders
  FOR DELETE USING (user_id = current_user_id() AND status = 'pending');
  -- Can only delete own PENDING orders (not completed/shipped!)
```

---

### Multi-tenant RLS — Complete Implementation

```sql
-- Schema: shared schema with tenant_id

-- Tables:
CREATE TABLE orders (
  id bigint GENERATED ALWAYS AS IDENTITY,
  tenant_id bigint NOT NULL,
  user_id bigint NOT NULL,
  amount numeric(12,2) NOT NULL,
  status text NOT NULL DEFAULT 'pending',
  created_at timestamptz DEFAULT now(),
  PRIMARY KEY (id, tenant_id)
);

CREATE TABLE users (
  id bigint GENERATED ALWAYS AS IDENTITY,
  tenant_id bigint NOT NULL,
  name text NOT NULL,
  email text NOT NULL,
  role text NOT NULL DEFAULT 'customer',
  PRIMARY KEY (id, tenant_id)
);

-- Enable RLS:
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders FORCE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE users FORCE ROW LEVEL SECURITY;

-- Helper function (SECURITY DEFINER for performance):
CREATE OR REPLACE FUNCTION current_tenant_id()
RETURNS bigint
LANGUAGE sql STABLE SECURITY DEFINER
SET search_path = public, pg_catalog AS $$
  SELECT nullif(current_setting('app.tenant_id', true), '')::bigint;
$$;

CREATE OR REPLACE FUNCTION current_user_id()
RETURNS bigint
LANGUAGE sql STABLE SECURITY DEFINER
SET search_path = public, pg_catalog AS $$
  SELECT nullif(current_setting('app.user_id', true), '')::bigint;
$$;

CREATE OR REPLACE FUNCTION current_user_role()
RETURNS text
LANGUAGE sql STABLE SECURITY DEFINER
SET search_path = public, pg_catalog AS $$
  SELECT current_setting('app.user_role', true);
$$;

-- Application role:
CREATE ROLE app_user WITH LOGIN PASSWORD 'apppass';
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_user;

-- Tenant isolation (hardest boundary):
CREATE POLICY tenant_isolation_orders ON orders
  AS RESTRICTIVE
  FOR ALL
  TO app_user
  USING (tenant_id = current_tenant_id())
  WITH CHECK (tenant_id = current_tenant_id());

CREATE POLICY tenant_isolation_users ON users
  AS RESTRICTIVE
  FOR ALL
  TO app_user
  USING (tenant_id = current_tenant_id())
  WITH CHECK (tenant_id = current_tenant_id());

-- User-level access within tenant:
CREATE POLICY user_own_orders ON orders
  AS PERMISSIVE
  FOR ALL
  TO app_user
  USING (
    user_id = current_user_id()
    OR current_user_role() IN ('admin', 'manager')  -- admins see all tenant orders
  )
  WITH CHECK (
    user_id = current_user_id()
    OR current_user_role() = 'admin'
  );

-- Application usage:
-- Connection established, then:
SET app.tenant_id = '101';
SET app.user_id = '42';
SET app.user_role = 'customer';

-- Now all queries automatically filtered:
SELECT * FROM orders;
-- Effective: WHERE tenant_id = 101 AND user_id = 42

-- Admin user:
SET app.tenant_id = '101';
SET app.user_id = '99';
SET app.user_role = 'admin';

SELECT * FROM orders;
-- Effective: WHERE tenant_id = 101 (no user_id filter for admin!)
```

---

### RLS and Performance

```sql
-- RLS adds WHERE clause → query planning affected

-- Good: index on policy column
CREATE INDEX ON orders (tenant_id, user_id);
-- RLS: WHERE tenant_id = 101 AND user_id = 42 → index used efficiently

-- Bad: no index on policy column
-- RLS adds WHERE tenant_id = 101 → Seq Scan on orders!
-- With 10M rows: catastrophically slow

-- RLS policy complexity matters:
-- Simple: USING (tenant_id = current_tenant_id())
-- → Fast: single column equality, index friendly

-- Complex: USING (EXISTS (SELECT 1 FROM user_permissions WHERE ...))
-- → Slow: subquery per row!
-- → Alternative: denormalize permission into main table

-- Helper function vs inline:
-- Inline:
USING (tenant_id = nullif(current_setting('app.tenant_id', true), '')::bigint)
-- Called once per query (STABLE), but inline text

-- Helper function (better for readability, same performance with STABLE):
USING (tenant_id = current_tenant_id())  -- function is STABLE
-- PostgreSQL inlines STABLE function in policies → single evaluation

-- Check RLS policy impact:
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE amount > 1000;
-- With RLS: plan shows additional filter from policy
-- "Filter: ((tenant_id = current_tenant_id()) AND (amount > '1000'::numeric))"
```

---

### RLS Bypass — When and Why

```sql
-- BYPASSRLS attribute:
CREATE ROLE super_admin WITH LOGIN BYPASSRLS;
-- OR:
ALTER ROLE super_admin BYPASSRLS;

-- BYPASSRLS role: ignores ALL RLS policies
-- Use: DBA maintenance, data migration, audit queries
-- Never give to application service accounts!

-- Superuser automatically bypasses RLS:
-- postgres superuser: BYPASSRLS implicit

-- Check if current user bypasses:
SELECT rolbypassrls FROM pg_roles WHERE rolname = current_user;

-- Table owner bypass (unless FORCE RLS):
-- FORCE RLS applied: owner also follows policies
-- Without FORCE: owner bypasses

-- Temporarily disable RLS for maintenance:
-- Option 1: SET ROLE to superuser
SET ROLE postgres;
SELECT * FROM orders;  -- sees all
RESET ROLE;

-- Option 2: Use superuser connection for maintenance
-- Option 3: BYPASSRLS role for specific maintenance tasks

-- Security audit: who can bypass RLS?
SELECT rolname, rolsuper, rolbypassrls
FROM pg_roles
WHERE rolsuper OR rolbypassrls
ORDER BY rolname;
-- Minimize this list!
```

---

### Column-Level Security

```sql
-- Column-level: GRANT specific columns only
GRANT SELECT (id, name, email, created_at) ON users TO api_readonly;
-- api_readonly cannot see: password_hash, ssn, date_of_birth, salary

-- Test:
SET ROLE api_readonly;
SELECT id, name, email FROM users;  -- OK
SELECT password_hash FROM users;    -- ERROR: permission denied for column "password_hash"
SELECT * FROM users;                -- ERROR: permission denied for column "password_hash"
RESET ROLE;

-- Column-level UPDATE:
GRANT UPDATE (email, phone) ON users TO user_profile_updater;
-- Can only update these two columns
UPDATE users SET email = 'new@email.com' WHERE id = 42;  -- OK
UPDATE users SET salary = 100000 WHERE id = 42;  -- ERROR!

-- Combined with RLS:
-- RLS: rows filtered by tenant
-- Column grant: specific columns visible
-- Double protection!

-- View as alternative to column-level grant:
CREATE VIEW users_safe AS
SELECT id, name, email, created_at, active
FROM users;
-- Simpler than column grants, easier to reason about
-- View definition clearly shows what's visible
```

---

## Topic 23 — Audit Logging

### Big Picture: Who Did What, When?

Audit logging হলো compliance, security, এবং debugging-এর জন্য। কিন্তু naive implementation করলে performance nightmare।

```
Audit-এর তিনটা layer:

1. Connection-level audit (pg_hba.conf + log_connections):
   "কে connect করলো, কখন, কোথা থেকে?"

2. Statement-level audit (log_statement, pgAudit):
   "কে কোন SQL চালালো?"

3. Data-level audit (trigger-based):
   "কোন row-এর কী value ছিলো, কী হলো, কে বদলালো?"
```

---

### PostgreSQL Built-in Logging

```
postgresql.conf logging parameters:

log_connections = on
  Every new connection logged:
  "LOG: connection received: host=10.0.1.50 port=52341"
  "LOG: connection authorized: user=app_service database=mydb"

log_disconnections = on
  Connection end logged with session duration:
  "LOG: disconnection: session time: 0:00:05.234 user=app_service database=mydb"

log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
  %t: timestamp
  %p: PID
  %l: log line number in session
  %u: username
  %d: database name
  %a: application_name
  %h: client hostname/IP

log_statement = 'all'     -- every statement
log_statement = 'ddl'     -- CREATE, ALTER, DROP, TRUNCATE
log_statement = 'mod'     -- DDL + INSERT, UPDATE, DELETE
log_statement = 'none'    -- nothing (use pgAudit instead)

log_min_duration_statement = 1000  -- statements taking > 1 second
log_duration = on          -- log duration of ALL statements
log_error_verbosity = 'verbose'  -- detailed error messages

log_checkpoints = on       -- checkpoint start/end
log_lock_waits = on        -- lock wait > deadlock_timeout
log_temp_files = 0         -- all temp files (0 = all, N = > N bytes)
log_autovacuum_min_duration = 0  -- all autovacuum operations
```

**Log destination:**

```
log_destination = 'stderr,csvlog,jsonlog'  -- PG 15+
logging_collector = on   -- enable log file collection
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d    -- rotate daily
log_rotation_size = 100MB -- or when > 100MB
log_file_mode = 0600     -- permissions

# For log analysis:
# csvlog: structured, parseable
# jsonlog: JSON format (PG 15+), easiest to process
```

---

### pgAudit Extension — Production-grade SQL Auditing

```sql
-- Installation:
-- apt install postgresql-17-pgaudit
-- or: compile from source

-- Enable in postgresql.conf:
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'read,write,ddl,role,function'
-- read: SELECT, COPY FROM
-- write: INSERT, UPDATE, DELETE, TRUNCATE, COPY TO
-- ddl: CREATE, ALTER, DROP, COMMENT, etc.
-- role: GRANT, REVOKE, CREATE/ALTER/DROP ROLE
-- function: Function calls
-- misc: FETCH, CHECKPOINT
-- misc_set: SET command
-- all: everything

-- Per-role audit (more granular):
pgaudit.log_catalog = off   -- don't audit system catalog queries
pgaudit.log_parameter = on  -- log bind parameters (for prepared statements)
pgaudit.log_relation = on   -- log each relation referenced
pgaudit.log_rows = off      -- don't log actual row data (pgaudit handles this)
pgaudit.log_statement_once = off  -- log statement for each row (or once)

-- Reload:
SELECT pg_reload_conf();

-- pgAudit output format:
-- AUDIT: SESSION,1,1,READ,SELECT,TABLE,public.orders,SELECT * FROM orders,<none>
-- AUDIT: SESSION,2,1,WRITE,INSERT,TABLE,public.orders,...

-- Object-level audit (per table):
CREATE EXTENSION pgaudit;
SELECT pgaudit.set_config('log', 'read,write', false);

-- Audit specific table via role:
CREATE ROLE orders_audit_role;
SELECT pgaudit.set_config('role', 'orders_audit_role', false);
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO orders_audit_role;
-- Now: all access to orders via orders_audit_role is audited
-- Application role inherits orders_audit_role → audited!
```

---

### Trigger-based Data Audit — Complete System

```sql
-- Complete audit system:

-- Audit table (immutable log):
CREATE TABLE audit_trail (
  id bigserial PRIMARY KEY,
  -- When
  event_time timestamptz NOT NULL DEFAULT clock_timestamp(),
  -- Who (DB level)
  db_user text NOT NULL DEFAULT current_user,
  app_name text DEFAULT current_setting('application_name', true),
  client_addr inet DEFAULT inet_client_addr(),
  -- Who (Application level)
  app_user_id bigint,
  app_user_name text,
  -- What
  schema_name text NOT NULL,
  table_name text NOT NULL,
  operation text NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
  -- Data
  row_id bigint,
  old_values jsonb,
  new_values jsonb,
  changed_columns text[],
  -- Session
  transaction_id bigint DEFAULT txid_current(),
  statement_id int DEFAULT txid_current_if_assigned()::text::bigint
);

-- Partition audit trail by month (grows fast!):
-- Actually create as partitioned:
CREATE TABLE audit_trail (
  id bigserial,
  event_time timestamptz NOT NULL DEFAULT clock_timestamp(),
  db_user text NOT NULL DEFAULT current_user,
  app_name text DEFAULT current_setting('application_name', true),
  client_addr inet DEFAULT inet_client_addr(),
  app_user_id bigint,
  app_user_name text,
  schema_name text NOT NULL,
  table_name text NOT NULL,
  operation text NOT NULL,
  row_id bigint,
  old_values jsonb,
  new_values jsonb,
  changed_columns text[],
  transaction_id bigint DEFAULT txid_current()
) PARTITION BY RANGE (event_time);

-- Monthly partitions:
CREATE TABLE audit_trail_2024_01 PARTITION OF audit_trail
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
-- ... etc

-- Indexes on audit table:
CREATE INDEX ON audit_trail (table_name, event_time);
CREATE INDEX ON audit_trail (app_user_id, event_time);
CREATE INDEX ON audit_trail (row_id, table_name, event_time);
CREATE INDEX ON audit_trail (transaction_id);

-- Generic audit function:
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS trigger
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public, pg_catalog AS $$
DECLARE
  v_old jsonb;
  v_new jsonb;
  v_changed_cols text[];
  v_row_id bigint;
  v_app_user_id bigint;
  v_app_user_name text;
BEGIN
  -- Get application-level user info (from session variables)
  v_app_user_id := nullif(current_setting('app.user_id', true), '')::bigint;
  v_app_user_name := current_setting('app.user_name', true);

  -- Build old/new JSON and find changes
  CASE TG_OP
    WHEN 'INSERT' THEN
      v_old := NULL;
      v_new := to_jsonb(NEW);
      v_row_id := NEW.id;
      v_changed_cols := NULL;

    WHEN 'UPDATE' THEN
      v_old := to_jsonb(OLD);
      v_new := to_jsonb(NEW);
      v_row_id := NEW.id;

      -- Only store changed columns (minimize storage)
      SELECT array_agg(key ORDER BY key)
      INTO v_changed_cols
      FROM jsonb_each(v_new) AS n(key, val)
      WHERE v_old->key IS DISTINCT FROM val
        AND key NOT IN ('updated_at', 'version');  -- exclude noisy system columns

      -- If only ignored columns changed, skip audit
      IF v_changed_cols IS NULL OR array_length(v_changed_cols, 1) = 0 THEN
        RETURN NEW;
      END IF;

      -- Store only changed column values (not full row)
      v_old := v_old - ARRAY(
        SELECT key FROM jsonb_object_keys(v_old) AS key
        WHERE NOT (key = ANY(v_changed_cols))
      );
      v_new := v_new - ARRAY(
        SELECT key FROM jsonb_object_keys(v_new) AS key
        WHERE NOT (key = ANY(v_changed_cols))
      );

    WHEN 'DELETE' THEN
      v_old := to_jsonb(OLD);
      v_new := NULL;
      v_row_id := OLD.id;
      v_changed_cols := NULL;
  END CASE;

  -- Insert audit record
  INSERT INTO audit_trail (
    schema_name, table_name, operation,
    row_id, old_values, new_values, changed_columns,
    app_user_id, app_user_name
  ) VALUES (
    TG_TABLE_SCHEMA, TG_TABLE_NAME, TG_OP,
    v_row_id, v_old, v_new, v_changed_cols,
    v_app_user_id, v_app_user_name
  );

  IF TG_OP = 'DELETE' THEN RETURN OLD;
  ELSE RETURN NEW;
  END IF;

EXCEPTION WHEN OTHERS THEN
  -- Don't let audit failure break the main operation!
  RAISE WARNING 'Audit failed for %.%: %', TG_TABLE_SCHEMA, TG_TABLE_NAME, SQLERRM;
  IF TG_OP = 'DELETE' THEN RETURN OLD;
  ELSE RETURN NEW;
  END IF;
END;
$$;

-- Apply audit to tables:
CREATE TRIGGER audit_orders
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

CREATE TRIGGER audit_users
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

-- Query audit trail:
-- Who changed order 42?
SELECT
  event_time,
  db_user,
  app_user_name,
  operation,
  changed_columns,
  old_values,
  new_values
FROM audit_trail
WHERE table_name = 'orders'
  AND row_id = 42
ORDER BY event_time;

-- All changes by user 99 in last hour:
SELECT *
FROM audit_trail
WHERE app_user_id = 99
  AND event_time > now() - interval '1 hour'
ORDER BY event_time DESC;

-- All changes in a transaction:
SELECT *
FROM audit_trail
WHERE transaction_id = 12345
ORDER BY id;
```

---

### Audit Performance Considerations

```sql
-- Audit trigger overhead:
-- Every DML → trigger → INSERT into audit_trail
-- For 10K orders/sec: 10K audit inserts/sec!

-- Mitigations:

-- 1. Only audit what's needed
-- Don't audit every column change, only sensitive ones:
CREATE TRIGGER audit_sensitive_fields
AFTER UPDATE OF amount, status, user_id ON orders
FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
-- Only fires when amount, status, or user_id changes
-- Other updates: no audit overhead

-- 2. Async audit (via LISTEN/NOTIFY + worker)
CREATE OR REPLACE FUNCTION audit_async()
RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  PERFORM pg_notify(
    'audit_channel',
    json_build_object(
      'table', TG_TABLE_NAME,
      'op', TG_OP,
      'id', CASE WHEN TG_OP = 'DELETE' THEN OLD.id ELSE NEW.id END,
      'ts', clock_timestamp()
    )::text
  );
  IF TG_OP = 'DELETE' THEN RETURN OLD; ELSE RETURN NEW; END IF;
END;
$$;
-- Worker listens on 'audit_channel', writes to audit store
-- Main transaction doesn't wait for audit write!
-- Trade-off: if worker dies, some audit events lost

-- 3. Partitioned audit table (mentioned above)
-- Old partitions: archive/compress
-- Current partition: small, fast inserts

-- 4. UNLOGGED audit table (controversial)
-- Ultra-fast inserts (no WAL)
-- But: data lost on crash!
-- OK if: audit data also in application logs (dual write)

-- 5. Batch audit (FOR EACH STATEMENT instead of ROW)
-- Less granular but much less overhead for bulk ops
```

---

### Audit Immutability — Prevent Tampering

```sql
-- Audit trail must be tamper-proof!
-- But if DBA can DELETE from audit_trail, it's not tamper-proof

-- Option 1: Append-only with trigger:
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS trigger
LANGUAGE plpgsql AS $$
BEGIN
  IF TG_OP = 'DELETE' OR TG_OP = 'UPDATE' THEN
    RAISE EXCEPTION 'Audit trail records cannot be modified or deleted'
      USING ERRCODE = 'insufficient_privilege';
  END IF;
  RETURN NEW;
END;
$$;

CREATE TRIGGER audit_trail_immutable
BEFORE UPDATE OR DELETE ON audit_trail
FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();

-- But: DBA can DROP the trigger!
-- Better: revoke all DML on audit_trail from everyone including DBA

-- Option 2: Separate database / server for audit
-- Audit records shipped to read-only audit server
-- DBA of main DB cannot touch audit server

-- Option 3: External audit log (S3, SIEM)
-- pg_audit → external SIEM (Splunk, ELK)
-- SIEM has separate access control
-- Main DB DBA has no write access to SIEM

-- Option 4: PostgreSQL logical replication to audit DB
-- Audit DB: read-only replica of audit_trail table
-- Main DB DBA cannot modify replica!

-- Compliance note:
-- For SOX, PCI-DSS, HIPAA: external immutable audit log required
-- Trigger-based + same DB: insufficient for strict compliance
-- Use pgAudit → external SIEM integration
```

---

### Block 5 Summary — Dots Connected

```
Topic 21 (Users, Roles, Privileges)
    │
    ├──► Topic 6 (Schema): Schema-level USAGE + CREATE permissions
    ├──► Topic 17 (Functions): SECURITY DEFINER function owner's privileges
    ├──► Topic 22 (RLS): Role membership determines which RLS policies apply
    └──► Topic 26 (Replication): Replication role requires REPLICATION attribute

Topic 22 (Row-Level Security)
    │
    ├──► Topic 6 (Multi-tenancy): RLS = shared schema tenant isolation mechanism
    ├──► Topic 11 (Index): Policy column needs index for performance
    ├──► Topic 12 (Query Execution): RLS policy injected at rewriter stage
    └──► Topic 17 (Functions): STABLE helper functions used in policies

Topic 23 (Audit Logging)
    │
    ├──► Topic 18 (Triggers): Audit trigger pattern
    ├──► Topic 20 (Partition): Audit trail partitioned by date for manageability
    ├──► Topic 21 (Roles): Audit captures db_user + app_user
    └──► Topic 28 (Monitoring): pg_stat_activity + audit_trail = complete picture
```
# Senior DBA Master Guide — Block 6
## Backup, Recovery & High Availability

---

## Topic 24 — Backup Strategies: Full, Incremental, Differential

### Big Picture: Backup-এর Philosophy

```
Backup একটা insurance policy।
কিন্তু insurance-এর মতোই:
  - কখনো দরকার না হলেও maintain করতে হয়
  - দরকার হওয়ার সময় ঠিকঠাক কাজ না করলে catastrophic
  - Test না করা backup = no backup

Senior DBA-র নিয়ম:
  "Backup নয়, RESTORE test করো।"
  মাসে একবার actual restore করো — শুধু তখনই জানা যাবে backup কাজ করে।

RTO vs RPO:
  RPO (Recovery Point Objective):
    "কতটুকু data হারানো acceptable?"
    RPO = 0: zero data loss (synchronous replication দরকার)
    RPO = 1 hour: last 1 hour-এর data হারাতে পারো
    RPO = 24 hours: daily backup যথেষ্ট

  RTO (Recovery Time Objective):
    "কত সময়ের মধ্যে system চালু করতে হবে?"
    RTO = 15 minutes: standby/failover দরকার
    RTO = 4 hours: PITR restore acceptable
    RTO = 24 hours: cold backup restore OK
```

---

### Physical vs Logical Backup — Core Difference

```
Physical Backup:
  File system level copy ($PGDATA directory contents)
  Binary format — human unreadable
  Consistent snapshot of database files

  Tool: pg_basebackup
  Restore: copy files back, start PostgreSQL

Logical Backup:
  SQL statements (CREATE TABLE, INSERT, etc.)
  Human readable text (or custom binary format)
  Selective export (table, schema, database)

  Tool: pg_dump, pg_dumpall
  Restore: execute SQL statements via psql/pg_restore

Physical বনাম Logical তুলনা:
┌──────────────────────┬────────────────────────┬──────────────────────┐
│ Characteristic       │ Physical                │ Logical              │
├──────────────────────┼────────────────────────┼──────────────────────┤
│ Speed (backup)       │ Fast (file copy)        │ Slow (query+format)  │
│ Speed (restore)      │ Fast (file copy)        │ Slow (SQL execution) │
│ Size                 │ = database size         │ ≤ database size      │
│ Cross-version        │ Same major version only │ Across versions ✓    │
│ Cross-platform       │ Same OS/arch typically  │ Cross-platform ✓     │
│ PITR possible        │ Yes (with WAL)          │ No                   │
│ Partial restore      │ Difficult               │ Easy (per table)     │
│ Bloat included       │ Yes (dead tuples)       │ No (only live data)  │
│ Indexes included     │ Yes                     │ DDL recreates them   │
│ Online backup        │ Yes (hot backup)        │ Yes (snapshot)       │
│ Consistency          │ Guaranteed with WAL     │ Transaction snapshot │
└──────────────────────┴────────────────────────┴──────────────────────┘
```

---

### pg_dump — Logical Backup Deep Dive

**How pg_dump works internally:**

```
pg_dump process:
1. Connect to database
2. SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
   SET TRANSACTION SNAPSHOT '...'  (export current snapshot)
   → Consistent point-in-time view of database
   → Other transactions continue! Hot backup!

3. Query pg_catalog for object definitions:
   Tables, views, functions, sequences, constraints, indexes, triggers...

4. For each table:
   COPY table TO stdout  → row data
   Or: SELECT * FROM table (for some formats)

5. Output: SQL file or custom binary format

Snapshot isolation guarantee:
   pg_dump sees database state at START of dump
   Changes after dump starts: NOT included
   Result: consistent, transactionally correct backup
```

```bash
# pg_dump formats:

# Plain SQL (default): human readable, largest file
pg_dump -h localhost -U postgres -d mydb -f mydb_backup.sql
# Output: CREATE TABLE ...; INSERT INTO ...; etc.
# Restore: psql -d newdb -f mydb_backup.sql

# Custom format: compressed, supports parallel restore, selective restore
pg_dump -h localhost -U postgres -d mydb -Fc -f mydb_backup.dump
# -Fc: Format Custom
# Default compression (gzip level 6)
# Can specify: -Z 9 (max compression) or -Z 0 (no compression)

# Directory format: parallel backup
pg_dump -h localhost -U postgres -d mydb -Fd -j 8 -f mydb_backup_dir/
# -Fd: Format Directory
# -j 8: 8 parallel workers (one file per table)
# Each table in separate file → parallel restore possible

# Tar format: portable, single file, no random access
pg_dump -h localhost -U postgres -d mydb -Ft -f mydb_backup.tar

# Restore custom/directory format:
pg_restore -h localhost -U postgres -d newdb -j 8 mydb_backup.dump
# -j 8: 8 parallel restore workers

# Selective backup:
pg_dump -h localhost -U postgres -d mydb \
  -t orders -t order_items \
  -Fc -f orders_backup.dump
# Only orders and order_items tables

# Exclude tables:
pg_dump -h localhost -U postgres -d mydb \
  -T 'audit_*' -T 'temp_*' \
  -Fc -f mydb_no_audit.dump
# Exclude tables matching pattern

# Schema only (no data):
pg_dump -h localhost -U postgres -d mydb \
  --schema-only -f schema_only.sql

# Data only (no DDL):
pg_dump -h localhost -U postgres -d mydb \
  --data-only -Fc -f data_only.dump

# Specific schema:
pg_dump -h localhost -U postgres -d mydb \
  -n analytics -Fc -f analytics_schema.dump

# Large table in parallel:
pg_dump -h localhost -U postgres -d mydb \
  -Fd -j 16 --no-privileges --no-acl \
  -f full_parallel_backup/

# Compression options (PostgreSQL 16+):
pg_dump ... --compress=lz4  # faster than gzip
pg_dump ... --compress=zstd # better ratio

# Dump size estimate (before doing actual dump):
psql -c "SELECT pg_size_pretty(pg_database_size('mydb'));"
```

**pg_dump Gotchas:**

```bash
# Gotcha 1: Large object (lo) handling
pg_dump -h localhost -U postgres -d mydb -b -f mydb_with_blobs.dump
# -b: include large objects (blobs)
# Without -b: large objects NOT backed up!

# Gotcha 2: Role/permission backup
pg_dump dumps table-level permissions
# But: role definitions NOT dumped by pg_dump!
pg_dumpall --globals-only -f globals.sql
# Dumps: roles, tablespaces (no data)

# Full cluster backup:
pg_dumpall -h localhost -U postgres -f full_cluster.sql
# All databases + roles + tablespaces
# Very large, sequential only

# Gotcha 3: Sequence current values
pg_dump includes: CREATE SEQUENCE ... START WITH current_value
# After restore: sequences continue from backup point
# If data was added since backup: new rows might conflict!
# Always reset sequences after data merge:
SELECT setval('orders_id_seq', (SELECT max(id) FROM orders));

# Gotcha 4: Foreign keys and restore order
pg_restore respects dependency order automatically
# Custom format: --disable-triggers option for data load
pg_restore -h localhost -U postgres -d newdb \
  --disable-triggers  # disable FK triggers during load
  mydb_backup.dump
# Re-enable after: pg_restore without --disable-triggers

# Gotcha 5: Parallel restore requires superuser or table owner
# Non-superuser parallel restore: worker processes need permissions
```

---

### pg_basebackup — Physical Backup Deep Dive

**How pg_basebackup works:**

```
pg_basebackup process:
1. Connect to PostgreSQL (replication connection)
2. SELECT pg_start_backup('label', fast, exclusive/non-exclusive)
   fast=true: immediate checkpoint (forced)
   fast=false: wait for next scheduled checkpoint (less I/O impact)

3. Copy $PGDATA files (while database is running!)
   Files being modified while copying → inconsistent pages?
   → WAL covers this! pg_start_backup marks start LSN
   → All changes during backup: in WAL
   → Restore: apply WAL from start LSN → consistent state

4. Copy pg_wal/ (WAL files generated during backup)
   -Xs option: stream WAL simultaneously
   -Xf option: fetch WAL after backup (must retain WAL)
   -Xn option: don't include WAL (manual WAL management)

5. SELECT pg_stop_backup()
   Returns: stop LSN, backup label
   Writes backup_label file to backup directory

6. Result: consistent base backup + WAL files
```

```bash
# Basic physical backup:
pg_basebackup \
  -h localhost \
  -U replicator \
  -D /backup/base/$(date +%Y%m%d_%H%M%S) \
  -Fp \          # format: plain (copy files as-is)
  -Xs \          # stream WAL during backup
  -P \           # show progress
  -R             # create recovery config (for standby setup)

# Compressed backup:
pg_basebackup \
  -h localhost \
  -U replicator \
  -D /backup/base/$(date +%Y%m%d_%H%M%S) \
  -Ft \          # format: tar
  -z \           # gzip compression
  -Xs \
  -P

# Modern compression (PostgreSQL 15+):
pg_basebackup \
  -h localhost \
  -U replicator \
  -D /backup/base/ \
  -Ft \
  --compress=server-lz4 \  # compress on server side
  -Xs -P

# Tablespace mapping (if custom tablespaces):
pg_basebackup \
  -h localhost \
  -U replicator \
  -D /backup/base/ \
  -Fp \
  -Xs \
  --tablespace-map=/old/ssd/path=/new/ssd/path \
  -P

# Checkpoint control:
pg_basebackup ... --checkpoint=fast     # immediate checkpoint (more I/O spike)
pg_basebackup ... --checkpoint=spread   # gentle, less I/O impact (slower start)

# Verify backup:
pg_basebackup ... --verify-checksums    # verify page checksums during backup
```

**Replication user setup:**

```sql
-- Replication user for backup:
CREATE ROLE replicator WITH LOGIN REPLICATION PASSWORD 'repl_pass';
-- REPLICATION attribute required for physical backup!

-- pg_hba.conf:
-- host  replication  replicator  backup_server_ip/32  scram-sha-256
```

---

### WAL Archiving — Foundation of PITR

```sql
-- postgresql.conf for archiving:
wal_level = replica         -- or logical
archive_mode = on
archive_command = 'test ! -f /wal_archive/%f && cp %p /wal_archive/%f'
-- test ! -f: don't overwrite existing (safety)
-- %p: source path
-- %f: filename

-- Restore command (for recovery):
restore_command = 'cp /wal_archive/%f %p'

-- Archive to S3:
archive_command = 'aws s3 cp %p s3://mybackup-bucket/wal/%f'
restore_command = 'aws s3 cp s3://mybackup-bucket/wal/%f %p'

-- Archive to S3 with wal-g (recommended tool):
archive_command = 'wal-g wal-push %p'
restore_command = 'wal-g wal-fetch %f %p'

-- Verify archiving works:
SELECT * FROM pg_stat_archiver;
-- archived_count: total archived
-- last_archived_wal: last successfully archived WAL file
-- last_archived_time: when
-- failed_count: failures (should be 0!)
-- last_failed_wal: which file failed
-- last_failed_time: when failed
-- last_failure_msg: error message

-- Archive lag monitoring:
SELECT
  archived_count,
  failed_count,
  last_archived_wal,
  now() - last_archived_time AS archive_lag
FROM pg_stat_archiver;
-- archive_lag > 5 minutes: investigate!
```

---

### Incremental Backup

**PostgreSQL 17 Native Incremental:**

```bash
# PostgreSQL 17+ has native incremental backup support
# Uses summarized WAL to know which blocks changed

# Enable WAL summarization:
# postgresql.conf:
# summarize_wal = on

# Initial full backup (creates manifest):
pg_basebackup \
  -h localhost \
  -U replicator \
  -D /backup/base_full \
  -Ft -Xs -P

# Incremental backup (only changed blocks):
pg_basebackup \
  --incremental=/backup/base_full/backup_manifest \
  -D /backup/incr_$(date +%Y%m%d) \
  -Ft -Xs -P

# Restore incremental backup (combine with pg_combinebackup):
pg_combinebackup \
  /backup/base_full \
  /backup/incr_20240115 \
  /backup/incr_20240116 \
  -o /restore/combined
# Creates a complete backup from full + incrementals
```

**Manual Incremental (pre-17):**

```bash
# Using WAL + base backup = continuous incremental
# Base backup + all WAL since = incremental effectively

# WAL-based incremental with wal-g:
wal-g backup-push $PGDATA          # full backup
# WAL continuously archived

# List backups:
wal-g backup-list

# Restore to specific time:
wal-g backup-fetch /restore/dir LATEST  # latest full backup
# Then PITR using archived WAL

# wal-g delta backup (incremental):
wal-g backup-push $PGDATA --full         # force full
wal-g backup-push $PGDATA                # delta (incremental)
# Delta: only pages changed since last backup
# Storage: much less than full backup
```

---

### Differential Backup

```bash
# Differential = changes since last FULL backup
# vs Incremental = changes since last backup (full or incremental)

# PostgreSQL-native: no built-in differential
# But: WAL-based recovery provides this effectively:
# Full backup + WAL from that point = differential equivalent

# rsync-based differential (filesystem level):
# First backup:
rsync -av $PGDATA/ /backup/full_$(date +%Y%m%d)/

# Differential (everything changed since full):
rsync -av --compare-dest=/backup/full_20240101/ \
  $PGDATA/ /backup/diff_$(date +%Y%m%d)/
# Only files different from full_20240101 copied

# Note: rsync on live database not safe without WAL!
# Only safe with pg_basebackup or after pg_start_backup/pg_stop_backup
```

---

### Backup Validation

```bash
# Test backup integrity:

# 1. Verify pg_dump backup:
pg_restore --list mydb_backup.dump | head -50  # list contents
pg_restore --verbose --dry-run mydb_backup.dump  # dry run

# 2. Restore to test instance:
createdb testrestore
pg_restore -d testrestore -j 4 mydb_backup.dump
# Run application smoke tests against testrestore
dropdb testrestore

# 3. Verify physical backup:
# Start PostgreSQL with backup data:
pg_ctl -D /backup/base_$(date +%Y%m%d) start
# If starts: backup valid
# pg_basebackup --verify-checksums: during backup time

# 4. Checksum verification:
pg_checksums --check -D /backup/base_20240101/
# Verifies page checksums (requires page_checksums enabled)
# Enable checksums: initdb --data-checksums
# Or: pg_checksums --enable -D $PGDATA (offline!)

# 5. Automated restore testing:
# Cron job monthly:
# 1. Restore latest backup to test server
# 2. Run pg_dumpall --schema-only on both
# 3. Diff schema dumps
# 4. Run basic SELECT queries
# 5. Alert if any step fails
```

---

### Backup Strategy for Different Scenarios

```
Scenario 1: Small database (< 100GB), low RPO (1 hour), high RTO (4 hours)
  Strategy:
    - Daily pg_dump (logical, to S3)
    - WAL archiving (continuous, to S3)
    - Retention: 30 days pg_dump + 7 days WAL
  Cost: low
  Restore: pg_dump restore + WAL replay to point-in-time

Scenario 2: Large database (> 1TB), low RPO (15 min), low RTO (30 min)
  Strategy:
    - Daily pg_basebackup (physical, to S3 with wal-g)
    - WAL archiving (continuous streaming)
    - Warm standby for fast failover
    - Weekly pg_dump for selective restore
  Cost: medium-high
  Restore: Failover to standby (minutes) OR PITR from wal-g

Scenario 3: Mission-critical (financial), RPO = 0, RTO = 1 min
  Strategy:
    - Synchronous streaming replication (RPO = 0)
    - Automatic failover (Patroni/PgBouncer)
    - Daily pg_basebackup + WAL archiving
    - Cross-region standby
  Cost: high
  Restore: Automatic failover (seconds), no data loss

Scenario 4: Analytics/reporting, high RPO acceptable (24 hours)
  Strategy:
    - Daily pg_dump after business hours
    - S3 storage with lifecycle policy (Glacier after 30 days)
    - No WAL archiving needed
  Cost: minimal
```

---

## Topic 25 — Point-in-Time Recovery (PITR)

### Big Picture: "Time Machine" for Database

PITR হলো database-এর সবচেয়ে powerful recovery mechanism। কোনো human error (accidental DROP TABLE, bad UPDATE), কোনো specific moment-এ ফিরে যাওয়া যায়।

```
Requirements for PITR:
  1. Base backup (pg_basebackup)
  2. WAL archive from base backup time to recovery target
  3. Both accessible at restore time

PITR timeline:
  Sunday 2:00 AM: pg_basebackup completes
  Sunday-Wednesday: WAL continuously archived
  Wednesday 3:00 PM: Accidental DROP TABLE orders!
  Wednesday 3:00:04 PM: DBA notices the error

  Goal: Restore to Wednesday 2:59:59 PM (before DROP)

  Process:
    1. Restore base backup (Sunday 2:00 AM state)
    2. Apply WAL: Sunday → Monday → Tuesday → Wednesday 2:59:59 PM
    3. Stop before Wednesday 3:00 PM
    4. Database at Wednesday 2:59:59 PM state!
```

---

### WAL Replay — How It Works

```
WAL file = sequence of records, each with LSN (Log Sequence Number)

Recovery reads WAL records sequentially:
  LSN 0/1000000: INSERT into orders (XID=100)
  LSN 0/1001000: UPDATE orders SET status='shipped' (XID=101)
  LSN 0/1002000: COMMIT XID=100
  LSN 0/1003000: COMMIT XID=101
  ...
  LSN 0/5000000: DROP TABLE orders (XID=9999) ← stop before this!
  LSN 0/5000100: COMMIT XID=9999

Recovery process:
  For each WAL record:
    Apply to data file (redo operation)
    Check: have we passed recovery target?
      Yes → stop
      No → continue

Recovery target types:
  Time:    stop before this timestamp
  XID:     stop before this transaction
  LSN:     stop at this WAL position
  Name:    stop at this restore point (pg_create_restore_point())
```

---

### PITR Step-by-Step — Complete Guide

**Step 1: Prepare the restore directory**

```bash
# Stop existing PostgreSQL (if running):
pg_ctl -D $PGDATA stop

# Backup current data (safety):
mv $PGDATA $PGDATA.corrupted.$(date +%Y%m%d_%H%M%S)

# Create new data directory:
mkdir -p $PGDATA
chmod 700 $PGDATA
```

**Step 2: Restore base backup**

```bash
# From pg_basebackup (plain format):
cp -a /backup/base_20240114_020000/. $PGDATA/
# Or rsync:
rsync -av /backup/base_20240114_020000/ $PGDATA/

# From pg_basebackup (tar format):
cd $PGDATA
tar xzf /backup/base_20240114_020000/base.tar.gz
tar xzf /backup/base_20240114_020000/pg_wal.tar.gz -C pg_wal/

# From wal-g:
wal-g backup-fetch $PGDATA LATEST
# or specific backup:
wal-g backup-fetch $PGDATA base_000000010000000000000002

# Ownership:
chown -R postgres:postgres $PGDATA
```

**Step 3: Configure recovery**

```bash
# Create recovery signal file:
touch $PGDATA/recovery.signal
# This tells PostgreSQL: start in recovery mode

# Configure postgresql.conf (or postgresql.auto.conf):
cat >> $PGDATA/postgresql.auto.conf << 'EOF'

# Recovery configuration:
restore_command = 'cp /wal_archive/%f %p'
# Or from S3:
# restore_command = 'aws s3 cp s3://mybackup/wal/%f %p'
# Or wal-g:
# restore_command = 'wal-g wal-fetch %f %p'

# Recovery target (choose ONE):

# Option A: Time-based
recovery_target_time = '2024-01-17 14:59:59'
recovery_target_timezone = 'Asia/Dhaka'

# Option B: Transaction-based
# recovery_target_xid = '12345'

# Option C: Restore point (named)
# recovery_target_name = 'before_batch_delete'

# Option D: LSN-based
# recovery_target_lsn = '0/5000000'

# Option E: Latest (default, no target = recover as far as possible)
# recovery_target = 'immediate'  # stop at end of backup consistency point

# What to do after reaching target:
recovery_target_action = 'promote'    # become primary after recovery
# recovery_target_action = 'pause'    # pause and wait for manual promote
# recovery_target_action = 'shutdown' # shutdown after recovery (inspect state)

# Include or exclude the target transaction:
recovery_target_inclusive = true      # include the target transaction
EOF
```

**Step 4: Start PostgreSQL in recovery mode**

```bash
# Start:
pg_ctl -D $PGDATA start

# Watch recovery progress in logs:
tail -f /var/log/postgresql/postgresql.log

# Expected log output:
# LOG: database system was shut down in recovery at 2024-01-14 02:00:00 UTC
# LOG: starting point-in-time recovery to 2024-01-17 14:59:59+06
# LOG: restored log file "000000010000000000000001" from archive
# LOG: redo starts at 0/1000000
# LOG: consistent recovery state reached at 0/1000100
# LOG: restored log file "000000010000000000000002" from archive
# ... (many WAL files)
# LOG: recovery stopping before commit of transaction 9999,
#      time 2024-01-17 15:00:00.123+06
# LOG: pausing at the end of recovery
# HINT: Execute pg_wal_replay_resume() to promote.

# Verify recovery point:
psql -c "SELECT now();"  -- current server time
psql -c "SELECT count(*) FROM orders;"  -- should have rows!
psql -c "SELECT pg_is_in_recovery();"  -- true = still in recovery

# If recovery_target_action = 'pause':
psql -c "SELECT pg_wal_replay_resume();"  -- promote to primary
```

**Step 5: Verify and promote**

```bash
# After recovery.signal removed (automatic after promote),
# PostgreSQL is now a normal primary.

# Verify:
psql -c "SELECT pg_is_in_recovery();"  -- false = promoted!
psql -c "SELECT count(*) FROM orders;"  -- data intact?
psql -c "SELECT max(created_at) FROM orders;"  -- latest data at correct time?

# Remove old WAL files from pg_wal (from before recovery):
# PostgreSQL handles this automatically after promotion

# Create new base backup immediately!
pg_basebackup -h localhost -U replicator -D /backup/base_new -Fp -Xs -P
```

---

### Restore Point — Named Recovery Targets

```sql
-- Create named restore point BEFORE risky operation:
SELECT pg_create_restore_point('before_schema_migration_v2');
-- Returns: restore point LSN

-- Before risky batch delete:
SELECT pg_create_restore_point('before_cleanup_2024_01_17');

-- Recovery to named point:
-- recovery_target_name = 'before_cleanup_2024_01_17'

-- Best practice: always create restore point before:
-- Schema migrations
-- Bulk data operations
-- Third-party integrations
-- Any potentially destructive operation
```

---

### Tablespace PITR — Additional Complexity

```bash
# Tablespaces in different directories:
# pg_basebackup creates tablespace_map
# During restore: must recreate tablespace directories

# Read tablespace_map from backup:
cat /backup/base/tablespace_map
# OID    /old/path/to/tablespace

# Create same directories on restore server:
mkdir -p /new/path/to/tablespace
chown postgres:postgres /new/path/to/tablespace

# Or: remap tablespace during restore:
pg_basebackup ... --tablespace-map=/old/path=/new/path
```

---

### PITR Monitoring and Testing

```sql
-- Monitor WAL archive completeness:
SELECT
  pg_walfile_name(pg_current_wal_lsn()) AS current_wal_file,
  last_archived_wal,
  last_archived_time,
  now() - last_archived_time AS archive_lag,
  failed_count
FROM pg_stat_archiver;

-- Estimate PITR restore time:
-- WAL files to replay = time_range / wal_rotation_rate
-- Each 16MB WAL file: ~seconds to replay (depends on transaction rate)
-- Rough estimate: 100MB WAL/minute replay rate

-- WAL retention check:
-- Must keep WAL from oldest base backup time
-- Too little: PITR window narrow
-- Too much: storage expensive

-- wal-g backup policy:
wal-g backup-list  -- see available backups
wal-g delete retain FULL 7  -- keep 7 full backups
wal-g delete --confirm  -- actually delete old backups

-- Test PITR quarterly:
-- 1. Create test restore point
-- 2. Insert test data
-- 3. Note timestamp
-- 4. Restore to test server up to timestamp
-- 5. Verify test data exists
-- 6. Verify data AFTER test point doesn't exist
```

---

## Topic 26 — Replication: Streaming, Logical

### Big Picture: Replication কেন এবং কখন কোনটা?

```
Replication-এর goals:
  High Availability (HA): Primary fails → Standby takes over
  Read Scaling: Read queries → Standbys (read replicas)
  Disaster Recovery: Geographic separation
  Zero-downtime upgrade: Logical replication-based migration

Types:
  Physical (Streaming): byte-level copy of WAL
    → Exact copy of primary
    → Same PostgreSQL version required
    → All databases replicated
    → Used for: HA, disaster recovery

  Logical: row-level changes
    → Selective (table by table)
    → Cross-version possible
    → Different schemas possible
    → Used for: upgrade, selective replication, CDC
```

---

### Streaming Replication — Physical Deep Dive

**How streaming replication works:**

```
Primary:
  1. Transactions happen → WAL written
  2. WAL Writer: WAL Buffer → pg_wal/ files
  3. WAL Sender process: reads pg_wal/, streams to standby

Standby:
  1. WAL Receiver process: receives WAL from primary
  2. Writes to pg_wal/ on standby
  3. Startup (recovery) process: reads pg_wal/, applies to data files
  4. hot_standby = on: allows read queries on standby!

Network protocol:
  Physical replication: uses replication protocol (not SQL protocol)
  Connection: replication type (pg_hba.conf: TYPE=host, DATABASE=replication)
  Streaming: continuous WAL stream (not discrete files)
  Fallback: if streaming breaks → WAL file fetching from archive
```

**Primary Configuration:**

```sql
-- postgresql.conf:
wal_level = replica           -- minimum for replication (or 'logical')
max_wal_senders = 10          -- max simultaneous replication connections
                              -- = number of standbys + extra for pg_basebackup
wal_keep_size = 1GB           -- keep this much WAL even if archived
                              -- Standby lag protection (if standby behind)
                              -- Better alternative: replication slots (below)

-- Replication user:
CREATE ROLE replicator
  WITH LOGIN REPLICATION PASSWORD 'strong_repl_password';

-- pg_hba.conf:
-- host replication replicator standby_ip/32 scram-sha-256
```

**Standby Configuration:**

```bash
# Initial standby setup:
# 1. Base backup from primary:
pg_basebackup \
  -h primary_host \
  -U replicator \
  -D $PGDATA \
  -Fp -Xs -R \  # -R: auto-create standby.signal + postgresql.auto.conf
  -P

# -R creates these automatically:
# standby.signal: tells PostgreSQL to start as standby
# postgresql.auto.conf contains:
# primary_conninfo = 'host=primary_host user=replicator password=...'
```

```
# postgresql.conf on standby:
hot_standby = on           # allow read queries (default on)
hot_standby_feedback = on  # tell primary about standby's oldest transaction
                           # prevents primary from VACUUM-ing rows standby needs
                           # Prevents "query conflicts" on standby
                           # Trade-off: primary vacuum less aggressive

max_standby_streaming_delay = 30s  # how long to delay conflicting queries
max_standby_archive_delay = 30s    # same for archive recovery

# Primary connection string (in auto.conf or manually):
primary_conninfo = 'host=primary_host port=5432
                    user=replicator password=strong_repl_password
                    application_name=standby1
                    sslmode=require'

# If using replication slot:
primary_slot_name = 'standby1_slot'
```

---

### Replication Slots — WAL Retention Safety Net

```sql
-- Problem without replication slot:
-- Standby falls behind (network issue, maintenance)
-- Primary: WAL older than wal_keep_size → DELETED
-- Standby tries to catch up: WAL file not found!
-- → Standby must be rebuilt from scratch!

-- Replication slot solution:
-- Primary tracks: "standby1 has read up to LSN X"
-- Primary: NEVER delete WAL needed by any slot
-- Standby can fall behind safely (as long as disk space allows)

-- Create physical replication slot on primary:
SELECT pg_create_physical_replication_slot('standby1_slot');
-- Returns: (slot_name, lsn)

-- View slots:
SELECT
  slot_name,
  slot_type,
  active,
  active_pid,
  restart_lsn,
  confirmed_flush_lsn,
  pg_size_pretty(pg_wal_lsn_diff(
    pg_current_wal_lsn(),
    restart_lsn
  )) AS lag_size
FROM pg_replication_slots;

-- Gotcha: Replication slot danger!
-- If standby dies and slot not dropped:
-- Primary keeps ALL WAL from slot's restart_lsn!
-- pg_wal/ directory grows unboundedly!
-- Disk full → PostgreSQL crash!

-- Monitor slot lag:
SELECT slot_name,
  pg_size_pretty(pg_wal_lsn_diff(
    pg_current_wal_lsn(), restart_lsn
  )) AS retained_wal
FROM pg_replication_slots
WHERE NOT active;  -- inactive slots are dangerous!

-- Alert if retained WAL > threshold:
SELECT slot_name
FROM pg_replication_slots
WHERE pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) > 10 * 1024^3;  -- > 10GB

-- Drop abandoned slot:
SELECT pg_drop_replication_slot('standby1_slot');
-- Only if standby truly dead and being rebuilt!
```

---

### Synchronous vs Asynchronous Replication

```sql
-- ASYNCHRONOUS (default):
-- Primary: COMMIT → return OK to client → send WAL to standby
-- Risk: Primary crashes after COMMIT but before WAL sent to standby
-- → Data loss possible (WAL in primary memory, not yet on standby)
-- RPO: seconds to minutes (depending on network/lag)

-- SYNCHRONOUS:
-- Primary: COMMIT → wait for standby acknowledgment → return OK to client
-- Risk: None (data confirmed on standby before client gets OK)
-- Cost: Latency! Each COMMIT waits for network round-trip to standby
-- RPO: 0 (no data loss)

-- Configure synchronous replication:
synchronous_standby_names = 'standby1'
-- Primary waits for standby1 to acknowledge each commit

-- Multiple standbys:
synchronous_standby_names = 'standby1,standby2'
-- Waits for BOTH (stricter)

-- Quorum-based (PostgreSQL 10+):
synchronous_standby_names = 'ANY 1 (standby1, standby2, standby3)'
-- Waits for ANY 1 of the 3 standbys
-- If standby1 down: waits for standby2 or standby3
-- More resilient than requiring specific standby

synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
-- Waits for first in list (priority order)

-- Synchronous commit levels:
synchronous_commit = on              -- wait for standby WAL write + flush
synchronous_commit = remote_write    -- wait for standby WAL write (not flush)
synchronous_commit = remote_apply    -- wait for standby to APPLY (replay)
-- remote_apply: strongest, slowest (standby query reflects committed data)
synchronous_commit = local           -- only wait for local WAL, async to standby
synchronous_commit = off             -- async everywhere

-- What happens if synchronous standby goes down?
-- Primary waits FOREVER for synchronous commit!
-- All transactions block!
-- Set timeout:
-- (no built-in timeout, application must handle via statement_timeout)
-- Better: use quorum 'ANY 1 of 2' so one standby down = still OK
```

---

### Replication Monitoring

```sql
-- On PRIMARY:
SELECT
  application_name,
  client_addr,
  state,                      -- 'startup', 'catchup', 'streaming', 'backup', 'stopping'
  sent_lsn,                   -- LSN sent to standby
  write_lsn,                  -- LSN written to standby disk
  flush_lsn,                  -- LSN flushed to standby disk
  replay_lsn,                 -- LSN replayed on standby (applied to data files)
  pg_size_pretty(
    pg_wal_lsn_diff(sent_lsn, replay_lsn)
  ) AS total_lag,
  write_lag,                  -- time lag for write
  flush_lag,                  -- time lag for flush
  replay_lag,                 -- time lag for replay (most useful)
  sync_state                  -- 'async', 'sync', 'quorum', 'potential'
FROM pg_stat_replication
ORDER BY application_name;

-- On STANDBY:
SELECT
  status,                     -- 'streaming' or 'stopped'
  received_lsn,               -- received from primary
  last_msg_send_time,         -- last message from primary
  last_msg_receipt_time,      -- last message received
  latest_end_lsn,             -- latest LSN
  latest_end_time
FROM pg_stat_wal_receiver;

-- Replication delay in time:
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay
FROM pg_stat_replication;  -- on primary

-- On standby:
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;

-- Check if standby is in recovery:
SELECT pg_is_in_recovery();  -- true = standby, false = primary

-- Standby LSN lag:
SELECT pg_wal_lsn_diff(
  pg_current_wal_lsn(),  -- primary's current LSN
  pg_last_wal_receive_lsn()  -- standby's received LSN
) AS lag_bytes;
```

---

### Logical Replication — Selective Row-Level Replication

**How logical replication works:**

```
Physical replication: byte-level WAL → exact copy
Logical replication: decoded changes (INSERT/UPDATE/DELETE) → selective apply

WAL → Logical Decoding → pgoutput plugin → Publication/Subscription

Logical decoding:
  WAL record: "page 5, offset 100, bytes changed: ..."
  Decoded: "Table orders, id=42, status changed from 'pending' to 'shipped'"
  
  Human-readable, table-specific, filterable!

wal_level = logical required (more WAL than replica)
```

**Publisher setup:**

```sql
-- PostgreSQL 10+
-- wal_level must be 'logical' on publisher

-- Create publication (what to publish):
CREATE PUBLICATION my_pub FOR TABLE orders, users;
-- Specific tables only

CREATE PUBLICATION all_tables_pub FOR ALL TABLES;
-- All tables (including future tables if using FOR ALL TABLES)

CREATE PUBLICATION orders_active_pub FOR TABLE orders
WHERE (status != 'cancelled');  -- Row filter (PostgreSQL 15+)
-- Only non-cancelled orders replicated!

-- Column filter (PostgreSQL 15+):
CREATE PUBLICATION users_safe_pub FOR TABLE users
  (id, name, email, created_at, active);
-- Only these columns replicated (no password_hash!)

-- Check publications:
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;

-- Replication user with publication privileges:
CREATE ROLE logical_repl WITH LOGIN REPLICATION PASSWORD 'logicalpass';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO logical_repl;
-- Subscriber connects as this user to read WAL
```

**Subscriber setup:**

```sql
-- Tables must exist on subscriber already (DDL not replicated!):
CREATE TABLE orders (
  id bigint PRIMARY KEY,
  user_id bigint,
  amount numeric(12,2),
  status text,
  created_at timestamptz
);

-- Create subscription:
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=primary_host dbname=mydb user=logical_repl password=logicalpass'
  PUBLICATION my_pub;

-- Options:
CREATE SUBSCRIPTION my_sub
  CONNECTION '...'
  PUBLICATION my_pub
  WITH (
    copy_data = true,         -- copy existing data (initial sync)
    create_slot = true,       -- create replication slot on publisher
    enabled = true,           -- start immediately
    slot_name = 'my_sub_slot',-- slot name on publisher
    synchronous_commit = 'off'-- subscriber doesn't wait for publisher confirm
  );

-- Check subscription status:
SELECT
  subname,
  subenabled,
  subslotname,
  subpublications
FROM pg_subscription;

-- Subscription progress:
SELECT
  subname,
  received_lsn,
  latest_end_lsn,
  latest_end_time
FROM pg_stat_subscription;

-- Worker status:
SELECT * FROM pg_stat_subscription_stats;

-- Replication origin tracking:
SELECT * FROM pg_replication_origin;
SELECT * FROM pg_replication_origin_status;
```

**Logical Replication Use Cases:**

```sql
-- Use case 1: Zero-downtime major version upgrade
-- Old server (PostgreSQL 14): publisher
-- New server (PostgreSQL 15): subscriber
-- 
-- Steps:
-- 1. Set up logical replication old → new
-- 2. Wait for initial sync
-- 3. Wait for lag → 0
-- 4. Point application to new server (minimal downtime: seconds)
-- 5. Drop subscription on new server → promote to primary

-- Use case 2: Selective table replication (microservices)
-- Orders service DB: full database
-- Analytics DB: only orders + order_items table (read replica)
CREATE PUBLICATION analytics_pub FOR TABLE orders, order_items, products;
-- Analytics connects as subscriber, gets only relevant tables

-- Use case 3: Change Data Capture (CDC) for external systems
-- Debezium: reads PostgreSQL logical replication stream
-- Sends changes to Kafka
-- Kafka consumers: Elasticsearch (search index), Redis (cache), etc.
-- wal_level = logical enables this

-- Limitations:
-- DDL NOT replicated! Schema changes must be done manually on subscriber
-- Sequences NOT replicated automatically
-- Large objects NOT replicated
-- TRUNCATE: replicated (PostgreSQL 11+)
-- UPDATE/DELETE: need PRIMARY KEY on table (or REPLICA IDENTITY FULL)
```

---

## Topic 27 — Failover & High Availability

### Big Picture: HA = Never Single Point of Failure

```
HA Architecture Levels:

Level 0: Single server
  SPOF: server failure = outage
  RTO: hours (restore from backup)
  RPO: hours (last backup)

Level 1: Hot Standby (async replication)
  SPOF: manual failover needed
  RTO: minutes (manual promote)
  RPO: seconds (replication lag)

Level 2: Synchronous Hot Standby
  SPOF: manual failover needed
  RTO: minutes
  RPO: 0

Level 3: Automatic Failover (Patroni/pg_auto_failover)
  SPOF: none (automated)
  RTO: seconds to minutes
  RPO: 0 (with synchronous) or seconds (async)

Level 4: Multi-region HA
  SPOF: even datacenter failure handled
  RTO: minutes
  RPO: seconds to minutes (cross-region latency)
```

---

### Manual Failover — Step by Step

```bash
# Scenario: Primary server crashed, standby must take over

# On STANDBY server:

# Step 1: Verify standby is up-to-date
psql -c "SELECT pg_is_in_recovery();"  -- must be true
psql -c "SELECT now() - pg_last_xact_replay_timestamp() AS lag;"
# Lag should be minimal (seconds, not minutes)

# Step 2: Promote standby to primary
pg_ctl promote -D $PGDATA
# OR:
touch $PGDATA/promote.signal  # PostgreSQL 12+ method
# OR in psql:
# SELECT pg_promote();  -- PostgreSQL 12+

# What happens during promote:
# 1. standby.signal file deleted
# 2. WAL replay stops
# 3. Timeline ID incremented (new timeline)
# 4. New WAL file started
# 5. PostgreSQL accepts write connections!

# Step 3: Verify promotion
psql -c "SELECT pg_is_in_recovery();"  -- must be FALSE now
psql -c "SELECT pg_current_wal_lsn();"  -- now advancing

# Step 4: Update application connection string
# Point applications to new primary (old standby)

# Step 5: Fix DNS / load balancer
# If using DNS: update A record to new primary IP
# PgBouncer: update host in pgbouncer.ini, RELOAD

# Step 6: Handle old primary (when it comes back)
# Old primary: might have data not on new primary (if async replication)
# DO NOT bring old primary online as primary!
# Convert to standby:
# 1. Stop old primary (pg_ctl stop)
# 2. pg_rewind to sync with new primary
# 3. Configure as standby
# 4. Start as standby

# pg_rewind: efficiently sync old primary with new primary
pg_rewind \
  --target-pgdata=$PGDATA \
  --source-server='host=new_primary_host user=postgres dbname=postgres' \
  --progress
# pg_rewind: uses WAL to find divergence point, sync only diverged blocks
# Much faster than full pg_basebackup!
# Requirement: wal_log_hints = on OR checksums enabled
```

---

### Patroni — Industry Standard HA

**Architecture:**

```
                    ┌─────────────────┐
                    │  DCS (etcd /    │
                    │  Consul / ZK)   │
                    │  Leader lock    │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
   ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
   │  Node 1     │    │  Node 2     │    │  Node 3     │
   │  Patroni    │    │  Patroni    │    │  Patroni    │
   │  PostgreSQL │    │  PostgreSQL │    │  PostgreSQL │
   │  (PRIMARY)  │    │  (STANDBY)  │    │  (STANDBY)  │
   └─────────────┘    └─────────────┘    └─────────────┘

DCS = Distributed Consensus Store (etcd, Consul, ZooKeeper)
  Leader lock: only one node holds the lock = PRIMARY

Patroni:
  Each node runs Patroni agent
  Patroni: manages PostgreSQL, health checks, DCS communication

Failover process:
  1. Primary node: health check fails (PostgreSQL down)
  2. Primary Patroni: releases DCS leader lock (or lock expires: TTL)
  3. Standby Patroni nodes: compete for DCS leader lock
  4. Winner: promotes PostgreSQL to primary
  5. Loser: reconfigures as standby of new primary
  Total time: TTL + election time ≈ 15-30 seconds
```

**Patroni configuration:**

```yaml
# /etc/patroni/patroni.yml

scope: postgres-ha-cluster   # cluster name (same on all nodes)
namespace: /db/              # DCS namespace
name: node1                  # this node's name (unique per node)

# DCS connection:
etcd3:
  host: etcd1:2379,etcd2:2379,etcd3:2379
  # protocol: https (production)
  # cacert: /etc/ssl/etcd/ca.crt
  # cert: /etc/ssl/etcd/client.crt
  # key: /etc/ssl/etcd/client.key

# Cluster behavior:
bootstrap:
  dcs:
    ttl: 30                         # leader lock TTL (seconds)
    loop_wait: 10                   # health check interval
    retry_timeout: 10               # DCS operation timeout
    maximum_lag_on_failover: 1048576 # max lag for failover candidate (bytes)
    master_start_timeout: 300       # max time to wait for primary startup
    synchronous_mode: false         # true = synchronous replication required
    synchronous_mode_strict: false  # true = no async fallback

  # Initial cluster setup:
  initdb:
    - encoding: UTF8
    - data-checksums         # enable page checksums!

  # Post-init SQL:
  post_bootstrap: |
    CREATE ROLE replicator WITH LOGIN REPLICATION PASSWORD 'replpass';

postgresql:
  listen: 0.0.0.0:5432        # listen on all interfaces
  connect_address: node1_ip:5432  # how other nodes connect to this
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  pgpass: /tmp/pgpass           # password file

  # Authentication:
  authentication:
    replication:
      username: replicator
      password: replpass
    superuser:
      username: postgres
      password: pgpass

  # PostgreSQL parameters:
  parameters:
    max_connections: 200
    shared_buffers: 4GB
    effective_cache_size: 12GB
    wal_level: replica
    hot_standby: "on"
    max_wal_senders: 10
    max_replication_slots: 10
    wal_log_hints: "on"        # required for pg_rewind!
    archive_mode: "on"
    archive_command: "wal-g wal-push %p"

  # pg_hba.conf:
  pg_hba:
    - host replication replicator 10.0.0.0/8 scram-sha-256
    - host all all 10.0.0.0/8 scram-sha-256

# REST API (for health checks and management):
restapi:
  listen: 0.0.0.0:8008
  connect_address: node1_ip:8008
  # auth: username:password (production)

# Tags (for routing):
tags:
  nofailover: false      # can be promoted
  noloadbalance: false   # can serve reads
  clonedfrom: ""
```

**Patroni operations:**

```bash
# Patroni CLI (patronictl):
patronictl -c /etc/patroni/patroni.yml list
# Cluster: postgres-ha-cluster
# Member  Host      Role    State   TL  Lag in MB
# node1   10.0.0.1  Leader  running  1   0
# node2   10.0.0.2  Replica running  1   0
# node3   10.0.0.3  Replica running  1   0

# Manual failover (controlled):
patronictl -c /etc/patroni/patroni.yml failover postgres-ha-cluster \
  --master node1 --candidate node2
# Graceful failover: node1 → node2

# Switchover (planned, with minimal interruption):
patronictl -c /etc/patroni/patroni.yml switchover postgres-ha-cluster \
  --master node1 --candidate node2 --scheduled now

# Pause automatic failover (for maintenance):
patronictl -c /etc/patroni/patroni.yml pause postgres-ha-cluster
# ... maintenance ...
patronictl -c /etc/patroni/patroni.yml resume postgres-ha-cluster

# Reinitialize a node (e.g., after disk corruption):
patronictl -c /etc/patroni/patroni.yml reinit postgres-ha-cluster node3

# Edit cluster configuration:
patronictl -c /etc/patroni/patroni.yml edit-config postgres-ha-cluster
# Opens editor with DCS-stored config
# Changes applied to all nodes!
```

---

### HAProxy + Patroni — Connection Routing

```
Application → HAProxy (load balancer)
              → Port 5000: Primary (read-write)
              → Port 5001: Standbys (read-only)

HAProxy health check: Patroni REST API
  GET http://patroni_node:8008/master
  200 OK: this node is primary
  503 Service Unavailable: this node is standby

  GET http://patroni_node:8008/replica
  200 OK: this node is standby
  503: this node is primary
```

```
# haproxy.cfg

global
  maxconn 100
  log stdout format raw local0

defaults
  log global
  mode tcp
  retries 2
  timeout client 30m
  timeout connect 4s
  timeout server 30m
  timeout check 5s

# Primary (read-write):
frontend primary
  bind *:5000
  default_backend primary_backend

backend primary_backend
  option httpchk GET /master
  http-check expect status 200
  default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
  server node1 10.0.0.1:5432 maxconn 100 check port 8008
  server node2 10.0.0.2:5432 maxconn 100 check port 8008
  server node3 10.0.0.3:5432 maxconn 100 check port 8008

# Standbys (read-only):
frontend replicas
  bind *:5001
  default_backend replica_backend

backend replica_backend
  balance roundrobin
  option httpchk GET /replica
  http-check expect status 200
  default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
  server node1 10.0.0.1:5432 maxconn 100 check port 8008
  server node2 10.0.0.2:5432 maxconn 100 check port 8008
  server node3 10.0.0.3:5432 maxconn 100 check port 8008
```

---

### PgBouncer + Patroni — Connection Pooling with HA

```ini
# pgbouncer.ini (on each application server or centrally)

[databases]
mydb = host=haproxy_vip port=5000 dbname=mydb  # primary

mydb_ro = host=haproxy_vip port=5001 dbname=mydb  # read replicas

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
reserve_pool_size = 5
reserve_pool_timeout = 3
server_idle_timeout = 600

# After failover: PgBouncer reconnects to new primary automatically
# HAProxy detects new primary via health check → PgBouncer → new primary
# Application: transparent failover!
```

---

### Split-Brain Prevention — Critical HA Concern

```
Split-brain: two nodes both think they are primary!
  Scenario: network partition between node1 (primary) and node2 (standby)
  node2: "primary is unreachable! I'll promote!"
  node1: "I'm still running, still accepting writes!"
  Both nodes: writing different data!
  Network restores: DISASTER — incompatible data!

Prevention via DCS (Distributed Consensus Store):
  Only one node can hold DCS leader lock at a time
  Network partition:
    node1: cannot reach DCS → CANNOT hold lock → STOPS accepting writes!
    node2: can reach DCS → gets lock → becomes primary
  
  This requires: DCS is more reliable than direct node-to-node network
  DCS cluster itself needs 3+ nodes for quorum

STONITH (Shoot The Other Node In The Head):
  When failover happens: kill the old primary!
  Fencing mechanisms:
    - IPMI/iDRAC: power off old primary's server
    - Cloud API: stop old primary's instance
    - SAN: fence old primary's disk access
  
  Why STONITH?
    "I see the old primary is unreachable" ≠ "old primary is down"
    Old primary might be up but network-partitioned
    Without STONITH: might still be writing data!
    With STONITH: guarantees old primary is dead before new primary starts writing

Patroni + watchdog:
  Patroni supports watchdog device (hardware or software)
  If Patroni loses DCS lock: watchdog reboots the server
  Ensures old primary cannot continue writing
```

---

### Disaster Recovery — Geographic HA

```
Multi-region setup:
  Region A (primary region):
    node1: Primary
    node2: Synchronous Standby
  
  Region B (DR region):
    node3: Asynchronous Standby (different datacenter)
  
  Network:
    node1 → node2: low latency (same region) → synchronous OK
    node1 → node3: high latency (cross-region) → async only

Failover scenarios:
  node1 fails: node2 auto-promotes (Patroni, same region)
  Region A fails: manual failover to node3
    - node3 may be slightly behind (async lag)
    - Acceptable RPO for DR scenario

Cross-region replication:
  wal_level = replica + streaming replication
  node3: primary_conninfo points to node1 (or node2 via VIP)
  Network: private WAN link or VPN (not public internet)
  SSL required: hostssl in pg_hba.conf

Monitoring cross-region:
  SELECT replay_lag FROM pg_stat_replication WHERE application_name = 'dr_node3';
  -- Cross-region lag: seconds to minutes (acceptable for DR)
  -- Intra-region lag: milliseconds (for HA)
```

---

### Block 6 Summary — Dots Connected

```
Topic 24 (Backup)
    │
    ├──► Topic 5 (WAL): pg_basebackup uses WAL + base for consistent backup
    ├──► Topic 20 (Partition): Partition-level backup via DETACH
    └──► Topic 25 (PITR): Physical backup = PITR foundation

Topic 25 (PITR)
    │
    ├──► Topic 5 (WAL): WAL archive + base backup = PITR capability
    ├──► Topic 3 (Transaction): Recovery to specific transaction XID
    └──► Topic 26 (Replication): Standby is continuous PITR from primary WAL

Topic 26 (Replication)
    │
    ├──► Topic 1 (Architecture): WAL Sender/Receiver background processes
    ├──► Topic 5 (WAL): Physical replication = WAL streaming
    ├──► Topic 4 (Concurrency): hot_standby_feedback prevents vacuum conflicts
    └──► Topic 27 (Failover): Standby is the failover target

Topic 27 (Failover & HA)
    │
    ├──► Topic 26 (Replication): HA requires replication to standby
    ├──► Topic 30 (Connection Pooling): PgBouncer enables transparent failover
    └──► Topic 28 (Monitoring): pg_stat_replication = HA health monitoring
```
# Senior DBA Master Guide — Block 7
## Monitoring & Maintenance

---

## Topic 28 — System Catalogs & pg_stat Views

### Big Picture: Database-এর X-Ray Vision

```
pg_catalog = PostgreSQL-এর internal metadata store
  Every object: table, index, function, constraint, role
  Everything stored in pg_catalog system tables

pg_stat_* = Runtime statistics views
  What's happening RIGHT NOW
  What happened since last stats reset
  Performance counters, activity, I/O

এই দুটো জানলে:
  Database-এর structure যেকোনো সময় inspect করতে পারবে
  Performance problem যেকোনো জায়গা থেকে diagnose করতে পারবে
  Tool ছাড়াই DBA কাজ করতে পারবে
```

---

### System Catalogs — Database-এর DNA

**Most important catalog tables:**

```sql
-- pg_class: সব database objects (tables, indexes, views, sequences, etc.)
SELECT
  oid,
  relname AS name,
  relnamespace::regnamespace AS schema,
  relkind,       -- r=table, i=index, v=view, m=matview, S=sequence,
                 -- p=partitioned table, I=partitioned index, c=composite type
  relowner::regrole AS owner,
  relpages,      -- disk pages (estimated)
  reltuples,     -- row count (estimated)
  relhasindex,   -- has any indexes?
  relispartition, -- is a partition?
  relpartbound,  -- partition bounds (if partition)
  relfrozenxid,  -- oldest unfrozen XID (XID wraparound tracking)
  reltoastrelid  -- OID of TOAST table
FROM pg_class
WHERE relnamespace = 'public'::regnamespace
  AND relkind = 'r'  -- tables only
ORDER BY relpages DESC;  -- largest tables first

-- pg_attribute: column information
SELECT
  attname AS column_name,
  atttypid::regtype AS data_type,
  attnum AS column_order,
  attnotnull AS not_null,
  atthasdef AS has_default,
  attlen AS storage_len,  -- -1 = variable
  attalign AS alignment,  -- c=char(1), s=short(2), i=int(4), d=double(8)
  attstorage,             -- p=plain, e=extended, m=main, x=external
  attisdropped            -- soft-deleted column (don't show)
FROM pg_attribute
WHERE attrelid = 'orders'::regclass
  AND attnum > 0            -- system columns excluded
  AND NOT attisdropped      -- not dropped
ORDER BY attnum;

-- pg_index: index information
SELECT
  i.indexrelid::regclass AS index_name,
  i.indrelid::regclass AS table_name,
  i.indisprimary AS is_primary,
  i.indisunique AS is_unique,
  i.indisvalid AS is_valid,    -- false = index being built or failed
  i.indisready AS is_ready,    -- false = not ready for queries
  i.indkey AS column_numbers,  -- attnum array
  pg_get_indexdef(i.indexrelid) AS index_definition,
  pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size
FROM pg_index i
WHERE i.indrelid = 'orders'::regclass;

-- pg_constraint: all constraints
SELECT
  conname AS constraint_name,
  contype,  -- p=primary key, u=unique, f=foreign key, c=check, n=not null
  condeferrable,
  condeferred,
  pg_get_constraintdef(oid, true) AS definition
FROM pg_constraint
WHERE conrelid = 'orders'::regclass
ORDER BY contype, conname;

-- pg_proc: functions and procedures
SELECT
  proname AS name,
  pronamespace::regnamespace AS schema,
  proowner::regrole AS owner,
  prolang::regprocedure AS language,
  prokind,        -- f=function, p=procedure, a=aggregate, w=window
  provolatile,    -- i=immutable, s=stable, v=volatile
  prosecdef,      -- security definer?
  proisstrict,    -- returns null if any input null?
  prorettype::regtype AS return_type,
  pronargs AS num_args,
  proargtypes::regtype[] AS arg_types,
  prosrc AS source_code
FROM pg_proc
WHERE pronamespace = 'public'::regnamespace
  AND prokind = 'f'  -- functions only
ORDER BY proname;

-- pg_namespace: schemas
SELECT
  oid,
  nspname AS schema_name,
  nspowner::regrole AS owner,
  nspacl AS access_privileges
FROM pg_namespace
WHERE nspname NOT LIKE 'pg_%'
  AND nspname != 'information_schema'
ORDER BY nspname;

-- pg_roles: all roles/users
SELECT
  rolname,
  rolsuper,
  rolinherit,
  rolcreaterole,
  rolcreatedb,
  rolcanlogin,
  rolreplication,
  rolbypassrls,
  rolconnlimit,
  rolvaliduntil,
  rolpassword IS NOT NULL AS has_password
FROM pg_roles
ORDER BY rolcanlogin DESC, rolname;

-- pg_settings: all configuration parameters
SELECT
  name,
  setting,
  unit,
  category,
  short_desc,
  context,      -- internal, postmaster, sighup, superuser, user, backend
  source,       -- default, configuration file, command line, etc.
  sourcefile,   -- config file path if set there
  sourceline    -- line number in config file
FROM pg_settings
WHERE source != 'default'  -- only non-default settings
ORDER BY category, name;
```

---

### pg_stat Views — Runtime Intelligence

**Activity and Connections:**

```sql
-- pg_stat_activity: current sessions
SELECT
  pid,
  usename AS username,
  application_name,
  client_addr,
  client_port,
  backend_start,
  xact_start,
  query_start,
  state_change,
  state,                -- active, idle, idle in transaction, idle in transaction (aborted)
  wait_event_type,      -- Lock, LWLock, IO, Client, Extension, etc.
  wait_event,           -- specific event name
  now() - query_start AS query_duration,
  now() - xact_start AS transaction_duration,
  left(query, 100) AS query_preview,  -- first 100 chars
  backend_type          -- client backend, autovacuum worker, background worker, etc.
FROM pg_stat_activity
WHERE pid != pg_backend_pid()  -- exclude current session
ORDER BY query_duration DESC NULLS LAST;

-- Long-running queries:
SELECT pid, usename, state, wait_event_type, wait_event,
  now() - query_start AS duration,
  query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '30 seconds'
  AND query NOT LIKE '%pg_stat_activity%'  -- exclude this query
ORDER BY duration DESC;

-- Idle in transaction (dangerous — holding locks, blocking vacuum):
SELECT pid, usename, state,
  now() - xact_start AS idle_duration,
  query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '5 minutes'
ORDER BY idle_duration DESC;

-- Lock waits (who is blocking whom):
SELECT
  blocked.pid AS blocked_pid,
  blocked.usename AS blocked_user,
  blocked.query AS blocked_query,
  now() - blocked.query_start AS blocked_duration,
  blocking.pid AS blocking_pid,
  blocking.usename AS blocking_user,
  blocking.query AS blocking_query,
  now() - blocking.query_start AS blocking_duration
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0
ORDER BY blocked_duration DESC;

-- Kill a specific query (graceful):
SELECT pg_cancel_backend(pid) FROM pg_stat_activity
WHERE pid = 12345;
-- pg_cancel_backend: sends SIGINT, query cancelled, transaction can continue

-- Kill a connection (force):
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE pid = 12345;
-- pg_terminate_backend: sends SIGTERM, connection terminated

-- Kill all idle-in-transaction older than 10 minutes:
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '10 minutes';
```

**Database-level Statistics:**

```sql
-- pg_stat_database: per-database statistics
SELECT
  datname,
  numbackends AS active_connections,
  xact_commit AS commits,
  xact_rollback AS rollbacks,
  round(xact_rollback::numeric /
    nullif(xact_commit + xact_rollback, 0) * 100, 2) AS rollback_pct,
  blks_read AS disk_reads,
  blks_hit AS buffer_hits,
  round(blks_hit::numeric /
    nullif(blks_hit + blks_read, 0) * 100, 2) AS cache_hit_pct,
  tup_returned AS rows_returned,
  tup_fetched AS rows_fetched,
  tup_inserted AS rows_inserted,
  tup_updated AS rows_updated,
  tup_deleted AS rows_deleted,
  conflicts AS replication_conflicts,
  temp_files AS temp_files_created,
  pg_size_pretty(temp_bytes) AS temp_space_used,
  deadlocks,
  stats_reset                -- when stats were last reset
FROM pg_stat_database
WHERE datname = current_database();

-- Reset stats (after tuning changes, fresh baseline):
SELECT pg_stat_reset();  -- current database
-- Or per-table:
SELECT pg_stat_reset_single_table_counters('orders'::regclass);
```

**Table-level Statistics:**

```sql
-- pg_stat_user_tables: table access patterns
SELECT
  schemaname,
  relname AS table_name,
  seq_scan,                  -- sequential scans (high = missing index?)
  seq_tup_read,              -- rows read by seq scan
  idx_scan,                  -- index scans
  idx_tup_fetch,             -- rows fetched by index scans
  n_tup_ins AS inserts,
  n_tup_upd AS updates,
  n_tup_del AS deletes,
  n_tup_hot_upd AS hot_updates,  -- HOT updates (good!)
  n_live_tup AS live_rows,
  n_dead_tup AS dead_rows,
  n_mod_since_analyze,       -- modifications since last analyze
  n_ins_since_vacuum,        -- inserts since last vacuum
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze,
  vacuum_count,
  autovacuum_count,
  analyze_count,
  autoanalyze_count
FROM pg_stat_user_tables
ORDER BY seq_scan DESC  -- tables with most sequential scans
LIMIT 20;

-- Tables with high sequential scan ratio (potential missing indexes):
SELECT
  relname,
  seq_scan,
  idx_scan,
  CASE WHEN seq_scan + idx_scan > 0
    THEN round(seq_scan::numeric / (seq_scan + idx_scan) * 100, 2)
    ELSE 0 END AS seq_scan_pct,
  n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 100  -- enough activity to be meaningful
  AND n_live_tup > 10000         -- not tiny tables (seq scan is fine for tiny)
ORDER BY seq_scan_pct DESC;

-- pg_statio_user_tables: I/O statistics
SELECT
  relname,
  heap_blks_read AS heap_disk_reads,
  heap_blks_hit AS heap_buffer_hits,
  round(heap_blks_hit::numeric /
    nullif(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS heap_cache_pct,
  idx_blks_read AS index_disk_reads,
  idx_blks_hit AS index_buffer_hits,
  round(idx_blks_hit::numeric /
    nullif(idx_blks_hit + idx_blks_read, 0) * 100, 2) AS index_cache_pct,
  toast_blks_read,
  toast_blks_hit
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;
```

**Index Statistics:**

```sql
-- pg_stat_user_indexes: index usage
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan AS times_used,
  idx_tup_read AS tuples_read,
  idx_tup_fetch AS tuples_fetched,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Unused indexes (prime candidates for removal):
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan AS usage_count,
  pg_size_pretty(pg_relation_size(indexrelid)) AS wasted_space
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%_pkey'       -- keep primary keys
  AND indexname NOT LIKE '%_unique'     -- keep unique constraints
ORDER BY pg_relation_size(indexrelid) DESC;

-- Duplicate/redundant indexes:
-- Index (a) is redundant if (a, b) exists AND (a) alone not used separately
SELECT
  t.tablename,
  ix.indexname AS smaller_index,
  ix2.indexname AS larger_index,
  pg_size_pretty(pg_relation_size(ix.indexrelid)) AS smaller_size
FROM pg_stat_user_indexes ix
JOIN pg_index i ON i.indexrelid = ix.indexrelid
JOIN pg_stat_user_indexes ix2 ON ix2.tablename = ix.tablename
  AND ix2.indexname != ix.indexname
JOIN pg_index i2 ON i2.indexrelid = ix2.indexrelid
JOIN pg_tables t ON t.tablename = ix.tablename
WHERE array_length(i.indkey, 1) < array_length(i2.indkey, 1)
  AND i.indkey[0] = i2.indkey[0]  -- same leading column
ORDER BY t.tablename, ix.indexname;
```

**Background Writer and Checkpoint Statistics:**

```sql
-- pg_stat_bgwriter: checkpoint and background write health
SELECT
  checkpoints_timed,    -- scheduled checkpoints (checkpoint_timeout)
  checkpoints_req,      -- forced checkpoints (max_wal_size exceeded)
  -- checkpoints_req high = max_wal_size too small, increase it
  checkpoint_write_time, -- ms spent writing dirty pages
  checkpoint_sync_time,  -- ms spent in fsync
  buffers_checkpoint,    -- pages written by checkpointer
  buffers_clean,         -- pages written by bgwriter
  maxwritten_clean,      -- bgwriter reached max (bgwriter_lru_maxpages)
  -- maxwritten_clean high = increase bgwriter_lru_maxpages
  buffers_backend,       -- pages written directly by backend processes
  -- buffers_backend high = bgwriter not keeping up
  buffers_backend_fsync, -- fsyncs by backend (should be 0)
  buffers_alloc,         -- new buffers allocated
  stats_reset
FROM pg_stat_bgwriter;

-- pg_stat_wal: WAL activity (PostgreSQL 14+)
SELECT
  wal_records,          -- total WAL records
  wal_fpi,              -- full page images (from full_page_writes)
  wal_bytes,            -- total WAL bytes
  wal_buffers_full,     -- times WAL buffer was full (increase wal_buffers if high)
  wal_write,            -- WAL writes to disk
  wal_sync,             -- WAL fsyncs
  wal_write_time,       -- ms spent writing
  wal_sync_time         -- ms spent in fsync (high = I/O bottleneck)
FROM pg_stat_wal;
```

---

### pg_stat_statements — Query Performance Analytics

```sql
-- Enable (requires restart):
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
-- pg_stat_statements.track = all  (top, all, none)
-- pg_stat_statements.max = 10000  (max queries tracked)

CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top slow queries by average execution time:
SELECT
  round(mean_exec_time::numeric, 2) AS avg_ms,
  round(stddev_exec_time::numeric, 2) AS stddev_ms,
  calls,
  round(total_exec_time::numeric, 2) AS total_ms,
  round(total_exec_time::numeric / calls / 1000, 2) AS avg_seconds,
  rows / calls AS avg_rows,
  shared_blks_hit + shared_blks_read AS total_blks,
  round(shared_blks_hit::numeric /
    nullif(shared_blks_hit + shared_blks_read, 0) * 100, 2) AS cache_hit_pct,
  left(query, 200) AS query
FROM pg_stat_statements
WHERE calls > 10          -- at least 10 calls (meaningful)
  AND mean_exec_time > 100  -- > 100ms average
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Top queries by total time (biggest performance impact):
SELECT
  round(total_exec_time::numeric, 2) AS total_ms,
  calls,
  round(mean_exec_time::numeric, 2) AS avg_ms,
  left(query, 200) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Queries with high I/O (cache misses):
SELECT
  round(mean_exec_time::numeric, 2) AS avg_ms,
  shared_blks_read AS disk_reads,
  shared_blks_hit AS cache_hits,
  round(shared_blks_hit::numeric /
    nullif(shared_blks_hit + shared_blks_read, 0) * 100, 2) AS cache_pct,
  calls,
  left(query, 200) AS query
FROM pg_stat_statements
WHERE shared_blks_read > 1000  -- significant disk reads
ORDER BY shared_blks_read DESC
LIMIT 20;

-- Queries with high variance (inconsistent performance):
SELECT
  round(mean_exec_time::numeric, 2) AS avg_ms,
  round(stddev_exec_time::numeric, 2) AS stddev_ms,
  round(stddev_exec_time / nullif(mean_exec_time, 0) * 100, 2) AS cv_pct,
  -- coefficient of variation > 100% = highly variable
  calls,
  left(query, 200) AS query
FROM pg_stat_statements
WHERE calls > 50
  AND mean_exec_time > 10
ORDER BY cv_pct DESC
LIMIT 20;

-- Reset stats for fresh baseline (after changes):
SELECT pg_stat_statements_reset();
```

---

### Locks Deep Dive

```sql
-- pg_locks: all current lock information
SELECT
  locktype,         -- relation, tuple, transactionid, page, advisory, etc.
  relation::regclass AS locked_object,
  mode,             -- AccessShareLock, RowExclusiveLock, AccessExclusiveLock, etc.
  granted,          -- true = lock held, false = waiting
  pid,
  transactionid,
  virtualtransaction,
  pg_stat_activity.query,
  pg_stat_activity.state
FROM pg_locks
JOIN pg_stat_activity USING (pid)
WHERE NOT granted  -- waiting locks (most interesting)
   OR mode IN ('AccessExclusiveLock', 'ShareLock', 'ExclusiveLock')
ORDER BY NOT granted DESC;

-- Full lock graph (who is blocking whom at all levels):
WITH lock_info AS (
  SELECT
    l.pid,
    l.locktype,
    l.relation,
    l.mode,
    l.granted,
    a.usename,
    a.state,
    a.query,
    now() - a.query_start AS query_age
  FROM pg_locks l
  JOIN pg_stat_activity a ON a.pid = l.pid
)
SELECT
  waiting.pid AS waiting_pid,
  waiting.usename AS waiting_user,
  waiting.query AS waiting_query,
  waiting.query_age AS waiting_duration,
  blocking.pid AS blocking_pid,
  blocking.usename AS blocking_user,
  blocking.query AS blocking_query,
  blocking.query_age AS blocking_duration,
  waiting.locktype AS lock_type,
  waiting.mode AS requested_mode
FROM lock_info AS waiting
JOIN lock_info AS blocking
  ON blocking.relation = waiting.relation
  AND blocking.granted = true
  AND waiting.granted = false
  AND blocking.pid != waiting.pid
ORDER BY waiting_duration DESC;

-- Table-level locks currently held:
SELECT
  a.pid,
  a.usename,
  l.mode,
  l.granted,
  a.query,
  now() - a.query_start AS duration
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.relation = 'orders'::regclass
ORDER BY NOT l.granted DESC, duration DESC;
```

---

### Comprehensive Monitoring Dashboard Query

```sql
-- Single query: database health overview
WITH db_stats AS (
  SELECT
    datname,
    numbackends AS connections,
    xact_commit AS commits,
    xact_rollback AS rollbacks,
    round(blks_hit::numeric / nullif(blks_hit + blks_read, 0) * 100, 2) AS cache_hit,
    deadlocks,
    temp_files,
    pg_size_pretty(pg_database_size(datname)) AS db_size
  FROM pg_stat_database
  WHERE datname = current_database()
),
table_stats AS (
  SELECT
    sum(seq_scan) AS total_seq_scans,
    sum(n_dead_tup) AS total_dead_tuples,
    sum(n_live_tup) AS total_live_tuples
  FROM pg_stat_user_tables
),
index_stats AS (
  SELECT count(*) AS unused_indexes
  FROM pg_stat_user_indexes
  WHERE idx_scan = 0 AND indexname NOT LIKE '%_pkey'
),
bgwriter AS (
  SELECT
    checkpoints_req,
    checkpoints_timed,
    round(checkpoints_req::numeric /
      nullif(checkpoints_req + checkpoints_timed, 0) * 100, 2) AS forced_ckpt_pct
  FROM pg_stat_bgwriter
)
SELECT
  d.*,
  t.total_seq_scans,
  t.total_dead_tuples,
  round(t.total_dead_tuples::numeric /
    nullif(t.total_live_tuples, 0) * 100, 2) AS dead_tuple_pct,
  i.unused_indexes,
  b.checkpoints_req AS forced_checkpoints,
  b.forced_ckpt_pct
FROM db_stats d
CROSS JOIN table_stats t
CROSS JOIN index_stats i
CROSS JOIN bgwriter b;
```

---

## Topic 29 — Bloat, Autovacuum & Table Maintenance

### Big Picture: Database-এর Long-term Health System

```
Maintenance ছাড়া PostgreSQL-এ কী হয়:

Week 1-4:   Moderate bloat builds up (dead tuples from UPDATE/DELETE)
Month 2-3:  Query performance degrades (seq scans slower, index bloat)
Month 4-6:  VACUUM struggles (autovacuum can't keep up)
Month 6+:   XID age warning messages in log
Year 1-2:   XID age critical — approaching wraparound
Year 2+:    Database refuses connections to prevent data loss!

Senior DBA: proactive maintenance, not reactive firefighting
```

---

### Bloat — Types and Detection

**Table bloat:**

```sql
-- Method 1: pg_stat_user_tables (fast, approximate)
SELECT
  schemaname,
  relname AS table_name,
  n_live_tup,
  n_dead_tup,
  round(n_dead_tup::numeric /
    nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS bloat_pct,
  pg_size_pretty(pg_relation_size(relid)) AS table_size,
  last_autovacuum,
  autovacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- Method 2: pgstattuple (exact, brief scan)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
  table_len / 1024 / 1024 AS size_mb,
  tuple_count AS live_tuples,
  tuple_percent AS live_pct,
  dead_tuple_count,
  dead_tuple_percent AS dead_pct,
  free_space / 1024 AS free_kb,
  free_percent
FROM pgstattuple('orders');

-- Method 3: Bloat estimation query (no lock, reasonably accurate)
-- Based on: expected row size vs actual page usage
WITH table_bloat AS (
  SELECT
    schemaname,
    tablename,
    (
      (relpages::bigint * 8192)  -- actual size
      - (reltuples * (
          23 +  -- tuple header
          CASE WHEN avg_width IS NULL THEN 0 ELSE avg_width END  -- avg row data
        ))
    ) / 1024 / 1024 AS estimated_bloat_mb,
    pg_size_pretty(pg_relation_size(
      (schemaname || '.' || tablename)::regclass
    )) AS actual_size
  FROM pg_tables
  JOIN pg_class ON relname = tablename
    AND relnamespace::regnamespace::text = schemaname
  LEFT JOIN (
    SELECT attrelid, sum(avg_width) AS avg_width
    FROM pg_stats
    JOIN pg_attribute ON attname = attname AND attrelid = (tablename || '.' || attname)::regclass
    GROUP BY attrelid
  ) s ON s.attrelid = pg_class.oid
  WHERE schemaname = 'public'
)
SELECT * FROM table_bloat
WHERE estimated_bloat_mb > 100  -- > 100MB bloat
ORDER BY estimated_bloat_mb DESC;
```

**Index bloat:**

```sql
-- pgstattuple for indexes:
SELECT * FROM pgstatindex('idx_orders_user_id');
-- leaf_pages: pages with actual data
-- empty_pages: pages with no data (after deletes)
-- deleted_pages: pages marked for reuse
-- avg_leaf_density: how full leaf pages are (< 70% = bloated)
-- leaf_fragmentation: out-of-order percentage

-- Quick index bloat check:
SELECT
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
  round(
    (pg_relation_size(indexrelid)::float /
     pg_total_relation_size(indrelid)::float) * 100,
  2) AS pct_of_table
FROM pg_stat_user_indexes
JOIN pg_index ON indexrelid = pg_stat_user_indexes.indexrelid
WHERE tablename = 'orders'
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

### Autovacuum — Deep Configuration

**How autovacuum decides when to run:**

```
For each table, autovacuum checks:

VACUUM trigger:
  dead_tuples > autovacuum_vacuum_threshold +
                autovacuum_vacuum_scale_factor × n_live_tup

  Default: 50 + 0.2 × reltuples
  
  1M row table: 50 + 200,000 = 200,050 dead tuples before vacuum!
  (20% bloat before cleanup)

ANALYZE trigger:
  modifications > autovacuum_analyze_threshold +
                  autovacuum_analyze_scale_factor × n_live_tup

  Default: 50 + 0.1 × reltuples
  
  1M row table: 50 + 100,000 = 100,050 modifications before analyze

FREEZE trigger (regardless of dead tuples):
  relfrozenxid age > autovacuum_freeze_max_age (default 200M)
  → Aggressive vacuum to freeze old tuples
```

**Autovacuum throttling:**

```
autovacuum_vacuum_cost_delay = 2ms    (sleep between work units)
autovacuum_vacuum_cost_limit = 200    (work units per sleep)

Work unit costs:
  vacuum_cost_page_hit = 1    (page from buffer: cheap)
  vacuum_cost_page_miss = 10  (page from disk: expensive)
  vacuum_cost_page_dirty = 20 (dirty page (needs write): most expensive)

Throttle calculation:
  After every 200 cost units: sleep 2ms

  Full speed example:
  200 buffer hits / 2ms = 100,000 hits/second
  
  Disk-heavy vacuum:
  200/10 = 20 disk reads / 2ms = 10,000 disk reads/second
  
  Mix of reads and dirty pages:
  200/(10+20) = ~6 pages / 2ms = 3,000 pages/second

  Too aggressive = I/O interference with live queries
  Too throttled = vacuum falls behind
```

```sql
-- Per-table autovacuum tuning:
ALTER TABLE large_orders SET (
  autovacuum_vacuum_scale_factor = 0.01,   -- 1% (not 20%)
  autovacuum_vacuum_threshold = 500,        -- 500 dead tuples minimum
  autovacuum_analyze_scale_factor = 0.005, -- 0.5%
  autovacuum_analyze_threshold = 250,
  autovacuum_vacuum_cost_delay = 2,         -- 2ms (same as global)
  autovacuum_vacuum_cost_limit = 800,       -- 4x more work per cycle (faster!)
  autovacuum_freeze_min_age = 10000000,     -- 10M (freeze sooner)
  autovacuum_freeze_max_age = 150000000     -- 150M (force freeze before global max)
);

-- Global autovacuum settings (postgresql.conf):
autovacuum = on
autovacuum_max_workers = 6              -- more workers for busy DB
autovacuum_naptime = 30s               -- check tables every 30s (not 1min)
autovacuum_vacuum_cost_delay = 2ms     -- PostgreSQL 13+ default
autovacuum_vacuum_cost_limit = 400     -- more work per cycle (2x default)
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.05  -- 5% (global reduction from 20%)
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.05

-- What's running right now:
SELECT
  pid,
  datname,
  usename,
  wait_event_type,
  wait_event,
  now() - xact_start AS duration,
  query
FROM pg_stat_activity
WHERE backend_type = 'autovacuum worker'
ORDER BY duration DESC;

-- Which tables need vacuum most urgently:
SELECT
  schemaname || '.' || relname AS table_name,
  n_dead_tup,
  n_live_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup, 0) * 100, 2) AS dead_ratio,
  last_autovacuum,
  -- When should autovacuum trigger?
  autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * n_live_tup AS threshold
FROM pg_stat_user_tables
JOIN pg_class ON relname = pg_class.relname
  AND relnamespace::regnamespace::text = schemaname
CROSS JOIN (
  SELECT
    current_setting('autovacuum_vacuum_threshold')::int AS autovacuum_vacuum_threshold,
    current_setting('autovacuum_vacuum_scale_factor')::float AS autovacuum_vacuum_scale_factor
) settings
WHERE n_dead_tup > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * n_live_tup
ORDER BY n_dead_tup DESC;
```

---

### Long Transaction and Vacuum Conflict

```sql
-- Long transactions BLOCK vacuum!
-- Vacuum cannot remove dead tuples visible to ANY active transaction.

-- Find transactions blocking vacuum:
SELECT
  pid,
  usename,
  state,
  now() - xact_start AS transaction_age,
  left(query, 100) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND now() - xact_start > interval '10 minutes'
  AND state != 'idle'
ORDER BY xact_start;

-- Find oldest transaction (vacuum blocker):
SELECT
  min(xact_start) AS oldest_transaction,
  now() - min(xact_start) AS age
FROM pg_stat_activity
WHERE xact_start IS NOT NULL;

-- The xmin horizon: oldest snapshot vacuum must preserve
SELECT
  backend_xmin,
  backend_xid,
  pg_stat_activity.pid,
  state,
  query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC;

-- Vacuum cannot proceed past this horizon!
-- Kill the old transaction (after verification):
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid = <blocking_pid>;

-- Idle in transaction timeout (prevention):
-- postgresql.conf:
-- idle_in_transaction_session_timeout = '10min'
-- After 10 minutes idle in transaction: auto-terminate
```

---

### pg_repack — Online Bloat Removal

```bash
# pg_repack: online VACUUM FULL alternative
# No long exclusive lock! Read-write continues during repack.

# Install:
apt install postgresql-17-repack
# Or from source: https://github.com/reorg/pg_repack

# How pg_repack works:
# 1. Create new table (empty copy)
# 2. INSERT all live tuples into new table (compacted)
# 3. Create trigger on original table: capture new changes
# 4. Apply captured changes to new table
# 5. Short exclusive lock: swap tables (old → new)
# 6. Drop old (bloated) table

# Repack single table:
pg_repack \
  -h localhost \
  -U postgres \
  -d mydb \
  -t orders \
  --no-order              # don't reorder (faster)
  # or with cluster:
  # --order-by "created_at"  # physical order = created_at (helps range scans)

# Repack entire database:
pg_repack -h localhost -U postgres -d mydb

# Repack indexes only (faster, no table lock):
pg_repack -h localhost -U postgres -d mydb -t orders --only-indexes

# Repack specific index:
pg_repack -h localhost -U postgres -d mydb \
  --index idx_orders_user_id

# Wait mode (don't start if lock conflicts):
pg_repack -h localhost -U postgres -d mydb -t orders \
  --wait-timeout 30       # wait max 30s for lock, then abort

# Dry run (show what would be done):
pg_repack -h localhost -U postgres -d mydb -t orders \
  --dry-run

# When to use pg_repack:
# dead_tuple_percent > 20% AND table is large (> 1GB)
# VACUUM cannot reclaim space (bloat not decreasing)
# Regular VACUUM is falling behind
# After large bulk DELETE

# When NOT to use pg_repack:
# Small table: VACUUM FULL (brief downtime) is simpler
# Active maintenance window: VACUUM FULL is OK then
# Very high transaction rate: extra trigger overhead
```

---

### XID Wraparound — Emergency Response

```sql
-- Monitoring XID age (critical metric!):
SELECT
  datname,
  age(datfrozenxid) AS xid_age,
  pg_size_pretty(pg_database_size(datname)) AS db_size,
  CASE
    WHEN age(datfrozenxid) > 1800000000 THEN 'CRITICAL'
    WHEN age(datfrozenxid) > 1500000000 THEN 'WARNING'
    WHEN age(datfrozenxid) > 1000000000 THEN 'WATCH'
    ELSE 'OK'
  END AS status
FROM pg_database
WHERE datname NOT IN ('template0')
ORDER BY age(datfrozenxid) DESC;

-- Per-table XID age:
SELECT
  n.nspname AS schema,
  c.relname AS table_name,
  age(c.relfrozenxid) AS xid_age,
  pg_size_pretty(pg_total_relation_size(c.oid)) AS table_size,
  CASE
    WHEN age(c.relfrozenxid) > 1800000000 THEN '!!! CRITICAL !!!'
    WHEN age(c.relfrozenxid) > 1500000000 THEN '!! WARNING !!'
    WHEN age(c.relfrozenxid) > 1000000000 THEN '! WATCH !'
    ELSE 'OK'
  END AS status
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'm')  -- tables and materialized views
  AND n.nspname NOT IN ('pg_catalog', 'information_schema', 'pg_toast')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 30;

-- Emergency freeze (if age approaching 2B):
-- First: check what's blocking vacuum
SELECT pid, state, xact_start, query FROM pg_stat_activity
WHERE xact_start IS NOT NULL ORDER BY xact_start;

-- Kill long transactions:
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE pid != pg_backend_pid()
  AND state IN ('active', 'idle in transaction')
  AND now() - xact_start > interval '5 minutes';

-- Emergency VACUUM FREEZE (affects performance, but prevents data loss):
VACUUM FREEZE VERBOSE <critical_table>;
-- Or entire database (very slow for large DB):
VACUUM FREEZE;

-- After emergency, investigate root cause:
-- Why wasn't autovacuum keeping up?
-- Long transactions? autovacuum disabled? scale_factor too high?
-- Fix the root cause, then tune autovacuum.
```

---

## Topic 30 — Connection Pooling

### Big Picture: Why PostgreSQL Needs a Pooler

```
PostgreSQL process model (Topic 1):
  Each connection = 1 OS process
  Process overhead: ~5-10MB RAM per process
  OS scheduling: context switch cost per process

Without pooler:
  Application: 500 concurrent requests
  → 500 PostgreSQL processes
  → 500 × 8MB = 4GB RAM just for process overhead!
  → OS scheduler: 500 processes → overhead
  → PostgreSQL shared memory lock contention
  → Performance DEGRADES with too many connections!

Optimal PostgreSQL connections:
  Research: peak performance ≈ 2-4 × CPU cores
  16-core server: ~64 connections optimal
  Beyond 200 connections: diminishing returns, increasing overhead

With PgBouncer:
  Application: 1000 connections → PgBouncer
  PgBouncer: 50 connections → PostgreSQL
  Application thinks it has 1000 connections
  PostgreSQL handles 50 (optimal)
  PgBouncer: multiplexes (queues if all 50 busy)
```

---

### PgBouncer — Architecture and Internals

**How PgBouncer works:**

```
Client connects to PgBouncer (port 6432)
PgBouncer: authentication, pool lookup

Pool modes determine when server connection is released:

SESSION mode:
  Client connects → server connection assigned
  Client disconnects → server connection released back to pool
  Client lifetime = server connection lifetime
  Good for: SET commands, prepared statements, LISTEN
  Not good for: many short-lived connections (no multiplexing benefit)

TRANSACTION mode (recommended):
  Client begins transaction → server connection assigned
  Transaction commits/rolls back → server connection released
  One server connection can serve MANY clients (sequentially)
  Good for: OLTP, short transactions
  Bad for: SET (session-level config lost between transactions!),
           prepared statements (may go to different server connection),
           LISTEN/NOTIFY

STATEMENT mode:
  Server connection released after EACH statement
  Transactions spanning multiple statements: IMPOSSIBLE
  Rarely used: only for truly stateless queries
```

```ini
# pgbouncer.ini — Production Configuration

[databases]
# Format: alias = connection_string
mydb = host=postgresql_primary port=5432 dbname=mydb
mydb_ro = host=postgresql_replica port=5432 dbname=mydb

# Database-specific pool settings:
mydb_analytics = host=analytics_host port=5432 dbname=analytics
  pool_size=5 reserve_pool_size=2

[pgbouncer]
# Network
listen_port = 6432
listen_addr = *                  # or specific IP: 10.0.1.100

# Authentication
auth_type = scram-sha-256        # recommended (PostgreSQL 10+)
auth_file = /etc/pgbouncer/userlist.txt
# auth_query = SELECT p_user, p_password FROM pg_shadow WHERE usename=$1
# auth_user = pgbouncer_auth_user  # user to run auth_query

# Pooling
pool_mode = transaction          # session / transaction / statement
max_client_conn = 2000           # max simultaneous client connections
default_pool_size = 20           # server connections per database+user combo

# Reserve pool (burst handling):
reserve_pool_size = 5            # extra connections for bursts
reserve_pool_timeout = 3         # seconds to wait before using reserve pool

# Timeouts
server_idle_timeout = 600        # close idle server connections after 10min
client_idle_timeout = 0          # disable (or set high for long-lived clients)
client_login_timeout = 15        # abort slow logins
query_wait_timeout = 120         # abort if client waits > 2min for server conn
query_timeout = 0                # no per-query timeout (set in PostgreSQL)

# Connection limits
server_reset_query = DISCARD ALL  # reset server connection state between clients
                                   # transaction mode: not needed (auto-reset)
server_check_query = select 1      # health check query
server_check_delay = 30            # check every 30 seconds

# TLS
# server_tls_sslmode = require
# server_tls_ca_file = /etc/ssl/certs/ca.crt
# client_tls_sslmode = require
# client_tls_key_file = /etc/pgbouncer/server.key
# client_tls_cert_file = /etc/pgbouncer/server.crt

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60               # stats log interval

# Admin
admin_users = pgbouncer_admin
stats_users = pgbouncer_stats
```

---

### PgBouncer — Transaction Mode Gotchas

```sql
-- Gotcha 1: SET commands don't persist
-- Session mode: SET search_path → stays for session
-- Transaction mode: SET search_path → only for this transaction!

-- Problem in application:
SET search_path = tenant_acme, public;  -- transaction ends → reset!
SELECT * FROM users;  -- might use wrong schema!

-- Fix: SET LOCAL (transaction-scoped, intentional):
BEGIN;
SET LOCAL search_path = tenant_acme, public;
SELECT * FROM users;  -- correct schema
COMMIT;
-- After commit: search_path resets anyway → no problem

-- Gotcha 2: Prepared statements
-- In session mode: PREPARE name AS SELECT ... → stays in session
-- In transaction mode: different transactions may go to different server conns!
-- Prepared statement from one server conn not visible on another

-- Fix: Use PostgreSQL protocol-level prepared statements
-- (most drivers handle this transparently)
-- Or: use query-level prepared statements (parse each time)
-- Or: use session mode if prepared statements critical

-- Gotcha 3: LISTEN/NOTIFY
-- LISTEN requires persistent server connection
-- Transaction mode: connection not persistent!
-- Fix: Use session mode for LISTEN connections
-- Separate pool: one for normal queries (transaction mode), one for NOTIFY (session mode)

-- Gotcha 4: Advisory locks
-- pg_advisory_lock(): session-level advisory lock
-- Transaction mode: server connection may change → lock released!
-- Fix: pg_advisory_xact_lock() (transaction-scoped) — works with transaction mode

-- Gotcha 5: Temporary tables
-- Temp tables: session-scoped
-- Transaction mode: next transaction on different server conn → temp table gone!
-- Fix: session mode for temp table sessions
-- Or: use real tables with session-identifying column

-- server_reset_query:
-- DISCARD ALL: clears session state between clients
-- Includes: SET parameters, prepared statements, advisory locks, temp tables
-- Ensure: each client gets clean state
-- Cost: small overhead per client assignment
```

---

### Connection Pool Sizing

```sql
-- Optimal pool size formula:
-- pool_size = (core_count × 2) + effective_spindle_count
-- For 16 cores, 2 SSDs: (16 × 2) + 2 = 34

-- But in practice: measure, don't guess

-- Current connection usage:
SELECT
  count(*) AS total_connections,
  count(*) FILTER (WHERE state = 'active') AS active,
  count(*) FILTER (WHERE state = 'idle') AS idle,
  count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_txn,
  count(*) FILTER (WHERE wait_event_type = 'Lock') AS lock_waiting
FROM pg_stat_activity
WHERE datname = current_database();

-- PgBouncer stats (from pgbouncer admin console):
-- SHOW POOLS;
-- database  | user       | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait
-- mydb      | app_user   | 45        | 3          | 20        | 0       | 5       | 0         | 0        | 0.15
--
-- cl_active: clients with server connection assigned
-- cl_waiting: clients waiting for server connection (> 0 = pool too small!)
-- sv_active: server connections active
-- sv_idle: idle server connections
-- maxwait: longest wait time (> 1s = pool too small!)

-- If maxwait > 1 second: increase default_pool_size
-- If sv_idle always 0 and cl_waiting > 0: definitely increase pool_size
-- If sv_idle always high: decrease pool_size (wasting connections)

-- Max connections PostgreSQL can handle:
SHOW max_connections;
-- subtract superuser_reserved_connections:
SHOW superuser_reserved_connections;

-- Total available for applications:
SELECT current_setting('max_connections')::int -
       current_setting('superuser_reserved_connections')::int AS available_connections;

-- PgBouncer total server connections:
-- Sum of all pool_sizes should not exceed PostgreSQL max_connections!
```

---

### PgBouncer Monitoring and Operations

```sql
-- PgBouncer has its own admin console:
-- psql -h pgbouncer_host -p 6432 -U pgbouncer_admin pgbouncer

-- In PgBouncer admin console:
SHOW STATS;
-- total_xact_count, total_query_count, total_received, total_sent
-- total_xact_time, total_query_time, total_wait_time

SHOW POOLS;
-- Connection pool status per database+user

SHOW CLIENTS;
-- All connected clients

SHOW SERVERS;
-- All server connections

SHOW DATABASES;
-- Database configuration

SHOW CONFIG;
-- Current configuration

-- Operations:
RELOAD;          -- reload config without restart
RECONNECT mydb;  -- reconnect all server connections to mydb
PAUSE mydb;      -- pause accepting queries on mydb (maintenance)
RESUME mydb;     -- resume
KILL mydb;       -- close all connections to mydb
SUSPEND;         -- suspend all pools (maintenance)

-- Force pool reconnect (after primary failover):
RECONNECT;  -- all databases

-- In application server shell:
psql -h localhost -p 6432 -U admin pgbouncer -c "SHOW POOLS"
psql -h localhost -p 6432 -U admin pgbouncer -c "RECONNECT"
```

---

## Topic 31 — Capacity Planning

### Big Picture: Proactive vs Reactive

```
Reactive DBA: "Database is slow, what's wrong?"
Proactive DBA: "Based on growth trends, we'll need 2x more storage in 6 months."

Capacity planning requires:
  1. Baseline metrics (current state)
  2. Growth trend (historical data)
  3. Projection (forecast)
  4. Headroom calculation (when to act)
  5. Action plan (what to do before limit)
```

---

### Storage Capacity

```sql
-- Current storage usage:
SELECT
  pg_size_pretty(pg_database_size(current_database())) AS database_size,
  pg_size_pretty(sum(pg_total_relation_size(oid))) AS all_tables_total
FROM pg_class
WHERE relkind IN ('r', 'm');

-- Per-table storage breakdown:
SELECT
  schemaname,
  relname AS table_name,
  pg_size_pretty(pg_relation_size(relid)) AS data_size,
  pg_size_pretty(pg_indexes_size(relid)) AS index_size,
  pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
  round(pg_total_relation_size(relid)::numeric /
    pg_database_size(current_database()) * 100, 2) AS pct_of_db,
  n_live_tup,
  pg_size_pretty(pg_total_relation_size(relid) /
    nullif(n_live_tup, 0)) AS bytes_per_row
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;

-- WAL size:
SELECT pg_size_pretty(sum(size)) AS wal_size
FROM pg_ls_waldir();

-- Track size over time (create history table):
CREATE TABLE size_history (
  captured_at timestamptz DEFAULT now(),
  table_name text,
  total_size_bytes bigint
);

-- Insert daily via cron:
INSERT INTO size_history (table_name, total_size_bytes)
SELECT relname, pg_total_relation_size(relid)
FROM pg_stat_user_tables;

-- Growth trend:
SELECT
  table_name,
  date_trunc('week', captured_at) AS week,
  max(total_size_bytes) AS size_bytes,
  pg_size_pretty(max(total_size_bytes)) AS size
FROM size_history
GROUP BY table_name, date_trunc('week', captured_at)
ORDER BY table_name, week;

-- Projected growth (simple linear regression):
WITH weekly_sizes AS (
  SELECT
    table_name,
    extract(epoch FROM captured_at) / 86400 AS day_number,
    total_size_bytes
  FROM size_history
  WHERE captured_at > now() - interval '90 days'
),
regression AS (
  SELECT
    table_name,
    regr_slope(total_size_bytes, day_number) AS bytes_per_day,
    max(total_size_bytes) AS current_size
  FROM weekly_sizes
  GROUP BY table_name
)
SELECT
  table_name,
  pg_size_pretty(current_size::bigint) AS current_size,
  pg_size_pretty((bytes_per_day * 30)::bigint) AS growth_per_month,
  pg_size_pretty((current_size + bytes_per_day * 90)::bigint) AS projected_90_days,
  pg_size_pretty((current_size + bytes_per_day * 180)::bigint) AS projected_6_months
FROM regression
WHERE bytes_per_day > 0
ORDER BY bytes_per_day DESC;
```

---

### Connection Capacity

```sql
-- Connection usage trend:
CREATE TABLE connection_history (
  captured_at timestamptz DEFAULT now(),
  total_connections int,
  active_connections int,
  idle_connections int,
  idle_in_transaction int,
  max_connections int
);

-- Insert every minute via cron:
INSERT INTO connection_history
SELECT
  now(),
  count(*),
  count(*) FILTER (WHERE state = 'active'),
  count(*) FILTER (WHERE state = 'idle'),
  count(*) FILTER (WHERE state = 'idle in transaction'),
  current_setting('max_connections')::int
FROM pg_stat_activity;

-- Peak connection analysis:
SELECT
  date_trunc('hour', captured_at) AS hour,
  max(total_connections) AS peak_connections,
  avg(total_connections) AS avg_connections,
  max(idle_in_transaction) AS peak_idle_in_txn,
  max_connections
FROM connection_history
WHERE captured_at > now() - interval '7 days'
GROUP BY date_trunc('hour', captured_at), max_connections
ORDER BY hour;

-- Alert: approaching max_connections
SELECT
  total_connections,
  max_connections,
  round(total_connections::numeric / max_connections * 100, 2) AS pct_used
FROM connection_history
ORDER BY captured_at DESC LIMIT 1;
```

---

### Query Performance Capacity

```sql
-- TPS (Transactions per second) over time:
SELECT
  date_trunc('minute', now()) AS minute,
  xact_commit + xact_rollback AS total_transactions
FROM pg_stat_database
WHERE datname = current_database();

-- Capture TPS history:
CREATE TABLE tps_history (
  captured_at timestamptz DEFAULT now(),
  xact_commit bigint,
  xact_rollback bigint
);

-- Every minute:
INSERT INTO tps_history
SELECT now(), xact_commit, xact_rollback
FROM pg_stat_database WHERE datname = current_database();

-- TPS calculation:
WITH deltas AS (
  SELECT
    captured_at,
    xact_commit - lag(xact_commit) OVER (ORDER BY captured_at) AS commits_delta,
    xact_rollback - lag(xact_rollback) OVER (ORDER BY captured_at) AS rollbacks_delta,
    extract(epoch FROM captured_at - lag(captured_at) OVER (ORDER BY captured_at)) AS seconds
  FROM tps_history
  WHERE captured_at > now() - interval '24 hours'
)
SELECT
  date_trunc('hour', captured_at) AS hour,
  round(sum(commits_delta) / sum(seconds), 2) AS avg_tps,
  round(max(commits_delta) / min(seconds), 2) AS peak_tps
FROM deltas
WHERE seconds > 0
GROUP BY date_trunc('hour', captured_at)
ORDER BY hour;

-- Cache hit ratio trend:
SELECT
  date_trunc('hour', now()) AS hour,
  round(blks_hit::numeric / nullif(blks_hit + blks_read, 0) * 100, 2) AS cache_hit_pct
FROM pg_stat_database
WHERE datname = current_database();
```

---

### Alerting Thresholds — Complete Reference

```sql
-- Create monitoring check function:
CREATE OR REPLACE FUNCTION check_database_health()
RETURNS TABLE(
  check_name text,
  status text,
  value text,
  threshold text,
  recommendation text
)
LANGUAGE plpgsql AS $$
BEGIN
  -- 1. Cache hit ratio
  RETURN QUERY
  SELECT
    'Cache Hit Ratio'::text,
    CASE WHEN ratio < 95 THEN 'CRITICAL'
         WHEN ratio < 98 THEN 'WARNING'
         ELSE 'OK' END,
    ratio::text || '%',
    '> 99%',
    CASE WHEN ratio < 95 THEN 'Increase shared_buffers or reduce data scanned'
         ELSE 'OK' END
  FROM (
    SELECT round(blks_hit::numeric / nullif(blks_hit + blks_read, 0) * 100, 2) AS ratio
    FROM pg_stat_database WHERE datname = current_database()
  ) r;

  -- 2. Connection usage
  RETURN QUERY
  SELECT
    'Connection Usage'::text,
    CASE WHEN pct > 90 THEN 'CRITICAL'
         WHEN pct > 75 THEN 'WARNING'
         ELSE 'OK' END,
    pct::text || '%',
    '< 75%',
    CASE WHEN pct > 75 THEN 'Increase max_connections or add PgBouncer'
         ELSE 'OK' END
  FROM (
    SELECT round(count(*)::numeric / current_setting('max_connections')::int * 100, 2) AS pct
    FROM pg_stat_activity
  ) c;

  -- 3. XID age (wraparound risk)
  RETURN QUERY
  SELECT
    'XID Wraparound Risk'::text,
    CASE WHEN max_age > 1800000000 THEN 'CRITICAL'
         WHEN max_age > 1500000000 THEN 'WARNING'
         WHEN max_age > 1000000000 THEN 'WATCH'
         ELSE 'OK' END,
    max_age::text,
    '< 1,000,000,000',
    CASE WHEN max_age > 1500000000 THEN 'Run VACUUM FREEZE immediately!'
         WHEN max_age > 1000000000 THEN 'Increase autovacuum aggressiveness'
         ELSE 'OK' END
  FROM (SELECT max(age(relfrozenxid)) AS max_age FROM pg_class WHERE relkind = 'r') x;

  -- 4. Dead tuple ratio
  RETURN QUERY
  SELECT
    'Dead Tuple Ratio'::text,
    CASE WHEN max_dead_pct > 20 THEN 'CRITICAL'
         WHEN max_dead_pct > 10 THEN 'WARNING'
         ELSE 'OK' END,
    max_dead_pct::text || '%',
    '< 10%',
    CASE WHEN max_dead_pct > 10 THEN 'Tune autovacuum or run VACUUM manually'
         ELSE 'OK' END
  FROM (
    SELECT max(round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2)) AS max_dead_pct
    FROM pg_stat_user_tables
  ) d;

  -- 5. Replication lag (if replica)
  IF EXISTS (SELECT 1 FROM pg_stat_replication) THEN
    RETURN QUERY
    SELECT
      'Replication Lag'::text,
      CASE WHEN max_lag > 100*1024*1024 THEN 'CRITICAL'  -- > 100MB
           WHEN max_lag > 10*1024*1024 THEN 'WARNING'    -- > 10MB
           ELSE 'OK' END,
      pg_size_pretty(max_lag),
      '< 10MB',
      CASE WHEN max_lag > 10*1024*1024 THEN 'Check standby network and performance'
           ELSE 'OK' END
    FROM (
      SELECT max(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS max_lag
      FROM pg_stat_replication
    ) r;
  END IF;

  -- 6. Long-running queries
  RETURN QUERY
  SELECT
    'Long Running Queries'::text,
    CASE WHEN long_query_count > 5 THEN 'CRITICAL'
         WHEN long_query_count > 0 THEN 'WARNING'
         ELSE 'OK' END,
    long_query_count::text,
    '0',
    CASE WHEN long_query_count > 0 THEN 'Investigate and kill if necessary'
         ELSE 'OK' END
  FROM (
    SELECT count(*) AS long_query_count
    FROM pg_stat_activity
    WHERE state = 'active'
      AND now() - query_start > interval '5 minutes'
      AND query NOT LIKE '%pg_stat_%'
  ) q;

  -- 7. Lock waits
  RETURN QUERY
  SELECT
    'Lock Waits'::text,
    CASE WHEN lock_wait_count > 10 THEN 'CRITICAL'
         WHEN lock_wait_count > 3 THEN 'WARNING'
         ELSE 'OK' END,
    lock_wait_count::text,
    '0',
    CASE WHEN lock_wait_count > 0 THEN 'Find and kill blocking queries'
         ELSE 'OK' END
  FROM (
    SELECT count(*) AS lock_wait_count
    FROM pg_stat_activity
    WHERE wait_event_type = 'Lock'
  ) l;
END;
$$;

-- Run health check:
SELECT * FROM check_database_health()
ORDER BY CASE status WHEN 'CRITICAL' THEN 1 WHEN 'WARNING' THEN 2 WHEN 'WATCH' THEN 3 ELSE 4 END;
```

---

### Configuration Reference — Production Defaults

```ini
# postgresql.conf — Production Template (16-32 core, 64GB RAM server)

# Memory
shared_buffers = 16GB                    # 25% of 64GB RAM
effective_cache_size = 48GB              # 75% of RAM (OS cache hint)
work_mem = 64MB                          # per sort/hash (conservative global)
maintenance_work_mem = 2GB               # VACUUM, CREATE INDEX
temp_file_limit = 50GB                   # limit runaway queries

# Write Performance
wal_buffers = 256MB                      # WAL buffer (auto from shared_buffers)
checkpoint_timeout = 15min               # less frequent checkpoints
max_wal_size = 8GB                       # before forced checkpoint
min_wal_size = 1GB
checkpoint_completion_target = 0.9       # spread I/O
synchronous_commit = on                  # durability first

# Parallel Query
max_parallel_workers_per_gather = 4
max_parallel_workers = 16
max_worker_processes = 24                # background + parallel

# Connections
max_connections = 200                    # use PgBouncer for more!
superuser_reserved_connections = 5

# WAL and Replication
wal_level = replica                      # or logical if needed
max_wal_senders = 10
wal_keep_size = 2GB

# Autovacuum (tuned)
autovacuum = on
autovacuum_max_workers = 6
autovacuum_naptime = 30s
autovacuum_vacuum_scale_factor = 0.05    # 5% not 20%
autovacuum_analyze_scale_factor = 0.05
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = 400

# Logging
log_min_duration_statement = 1000        # log queries > 1 second
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 10MB                    # log temp files > 10MB
log_autovacuum_min_duration = 0          # log all autovacuum
log_line_prefix = '%t [%p-%l] %q%u@%d '

# pg_stat_statements
shared_preload_libraries = 'pg_stat_statements,auto_explain'
pg_stat_statements.track = all
pg_stat_statements.max = 10000

# auto_explain
auto_explain.log_min_duration = 5000     # log plans for queries > 5s
auto_explain.log_analyze = true
auto_explain.log_buffers = true
auto_explain.log_format = json

# SSD optimization
random_page_cost = 1.1                   # SSD: nearly sequential cost
effective_io_concurrency = 200           # SSD: many concurrent I/O
```

---

### Block 7 Summary — Dots Connected

```
Topic 28 (System Catalogs & pg_stat)
    │
    ├──► ALL topics: Every component has pg_stat_* views
    ├──► Topic 4 (Concurrency): pg_locks for lock monitoring
    ├──► Topic 11 (Index): pg_stat_user_indexes for usage analysis
    └──► Topic 15 (Vacuum): pg_stat_user_tables for vacuum health

Topic 29 (Bloat & Autovacuum)
    │
    ├──► Topic 2 (Physical Storage): Dead tuples = physical bloat
    ├──► Topic 4 (Concurrency): Long transactions block vacuum
    ├──► Topic 5 (WAL): VACUUM also writes WAL
    └──► Topic 15 (Statistics): Autovacuum triggers ANALYZE too

Topic 30 (Connection Pooling)
    │
    ├──► Topic 1 (Architecture): Process model = why pooler needed
    ├──► Topic 22 (RLS): SET LOCAL for RLS in transaction mode
    └──► Topic 27 (Failover): PgBouncer reconnects after failover

Topic 31 (Capacity Planning)
    │
    ├──► ALL topics: Capacity planning uses metrics from all areas
    ├──► Topic 20 (Partition): Partition strategy for data growth
    └──► Topic 26 (Replication): Replica for read scaling capacity
```

---

# APPENDIX — Complete Quick Reference

## Essential Diagnostic Queries

```sql
-- 1. Database health snapshot
SELECT datname, numbackends,
  round(blks_hit::numeric/nullif(blks_hit+blks_read,0)*100,2) AS cache_hit_pct,
  xact_commit, xact_rollback, deadlocks, temp_files,
  pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_stat_database WHERE datname = current_database();

-- 2. Active queries with duration
SELECT pid, usename, state, wait_event_type, wait_event,
  now()-query_start AS duration, left(query,100) AS query
FROM pg_stat_activity WHERE state!='idle' AND pid!=pg_backend_pid()
ORDER BY duration DESC NULLS LAST;

-- 3. Lock conflicts
SELECT blocked.pid, blocked.query, blocking.pid, blocking.query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid=ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid))>0;

-- 4. Table sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS total,
  pg_size_pretty(pg_relation_size(relid)) AS data,
  pg_size_pretty(pg_indexes_size(relid)) AS indexes
FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 20;

-- 5. Index usage
SELECT tablename, indexname, idx_scan,
  pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes ORDER BY idx_scan DESC;

-- 6. Vacuum health
SELECT relname, n_dead_tup, n_live_tup,
  round(n_dead_tup::numeric/nullif(n_live_tup,0)*100,2) AS dead_pct,
  last_autovacuum, autovacuum_count
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- 7. XID age (wraparound risk)
SELECT relname, age(relfrozenxid) AS xid_age,
  pg_size_pretty(pg_total_relation_size(oid)) AS size
FROM pg_class WHERE relkind='r' ORDER BY age(relfrozenxid) DESC LIMIT 10;

-- 8. Replication lag
SELECT application_name, state,
  pg_size_pretty(pg_wal_lsn_diff(sent_lsn,replay_lsn)) AS total_lag,
  replay_lag
FROM pg_stat_replication ORDER BY replay_lag DESC;

-- 9. Slow queries (pg_stat_statements)
SELECT round(mean_exec_time::numeric,2) AS avg_ms, calls,
  round(total_exec_time::numeric,2) AS total_ms,
  left(query,150) AS query
FROM pg_stat_statements ORDER BY mean_exec_time DESC LIMIT 10;

-- 10. Cache hit per table
SELECT relname,
  round(heap_blks_hit::numeric/nullif(heap_blks_hit+heap_blks_read,0)*100,2) AS cache_pct
FROM pg_statio_user_tables ORDER BY heap_blks_read DESC LIMIT 20;
```

## Emergency Commands

```sql
-- Kill specific query:
SELECT pg_cancel_backend(pid);   -- graceful (SIGINT)
SELECT pg_terminate_backend(pid); -- force (SIGTERM)

-- Kill all idle-in-transaction:
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE state='idle in transaction' AND now()-xact_start > interval '10 minutes';

-- Emergency vacuum:
VACUUM FREEZE VERBOSE tablename;
VACUUM ANALYZE tablename;

-- Check what's blocking vacuum:
SELECT pid, state, xact_start, left(query,100) FROM pg_stat_activity
WHERE xact_start IS NOT NULL ORDER BY xact_start LIMIT 5;

-- Force checkpoint:
CHECKPOINT;

-- Reload config (no restart):
SELECT pg_reload_conf();

-- Promote standby:
SELECT pg_promote();   -- PostgreSQL 12+
-- or: touch $PGDATA/promote.signal

-- Check if primary or standby:
SELECT pg_is_in_recovery();  -- false=primary, true=standby
```
