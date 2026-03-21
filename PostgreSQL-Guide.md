# PostgreSQL 16 — Complete Study Guide

> **Environment:** Rocky Linux 9 | PostgreSQL 16
> **Coverage:** Architecture · Configuration · Data Types · Indexes · Transactions · Query Optimization · Backup & Recovery · Monitoring · Security · High Availability · Patroni · pgBouncer · pgBackRest · Barman

---

## Table of Contents

1. [Architecture](#1-architecture)
   - 1.1 Overall Architecture · 1.2 Process Model · 1.3 Shared Memory
   - 1.4 Background Processes · 1.5 File System Layout · 1.6 WAL
   - 1.7 MVCC · 1.8 Read Path · 1.9 Write Path · 1.10 UPDATE · 1.11 DELETE
   - 1.12 VACUUM · 1.13 Crash Recovery · 1.14 Concurrent Read+Write · 1.15 Summary
2. [Configuration & Parameters](#2-configuration--parameters)
   - 2.1 Config Files · 2.2 Memory · 2.3 WAL · 2.4 Connections
   - 2.5 Query Planner · 2.6 Logging · 2.7 Replication · 2.8 Runtime Change
3. [Data Types](#3-data-types)
   - 3.1 Numeric · 3.2 Character · 3.3 DateTime · 3.4 Boolean · 3.5 JSON/JSONB
   - 3.6 Arrays · 3.7 UUID · 3.8 Ranges · 3.9 Table Design
4. [Indexes](#4-indexes)
   - 4.1 B-Tree · 4.2 Hash · 4.3 GIN · 4.4 GiST · 4.5 BRIN · 4.6 Partial
   - 4.7 Expression · 4.8 Covering · 4.9 EXPLAIN · 4.10 Design Rules
5. [Transactions & Locking](#5-transactions--locking)
   - 5.1 ACID · 5.2 Commands · 5.3 Isolation Levels · 5.4 MVCC Deep Dive
   - 5.5 Lock Types · 5.6 Deadlock · 5.7 Advisory Locks · 5.8 Monitoring
6. [Query Optimization](#6-query-optimization)
   - 6.1 Statistics · 6.2 EXPLAIN · 6.3 Slow Patterns · 6.4 Planner Settings
   - 6.5 Partitioning · 6.6 CTEs · 6.7 Parallel Query · 6.8 Checklist
7. [Backup & Recovery](#7-backup--recovery)
   - 7.1 pg_dump · 7.2 pg_dumpall · 7.3 pg_restore · 7.4 pg_basebackup
   - 7.5 WAL Archiving · 7.6 PITR · 7.7 pgBackRest · 7.8 Barman · 7.9 Best Practices
8. [Monitoring](#8-monitoring)
   - 8.1 OS Level · 8.2 pg_stat_activity · 8.3 pg_stat_user_tables
   - 8.4 Replication Monitoring · 8.5 VACUUM Monitoring · 8.6 Lock Monitoring
   - 8.7 Index Usage · 8.8 Dashboard Script · 8.9 Thresholds
9. [Security](#9-security)
   - 9.1 Authentication (pg_hba.conf) · 9.2 Roles & Privileges
   - 9.3 Row Level Security · 9.4 Column Security · 9.5 SSL/TLS
   - 9.6 pgcrypto · 9.7 Audit Logging (pgaudit) · 9.8 Checklist
10. [High Availability & Replication](#10-high-availability--replication)
    - 10.1 RPO/RTO · 10.2 Streaming Replication · 10.3 Install to Replication (E2E)
    - 10.4 Replication Slots · 10.5 Synchronous Replication
    - 10.6 Logical Replication · 10.7 Patroni + etcd + HAProxy
    - 10.8 pgBouncer · 10.9 pgPool-II · 10.10 Failover · 10.11 Quick Reference

---

# 1. Architecture

## 1.1 Overall Architecture

PostgreSQL এর architecture MySQL এর চেয়ে আলাদা — **Process-based** (thread নয়), প্রতিটা connection এ আলাদা OS process।

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                │
│              Application / psql / JDBC / libpq                      │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ TCP/IP (port 5432) or Unix Socket
┌─────────────────────────────▼───────────────────────────────────────┐
│                       POSTMASTER (Listener)                         │
│              Master process — সব কিছু manage করে                    │
│         নতুন connection এ নতুন Backend Process fork করে             │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ fork()
┌─────────────────────────────▼───────────────────────────────────────┐
│                   BACKEND PROCESS (per connection)                  │
│        Parser → Rewriter → Planner/Optimizer → Executor             │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                        SHARED MEMORY                                │
│  ┌──────────────────────┐  ┌───────────────────┐  ┌─────────────┐  │
│  │    Shared Buffers    │  │    WAL Buffers     │  │  Lock Table │  │
│  │  (Data Page Cache)   │  │  (Write-Ahead Log) │  │  Clog/CSN  │  │
│  └──────────────────────┘  └───────────────────┘  └─────────────┘  │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                    BACKGROUND PROCESSES                             │
│  WAL Writer · Checkpointer · Background Writer · Autovacuum         │
│  Stats Collector · WAL Receiver · WAL Sender · Archiver             │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                          FILE SYSTEM                                │
│   base/ · pg_wal/ · pg_xact/ · pg_undo/ · postgresql.conf          │
│   pg_hba.conf · pg_ident.conf · PG_VERSION · postmaster.pid        │
└─────────────────────────────────────────────────────────────────────┘
```

**MySQL vs PostgreSQL — Key Architecture Difference:**

| | MySQL | PostgreSQL |
|---|---|---|
| Concurrency model | Thread-based | Process-based (fork) |
| Connection overhead | Low (thread reuse) | Higher (new process) |
| Connection pooling | Built-in Thread Cache | External (pgBouncer) needed |
| Storage engine | Pluggable (InnoDB, MyISAM) | Single engine (Heap) |
| MVCC implementation | Undo Log (in tablespace) | Tuple versioning (in heap) |
| Vacuum needed | No | Yes (dead tuple cleanup) |

---

## 1.2 Process Model

```
Postmaster (PID 1234)
    │
    ├── Background Writer    ← dirty pages flush
    ├── Checkpointer         ← checkpoint নেয়
    ├── WAL Writer           ← WAL buffer → disk
    ├── Autovacuum Launcher  ← vacuum scheduling
    │   ├── Autovacuum Worker 1
    │   └── Autovacuum Worker 2
    ├── Stats Collector      ← statistics আপডেট
    ├── WAL Sender           ← replication এ binlog পাঠায়
    ├── Archiver             ← WAL archive করে
    │
    ├── Backend Process (connection 1) ← Client A
    ├── Backend Process (connection 2) ← Client B
    └── Backend Process (connection N) ← Client N
```

```sql
-- Running processes দেখো
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity
ORDER BY state;

-- Background processes
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE query = '<idle in transaction>' OR backend_type != 'client backend';
```

---

## 1.3 Shared Memory Components

### Shared Buffers

PostgreSQL এর সবচেয়ে critical memory component। Disk থেকে read করা data pages এখানে cache হয়।

```
Shared Buffers
├── Clock-Sweep Algorithm (LRU এর variant)
│   ├── Usage count প্রতিটা page এর জন্য
│   ├── Access হলে count বাড়ে (max 5)
│   └── Eviction candidate: usage_count=0 এ নামলে
└── Buffer Descriptor (প্রতিটা page এর metadata)
    ├── Buffer State (free/valid/dirty)
    ├── Refcount (কতজন use করছে)
    └── Usage Count
```

```sql
-- Shared Buffers hit rate
SELECT
    sum(heap_blks_hit)  AS heap_hit,
    sum(heap_blks_read) AS heap_read,
    ROUND(sum(heap_blks_hit)::numeric /
        NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0) * 100, 2
    ) AS hit_rate_pct
FROM pg_statio_user_tables;
```

### WAL Buffers
Write-Ahead Log এর in-memory buffer। Transaction changes প্রথমে এখানে লেখা হয়।

### CLOG (Commit Log / pg_xact)
কোন transaction committed, কোন aborted এর record রাখে।

```
প্রতিটা transaction এর status:
  IN_PROGRESS  → চলছে
  COMMITTED    → সফল
  ABORTED      → rollback হয়েছে
  SUB_COMMITTED → sub-transaction
```

### Lock Table
Row-level এবং table-level lock information Shared Memory তে রাখা হয়।

---

## 1.4 Background Processes

```
WAL Writer:
  WAL Buffers থেকে pg_wal/ এ লেখে
  wal_writer_delay (default 200ms) interval এ
  Commit এ synchronously flush করে (synchronous_commit=on)

Checkpointer:
  Dirty pages disk এ flush করে
  checkpoint_timeout (default 5min) বা
  max_wal_size (default 1GB) পূর্ণ হলে
  Checkpoint LSN record করে recovery তে কাজ লাগে

Background Writer (bgwriter):
  Checkpointer এর আগেই dirty pages flush শুরু করে
  Checkpointer এর কাজ কমায়
  bgwriter_delay (default 200ms)

Autovacuum Launcher:
  Table statistics দেখে
  Dead tuple threshold পার হলে Worker spawn করে
  autovacuum_naptime (default 1min)

Autovacuum Worker:
  Dead tuples reclaim করে (VACUUM)
  Statistics update করে (ANALYZE)
  Table bloat কমায়

Stats Collector:
  Table access, index usage statistics collect করে
  pg_stat_* views এর data source

WAL Sender:
  Replica কে WAL stream পাঠায়
  Replication connection এ কাজ করে

Archiver:
  Completed WAL segments archive করে
  archive_command execute করে
```

---

## 1.5 File System Layout

```
$PGDATA (/var/lib/pgsql/16/data/)
│
├── postgresql.conf      → Main configuration
├── pg_hba.conf          → Client authentication rules
├── pg_ident.conf        → User name mapping
├── PG_VERSION           → PostgreSQL version number
├── postmaster.pid       → Running server PID
├── postmaster.opts      → Server startup options
│
├── base/                → Database files
│   ├── 1/               → template1 database (OID=1)
│   ├── 16384/           → mydb database (OID=16384)
│   │   ├── 16385        → users table (file = relation OID)
│   │   ├── 16385_fsm    → Free Space Map
│   │   ├── 16385_vm     → Visibility Map
│   │   └── 16386        → users_pkey index
│   └── ...
│
├── pg_wal/              → Write-Ahead Log segments
│   ├── 000000010000000000000001
│   ├── 000000010000000000000002
│   └── ...
│
├── pg_xact/             → Transaction commit status (CLOG)
├── pg_multixact/        → Multi-transaction status
├── pg_subtrans/         → Sub-transaction status
├── pg_stat_tmp/         → Temporary stats files
├── pg_tblspc/           → Tablespace symlinks
│
├── global/              → Cluster-wide tables
│   ├── pg_control       → Control file (critical!)
│   └── pg_filenode.map
│
└── pg_log/ (বা log/)    → Server log files
```

**Key files:**
| File | কাজ |
|---|---|
| `postgresql.conf` | সব configuration parameter |
| `pg_hba.conf` | কে কোথা থেকে কীভাবে connect করতে পারবে |
| `pg_control` | Checkpoint info, cluster state (corrupt হলে DB start হবে না) |
| `pg_wal/` | WAL segments — replication এবং crash recovery |
| `pg_xact/` | Transaction status — MVCC এর জন্য critical |
| `base/OID/RelFileNode` | Actual table এবং index data |
| `*_fsm` | Free Space Map — INSERT কোথায় space পাবে |
| `*_vm` | Visibility Map — VACUUM optimization |

---

## 1.6 WAL (Write-Ahead Log)

MySQL এর Redo Log এর equivalent।

```
WAL এর মূল নীতি:
  Data page disk এ লেখার আগে
  সেই change WAL এ লিখতে হবে

কেন?
  Crash হলে WAL দিয়ে data recover করা যাবে
  WAL sequential write → data files random write
  Sequential write অনেক faster
```

### WAL Segment

```
প্রতিটা WAL file = 16MB (default)
Filename: 000000010000000000000001
          │────────│─────────────│
          timeline  LSN (hex)

pg_walfile_name(pg_current_wal_lsn()); -- current WAL file
```

### LSN (Log Sequence Number)

```sql
-- Current WAL position
SELECT pg_current_wal_lsn();

-- WAL file location
SELECT pg_walfile_name(pg_current_wal_lsn());

-- Replica lag (Master এ)
SELECT
    application_name,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    (sent_lsn - replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

### WAL এর কাজ

```
1. Durability (Crash Recovery):
   COMMIT → WAL flush → "Transaction committed"
   Crash → WAL replay → Data recovered

2. Replication (Streaming):
   WAL segments → WAL Sender → WAL Receiver (Replica)

3. PITR (Point-in-Time Recovery):
   Base backup + WAL segments = যেকোনো সময়ে restore
```

### `synchronous_commit` — Durability vs Performance

```ini
synchronous_commit = on      # WAL disk এ flush হওয়ার পর commit ack
                              # Safest, slowest
synchronous_commit = off     # WAL flush এর আগেই ack
                              # Faster, last few transactions হারাতে পারে
synchronous_commit = local   # Local disk flush, Replica না হলেও ack
synchronous_commit = remote_write  # Replica memory তে লেখা, disk নয়
synchronous_commit = remote_apply  # Replica apply করার পর ack
```

---

## 1.7 MVCC — PostgreSQL এর Implementation

MySQL এ MVCC Undo Log দিয়ে। PostgreSQL এ MVCC **Tuple Versioning** দিয়ে।

### Tuple Header

প্রতিটা row (tuple) এ hidden system columns থাকে:

```
┌────────────┬────────────┬───────────────────────────────┐
│  xmin      │  xmax      │  ctid · cmin · cmax · infomask│
├────────────┼────────────┼───────────────────────────────┤
│ Transaction│ Transaction│ Physical location, command ID │
│ that INSERT│ that DELETE│ visibility flags               │
│ this tuple │ this tuple │                               │
└────────────┴────────────┴───────────────────────────────┘
```

```sql
-- Hidden columns দেখো
SELECT xmin, xmax, ctid, * FROM users LIMIT 5;
```

### কীভাবে কাজ করে

```
INSERT (id=1, name='Alice'):
  xmin=100, xmax=0
  (Transaction 100 এ insert, কেউ delete করেনি)

UPDATE (salary=50000 WHERE id=1):
  Old tuple: xmin=100, xmax=200 (Transaction 200 delete করেছে)
  New tuple: xmin=200, xmax=0   (Transaction 200 insert করেছে)

DELETE (WHERE id=1):
  tuple: xmin=100, xmax=300 (Transaction 300 delete করেছে)
  Tuple physically মুছে যায় না — xmax set হয়
  VACUUM পরে physically মুছবে

Transaction A (snapshot time = TXN 150):
  xmin=100 (<=150) এবং xmax=0 বা xmax>150
  → salary=40000 দেখবে (Transaction 200 এর পরিবর্তন দেখবে না)
```

**MySQL এর সাথে পার্থক্য:**
```
MySQL MVCC:  Old version Undo Log এ, current version heap এ
PostgreSQL:  সব version একই heap এ (dead tuple হিসেবে)
             → VACUUM দরকার dead tuples সরাতে
             → Table bloat হতে পারে VACUUM না চললে
```

---

## 1.8 Complete Read Path (Full Cycle)

**Query:** `SELECT * FROM users WHERE id = 5`

```
Step 1: Client → Postmaster
  TCP connection (port 5432)
  Postmaster নতুন Backend Process fork করে
  Authentication: pg_hba.conf rules check করে

Step 2: Backend — Parser
  SQL → Parse Tree
  Syntax check

Step 3: Backend — Rewriter
  View rewrite করে actual table query তে
  Rule system apply করে

Step 4: Backend — Planner/Optimizer
  Statistics (pg_statistic) দেখে
  Multiple execution plans generate করে
  Cost estimate করে (seq_page_cost, random_page_cost)
  Cheapest plan choose করে
  Plan: Index Scan on users_pkey (cost=0.29..8.30 rows=1)

Step 5: Executor → Shared Buffers Check
  ┌──────────────────────────────────────────────┐
  │ Shared Buffers এ page আছে? (buffer hit)       │
  │                                              │
  │ YES → Page থেকে tuple read (no disk I/O)     │
  │                                              │
  │ NO  → OS Page Cache (double buffering) →     │
  │       Disk থেকে read                         │
  │       Shared Buffers এ load                  │
  └──────────────────────────────────────────────┘

Step 6: MVCC Visibility Check
  Tuple এর xmin এবং xmax দেখো
  Current transaction এর snapshot এর সাথে compare
  CLOG (pg_xact) এ transaction status check করো
  → Visible হলে return

Step 7: Result → Client
  Tuple → Executor → Result set → Network → Client

Component States during READ:
  Shared Buffers : Page access হলে usage count বাড়ে
  CLOG           : Transaction status read হয়
  WAL            : Untouched (read এ WAL লেখা হয় না)
  Lock           : AccessShareLock on table (SELECT)
```

---

## 1.9 Complete Write Path (Full Cycle)

**Query:** `INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')`

```
Step 1-4: Connection → Parse → Rewrite → Plan

Step 5: Transaction শুরু
  Transaction ID (XID) assign হয়
  BEGIN এ snapshot নেওয়া হয়

Step 6: Shared Buffers এ Target Page খোঁজো
  Free Space Map (FSM) দেখো কোন page এ space আছে
  Page Shared Buffers এ আছে কিনা দেখো
  না থাকলে disk থেকে load করো

Step 7: Tuple লেখো (Heap)
  Page এ নতুন tuple insert করো:
    xmin = current XID
    xmax = 0
    ctid = (page_number, tuple_offset)
  Page state: Clean → Dirty

Step 8: Index Update
  PRIMARY KEY index (B-Tree) এ নতুন entry
  প্রতিটা index এ entry

Step 9: WAL লেখো
  WAL Record তৈরি: "Page X তে এই tuple insert হয়েছে"
  WAL Buffers এ লেখো (memory)
  Synchronous commit এ → disk flush

Step 10: COMMIT
  WAL Buffer → pg_wal/ disk flush (synchronous_commit=on)
  CLOG (pg_xact) এ XID = COMMITTED mark করো
  Lock release

Note: Data page (Shared Buffers) এখনো dirty
      Background Writer বা Checkpointer পরে disk এ লিখবে

Component States during WRITE:
  Shared Buffers : Target page dirty হয়
  WAL Buffers    : Change record লেখা হয়
  WAL on Disk    : COMMIT এ flush
  CLOG           : XID = COMMITTED
  FSM            : Space usage update
  Locks          : RowExclusiveLock
```

---

## 1.10 UPDATE Operation

**Query:** `UPDATE users SET salary = 50000 WHERE id = 5`

```
PostgreSQL UPDATE = DELETE + INSERT (in-place নয়!)

Before: (xmin=100, xmax=0, id=5, salary=40000)

Step 1: Row খোঁজো, RowExclusiveLock নাও

Step 2: পুরনো tuple "delete" করো (soft):
  Old tuple: xmin=100, xmax=200
  (xmax = current XID, মানে এই transaction এ delete হয়েছে)

Step 3: নতুন tuple insert করো:
  New tuple: xmin=200, xmax=0, id=5, salary=50000
  ctid of old tuple → new tuple point করে (HOT update possible)

Step 4: Index Update
  Non-HOT: সব index এ নতুন entry, পুরনো entry ও থাকে
  HOT (Heap Only Tuple): Index update নেই, শুধু heap
    → HOT হওয়ার condition:
       - Update করা column এ কোনো index নেই
       - নতুন tuple একই page এ জায়গা পায়

Step 5: WAL লেখো, COMMIT

After:
  Heap এ দুটো tuple (পুরনো dead + নতুন live)
  VACUUM পরে dead tuple সরাবে
  → Table bloat এড়াতে AUTOVACUUM চলতে হবে
```

---

## 1.11 DELETE Operation

```
DELETE FROM users WHERE id = 5;

Step 1: Row খোঁজো, Lock নাও

Step 2: Soft Delete:
  tuple: xmin=100, xmax=300
  (xmax = current XID)
  Physically মুছে যায় না

Step 3: Index এর entry এখনো আছে
  Index → Heap এ dead tuple point করে
  VACUUM পরে index entry এবং heap tuple দুটোই সরাবে

Step 4: WAL, COMMIT

VACUUM এর কাজ:
  Dead tuple এর space reclaim করে
  Index এর dead entry সরায়
  Visibility Map আপডেট করে
  Statistics আপডেট করে (ANALYZE হলে)
  Table এবং Index bloat কমায়
```

---

## 1.12 VACUUM — PostgreSQL Unique Requirement

MySQL তে VACUUM দরকার নেই কারণ Undo Log আলাদা। PostgreSQL এ dead tuples heap এ থাকে তাই VACUUM লাগে।

### কী করে VACUUM

```
1. Dead tuples (xmax != 0, transaction committed/aborted) চিহ্নিত করে
2. সেই space free করে (FSM আপডেট করে)
3. Dead index entries সরায়
4. Visibility Map (VM) আপডেট করে
5. Transaction ID Wraparound থেকে রক্ষা করে

VACUUM FULL:
  পুরো table rewrite করে
  Disk space ফেরত দেয় OS কে
  Exclusive lock নেয় — production এ সাবধানে ব্যবহার করো
  ALTER TABLE, CLUSTER এর মতো কাজ করে

VACUUM ANALYZE:
  VACUUM + Statistics update
```

### Transaction ID Wraparound — Critical Problem

```
XID 4 bytes = maximum ~4.2 billion transactions
XID exhausted হলে → PostgreSQL SHUTDOWN করে!

Prevention:
  autovacuum চলতে দাও সবসময়
  age(datfrozenxid) monitor করো
  age > 1.5 billion → আগে VACUUM FREEZE করো
```

```sql
-- Database এর XID age দেখো
SELECT datname, age(datfrozenxid) AS xid_age,
    2^31 - age(datfrozenxid) AS xids_remaining
FROM pg_database
ORDER BY xid_age DESC;

-- Table এর XID age
SELECT schemaname, relname, n_dead_tup, n_live_tup,
    last_autovacuum, age(relfrozenxid) AS xid_age
FROM pg_stat_user_tables
ORDER BY age(relfrozenxid) DESC LIMIT 20;

-- Manual VACUUM
VACUUM VERBOSE ANALYZE users;
VACUUM FREEZE users;         -- XID freeze করো
```

### Autovacuum Tuning

```ini
# postgresql.conf
autovacuum                  = on
autovacuum_naptime          = 1min      # check interval
autovacuum_vacuum_threshold = 50        # min dead tuples
autovacuum_vacuum_scale_factor = 0.02  # table এর 2%
# trigger = threshold + scale_factor * n_live_tup

autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.05

autovacuum_vacuum_cost_delay = 2ms     # throttle
autovacuum_max_workers       = 3

# Table level override:
ALTER TABLE big_table SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 100
);
```

---

## 1.13 Crash Recovery

```
Crash হলে PostgreSQL restart এ:

Step 1: pg_control file পড়ো
  Last checkpoint LSN জানো
  Cluster state: shutdown/in_production/in_crash_recovery

Step 2: WAL Redo (Recovery)
  Last checkpoint থেকে WAL replay শুরু
  WAL এর সব record reapply
  Committed transaction এর changes restore

Step 3: WAL Undo
  Crash এর সময় uncommitted transaction ছিল
  CLOG এ IN_PROGRESS → ABORTED mark করো
  (PostgreSQL physically undo করে না, CLOG update করাই যথেষ্ট)
  পরে VACUUM dead tuples সরাবে

Step 4: Normal operation

Recovery speed নির্ভর করে:
  Last checkpoint থেকে crash পর্যন্ত কতটা WAL জমেছে
  checkpoint_timeout এবং max_wal_size ছোট রাখলে recovery fast
```

---

## 1.14 Concurrent Read + Write

```
Transaction A (XID=100): SELECT salary FROM users WHERE id=5
Transaction B (XID=200): UPDATE users SET salary=50000 WHERE id=5

Timeline:
  T=1: A শুরু, snapshot: latest committed = XID 99
  T=2: B শুরু, UPDATE করে:
       Old tuple: xmin=50, xmax=200 (dead for B)
       New tuple: xmin=200, xmax=0
  T=3: B COMMIT → CLOG: XID 200 = COMMITTED
  T=4: A SELECT করে

A কী দেখবে?
  READ COMMITTED:
    Heap এ দুটো tuple দেখবে
    New tuple: xmin=200, CLOG check → 200 COMMITTED
    A এর snapshot নেওয়া হয় প্রতিটা statement এ
    → salary=50000 দেখবে

  REPEATABLE READ / SERIALIZABLE:
    A এর snapshot: XID 99 এর আগে
    New tuple: xmin=200 > 99 → NOT VISIBLE
    Old tuple: xmin=50 <=99, xmax=200 > 99 → VISIBLE
    → salary=40000 দেখবে (transaction শুরুর মতো)

Writer কখনো Reader কে block করে না (MVCC)
Reader কখনো Writer কে block করে না
```

---

## 1.15 Component Summary

| Component | MySQL Equivalent | Role |
|---|---|---|
| Shared Buffers | Buffer Pool | Data page cache |
| WAL Buffers | Log Buffer | WAL before disk |
| pg_wal/ | ib_logfile* | Crash recovery, replication |
| pg_xact/ (CLOG) | InnoDB trx system | Transaction commit status |
| FSM | InnoDB free space | Free space tracking for INSERT |
| Visibility Map | — | VACUUM optimization |
| Backend Process | Thread | Per-connection SQL processing |
| Checkpointer | InnoDB Master Thread | Flush dirty pages |
| Background Writer | InnoDB Page Cleaner | Dirty page flush |
| Autovacuum | InnoDB Purge Thread | Dead tuple cleanup |
| WAL Sender | MySQL Binlog Dump Thread | Replication |
| WAL Receiver | MySQL IO Thread | Replication |
| Archiver | — | WAL archiving for PITR |
| pg_hba.conf | mysql.user host column | Authentication rules |
| VACUUM | InnoDB Purge (automatic) | Space reclamation |

---

# 2. Configuration & Parameters

## 2.1 Configuration Files

```
$PGDATA/
├── postgresql.conf      → Main config (সব parameter)
├── postgresql.auto.conf → ALTER SYSTEM এর changes (override করে)
├── pg_hba.conf          → Authentication rules
└── pg_ident.conf        → OS username → PG username mapping
```

```sql
-- Config file location
SHOW config_file;
SHOW hba_file;
SHOW data_directory;

-- Reload config (restart ছাড়া)
SELECT pg_reload_conf();
-- অথবা:
-- sudo systemctl reload postgresql-16
-- sudo -u postgres pg_ctl reload -D /var/lib/pgsql/16/data

-- Current parameter values
SHOW shared_buffers;
SHOW ALL;
SELECT name, setting, unit, context FROM pg_settings WHERE name LIKE '%buffer%';
```

**`context` column মানে:**
```
internal     → compile-time, পরিবর্তন করা যায় না
postmaster   → restart লাগবে
sighup       → reload যথেষ্ট (pg_reload_conf())
superuser    → superuser SET করতে পারে
user         → যেকোনো user SET করতে পারে
```

---

## 2.2 Memory Parameters

```ini
# postgresql.conf

shared_buffers          = 4GB       # RAM এর 25%
                                    # MySQL এর innodb_buffer_pool_size এর মতো
                                    # কিন্তু OS Page Cache ও আছে
                                    # তাই 25% ভালো (MySQL এর মতো 70% না)

work_mem                = 64MB      # প্রতিটা sort/hash operation এর জন্য
                                    # Per-query, per-operation
                                    # 100 connections × 5 sort = 100×5×64MB = 32GB!
                                    # সাবধানে বাড়াও
                                    # MySQL এর sort_buffer_size এর মতো

maintenance_work_mem    = 512MB     # VACUUM, CREATE INDEX, REINDEX এর জন্য
                                    # বড় দিলে maintenance fast হয়

effective_cache_size    = 12GB      # RAM এর 75%
                                    # Optimizer কে বলে OS cache কতটা available
                                    # Actual memory allocate করে না — শুধু hint

wal_buffers             = 64MB      # WAL এর in-memory buffer
                                    # Default (-1) = shared_buffers এর 1/32
                                    # Minimum 64MB recommended

max_connections         = 200       # প্রতিটা connection ~5-10MB shared memory
                                    # High connection → pgBouncer ব্যবহার করো

temp_buffers            = 8MB       # Per-session temporary table buffer
```

**Memory Rule of Thumb (16GB RAM):**
```
shared_buffers       = 4GB   (25%)
effective_cache_size = 12GB  (75%)
work_mem             = 64MB  (conservative)
maintenance_work_mem = 1GB
wal_buffers          = 64MB
```

---

## 2.3 WAL Parameters

```ini
# Durability
synchronous_commit     = on         # on/off/local/remote_write/remote_apply
wal_level              = replica    # minimal/replica/logical
                                    # replica → replication possible
                                    # logical → logical replication possible

# Performance
checkpoint_timeout     = 5min       # Maximum time between checkpoints
max_wal_size           = 1GB        # Maximum WAL size before checkpoint
min_wal_size           = 80MB       # Minimum WAL size to retain
wal_compression        = on         # WAL compress করো → disk এবং network কমে
wal_writer_delay       = 200ms      # WAL Writer sleep interval

# Archiving (PITR এর জন্য)
archive_mode           = on
archive_command        = 'cp %p /archive/wal/%f'
archive_timeout        = 300        # Force archive every 5 min

# Replication
max_wal_senders        = 10         # Maximum WAL Sender processes
wal_keep_size          = 1GB        # Replica এর জন্য WAL retain করো
max_replication_slots  = 10
```

---

## 2.4 Connection Parameters

```ini
listen_addresses       = '*'        # সব interface এ listen
                                    # specific IP: '172.16.93.140'
port                   = 5432
max_connections        = 200

# Connection timeout
tcp_keepalives_idle    = 60
tcp_keepalives_interval = 10
tcp_keepalives_count   = 6

# SSL
ssl                    = on
ssl_cert_file          = 'server.crt'
ssl_key_file           = 'server.key'
```

---

## 2.5 Query Planner Parameters

```ini
# Cost settings — Hardware অনুযায়ী tune করো
random_page_cost       = 1.1        # SSD এ (HDD এর জন্য 4.0)
seq_page_cost          = 1.0        # Sequential read cost
cpu_tuple_cost         = 0.01       # CPU প্রতি tuple process
cpu_index_tuple_cost   = 0.005
cpu_operator_cost      = 0.0025

# Parallel query
max_parallel_workers_per_gather = 4
max_parallel_workers            = 8
parallel_tuple_cost             = 0.1
min_parallel_table_scan_size    = 8MB

# Statistics
default_statistics_target       = 100  # আরো accurate statistics
                                        # High cardinality column এ বাড়াও
```

```sql
-- Specific column এ statistics target বাড়াও
ALTER TABLE orders ALTER COLUMN user_id SET STATISTICS 500;
ANALYZE orders;
```

---

## 2.6 Logging Parameters

```ini
log_destination        = 'stderr'   # stderr/csvlog/jsonlog/syslog
logging_collector      = on         # File এ log collect করো
log_directory          = 'log'      # $PGDATA/log/
log_filename           = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age       = 1d
log_rotation_size      = 100MB

# What to log
log_min_duration_statement = 1000   # 1 second এর বেশি query log করো
                                    # MySQL এর slow_query_log এর মতো
log_line_prefix        = '%m [%p] %q%u@%d '
                        # %m=timestamp, %p=PID, %u=user, %d=database

log_connections        = on         # Connection log
log_disconnections     = on         # Disconnect log
log_lock_waits         = on         # Lock wait log
log_temp_files         = 0          # সব temp file log (0=all)
log_autovacuum_min_duration = 250ms # Slow autovacuum log

# Debug
log_checkpoints        = on
log_recovery_conflict_waits = on
```

---

## 2.7 Replication Parameters

```ini
# Primary Server
wal_level              = replica
max_wal_senders        = 10
wal_keep_size          = 1GB
max_replication_slots  = 10
synchronous_standby_names = ''      # Async replication

# Standby Server
hot_standby            = on         # Replica তে read query allow
hot_standby_feedback   = on         # Replica query চলাকালীন Master এ vacuum conflict এড়ানো
wal_receiver_timeout   = 60s
recovery_min_apply_delay = 0        # Delayed standby (0=no delay)
                                    # '30min' → 30 মিনিট আগের data apply
```

---

## 2.8 ALTER SYSTEM — Runtime Persistent Change

```sql
-- Persistent change (postgresql.auto.conf এ লেখে)
ALTER SYSTEM SET shared_buffers = '4GB';
ALTER SYSTEM SET work_mem = '64MB';

-- Reload (restart ছাড়া)
SELECT pg_reload_conf();

-- Restart required parameter
ALTER SYSTEM SET max_connections = 300;
-- Restart লাগবে: sudo systemctl restart postgresql-16

-- Reset
ALTER SYSTEM RESET work_mem;
ALTER SYSTEM RESET ALL;

-- Current effective value
SHOW work_mem;
SELECT current_setting('work_mem');
SELECT pg_reload_conf();
```

---

## 2.9 Production postgresql.conf

```ini
# Connection
listen_addresses         = '*'
port                     = 5432
max_connections          = 200

# Memory (16GB RAM)
shared_buffers           = 4GB
work_mem                 = 64MB
maintenance_work_mem     = 1GB
effective_cache_size     = 12GB
wal_buffers              = 64MB

# WAL / Durability
wal_level                = replica
synchronous_commit       = on
checkpoint_timeout       = 5min
max_wal_size             = 2GB
min_wal_size             = 256MB
wal_compression          = on

# Archiving
archive_mode             = on
archive_command          = 'pgbackrest --stanza=main archive-push %p'

# Replication
max_wal_senders          = 10
max_replication_slots    = 10
wal_keep_size            = 1GB
hot_standby              = on

# Query Planner
random_page_cost         = 1.1        # SSD
effective_io_concurrency = 200        # SSD IOPS
default_statistics_target = 100

# Logging
logging_collector        = on
log_directory            = 'log'
log_filename             = 'postgresql-%Y-%m-%d.log'
log_min_duration_statement = 1000
log_checkpoints          = on
log_lock_waits           = on
log_autovacuum_min_duration = 250ms
log_line_prefix          = '%m [%p] %q%u@%d '

# Autovacuum
autovacuum               = on
autovacuum_max_workers   = 5
autovacuum_naptime       = 30s
autovacuum_vacuum_scale_factor = 0.02
autovacuum_analyze_scale_factor = 0.01

# Locale
datestyle                = 'iso, mdy'
timezone                 = 'UTC'
default_text_search_config = 'pg_catalog.english'
```

---

# 3. Data Types

## 3.1 Numeric Types

```
Integer:
┌──────────────┬──────────┬──────────────────────────────────────┐
│ Type         │ Storage  │ Range                                │
├──────────────┼──────────┼──────────────────────────────────────┤
│ SMALLINT     │ 2 bytes  │ -32,768 to 32,767                    │
│ INTEGER      │ 4 bytes  │ -2,147,483,648 to 2,147,483,647      │
│ BIGINT       │ 8 bytes  │ -9.2 quintillion to 9.2Q             │
│ SERIAL       │ 4 bytes  │ Auto-increment INTEGER (deprecated)  │
│ BIGSERIAL    │ 8 bytes  │ Auto-increment BIGINT (deprecated)   │
└──────────────┴──────────┴──────────────────────────────────────┘
```

**MySQL vs PostgreSQL:**
```
MySQL:      TINYINT, SMALLINT, MEDIUMINT, INT, BIGINT
PostgreSQL: SMALLINT, INTEGER, BIGINT (কোনো TINYINT নেই)
            UNSIGNED নেই PostgreSQL এ
```

**Auto Increment — PostgreSQL 10+ এ IDENTITY preferred:**
```sql
-- Old way (SERIAL)
id SERIAL PRIMARY KEY          -- sequence create করে implicitly

-- New way (IDENTITY) — SQL standard
id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY
id BIGINT  GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY

-- Sequence explicitly
CREATE SEQUENCE users_id_seq;
id INTEGER DEFAULT nextval('users_id_seq') PRIMARY KEY
```

**Floating Point:**
```sql
REAL           -- 4 bytes, approximate (~6 decimal digits)
DOUBLE PRECISION -- 8 bytes, approximate (~15 decimal digits)
NUMERIC(p,s)   -- Exact, p total digits, s after decimal
DECIMAL(p,s)   -- NUMERIC এর alias

-- টাকার জন্য সবসময় NUMERIC:
price   NUMERIC(10, 2)   -- 99999999.99, exact
salary  NUMERIC(12, 2)
```

---

## 3.2 Character Types

```sql
CHAR(n)        -- Fixed length, space padded
VARCHAR(n)     -- Variable length, max n
TEXT           -- Unlimited length (PostgreSQL specific, highly efficient)
               -- MySQL TEXT এর মতো কিন্তু আরো efficient
               -- Index, default value — সব করা যায়
               -- PostgreSQL এ VARCHAR(255) এর চেয়ে TEXT + CHECK ভালো
```

**PostgreSQL এ TEXT efficient:**
```sql
-- MySQL তে TEXT এ সমস্যা আছে, PostgreSQL এ নেই
name TEXT NOT NULL CHECK (length(name) <= 200)
-- VARCHAR(200) এর চেয়ে TEXT + CHECK পছন্দ করে অনেক PostgreSQL DBA
```

---

## 3.3 Date and Time Types

```
DATE                → 4 bytes  → '2024-03-08'
TIME                → 8 bytes  → '14:30:00'
TIME WITH TIME ZONE → 12 bytes → '14:30:00+06'
TIMESTAMP           → 8 bytes  → '2024-03-08 14:30:00' (no timezone)
TIMESTAMPTZ         → 8 bytes  → '2024-03-08 14:30:00+06' (UTC stored)
INTERVAL            → 16 bytes → '1 year 2 months 3 days'
```

**MySQL vs PostgreSQL:**
```
MySQL DATETIME  ≈ PostgreSQL TIMESTAMP
MySQL TIMESTAMP ≈ PostgreSQL TIMESTAMPTZ

PostgreSQL TIMESTAMPTZ সবসময় UTC তে store করে
SET timezone = 'Asia/Dhaka'; করলে display এ +06 দেখাবে
```

```sql
-- Best practice
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()

-- Trigger দিয়ে updated_at auto-update:
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_updated_at
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

## 3.4 Boolean

```sql
is_active BOOLEAN NOT NULL DEFAULT TRUE

-- Values:
TRUE  / 't' / 'true'  / 'yes' / 'on'  / '1'
FALSE / 'f' / 'false' / 'no'  / 'off' / '0'
```

MySQL তে BOOLEAN = TINYINT(1)। PostgreSQL এ আলাদা data type, 1 byte।

---

## 3.5 JSON vs JSONB

```sql
JSON   → Text হিসেবে store, insert order preserve, parse on read
JSONB  → Binary format, parse on write, index possible, faster queries
       → সাধারণত JSONB ই ব্যবহার করো

-- JSONB operators:
->   → JSON object field (returns JSON)
->>  → JSON object field (returns TEXT)
#>   → Nested path (returns JSON)
#>>  → Nested path (returns TEXT)
@>   → Contains
<@   → Is contained by
?    → Key exists
```

```sql
CREATE TABLE products (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name     TEXT NOT NULL,
    metadata JSONB
);

INSERT INTO products (name, metadata)
VALUES ('Laptop', '{"brand": "Dell", "specs": {"ram": 16, "ssd": 512}}');

-- Query
SELECT metadata->>'brand' AS brand FROM products;
SELECT metadata#>>'{specs,ram}' AS ram FROM products;
SELECT * FROM products WHERE metadata @> '{"brand": "Dell"}';
SELECT * FROM products WHERE metadata ? 'brand';

-- Index on JSONB (GIN)
CREATE INDEX idx_metadata ON products USING GIN(metadata);
CREATE INDEX idx_brand ON products USING GIN((metadata->>'brand'));

-- Generated column (MySQL এর মতো)
ALTER TABLE products
ADD COLUMN brand TEXT GENERATED ALWAYS AS (metadata->>'brand') STORED;
CREATE INDEX idx_brand ON products(brand);
```

---

## 3.6 Arrays

PostgreSQL এর unique feature — column এ array store করা।

```sql
CREATE TABLE posts (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title    TEXT NOT NULL,
    tags     TEXT[],
    scores   INTEGER[]
);

INSERT INTO posts (title, tags) VALUES ('PostgreSQL', ARRAY['db', 'sql', 'open-source']);

-- Query
SELECT * FROM posts WHERE 'db' = ANY(tags);
SELECT * FROM posts WHERE tags @> ARRAY['db', 'sql'];
SELECT array_length(tags, 1) FROM posts;
SELECT unnest(tags) AS tag FROM posts;  -- array → rows

-- GIN Index
CREATE INDEX idx_tags ON posts USING GIN(tags);
```

---

## 3.7 UUID

```sql
-- Extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

id UUID DEFAULT uuid_generate_v4() PRIMARY KEY
-- অথবা PostgreSQL 13+:
id UUID DEFAULT gen_random_uuid() PRIMARY KEY
```

MySQL তে UUID() function আছে কিন্তু primary key হিসেবে সরাসরি efficient নয়।

---

## 3.8 Range Types

PostgreSQL unique — range of values এক column এ।

```sql
DATERANGE     -- [2024-01-01, 2024-12-31]
TSRANGE       -- timestamp range
TSTZRANGE     -- timestamptz range
INT4RANGE     -- integer range
INT8RANGE     -- bigint range
NUMRANGE      -- numeric range

-- Example: Hotel booking
CREATE TABLE bookings (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    room_id     INTEGER,
    stay_period DATERANGE,
    EXCLUDE USING GIST (room_id WITH =, stay_period WITH &&)
    -- একই room এ overlapping booking prevent
);

INSERT INTO bookings (room_id, stay_period)
VALUES (101, '[2024-03-01, 2024-03-07)');

SELECT * FROM bookings WHERE stay_period @> '2024-03-04'::date;
```

---

## 3.9 Practical Table Design

```sql
CREATE TABLE users (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username     TEXT NOT NULL UNIQUE CHECK (length(username) BETWEEN 3 AND 50),
    email        TEXT NOT NULL UNIQUE CHECK (email ~* '^[^@]+@[^@]+\.[^@]+$'),
    full_name    TEXT NOT NULL,
    bio          TEXT,
    country_code CHAR(2) NOT NULL DEFAULT 'BD',
    status       TEXT NOT NULL DEFAULT 'active'
                     CHECK (status IN ('active', 'inactive', 'banned')),
    age          SMALLINT CHECK (age >= 0 AND age <= 150),
    login_count  INTEGER NOT NULL DEFAULT 0,
    balance      NUMERIC(12, 2) NOT NULL DEFAULT 0.00,
    date_of_birth DATE,
    metadata     JSONB,
    tags         TEXT[],
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_login_at TIMESTAMPTZ,
    is_verified  BOOLEAN NOT NULL DEFAULT FALSE
);

-- updated_at trigger
CREATE TRIGGER trg_users_updated_at
BEFORE UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

---

# 4. Indexes

## 4.1 B-Tree Index (Default)

```sql
-- Default index type
CREATE INDEX idx_email ON users(email);
CREATE UNIQUE INDEX idx_username ON users(username);

-- Use case: =, <, >, <=, >=, BETWEEN, IN, LIKE 'prefix%'
-- Composite (left-prefix rule — MySQL এর মতো)
CREATE INDEX idx_country_status ON users(country_code, status);
```

**Left-Prefix Rule:**
```sql
-- (A, B, C) index কাজ করে:
WHERE A = ...
WHERE A = ... AND B = ...
WHERE A = ... AND B = ... AND C = ...
WHERE A = ... ORDER BY B
-- কাজ করে না:
WHERE B = ...          -- A নেই
WHERE C = ...          -- A, B নেই
```

---

## 4.2 Hash Index

```sql
CREATE INDEX idx_id_hash ON users USING HASH(id);
-- শুধু equality (=) এ কাজ করে
-- B-Tree এর চেয়ে faster for equality
-- Range query, sorting এ কাজ করে না
-- WAL logged (PostgreSQL 10+), safe to use
```

---

## 4.3 GIN (Generalized Inverted Index)

Full-text search, array, JSONB এর জন্য।

```sql
-- JSONB index
CREATE INDEX idx_metadata ON products USING GIN(metadata);

-- Array index
CREATE INDEX idx_tags ON posts USING GIN(tags);

-- Full-text search
CREATE INDEX idx_fts ON articles USING GIN(to_tsvector('english', content));

-- tsvector column দিয়ে (better)
ALTER TABLE articles ADD COLUMN content_tsv TSVECTOR
    GENERATED ALWAYS AS (to_tsvector('english', content)) STORED;
CREATE INDEX idx_fts ON articles USING GIN(content_tsv);

SELECT * FROM articles WHERE content_tsv @@ to_tsquery('postgresql & index');
```

---

## 4.4 GiST (Generalized Search Tree)

Geometric data, range types, full-text search।

```sql
-- Range type overlap query
CREATE INDEX idx_stay ON bookings USING GIST(stay_period);
SELECT * FROM bookings WHERE stay_period && '[2024-03-01, 2024-03-07)';

-- PostGIS (geographic)
CREATE INDEX idx_location ON places USING GIST(coordinates);

-- Full-text (GIN এর চেয়ে slower update, faster query)
CREATE INDEX idx_fts ON articles USING GIST(to_tsvector('english', content));
```

---

## 4.5 BRIN (Block Range Index)

বড় table এ naturally ordered data এর জন্য (timestamp, sequence)।

```sql
-- Time-series table এ
CREATE INDEX idx_created ON events USING BRIN(created_at);
-- B-Tree এর চেয়ে অনেক ছোট size
-- Sequential data তে effective
-- Random data তে কাজ করে না
```

---

## 4.6 Partial Index

Condition সহ index — subset of rows।

```sql
-- শুধু active user এর email এ index
CREATE INDEX idx_active_email ON users(email)
WHERE status = 'active';

-- শুধু pending orders এ index
CREATE INDEX idx_pending_orders ON orders(created_at)
WHERE status = 'pending';

-- Query তে WHERE condition match করলেই index use হবে
SELECT * FROM users WHERE email = 'x@x.com' AND status = 'active';
-- → Partial index use করবে (ছোট index, faster)
```

---

## 4.7 Expression Index

Function এর result এ index।

```sql
-- Case-insensitive email search
CREATE INDEX idx_lower_email ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- → Index use করবে

-- Date part index
CREATE INDEX idx_year ON orders(EXTRACT(YEAR FROM created_at));
SELECT * FROM orders WHERE EXTRACT(YEAR FROM created_at) = 2024;

-- MySQL তে function on column এ index কাজ করে না
-- PostgreSQL এ Expression Index দিয়ে সম্ভব
```

---

## 4.8 Covering Index (INCLUDE)

```sql
-- PostgreSQL 11+ এ INCLUDE clause
CREATE INDEX idx_user_covering ON orders(user_id) INCLUDE (status, total);
-- user_id এ filter, status এবং total table touch ছাড়াই পাওয়া যাবে

SELECT status, total FROM orders WHERE user_id = 5;
-- Index Only Scan — heap access নেই
```

---

## 4.9 EXPLAIN — Execution Plan Analysis

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE email = 'test@example.com';
EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...;
```

**Scan Types:**
```
Seq Scan         → Full table scan ← এটা দেখলেই সমস্যা (বড় table)
Index Scan       → Index → Heap fetch
Index Only Scan  → Index মাত্র (Covering Index) ✅
Bitmap Heap Scan → Bitmap of matching pages → Heap (multiple rows)
Bitmap Index Scan → Index → Bitmap তৈরি
```

**Join Types:**
```
Nested Loop   → Small inner table
Hash Join     → Large table, equality join
Merge Join    → Both sides sorted
```

**Important fields:**
```
cost=0.00..8.30      → (startup_cost..total_cost)
rows=1               → Estimated rows
actual time=0.1..0.2 → Real time (ANALYZE এ)
actual rows=1        → Real rows (ANALYZE এ)
Buffers: hit=5       → Shared Buffers hit/read
```

```sql
-- Slow query এর plan বুঝতে
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, count(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.country_code = 'BD'
GROUP BY u.name;
```

---

## 4.10 Index Design Rules

```sql
-- Rule 1: WHERE, JOIN column এ index
CREATE INDEX idx_user_id ON orders(user_id);

-- Rule 2: ORDER BY column এ (filesort এড়াতে)
CREATE INDEX idx_created ON orders(created_at DESC);

-- Rule 3: Partial Index for filtered queries
CREATE INDEX idx_active ON users(email) WHERE is_active = TRUE;

-- Rule 4: Expression Index for function queries
CREATE INDEX idx_lower ON users(LOWER(email));

-- Rule 5: Covering Index for Index Only Scan
CREATE INDEX idx_cov ON orders(user_id) INCLUDE (status, created_at);

-- Rule 6: Composite → equality আগে, range পরে
CREATE INDEX idx_user_status_date ON orders(user_id, status, created_at);

-- Unused index খোঁজো
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%pkey%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index bloat দেখো
SELECT indexname, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
```

---

# 5. Transactions & Locking

## 5.1 ACID in PostgreSQL

**Atomicity:** WAL এবং CLOG দিয়ে। Crash হলে incomplete transaction এর XID CLOG এ ABORTED হয়।

**Consistency:** CHECK constraints, triggers, foreign keys enforce করা হয়।

**Isolation:** MVCC দিয়ে। Tuple versioning।

**Durability:** WAL synchronous flush (synchronous_commit=on)।

---

## 5.2 Basic Commands

```sql
BEGIN;                    -- Transaction শুরু
BEGIN ISOLATION LEVEL REPEATABLE READ;
COMMIT;
ROLLBACK;

SAVEPOINT my_save;
ROLLBACK TO SAVEPOINT my_save;
RELEASE SAVEPOINT my_save;

-- Default: autocommit ON (প্রতিটা statement নিজেই transaction)
-- BEGIN এ override হয়
```

---

## 5.3 Isolation Levels

**চারটা সমস্যা:**

| সমস্যা | মানে |
|---|---|
| Dirty Read | Uncommitted data পড়া |
| Non-Repeatable Read | Same query দুইবার আলাদা result |
| Phantom Read | Range query তে নতুন row দেখা |
| Serialization Anomaly | Concurrent transaction এ logical inconsistency |

**PostgreSQL Isolation:**

| Level | Dirty Read | Non-Repeatable | Phantom | Serialization |
|---|---|---|---|---|
| READ COMMITTED (default) | না | হয় | হয় | হয় |
| REPEATABLE READ | না | না | না* | হয় |
| SERIALIZABLE | না | না | না | না |

*PostgreSQL এ REPEATABLE READ Phantom Read ও prevent করে।

**PostgreSQL তে READ UNCOMMITTED নেই** — minimum READ COMMITTED।

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Default change
SET default_transaction_isolation = 'repeatable read';
-- postgresql.conf:
-- default_transaction_isolation = 'read committed'
```

**SSI (Serializable Snapshot Isolation):**
PostgreSQL এর SERIALIZABLE level সত্যিকারের serializable — Predicate Lock ব্যবহার করে। MySQL SERIALIZABLE এর চেয়ে অনেক smart।

---

## 5.4 MVCC Deep Dive

```sql
-- Transaction ID দেখো
SELECT txid_current();

-- Snapshot দেখো
SELECT txid_current_snapshot();
-- Returns: xmin:xmax:xip_list
-- xmin: oldest active transaction
-- xmax: next XID to be assigned
-- xip_list: active transactions list

-- Tuple visibility চেক
SELECT xmin, xmax, ctid, * FROM users WHERE id = 5;
```

---

## 5.5 Lock Types

**Table-Level Locks (LOCK TABLE):**
```
ACCESS SHARE          → SELECT
ROW SHARE             → SELECT FOR UPDATE/SHARE
ROW EXCLUSIVE         → INSERT, UPDATE, DELETE
SHARE UPDATE EXCLUSIVE → VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY
SHARE                 → CREATE INDEX (non-concurrent)
SHARE ROW EXCLUSIVE   → Rare
EXCLUSIVE             → Rare
ACCESS EXCLUSIVE      → DROP TABLE, TRUNCATE, REINDEX, ALTER TABLE ← সবচেয়ে strict
```

```sql
-- Explicit table lock
LOCK TABLE users IN ACCESS EXCLUSIVE MODE;
LOCK TABLE users IN ROW EXCLUSIVE MODE;
```

**Row-Level Locks:**
```sql
SELECT * FROM users WHERE id = 5 FOR UPDATE;          -- Exclusive row lock
SELECT * FROM users WHERE id = 5 FOR SHARE;           -- Shared row lock
SELECT * FROM users WHERE id = 5 FOR UPDATE SKIP LOCKED; -- Lock নেওয়া row skip
SELECT * FROM users WHERE id = 5 FOR UPDATE NOWAIT;   -- Lock না পেলে error

-- SKIP LOCKED — Job Queue pattern:
SELECT * FROM job_queue WHERE status = 'pending'
ORDER BY created_at LIMIT 1
FOR UPDATE SKIP LOCKED;
-- অন্য worker এর নেওয়া job skip করে
```

---

## 5.6 Deadlock

```sql
-- Deadlock log দেখো
-- postgresql.conf: log_lock_waits = on
-- ERROR: deadlock detected
-- DETAIL: Process 12345 waits for ShareLock on transaction 56789

-- Prevention:
-- সবসময় একই order এ rows access করো
-- Transaction ছোট রাখো
-- NOWAIT বা SKIP LOCKED ব্যবহার করো
```

---

## 5.7 Advisory Locks

Application-level custom lock mechanism। MySQL তে নেই।

```sql
-- Session-level (connection শেষে auto-release)
SELECT pg_advisory_lock(12345);        -- exclusive
SELECT pg_advisory_lock_shared(12345); -- shared
SELECT pg_advisory_unlock(12345);

-- Transaction-level (transaction শেষে auto-release)
SELECT pg_advisory_xact_lock(12345);

-- Non-blocking
SELECT pg_try_advisory_lock(12345);    -- TRUE/FALSE return করে
-- Use case: "শুধু একটা process চলুক" গ্যারান্টি
```

---

## 5.8 Lock Monitoring

```sql
-- Current locks
SELECT
    pid, locktype, relation::regclass, mode, granted, query
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE relation IS NOT NULL
ORDER BY granted;

-- Lock waits
SELECT
    blocked.pid     AS blocked_pid,
    blocked.query   AS blocked_query,
    blocker.pid     AS blocker_pid,
    blocker.query   AS blocker_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocker
    ON blocker.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Long running transactions
SELECT pid, now() - pg_stat_activity.xact_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - xact_start) > INTERVAL '5 minutes'
ORDER BY duration DESC;

-- Kill stuck process
SELECT pg_terminate_backend(pid);
SELECT pg_cancel_backend(pid);   -- Query cancel, connection রাখে
```

---

# 6. Query Optimization

## 6.1 Statistics and ANALYZE

```sql
-- Statistics দেখো
SELECT attname, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'orders';

-- Statistics update করো
ANALYZE orders;
ANALYZE VERBOSE orders;

-- Specific column এ more statistics
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ANALYZE orders;
```

---

## 6.2 EXPLAIN — Full Usage

```sql
-- Basic plan
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- Actual execution + buffer info
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- JSON format (pg_badger, pgBadger tools এ useful)
EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...;

-- All options
EXPLAIN (
    ANALYZE true,
    VERBOSE true,
    COSTS true,
    SETTINGS true,
    BUFFERS true,
    WAL true,
    TIMING true,
    SUMMARY true,
    FORMAT text
) SELECT ...;
```

**Plan পড়ার নিয়ম:**
```
EXPLAIN output নিচ থেকে উপরে পড়তে হয়
সবচেয়ে ভেতরের (indented) node আগে execute হয়

Seq Scan on orders (cost=0.00..45231.00 rows=1000000 width=64)
                    └── startup_cost..total_cost  rows  bytes_per_row
```

---

## 6.3 Common Slow Patterns

### Pattern 1 — Sequential Scan
```sql
-- ❌
EXPLAIN SELECT * FROM orders WHERE user_id = 5;
-- Seq Scan (no index)

-- ✅
CREATE INDEX idx_user_id ON orders(user_id);
-- Index Scan
```

### Pattern 2 — Function on Column
```sql
-- ❌ index কাজ করে না
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
SELECT * FROM orders WHERE DATE(created_at) = '2024-03-08';

-- ✅ Expression Index
CREATE INDEX idx_lower_email ON users(LOWER(email));
SELECT * FROM orders
WHERE created_at >= '2024-03-08' AND created_at < '2024-03-09';
```

### Pattern 3 — LIKE with leading wildcard
```sql
-- ❌
SELECT * FROM products WHERE name LIKE '%phone%';

-- ✅ Full-Text Search
CREATE INDEX idx_fts ON products USING GIN(to_tsvector('english', name));
SELECT * FROM products WHERE to_tsvector('english', name) @@ to_tsquery('phone');

-- ✅ pg_trgm (partial LIKE)
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_trgm ON products USING GIN(name gin_trgm_ops);
SELECT * FROM products WHERE name LIKE '%phone%'; -- এখন index use করবে
```

### Pattern 4 — N+1 Query
```sql
-- ❌
SELECT id FROM users WHERE country = 'BD';
-- Loop করে N queries:
SELECT COUNT(*) FROM orders WHERE user_id = ?;

-- ✅ JOIN
SELECT u.id, u.name, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.country = 'BD'
GROUP BY u.id, u.name;
```

### Pattern 5 — Large OFFSET
```sql
-- ❌
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 1000000;

-- ✅ Keyset Pagination
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

### Pattern 6 — Implicit Cast
```sql
-- ❌ user_id INTEGER কিন্তু string দিলে
SELECT * FROM users WHERE user_id = '5';  -- implicit cast

-- ✅
SELECT * FROM users WHERE user_id = 5;
```

---

## 6.4 Planner Settings Override

```sql
-- Index Force করো
SET enable_seqscan = off;   -- seq scan disable → index prefer
EXPLAIN SELECT ...;
SET enable_seqscan = on;    -- reset

-- Per-query hint (pg_hint_plan extension)
/*+ IndexScan(users users_email_idx) */ SELECT * FROM users WHERE email = ...;
```

---

## 6.5 Table Partitioning

Large table কে ছোট physical parts এ ভাগ করা।

```sql
-- Range Partitioning
CREATE TABLE orders (
    id         BIGINT GENERATED ALWAYS AS IDENTITY,
    user_id    INTEGER,
    total      NUMERIC(10,2),
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- List Partitioning
CREATE TABLE users_by_country (
    id          BIGINT,
    country     TEXT NOT NULL
) PARTITION BY LIST (country);

CREATE TABLE users_bd PARTITION OF users_by_country FOR VALUES IN ('BD');
CREATE TABLE users_us PARTITION OF users_by_country FOR VALUES IN ('US');
CREATE TABLE users_other PARTITION OF users_by_country DEFAULT;

-- Hash Partitioning
CREATE TABLE events (id BIGINT, data TEXT)
PARTITION BY HASH (id);
CREATE TABLE events_0 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_1 PARTITION OF events FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ...

-- Partition pruning দেখো
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-01-01';
-- শুধু orders_2024 scan করবে — Partition Pruning
```

---

## 6.6 CTEs (Common Table Expressions)

```sql
-- Basic CTE
WITH active_users AS (
    SELECT id, name FROM users WHERE status = 'active'
),
user_orders AS (
    SELECT user_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY user_id
)
SELECT u.name, COALESCE(uo.order_count, 0) AS orders
FROM active_users u
LEFT JOIN user_orders uo ON u.id = uo.user_id;

-- Recursive CTE (MySQL 8.0 এও আছে)
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS level
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, ct.level + 1
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree ORDER BY level, name;

-- MATERIALIZED hint (PostgreSQL 12+)
WITH data AS MATERIALIZED (
    SELECT * FROM large_table WHERE condition = true
)
SELECT * FROM data WHERE other = 'x';
-- NOT MATERIALIZED: CTE inline করো (optimizer decide করতে পারে)
```

---

## 6.7 Parallel Query

```ini
max_parallel_workers_per_gather = 4
max_parallel_workers            = 8
```

```sql
EXPLAIN SELECT COUNT(*) FROM large_table;
-- Gather (Parallel Seq Scan)

-- Disable করো (debug এর জন্য)
SET max_parallel_workers_per_gather = 0;
```

---

## 6.8 Optimization Checklist

```
Query:
  ✅ SELECT * এড়াও
  ✅ WHERE column এ function ব্যবহার করো না (Expression Index দাও)
  ✅ JOIN column এ index দাও
  ✅ N+1 query এড়াও
  ✅ Keyset Pagination ব্যবহার করো

Index:
  ✅ B-Tree: equality, range, sort
  ✅ GIN: JSONB, array, full-text
  ✅ Partial Index: low cardinality filter
  ✅ Expression Index: function on column
  ✅ Covering Index (INCLUDE): Index Only Scan
  ✅ Unused index DROP করো

Maintenance:
  ✅ VACUUM ANALYZE নিয়মিত চালাও
  ✅ Autovacuum properly configured
  ✅ Statistics up-to-date
  ✅ Table bloat monitor করো
```

---

# 7. Backup & Recovery

## 7.1 pg_dump — Logical Backup

Single database backup।

```bash
# Basic
pg_dump -U postgres mydb > mydb_backup.sql

# Compressed
pg_dump -U postgres -Fc mydb > mydb_backup.dump
# -Fc = custom format (compressed, parallel restore possible)
# -Ft = tar format
# -Fd = directory format (parallel pg_dump)
# -Fp = plain SQL (default)

# Schema only
pg_dump -U postgres -s mydb > mydb_schema.sql

# Data only
pg_dump -U postgres -a mydb > mydb_data.sql

# Specific table
pg_dump -U postgres -t users mydb > users_backup.sql

# Exclude table
pg_dump -U postgres -T logs mydb > mydb_no_logs.sql

# Parallel (directory format only)
pg_dump -U postgres -Fd -j 4 -f /backup/mydb_dir mydb
# -j 4 = 4 parallel jobs

# With progress
pg_dump -U postgres -Fc --verbose mydb > mydb_backup.dump 2>&1

# Remote
PGPASSWORD=secret pg_dump -U postgres -h 172.16.93.140 -p 5432 mydb | gzip > backup.sql.gz
```

---

## 7.2 pg_dumpall — Cluster-wide Backup

Roles, tablespaces এবং সব database backup।

```bash
# Full cluster backup
pg_dumpall -U postgres > cluster_backup.sql

# Globals only (roles, tablespaces — no data)
pg_dumpall -U postgres --globals-only > globals_backup.sql

# Schema only
pg_dumpall -U postgres --schema-only > schema_backup.sql

# Compressed
pg_dumpall -U postgres | gzip > cluster_backup_$(date +%Y%m%d).sql.gz
```

---

## 7.3 pg_restore

```bash
# Custom format restore
pg_restore -U postgres -d mydb mydb_backup.dump

# Create database and restore
pg_restore -U postgres -C -d postgres mydb_backup.dump
# -C = CREATE DATABASE, -d postgres = connect এখানে, DB create করে নতুনটায় restore

# Specific table restore
pg_restore -U postgres -d mydb -t users mydb_backup.dump

# Schema only
pg_restore -U postgres -d mydb -s mydb_backup.dump

# Data only
pg_restore -U postgres -d mydb -a mydb_backup.dump

# Parallel restore
pg_restore -U postgres -d mydb -j 4 mydb_backup.dump
# -j 4 = 4 parallel jobs (custom/directory format)

# Verbose
pg_restore -U postgres -d mydb --verbose mydb_backup.dump

# List archive contents
pg_restore -l mydb_backup.dump

# Restore plain SQL
psql -U postgres -d mydb < mydb_backup.sql
```

---

## 7.4 pg_basebackup — Physical Backup

Full cluster physical backup। PITR এর base।

```bash
# Basic
pg_basebackup -U replicator -h localhost -D /backup/base -P

# Compressed tar
pg_basebackup -U replicator -h localhost \
    -D /backup/base \
    -Ft -z \
    -P --wal-method=stream

# Options:
# -D = destination directory
# -Ft = tar format
# -z = gzip compress
# -P = progress
# --wal-method=stream → WAL stream করে backup এর সময়
# --wal-method=fetch  → Backup শেষে WAL fetch
# --checkpoint=fast   → Fast checkpoint trigger

# Replication user তৈরি করো
CREATE USER replicator REPLICATION LOGIN PASSWORD 'ReplicaPass@2024';
# pg_hba.conf এ:
# host replication replicator 172.16.93.0/24 md5

# Verify backup
pg_basebackup --verify -D /backup/base
```

---

## 7.5 WAL Archiving

PITR এর জন্য WAL segments archive করা।

```ini
# postgresql.conf
wal_level     = replica
archive_mode  = on
archive_command = 'test ! -f /archive/wal/%f && cp %p /archive/wal/%f'
# %p = path to WAL file
# %f = filename only
# test ! -f → file না থাকলেই copy (re-archive prevent)

archive_timeout = 300  # Force archive every 5 minutes
                       # Idle server এও WAL archive হবে
```

```bash
# Archive directory তৈরি করো
mkdir -p /archive/wal
chown postgres:postgres /archive/wal

# Archiving কাজ করছে কিনা দেখো
SELECT * FROM pg_stat_archiver;
-- last_archived_wal: শেষ archive হওয়া WAL
-- failed_count: failure count → 0 হওয়া উচিত
```

---

## 7.6 Point-in-Time Recovery (PITR)

**Scenario:** সকাল ১০টায় ভুলে `DELETE FROM orders WHERE 1=1` হয়েছে।

```bash
# Step 1: Server stop করো
sudo systemctl stop postgresql-16

# Step 2: Current data directory backup (safety)
mv /var/lib/pgsql/16/data /var/lib/pgsql/16/data.broken

# Step 3: Base backup থেকে restore করো
mkdir -p /var/lib/pgsql/16/data
tar -xzf /backup/base/base.tar.gz -C /var/lib/pgsql/16/data
chown -R postgres:postgres /var/lib/pgsql/16/data

# Step 4: Recovery configuration তৈরি করো
cat > /var/lib/pgsql/16/data/postgresql.conf << 'EOF'
# Recovery settings
restore_command = 'cp /archive/wal/%f %p'
recovery_target_time = '2024-03-08 09:59:59'   # DELETE এর ঠিক আগে
recovery_target_action = 'promote'              # Recover হলে primary হয়ে যাবে
EOF

# অথবা specific transaction ID তে stop:
# recovery_target_xid = '12345'

# Step 5: recovery signal file তৈরি করো (PostgreSQL 12+)
touch /var/lib/pgsql/16/data/recovery.signal

# Step 6: Start করো
sudo systemctl start postgresql-16
# Log দেখো
sudo tail -f /var/lib/pgsql/16/data/log/postgresql-*.log
# "recovery stopping before commit ..." দেখবে
# তারপর "database system is ready"

# Step 7: Recovery complete করো (যদি promote না হয়েছে)
SELECT pg_wal_replay_resume();

# Step 8: Data verify করো
SELECT COUNT(*) FROM orders;
```

---

## 7.7 pgBackRest — Production Backup Solution

MySQL এ XtraBackup এর মতো। Full, differential, incremental backup + PITR।

```bash
# Install (Rocky Linux 9)
sudo dnf install -y pgbackrest

# Configuration file
sudo nano /etc/pgbackrest/pgbackrest.conf
```

```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2          # 2টা full backup রাখো
repo1-retention-diff=14         # 14টা differential
log-level-console=info
log-level-file=debug
start-fast=y                    # Fast checkpoint

[main]                          # Stanza name
pg1-path=/var/lib/pgsql/16/data
pg1-port=5432
pg1-user=postgres
```

```bash
# Stanza তৈরি করো
sudo -u postgres pgbackrest --stanza=main stanza-create

# postgresql.conf এ archive command আপডেট করো
archive_command = 'pgbackrest --stanza=main archive-push %p'
# Reload: SELECT pg_reload_conf();

# Full backup
sudo -u postgres pgbackrest --stanza=main --type=full backup

# Differential backup (last full থেকে)
sudo -u postgres pgbackrest --stanza=main --type=diff backup

# Incremental backup (last backup থেকে)
sudo -u postgres pgbackrest --stanza=main --type=incr backup

# Backup list দেখো
sudo -u postgres pgbackrest --stanza=main info

# Verify backup
sudo -u postgres pgbackrest --stanza=main check

# Restore — Full
sudo systemctl stop postgresql-16
sudo -u postgres pgbackrest --stanza=main restore

# PITR restore
sudo -u postgres pgbackrest --stanza=main restore \
    --target="2024-03-08 09:59:59" \
    --target-action=promote \
    --type=time

sudo systemctl start postgresql-16

# Automated backup (cron)
# Full: রবিবার রাত ১টায়
0 1 * * 0 postgres pgbackrest --stanza=main --type=full backup
# Diff: সোম-শনি রাত ১টায়
0 1 * * 1-6 postgres pgbackrest --stanza=main --type=diff backup
```

**pgBackRest এর সুবিধা:**
- Parallel backup এবং restore
- Compression (lz4, zstd)
- Encryption
- Remote backup (S3, Azure, GCS)
- WAL push/get built-in
- Retention management automatic

---

## 7.8 Barman — Backup and Recovery Manager

Remote backup management, multiple server handle করে।

```bash
# Install (backup server এ)
sudo dnf install -y barman

# /etc/barman.conf
[barman]
barman_home = /var/lib/barman
barman_user = barman
log_file = /var/log/barman/barman.log
compression = gzip
reuse_backup = link
backup_method = rsync

# /etc/barman.d/pgserver.conf (per server)
[pgserver]
description = "Main PostgreSQL Server"
conninfo = host=172.16.93.140 user=barman dbname=postgres
ssh_command = ssh postgres@172.16.93.140
backup_method = rsync
archiver = on
streaming_archiver = on
streaming_conninfo = host=172.16.93.140 user=streaming_barman
slot_name = barman_slot
```

```bash
# PostgreSQL Server এ setup
CREATE USER barman SUPERUSER PASSWORD 'BarmanPass@2024';
CREATE USER streaming_barman REPLICATION PASSWORD 'StreamPass@2024';

# pg_hba.conf এ:
# host all barman 172.16.93.150/32 md5
# host replication streaming_barman 172.16.93.150/32 md5

# Barman server এ
barman check pgserver           # Connection check
barman switch-wal pgserver      # WAL switch trigger
barman cron                     # Maintenance run
barman backup pgserver          # Backup নাও
barman list-backup pgserver     # Backup list
barman show-backup pgserver latest  # Backup details

# Restore
barman recover pgserver latest /var/lib/pgsql/16/data \
    --target-time "2024-03-08 09:59:59" \
    --remote-ssh-command "ssh postgres@172.16.93.140"
```

---

## 7.9 Backup Best Practices

```
Strategy:
  ✅ 3-2-1 Rule: 3 copies, 2 media, 1 offsite
  ✅ Daily full backup (pgBackRest) + Continuous WAL archiving
  ✅ Monthly restore test on separate server
  ✅ PITR capability verify করো

pgBackRest vs Barman:
  pgBackRest: Single server, simple, S3 support ✅
  Barman:     Multi-server management, enterprise features

pg_dump vs pg_basebackup:
  pg_dump:      Single database, portable, version agnostic
  pg_basebackup: Full cluster, physical, PITR possible

Performance:
  ✅ Replica থেকে backup নাও (Primary এ load নেই)
  ✅ Off-peak hours এ
  ✅ Compression always on
  ✅ Parallel backup/restore use করো
```

---

# 8. Monitoring

## 8.1 OS Level

```bash
# CPU
top -u postgres
ps aux | grep postgres

# Memory
free -h
cat /proc/$(pgrep -f "postmaster")/status | grep VmRSS

# Disk
df -h /var/lib/pgsql
iostat -x 1 5

# PostgreSQL process count
ps aux | grep "postgres:" | wc -l
```

---

## 8.2 pg_stat_activity — Connection and Query Monitoring

```sql
-- সব active connections
SELECT pid, usename, application_name, client_addr,
    state, wait_event_type, wait_event,
    now() - xact_start AS transaction_age,
    now() - state_change AS state_age,
    LEFT(query, 100) AS query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY transaction_age DESC NULLS LAST;

-- Connection count by state
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;

-- Idle in transaction (dangerous — holding locks)
SELECT pid, usename, now() - xact_start AS idle_duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_duration DESC;

-- Long running queries
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE (now() - query_start) > INTERVAL '5 minutes'
ORDER BY duration DESC;

-- Connection utilization
SELECT
    count(*) AS current_connections,
    max_conn,
    ROUND(count(*) * 100.0 / max_conn, 2) AS utilization_pct
FROM pg_stat_activity,
     (SELECT setting::int AS max_conn FROM pg_settings WHERE name = 'max_connections') mc
GROUP BY max_conn;
```

---

## 8.3 pg_stat_user_tables — Table Health

```sql
-- Table access statistics
SELECT
    relname AS table_name,
    seq_scan,
    seq_tup_read,
    idx_scan,
    idx_tup_fetch,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_vacuum,
    last_autovacuum,
    last_analyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Tables needing VACUUM (dead > 10%)
SELECT relname, n_dead_tup, n_live_tup,
    ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
  AND n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0) > 10
ORDER BY dead_pct DESC;

-- Table sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                   pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

---

## 8.4 Replication Monitoring

```sql
-- Primary এ: Replica status
SELECT
    application_name,
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag_size,
    write_lag,
    flush_lag,
    replay_lag,
    sync_state
FROM pg_stat_replication;

-- Replica এ: Recovery status
SELECT
    pg_is_in_recovery() AS is_replica,
    pg_last_wal_receive_lsn() AS received_lsn,
    pg_last_wal_replay_lsn()  AS replayed_lsn,
    pg_last_xact_replay_timestamp() AS last_replay_time,
    now() - pg_last_xact_replay_timestamp() AS replication_delay;

-- Replication slots
SELECT
    slot_name,
    slot_type,
    active,
    restart_lsn,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag_size
FROM pg_replication_slots;
-- WARNING: inactive slot এ WAL জমে → disk full হতে পারে
-- Monitor করো এবং unused slot DROP করো
```

---

## 8.5 VACUUM Monitoring

```sql
-- VACUUM progress (currently running)
SELECT
    p.pid,
    relid::regclass AS table_name,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    ROUND(heap_blks_scanned * 100.0 / NULLIF(heap_blks_total, 0), 2) AS progress_pct
FROM pg_stat_progress_vacuum p
JOIN pg_stat_activity a ON p.pid = a.pid;

-- Autovacuum activity
SELECT schemaname, relname, n_dead_tup, n_live_tup,
    last_autovacuum, last_autoanalyze,
    autovacuum_count, autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 20;

-- XID wraparound risk
SELECT datname,
    age(datfrozenxid) AS xid_age,
    2000000000 - age(datfrozenxid) AS safe_transactions_remaining
FROM pg_database
ORDER BY xid_age DESC;
-- age > 1.5 billion → emergency VACUUM needed!
```

---

## 8.6 Lock Monitoring

```sql
-- Blocked queries
SELECT
    blocked.pid         AS blocked_pid,
    blocked.usename     AS blocked_user,
    blocked.query       AS blocked_query,
    blocking.pid        AS blocking_pid,
    blocking.usename    AS blocking_user,
    blocking.query      AS blocking_query
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

---

## 8.7 Index Usage Monitoring

```sql
-- Unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%pkey%'
  AND indexname NOT LIKE '%unique%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Duplicate / redundant indexes
SELECT
    indrelid::regclass AS table_name,
    array_agg(indexrelid::regclass ORDER BY indexrelid) AS overlapping_indexes
FROM pg_index
GROUP BY indrelid, indkey
HAVING COUNT(*) > 1;

-- Index hit rate
SELECT
    schemaname,
    tablename,
    idx_scan,
    seq_scan,
    ROUND(idx_scan * 100.0 / NULLIF(idx_scan + seq_scan, 0), 2) AS idx_scan_pct
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 0
ORDER BY seq_scan DESC LIMIT 20;
```

---

## 8.8 Monitoring Dashboard Script

```bash
#!/bin/bash
# /usr/local/bin/pg_health_check.sh
PSQL="psql -U postgres -t -A -c"

echo "========================================="
echo " PostgreSQL Health Check - $(date)"
echo "========================================="

echo ""
echo "--- VERSION ---"
$PSQL "SELECT version();"

echo ""
echo "--- CONNECTIONS ---"
$PSQL "SELECT state, count(*) FROM pg_stat_activity GROUP BY state ORDER BY count DESC;"
$PSQL "SELECT count(*) AS total,
    (SELECT setting FROM pg_settings WHERE name='max_connections') AS max_allowed
    FROM pg_stat_activity;"

echo ""
echo "--- BUFFER HIT RATE ---"
$PSQL "SELECT ROUND(sum(heap_blks_hit) * 100.0 /
    NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) AS hit_rate_pct
    FROM pg_statio_user_tables;"

echo ""
echo "--- LONG RUNNING QUERIES (>5min) ---"
$PSQL "SELECT pid, now()-query_start AS duration, LEFT(query,80)
    FROM pg_stat_activity
    WHERE (now()-query_start) > INTERVAL '5 min' AND state != 'idle'
    ORDER BY duration DESC;"

echo ""
echo "--- REPLICATION STATUS ---"
$PSQL "SELECT application_name, state,
    pg_size_pretty(sent_lsn - replay_lsn) AS lag,
    replay_lag FROM pg_stat_replication;" 2>/dev/null || \
$PSQL "SELECT pg_is_in_recovery() AS is_replica,
    now() - pg_last_xact_replay_timestamp() AS replication_delay;"

echo ""
echo "--- TOP DEAD TUPLE TABLES ---"
$PSQL "SELECT relname, n_dead_tup, n_live_tup,
    ROUND(n_dead_tup*100.0/NULLIF(n_live_tup+n_dead_tup,0),2) AS dead_pct
    FROM pg_stat_user_tables
    WHERE n_dead_tup > 1000 ORDER BY dead_pct DESC LIMIT 5;"

echo ""
echo "--- DISK USAGE ---"
df -h /var/lib/pgsql 2>/dev/null || df -h /
$PSQL "SELECT pg_size_pretty(pg_database_size(current_database())) AS db_size;"

echo ""
echo "--- XID WRAPAROUND RISK ---"
$PSQL "SELECT datname, age(datfrozenxid) AS xid_age
    FROM pg_database ORDER BY xid_age DESC LIMIT 3;"

echo "========================================="
```

```bash
chmod +x /usr/local/bin/pg_health_check.sh
# Cron এ প্রতি 5 মিনিটে
*/5 * * * * postgres /usr/local/bin/pg_health_check.sh >> /var/log/pg_health.log 2>&1
```

---

## 8.9 Key Metrics Thresholds

| Metric | Healthy | Warning | Critical |
|---|---|---|---|
| Buffer Hit Rate | > 99% | 95-99% | < 95% |
| Connection Utilization | < 70% | 70-85% | > 85% |
| Replication Lag (bytes) | < 1MB | 1-100MB | > 100MB |
| Replication Lag (time) | < 5s | 5-60s | > 60s |
| Dead Tuple % | < 5% | 5-20% | > 20% |
| XID Age | < 500M | 500M-1.5B | > 1.5B |
| Disk Usage | < 70% | 70-85% | > 85% |
| Idle in Transaction | 0 | 1-5 | > 5 |

---

# 9. Security

## 9.1 Authentication — pg_hba.conf

MySQL এ user definition এই host বলা হয় (`user@host`)। PostgreSQL এ আলাদা `pg_hba.conf` file এ rules লেখা হয়।

```
# TYPE   DATABASE   USER       ADDRESS          METHOD
local    all        postgres                    peer
local    all        all                         md5
host     all        all        127.0.0.1/32     scram-sha-256
host     all        all        172.16.93.0/24   scram-sha-256
host     replication replicator 172.16.93.0/24  scram-sha-256
hostssl  all        all        0.0.0.0/0        scram-sha-256
```

**TYPE:**
```
local   → Unix socket connection
host    → TCP/IP (SSL বা non-SSL)
hostssl → TCP/IP SSL only
hostnossl → TCP/IP non-SSL only
```

**METHOD:**
```
trust         → Password ছাড়া (testing only, কখনো production এ না!)
reject        → সবসময় reject
md5           → MD5 password hash
scram-sha-256 → Secure (PostgreSQL 10+, recommended)
peer          → OS username = PG username (local only)
ident         → ident server username mapping
ldap          → LDAP authentication
cert          → SSL certificate
```

```bash
# pg_hba.conf reload (restart ছাড়া)
sudo systemctl reload postgresql-16
# অথবা:
psql -c "SELECT pg_reload_conf();"
```

---

## 9.2 Roles & Privileges

PostgreSQL এ **User** এবং **Role** same concept। `CREATE USER` = `CREATE ROLE WITH LOGIN`।

```sql
-- Role তৈরি
CREATE ROLE app_readonly;
CREATE ROLE app_readwrite;
CREATE ROLE app_admin;

-- Login user তৈরি
CREATE USER appuser   PASSWORD 'AppPass@2024!' IN ROLE app_readwrite;
CREATE USER reporter  PASSWORD 'Report@2024!'  IN ROLE app_readonly;
CREATE USER dbadmin   PASSWORD 'Admin@2024!'   IN ROLE app_admin;

-- Role এ privileges দাও
GRANT CONNECT ON DATABASE mydb TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_readonly;

GRANT CONNECT ON DATABASE mydb TO app_readwrite;
GRANT USAGE ON SCHEMA public TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE ON SEQUENCES TO app_readwrite;

-- Specific table
GRANT SELECT ON orders TO app_readonly;
GRANT SELECT, INSERT ON orders TO app_readwrite;

-- Specific column
GRANT SELECT (id, name, email) ON users TO reporter;
REVOKE SELECT (salary) ON users FROM reporter;

-- Role দেখো
\du                          -- psql এ
SELECT rolname, rolsuper, rolcreatedb, rolcanlogin
FROM pg_roles ORDER BY rolname;

-- User এর privileges দেখো
\dp users                    -- table privileges
SELECT grantee, privilege_type FROM information_schema.role_table_grants
WHERE table_name = 'users';
```

### Schema-based Access Control

```sql
-- প্রতিটা app এর আলাদা schema
CREATE SCHEMA app1;
CREATE SCHEMA app2;

GRANT USAGE ON SCHEMA app1 TO app1user;
GRANT ALL ON ALL TABLES IN SCHEMA app1 TO app1user;
ALTER DEFAULT PRIVILEGES IN SCHEMA app1 GRANT ALL ON TABLES TO app1user;

-- app1user শুধু app1 schema দেখতে পারবে
SET search_path TO app1;
```

---

## 9.3 Row Level Security (RLS)

MySQL তে নেই। PostgreSQL unique feature।

```sql
-- RLS enable করো
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy তৈরি করো
CREATE POLICY orders_isolation ON orders
    USING (user_id = current_user_id());
    -- প্রতিটা user শুধু নিজের orders দেখবে

-- অথবা context variable দিয়ে
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.tenant_id')::integer);

-- Application এ:
SET app.tenant_id = '42';
SELECT * FROM orders;  -- শুধু tenant_id=42 এর data আসবে

-- Superuser bypass করতে:
ALTER TABLE orders FORCE ROW LEVEL SECURITY;

-- Policy দেখো
SELECT * FROM pg_policies WHERE tablename = 'orders';

-- RLS disable করো
ALTER TABLE orders DISABLE ROW LEVEL SECURITY;
```

---

## 9.4 Column Level Security

```sql
-- Column টা hide করতে VIEW ব্যবহার করো
CREATE VIEW users_public AS
    SELECT id, username, email, created_at FROM users;
    -- salary column বাদ

GRANT SELECT ON users_public TO reporter;
REVOKE SELECT ON users FROM reporter;
```

---

## 9.5 SSL/TLS

```ini
# postgresql.conf
ssl            = on
ssl_cert_file  = 'server.crt'
ssl_key_file   = 'server.key'
ssl_ca_file    = 'root.crt'    # Client certificate verify করতে
```

```bash
# Self-signed certificate
openssl req -new -x509 -days 365 -nodes \
    -out /var/lib/pgsql/16/data/server.crt \
    -keyout /var/lib/pgsql/16/data/server.key
chmod 600 /var/lib/pgsql/16/data/server.key
chown postgres:postgres /var/lib/pgsql/16/data/server.*
```

```sql
-- SSL connection require করো (pg_hba.conf)
-- hostssl all all 0.0.0.0/0 scram-sha-256

-- User এ SSL require
ALTER USER appuser SET ssl = on;

-- Current connection SSL check
SELECT ssl, version, cipher, bits FROM pg_stat_ssl WHERE pid = pg_backend_pid();
```

---

## 9.6 pgcrypto — Data Encryption

```sql
CREATE EXTENSION pgcrypto;

-- Symmetric encryption
INSERT INTO sensitive (ssn) VALUES (pgp_sym_encrypt('123-45-6789', 'encryption_key'));
SELECT pgp_sym_decrypt(ssn, 'encryption_key') FROM sensitive;

-- Password hashing
INSERT INTO users (password_hash) VALUES (crypt('mypassword', gen_salt('bf', 10)));
-- bf = bcrypt, 10 = work factor

-- Verify password
SELECT * FROM users WHERE password_hash = crypt('mypassword', password_hash);

-- Random data
SELECT gen_random_uuid();
SELECT gen_random_bytes(32);  -- 32 random bytes

-- Hash
SELECT encode(digest('data', 'sha256'), 'hex');
```

---

## 9.7 pgAudit — Audit Logging

Production grade audit logging। MySQL এর Audit Plugin এর equivalent।

```bash
# Install
sudo dnf install -y pgaudit16_16   # version depends on PG version
# অথবা: sudo dnf install -y pgaudit
```

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'
# Options: read, write, function, role, ddl, misc, all
# write = INSERT, UPDATE, DELETE, TRUNCATE
# ddl   = CREATE, DROP, ALTER

pgaudit.log_catalog = off       # System catalog queries log করো না
pgaudit.log_relation = on       # প্রতিটা relation আলাদা log
pgaudit.log_statement_once = off
```

```sql
-- Extension load করো (restart এর পরে)
CREATE EXTENSION pgaudit;

-- Session level audit
SET pgaudit.log = 'all';

-- Specific role এর audit
CREATE ROLE auditor;
GRANT auditor TO appuser;
-- pgaudit.log_role = 'auditor';
-- auditor role এর সব query log হবে
```

```bash
# Audit log দেখো
sudo tail -f /var/lib/pgsql/16/data/log/postgresql-*.log | grep AUDIT
# Format: AUDIT: SESSION,1,1,DDL,CREATE TABLE,,,CREATE TABLE...
```

---

## 9.8 Security Checklist

```
Authentication:
  ✅ pg_hba.conf এ trust method নেই (production এ)
  ✅ scram-sha-256 authentication (md5 এর চেয়ে secure)
  ✅ Remote postgres superuser access নেই
  ✅ SSL/TLS enabled for remote connections

Roles & Privileges:
  ✅ Least privilege principle
  ✅ Application user এর শুধু দরকারি privilege
  ✅ Role-based access (user কে সরাসরি privilege না)
  ✅ Schema separation for multi-tenant
  ✅ Row Level Security for data isolation

Data:
  ✅ Sensitive column pgcrypto দিয়ে encrypt
  ✅ Password hashing (crypt + bcrypt)
  ✅ Backup encrypt করো (pgBackRest encryption)

Audit:
  ✅ pgaudit installed and configured
  ✅ write এবং ddl operations log হচ্ছে
  ✅ Log regularly review করা হচ্ছে

Network:
  ✅ listen_addresses specific IP তে
  ✅ Firewall port 5432 restrict করো
  ✅ pg_hba.conf এ specific CIDR blocks
```

---

# 10. High Availability & Replication

## 10.1 RPO/RTO

```
RPO (Recovery Point Objective): কতটুকু data loss acceptable?
  → Streaming replication: RPO ≈ 0 (synchronous mode)
  → WAL archiving: RPO = archive_timeout (default 5min)

RTO (Recovery Time Objective): কতক্ষণে service restore?
  → Manual failover: 5-30 minutes
  → Patroni: 30-60 seconds automatic
```

---

## 10.2 Streaming Replication — কীভাবে কাজ করে

```
Primary:
  INSERT/UPDATE/DELETE
       │
       ▼
  WAL Buffer → WAL File (pg_wal/)
       │
       │ WAL Sender Process → Port 5432
       ▼
Standby:
  WAL Receiver Process → Receives WAL stream
       │
       ▼
  pg_wal/ (local) → WAL Applier → Database
```

```
Physical (Streaming) Replication:
  - Block-level replication (byte for byte copy)
  - Primary এর exact replica
  - Read queries on standby (hot_standby=on)
  - Schema ভিন্ন করা যায় না

Logical Replication:
  - Table-level, row-level changes replicate
  - Different schema, different PG version possible
  - Filter করা যায় (specific table)
  - Bidirectional possible
```

---

## 10.3 Install to Streaming Replication — End-to-End

### উভয় Server এ PostgreSQL Install করো

```bash
# Rocky Linux 9
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib

# Initialize (Primary এ)
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl start postgresql-16
sudo systemctl enable postgresql-16

# Temporary access
sudo -u postgres psql
ALTER USER postgres PASSWORD 'PostgresPass@2024!';
\q
```

### Primary Server (172.16.93.140) Configure করো

```bash
sudo nano /var/lib/pgsql/16/data/postgresql.conf
```

```ini
listen_addresses     = '*'
wal_level            = replica
max_wal_senders      = 10
wal_keep_size        = 1GB
max_replication_slots = 10
hot_standby          = on
archive_mode         = on
archive_command      = 'cp %p /archive/wal/%f'
```

```bash
# pg_hba.conf এ replication user allow করো
sudo nano /var/lib/pgsql/16/data/pg_hba.conf
```

```
# replication connection
host    replication     replicator      172.16.93.141/32        scram-sha-256
```

```sql
-- Replication user তৈরি
CREATE USER replicator REPLICATION LOGIN PASSWORD 'ReplicaPass@2024!';

-- Firewall
```

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=172.16.93.141 port port=5432 protocol=tcp accept'
sudo firewall-cmd --reload
sudo systemctl restart postgresql-16
```

### Standby Server (172.16.93.141) Setup করো

```bash
# Primary এর data copy করো
sudo systemctl stop postgresql-16
sudo rm -rf /var/lib/pgsql/16/data/*

sudo -u postgres pg_basebackup \
    -h 172.16.93.140 \
    -U replicator \
    -D /var/lib/pgsql/16/data \
    -P -Xs -R
# -Xs = WAL stream করো backup সময়
# -R  = standby.signal এবং primary_conninfo auto-configure করে

# pg_basebackup automatically তৈরি করে:
# standby.signal → PostgreSQL কে বলে এটা standby
# postgresql.auto.conf এ:
#   primary_conninfo = 'host=172.16.93.140 user=replicator ...'
```

```bash
# Standby এর postgresql.conf এ যোগ করো
sudo nano /var/lib/pgsql/16/data/postgresql.conf
```

```ini
hot_standby = on
```

```bash
sudo systemctl start postgresql-16
sudo systemctl enable postgresql-16

# Verify
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# t মানে standby mode
```

```sql
-- Primary এ replication status
SELECT application_name, client_addr, state, sync_state,
    sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;

-- Standby এ lag দেখো
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;
```

### Live Test

```sql
-- Primary এ
CREATE DATABASE repl_test;
\c repl_test
CREATE TABLE t1 (id SERIAL, msg TEXT);
INSERT INTO t1 (msg) VALUES ('Replication works!');

-- Standby এ
\c repl_test
SELECT * FROM t1;
-- Row দেখা গেলে ✅

-- Standby তে write করার চেষ্টা
INSERT INTO t1 (msg) VALUES ('test');
-- ERROR: cannot execute INSERT in a read-only transaction ✅
```

---

## 10.4 Replication Slots

Slot না থাকলে Replica lag করলে Primary WAL delete করতে পারে → Replica sync হারায়।

```sql
-- Physical replication slot তৈরি করো (Primary এ)
SELECT pg_create_physical_replication_slot('standby_slot');

-- Standby এ use করো (postgresql.auto.conf এ)
primary_slot_name = 'standby_slot'

-- Slot status দেখো
SELECT slot_name, slot_type, active, restart_lsn,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots;

-- WARNING: Inactive slot WAL জমাতে থাকে → disk full!
-- Monitor করো এবং drop করো inactive slot:
SELECT pg_drop_replication_slot('standby_slot');
```

---

## 10.5 Synchronous Replication

```ini
# Primary এ postgresql.conf
synchronous_standby_names = 'standby1'          # One standby
synchronous_standby_names = 'ANY 1 (s1, s2)'    # Any 1 of s1, s2
synchronous_standby_names = 'FIRST 1 (s1, s2)'  # First of ordered list
synchronous_commit        = remote_apply         # Strictest
```

```sql
-- Standby এ application_name set করো (postgresql.auto.conf)
-- primary_conninfo = '... application_name=standby1'

-- Sync status দেখো
SELECT application_name, sync_state FROM pg_stat_replication;
-- sync_state: async/sync/quorum/potential
```

---

## 10.6 Logical Replication

```sql
-- Publisher (Primary) এ
CREATE PUBLICATION mypub FOR TABLE users, orders;
-- অথবা সব table:
CREATE PUBLICATION mypub FOR ALL TABLES;

-- Subscriber এ
CREATE SUBSCRIPTION mysub
    CONNECTION 'host=172.16.93.140 dbname=mydb user=replicator password=xxx'
    PUBLICATION mypub;

-- Status দেখো
SELECT * FROM pg_stat_replication;  -- Publisher এ
SELECT * FROM pg_stat_subscription; -- Subscriber এ

-- Use cases:
-- Version upgrade (10→16 migration)
-- Selective table replication
-- Cross-database replication
-- Bidirectional replication
```

---

## 10.7 Patroni + etcd + HAProxy — Production HA

Patroni হলো PostgreSQL High Availability solution যেটা automatic failover করে।

### Architecture

```
                    ┌─────────────────┐
                    │    HAProxy       │
                    │   (Port 5000)    │
                    │  Load Balancer   │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │  PostgreSQL  │ │  PostgreSQL  │ │  PostgreSQL  │
    │   Node 1     │ │   Node 2     │ │   Node 3     │
    │  (Primary)   │ │  (Standby)   │ │  (Standby)   │
    │  + Patroni   │ │  + Patroni   │ │  + Patroni   │
    └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
           │                │                │
           └────────────────┴────────────────┘
                            │
              ┌─────────────▼─────────────┐
              │      etcd Cluster          │
              │  (Distributed Config Store)│
              │  Node 1 · Node 2 · Node 3  │
              └───────────────────────────┘
```

**Component Roles:**
```
Patroni:    PostgreSQL এর HA agent — election, failover, config
etcd:       Distributed key-value store — leader election এর consensus
HAProxy:    TCP load balancer — write → Primary, read → Standbys
```

### etcd Install এবং Configure

```bash
# সব তিনটা node এ
sudo dnf install -y etcd

# /etc/etcd/etcd.conf.yml (Node 1: 172.16.93.140)
```

```yaml
name: etcd1
data-dir: /var/lib/etcd
listen-client-urls: http://172.16.93.140:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.16.93.140:2379
listen-peer-urls: http://172.16.93.140:2380
initial-advertise-peer-urls: http://172.16.93.140:2380
initial-cluster: etcd1=http://172.16.93.140:2380,etcd2=http://172.16.93.141:2380,etcd3=http://172.16.93.142:2380
initial-cluster-token: pg-etcd-cluster
initial-cluster-state: new
```

```bash
sudo systemctl start etcd
sudo systemctl enable etcd

# etcd health check
etcdctl --endpoints=http://172.16.93.140:2379 endpoint health
etcdctl --endpoints=http://172.16.93.140:2379 member list
```

### Patroni Install এবং Configure

```bash
# সব PostgreSQL node এ
sudo pip3 install patroni[etcd] psycopg2-binary

# /etc/patroni/patroni.yml (Node 1)
```

```yaml
scope: postgres-cluster
namespace: /service/
name: pg-node1

restapi:
  listen: 172.16.93.140:8008
  connect_address: 172.16.93.140:8008

etcd:
  hosts: 172.16.93.140:2379,172.16.93.141:2379,172.16.93.142:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576   # 1MB
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        max_connections: 200
        shared_buffers: 4GB
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        wal_keep_size: 1GB

  initdb:
    - encoding: UTF8
    - data-checksums

  pg_hba:
    - host replication replicator 0.0.0.0/0 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

  users:
    admin:
      password: Admin@2024!
      options: [createdb, createrole]

postgresql:
  listen: 172.16.93.140:5432
  connect_address: 172.16.93.140:5432
  data_dir: /var/lib/pgsql/16/data
  bin_dir: /usr/pgsql-16/bin
  authentication:
    replication:
      username: replicator
      password: ReplicaPass@2024!
    superuser:
      username: postgres
      password: PostgresPass@2024!

tags:
  nofailover: false
  noloadbalance: false
  clonedfrom: false
  nosync: false
```

```bash
# Patroni service
sudo cat > /etc/systemd/system/patroni.service << 'EOF'
[Unit]
Description=Patroni PostgreSQL HA
After=syslog.target network.target

[Service]
Type=simple
User=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start patroni
sudo systemctl enable patroni

# Cluster status দেখো
patronictl -c /etc/patroni/patroni.yml list
# Member │ Host            │ Role    │ State   │ TL │ Lag in MB
# ───────┼─────────────────┼─────────┼─────────┼────┼──────────
# node1  │ 172.16.93.140:5432 │ Leader │ running │  1 │           |
# node2  │ 172.16.93.141:5432 │ Replica│ running │  1 │         0 |
# node3  │ 172.16.93.142:5432 │ Replica│ running │  1 │         0 |
```

### HAProxy Configure করো

```bash
sudo dnf install -y haproxy

sudo nano /etc/haproxy/haproxy.cfg
```

```ini
global
    maxconn 100
    log /dev/log local0

defaults
    log global
    mode tcp
    retries 2
    timeout client  30m
    timeout connect 4s
    timeout server  30m
    option clitcpka

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

# Write traffic → Leader only (Port 5000)
frontend postgresql_write
    bind *:5000
    default_backend postgresql_primary

backend postgresql_primary
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg1 172.16.93.140:5432 maxconn 100 check port 8008
    server pg2 172.16.93.141:5432 maxconn 100 check port 8008
    server pg3 172.16.93.142:5432 maxconn 100 check port 8008
    # Patroni REST API (port 8008) দিয়ে Primary কে identify করে
    # Primary → HTTP 200 return করে
    # Standby → HTTP 503 return করে

# Read traffic → All nodes (Port 5001)
frontend postgresql_read
    bind *:5001
    default_backend postgresql_replicas

backend postgresql_replicas
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg1 172.16.93.140:5432 maxconn 100 check port 8008
    server pg2 172.16.93.141:5432 maxconn 100 check port 8008
    server pg3 172.16.93.142:5432 maxconn 100 check port 8008
```

```bash
sudo systemctl start haproxy
sudo systemctl enable haproxy

# Application connect করে:
# Write: haproxy_ip:5000 → automatically Primary
# Read:  haproxy_ip:5001 → any node (load balanced)
```

### Patroni Operations

```bash
# Cluster status
patronictl -c /etc/patroni/patroni.yml list

# Manual failover (planned)
patronictl -c /etc/patroni/patroni.yml failover postgres-cluster

# Switchover (planned, choose target)
patronictl -c /etc/patroni/patroni.yml switchover postgres-cluster \
    --master pg-node1 --candidate pg-node2

# Reinitialize a node
patronictl -c /etc/patroni/patroni.yml reinit postgres-cluster pg-node2

# Restart PostgreSQL via Patroni
patronictl -c /etc/patroni/patroni.yml restart postgres-cluster

# Config change (cluster-wide)
patronictl -c /etc/patroni/patroni.yml edit-config

# Pause automatic failover (maintenance)
patronictl -c /etc/patroni/patroni.yml pause postgres-cluster
patronictl -c /etc/patroni/patroni.yml resume postgres-cluster

# History
patronictl -c /etc/patroni/patroni.yml history postgres-cluster
```

---

## 10.8 pgBouncer — Connection Pooler

PostgreSQL তে প্রতিটা connection এ নতুন process fork হয় (~5-10MB)। 1000 connection = 5-10GB RAM। pgBouncer connection pool করে।

```bash
sudo dnf install -y pgbouncer

sudo nano /etc/pgbouncer/pgbouncer.ini
```

```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr        = *
listen_port        = 6432
auth_type          = scram-sha-256
auth_file          = /etc/pgbouncer/userlist.txt

# Pool modes:
# session    → One server connection per client session (MySQL Thread Cache মতো)
# transaction → Server connection শুধু transaction সময় (ভালো concurrency)
# statement  → Per statement (autocommit only)
pool_mode          = transaction

# Pool sizes
default_pool_size  = 20        # Per database per user
max_client_conn    = 1000      # pgBouncer তে max clients
reserve_pool_size  = 5
server_idle_timeout = 600

# Logging
logfile            = /var/log/pgbouncer/pgbouncer.log
pidfile            = /var/run/pgbouncer/pgbouncer.pid

# Admin
admin_users        = pgbouncer_admin
stats_users        = pgbouncer_stats
```

```bash
# userlist.txt — pgBouncer এর user/password
sudo -u postgres psql -c "SELECT concat('\"', usename, '\" \"', passwd, '\"') FROM pg_shadow WHERE usename = 'appuser';"
# output:
# "appuser" "SCRAM-SHA-256$..."
echo '"appuser" "SCRAM-SHA-256$..."' | sudo tee -a /etc/pgbouncer/userlist.txt
echo '"pgbouncer_admin" "admin_password"' | sudo tee -a /etc/pgbouncer/userlist.txt

sudo systemctl start pgbouncer
sudo systemctl enable pgbouncer

# Application connects to port 6432 (pgBouncer) instead of 5432

# Admin console
psql -U pgbouncer_admin -p 6432 pgbouncer
# Stats দেখো
SHOW STATS;
SHOW POOLS;
SHOW CLIENTS;
SHOW SERVERS;
SHOW DATABASES;

# Reload config
RELOAD;
```

**pgBouncer Pool Modes:**
```
Session Mode:
  Client → pgBouncer → PostgreSQL Backend (hold until disconnect)
  MySQL Thread Cache এর মতো
  Transaction aware, SET commands safe
  Concurrency improvement কম

Transaction Mode:
  Client → pgBouncer → PostgreSQL Backend (hold only during transaction)
  Best concurrency improvement
  1000 clients, 20 actual PostgreSQL connections
  SET, LISTEN, NOTIFY কাজ করে না session level

Statement Mode:
  প্রতিটা statement এর পর connection return
  autocommit only
  Very rare use case
```

---

## 10.9 pgPool-II

pgBouncer এর চেয়ে more features — connection pooling + load balancing + query routing।

```bash
sudo dnf install -y pgpool-II-pg16

sudo nano /etc/pgpool-II/pgpool.conf
```

```ini
listen_addresses = '*'
port = 9999
backend_hostname0 = '172.16.93.140'   # Primary
backend_port0     = 5432
backend_weight0   = 1
backend_flag0     = 'ALWAYS_PRIMARY'

backend_hostname1 = '172.16.93.141'   # Standby
backend_port1     = 5432
backend_weight1   = 1
backend_flag1     = 'DISALLOW_TO_FAILOVER'

# Load balancing
load_balance_mode = on

# Connection pool
num_init_children = 32
max_pool          = 4

# Read/Write split
primary_routing_query_pattern_list = 'INSERT,UPDATE,DELETE,SELECT FOR UPDATE'
```

**pgBouncer vs pgPool-II:**
```
pgBouncer:
  + Lightweight, simple
  + Transaction mode pooling (best)
  + 99% production choice for pooling

pgPool-II:
  + Read/Write splitting built-in
  + Watchdog (HA for pgPool itself)
  + Parallel query
  - Complex configuration
  - Heavier than pgBouncer
  - Most teams prefer pgBouncer + HAProxy separately
```

---

## 10.10 Failover

### Manual Failover

```bash
# Primary down হলে Standby promote করো

# Option 1: pg_ctl
sudo -u postgres /usr/pgsql-16/bin/pg_ctl promote -D /var/lib/pgsql/16/data

# Option 2: trigger file
sudo -u postgres touch /tmp/promote_trigger
# postgresql.conf এ: promote_trigger_file = '/tmp/promote_trigger'

# Option 3: SQL function
sudo -u postgres psql -c "SELECT pg_promote();"

# Verify: Standby → Primary
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# f = Primary
```

### Automatic Failover (Patroni)

```bash
# Primary down হলে Patroni automatically:
# 1. etcd থেকে leader lock release detect করে
# 2. Candidate election (most up-to-date standby)
# 3. Winner promote করে PostgreSQL কে
# 4. Other standbys নতুন Primary এর সাথে replication শুরু করে
# 5. HAProxy health check (Patroni REST API) → traffic switch

# এই পুরো process ~30-60 seconds এ হয়

# Failover history দেখো
patronictl -c /etc/patroni/patroni.yml history postgres-cluster
```

---

## 10.11 Replication Topology Comparison

| Topology | Setup | Failover | Read Scale | Use Case |
|---|---|---|---|---|
| Primary + Standby | Easy | Manual | Yes | Small app |
| Primary + Multi-Standby | Medium | Manual | Yes | Read heavy |
| Synchronous Replication | Medium | Manual, 0 data loss | Yes | Financial |
| Patroni + etcd + HAProxy | Hard | Auto (30-60s) | Yes | Production HA |
| Logical Replication | Medium | N/A | Selective | Migration, filtering |
| pgPool-II | Medium | Built-in | Yes + RW split | All-in-one |

---

## 10.12 Troubleshooting Replication

### Standby Not Connecting

```bash
# pg_hba.conf check
sudo cat /var/lib/pgsql/16/data/pg_hba.conf | grep replication

# Primary connection test from Standby
psql -h 172.16.93.140 -U replicator -d replication -c "SELECT 1;"

# Error দেখো
sudo tail -50 /var/lib/pgsql/16/data/log/postgresql-*.log
```

| Error | Cause | Solution |
|---|---|---|
| `no pg_hba.conf entry for replication` | pg_hba.conf এ replication line নেই | pg_hba.conf এ add করো, reload |
| `authentication failed` | Wrong password | Replication user password reset |
| `connection refused` | Firewall বা wrong port | Firewall check, pg_hba.conf |
| `requested WAL segment has been removed` | WAL deleted before Replica received | wal_keep_size বাড়াও বা Replication Slot ব্যবহার করো |

### pg_rewind — Failed Primary Rejoin

```bash
# Primary ছিল, crash হয়েছে, নতুন Primary হয়েছে
# পুরনো Primary কে standby হিসেবে rejoin করাতে চাই

# নতুন Primary তে pg_rewind permission
ALTER USER postgres WITH SUPERUSER;  # অথবা pg_rewind role

# পুরনো Primary এ (stop করো)
sudo systemctl stop postgresql-16

# pg_rewind চালাও
sudo -u postgres /usr/pgsql-16/bin/pg_rewind \
    --target-pgdata=/var/lib/pgsql/16/data \
    --source-server='host=172.16.93.141 user=postgres'

# standby.signal তৈরি করো
sudo -u postgres touch /var/lib/pgsql/16/data/standby.signal
# primary_conninfo postgresql.auto.conf এ আপডেট করো
sudo systemctl start postgresql-16
```

---

## 10.13 Quick Reference Commands

**Installation:**

```bash
# Repository
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib

# Initialize
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl start postgresql-16 && sudo systemctl enable postgresql-16
```

**psql Essential Commands:**

| Command | কাজ |
|---|---|
| `\l` | List databases |
| `\c dbname` | Connect to database |
| `\dt` | List tables |
| `\d tablename` | Table structure |
| `\di` | List indexes |
| `\du` | List roles/users |
| `\dp tablename` | Table privileges |
| `\timing` | Query timing on/off |
| `\x` | Expanded display on/off |
| `\copy` | Client-side copy |
| `\i file.sql` | Execute SQL file |
| `\! command` | Shell command |
| `\q` | Quit |

**Primary Server:**

| Query | কাজ |
|---|---|
| `SELECT pg_current_wal_lsn();` | Current WAL position |
| `SELECT * FROM pg_stat_replication;` | Replica status |
| `SELECT * FROM pg_replication_slots;` | Replication slots |
| `SELECT pg_switch_wal();` | Force WAL switch |
| `CHECKPOINT;` | Force checkpoint |

**Standby Server:**

| Query | কাজ |
|---|---|
| `SELECT pg_is_in_recovery();` | Is standby? (t/f) |
| `SELECT pg_last_wal_receive_lsn();` | Last received WAL |
| `SELECT pg_last_wal_replay_lsn();` | Last applied WAL |
| `SELECT now() - pg_last_xact_replay_timestamp();` | Replication lag |
| `SELECT pg_promote();` | Promote to primary |
| `SELECT pg_wal_replay_resume();` | Resume recovery |

**Both:**

| Query | কাজ |
|---|---|
| `SHOW data_directory;` | Data directory |
| `SHOW config_file;` | Config file path |
| `SELECT pg_reload_conf();` | Reload config |
| `EXPLAIN (ANALYZE, BUFFERS) query;` | Execution plan |
| `VACUUM VERBOSE ANALYZE table;` | Vacuum + stats |
| `SELECT pg_terminate_backend(pid);` | Kill connection |
| `SELECT pg_cancel_backend(pid);` | Cancel query |

**Patroni:**

| Command | কাজ |
|---|---|
| `patronictl list` | Cluster status |
| `patronictl failover cluster` | Failover |
| `patronictl switchover cluster` | Planned switchover |
| `patronictl pause cluster` | Pause auto-failover |
| `patronictl resume cluster` | Resume |
| `patronictl history cluster` | Failover history |

---

*PostgreSQL 16 | Rocky Linux 9 | Complete Study Reference*
