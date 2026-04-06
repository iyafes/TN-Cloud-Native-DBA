# PostgreSQL 17 — Production Setup & Migration Guide
**Rocky Linux 9 | Binary Install | Standalone DB Server**
**Migration: PG 18 (Docker, Combined Server) → PG 17 (Binary, New DB Server)**

---

> **এই document টা শুধু তোমার জন্য।**
> প্রতিটা step এ কী হচ্ছে, কেন হচ্ছে, কোথায় সতর্ক থাকতে হবে — সব বিস্তারিত আছে।
> যেখানে তোমার specific value দরকার সেখানে `<এইভাবে>` placeholder আছে — run করার আগে অবশ্যই replace করতে হবে।

---

## ⚠️ Migration সম্পর্কে একটা Critical কথা আগে

তোমার combined server এ চলছে **PostgreSQL 18**, নতুন server এ install হবে **PostgreSQL 17**। এটা একটা **major version downgrade**।

PostgreSQL এ `pg_upgrade` tool দিয়ে upgrade করা যায়, কিন্তু **downgrade করা যায় না**। তাই এখানে একমাত্র safe method হলো:

```
pg_dump (PG18 থেকে SQL বের করো)
       ↓
transfer to new server
       ↓
pg_restore (PG17 তে restore করো)
```

এই method এ:
- ✅ কোনো data loss নেই
- ✅ Downtime controlled — তুমি ঠিক করো কখন নামাবে
- ✅ Rollback সহজ — combined server এর DB কখনো বন্ধ করা হয় না migration এর সময়
- ⚠️ PG 18-specific কোনো feature বা syntax use হলে restore এ error আসতে পারে — Senior DBA কে আগে জিজ্ঞেস করো

---

## পুরো কাজের ধারা (Overview)

```
Part A — New DB Server Setup
─────────────────────────────
Repository add
       ↓
PostgreSQL 17 install
       ↓
Database cluster initialize (initdb)
       ↓
postgresql.conf configure
       ↓
pg_hba.conf configure
       ↓
Service start
       ↓
postgres user password set
       ↓
Firewall port open
       ↓
pgBackRest install + configure
       ↓
New server ready ✓

Part B — Migration (Downtime Window)
──────────────────────────────────────
[Downtime শুরু]
Combined server এ app containers বন্ধ
       ↓
Combined server এ final pg_dump
       ↓
Dump file new server এ transfer
       ↓
New server এ restore
       ↓
Data verify
       ↓
Backend team connection string update করে
       ↓
Application উঠিয়ে test
       ↓
[Downtime শেষ]
```

---

# Part A — PostgreSQL 17 Setup

---

## ধাপ ১ — Repository Add করা

### কী হচ্ছে এখানে?

Rocky Linux 9 এর default repo তে PostgreSQL এর পুরনো version থাকে। PostgreSQL Global Development Group (PGDG) এর official repository আলাদা করে add করতে হয়।

### Command

```bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
```

এই command টা একটা `.rpm` package সরাসরি URL থেকে install করে — এই package ই PGDG এর সব version এর repository configure করে দেয়।

এরপর Rocky Linux এর built-in PostgreSQL module disable করো — না করলে conflict হবে:

```bash
dnf -qy module disable postgresql
```

**কেন disable করতে হয়:** Rocky এর AppStream এ নিজস্ব একটা postgresql module আছে। দুটো একসাথে থাকলে `dnf` confused হয় কোনটা থেকে install করবে। PGDG version ব্যবহার করতে হলে AppStream এরটা disable করতেই হবে।

### Verify করো

```bash
dnf repolist | grep pgdg
```

`pgdg17` সহ কয়েকটা repo দেখাবে — মানে সফল।

---

## ধাপ ২ — PostgreSQL 17 Install করা

### কী install হয়

| Package | কী করে |
|---------|---------|
| `postgresql17-server` | `postgres` daemon, data directory management |
| `postgresql17` | `psql` client, `pg_dump`, `pg_restore`, সব utilities |

### Command

```bash
dnf install -y postgresql17-server postgresql17
```

### Verify করো

```bash
psql --version
# অথবা full path দিয়ে
/usr/pgsql-17/bin/psql --version
```

`PostgreSQL 17.x` দেখালে সফল।

---

## ধাপ ৩ — Database Cluster Initialize করা (initdb)

### কী হচ্ছে এখানে?

এটা PostgreSQL এর একটা unique step — MongoDB বা MySQL এ এরকম নেই। Install হওয়া মানেই PostgreSQL ready না। একটা "database cluster" তৈরি করতে হয় — এই process কে বলা হয় `initdb`।

`initdb` চলার সময় যা হয়:
- `/var/lib/pgsql/17/data/` directory তৈরি হয়
- System tables তৈরি হয় (pg_database, pg_user ইত্যাদি)
- Config files তৈরি হয় (`postgresql.conf`, `pg_hba.conf`)
- WAL (Write-Ahead Log) directory তৈরি হয়

**এটা একবারই করতে হয় — fresh install এর পর।**

### Command

```bash
/usr/pgsql-17/bin/postgresql-17-setup initdb
```

### Output দেখবে

```
Initializing database ... OK
```

### এরপর যা তৈরি হয়

```
/var/lib/pgsql/17/data/
├── postgresql.conf      ← main configuration
├── pg_hba.conf          ← authentication rules
├── pg_ident.conf        ← ident mapping (সাধারণত দরকার হয় না)
├── base/                ← actual database files
├── pg_wal/              ← write-ahead log files
└── log/                 ← log files (logging_collector on হলে)
```

---

## ধাপ ৪ — postgresql.conf Configure করা

### কী হচ্ছে এখানে?

এটা PostgreSQL এর main config file। Server কীভাবে চলবে সব এখানে define করা থাকে — কোন IP শুনবে, কতটুকু memory নেবে, log কীভাবে লিখবে, backup এর জন্য WAL কতটুকু রাখবে।

### File খোলো

```bash
vi /var/lib/pgsql/17/data/postgresql.conf
```

### গুরুত্বপূর্ণ settings এবং তাদের ব্যাখ্যা

নিচের প্রতিটা setting খুঁজে বের করো (file এ আগে থেকেই আছে, comment করা — `#` সরিয়ে enable করতে হবে) এবং value set করো:

---

**Connection settings:**

```ini
listen_addresses = '*'
```

কোন IP তে PostgreSQL "কান পাতবে"। `'*'` মানে সব interface। MongoDB এর `bindIp: 0.0.0.0` এর equivalent।

> **Production এ আরো specific করা যায়:** `'127.0.0.1,<db-server-ip>'` — শুধু localhost এবং DB server এর নিজের IP। তোমার Senior DBA এর preference জিজ্ঞেস করো।

```ini
port = 5432
max_connections = 200
```

`max_connections` — একসাথে সর্বোচ্চ কতটা client connect করতে পারবে। Backend এর connection pool size এর চেয়ে বেশি রাখো।

---

**Memory settings:**

```ini
shared_buffers = 256MB
effective_cache_size = 1GB
work_mem = 4MB
maintenance_work_mem = 64MB
```

> **⚠️ এই values server এর actual RAM এর উপর নির্ভর করে।** নিচে formula:
>
> | Parameter | Rule of thumb |
> |-----------|---------------|
> | `shared_buffers` | Total RAM এর 25% |
> | `effective_cache_size` | Total RAM এর 75% |
> | `work_mem` | (RAM - shared_buffers) / max_connections |
> | `maintenance_work_mem` | RAM এর 5%, max 1GB |
>
> Server এর RAM দেখো: `free -h`
>
> Senior DBA এর কাছ থেকে company standard values নাও।

`shared_buffers` — PostgreSQL এর নিজস্ব cache। Data এখান থেকে পড়লে disk এ যেতে হয় না — fast।

`effective_cache_size` — OS এর disk cache কতটুকু আছে সেটার estimate। PostgreSQL নিজে এটা allocate করে না, শুধু query planner কে hint দেয়।

`work_mem` — প্রতিটা sort/hash operation এর জন্য memory। বেশি দিলে query fast, কিন্তু অনেক connection থাকলে total memory বেশি লাগে।

---

**WAL settings — backup এর জন্য জরুরি:**

```ini
wal_level = replica
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
max_wal_senders = 3
wal_keep_size = 1GB
```

`wal_level = replica` — WAL এ কতটুকু information লিখবে। `replica` level এ replication এবং PITR (Point-in-Time Recovery) এর জন্য যথেষ্ট information থাকে। এটা না হলে pgBackRest কাজ করবে না।

`archive_mode = on` — WAL files গুলো archive করবে।

`archive_command` — প্রতিটা WAL file complete হলে এই command run হবে। `%p` এর জায়গায় WAL file এর path বসে। pgBackRest configure হওয়ার আগে এই command fail করবে — তাই pgBackRest setup (ধাপ ৮) করার পরে service restart করতে হবে।

`max_wal_senders` — কতটা replication connection allow করবে।

`wal_keep_size` — Minimum কতটুকু WAL disk এ রাখবে।

---

**Logging settings:**

```ini
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d
log_rotation_size = 0
log_min_duration_statement = 1000
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Asia/Dhaka'
```

`logging_collector = on` — Background process এ log লেখে। এটা off থাকলে log file এ যায় না।

`log_min_duration_statement = 1000` — 1000ms (1 second) এর বেশি সময় নেওয়া query log এ লেখে। Slow query diagnose করতে কাজে লাগে।

`log_line_prefix` — প্রতিটা log line এর আগে কী লিখবে। `%m` = timestamp, `%p` = process ID, `%u` = username, `%d` = database name।

`log_timezone` — Log এর timestamp কোন timezone এ। তোমার server যদি UTC তে থাকে তাহলে `'UTC'` দাও — Senior DBA কে জিজ্ঞেস করো।

---

**Timezone:**

```ini
timezone = 'Asia/Dhaka'
```

> **⚠️ Server কোন timezone এ configured সেটার সাথে match করা ভালো।** Server এর timezone দেখো: `timedatectl`। Senior DBA এর preference জিজ্ঞেস করো।

---

## ধাপ ৫ — pg_hba.conf Configure করা

### কী হচ্ছে এখানে?

`pg_hba.conf` (Host-Based Authentication) PostgreSQL এর authentication rule file। এটা MongoDB এর `security: authorization: enabled` এর চেয়ে অনেক বেশি granular।

এখানে line by line define করা হয়:
- কোন **type** এর connection (local/network)
- কোন **database** এ
- কোন **user**
- কোন **IP** থেকে
- কোন **method** এ authenticate করবে

### File খোলো

```bash
vi /var/lib/pgsql/17/data/pg_hba.conf
```

### File এর পুরো content replace করো

```conf
# TYPE    DATABASE    USER        ADDRESS                  METHOD

# Local OS — postgres OS user peer auth দিয়ে password ছাড়া ঢুকতে পারবে
# এটা DBA এর জন্য server এ বসে কাজ করার সময় লাগে
local     all         postgres                             peer

# Localhost থেকে সব user password দিয়ে
host      all         all         127.0.0.1/32             md5

# Backend/application server থেকে
host      all         all         <backend-server-ip>/32   md5

# DBA এর machine থেকে pgAdmin বা psql দিয়ে connect
host      all         postgres    <dba-machine-ip>/32      md5
```

### Authentication method এর মানে

| Method | কী করে | কখন ব্যবহার |
|--------|---------|-------------|
| `peer` | OS এর current user এর নামের সাথে PG user নাম match করলে password ছাড়া allow | শুধু local, DBA এর জন্য |
| `md5` | Password hash দিয়ে verify করে | Remote connection সব জায়গায় |
| `scram-sha-256` | md5 এর চেয়ে বেশি secure — PG 14+ এ recommended | Senior DBA এর preference অনুযায়ী |

> **⚠️ Replace করতে হবে:**
> - `<backend-server-ip>` — Backend application server এর IP। DevOps/backend team এর কাছ থেকে নাও।
> - `<dba-machine-ip>` — তোমার নিজের machine এর IP। `curl ifconfig.me` দিয়ে পাবে।

### Connection type এর মানে

`local` — Unix socket connection। Same machine থেকে, same OS এর ভেতর দিয়ে। Network involve করে না।

`host` — TCP/IP connection। এটা localhost এবং remote দুটোই হতে পারে — address দিয়ে distinguish করা হয়।

---

## ধাপ ৬ — Service Start করা

```bash
# Service চালু করো
systemctl start postgresql-17

# Server reboot হলে automatically উঠবে
systemctl enable postgresql-17

# Status দেখো
systemctl status postgresql-17
```

`active (running)` দেখলে সফল।

যদি failed হয়:

```bash
# কারণ এখানে থাকবে
journalctl -u postgresql-17 -n 50
# অথবা
tail -50 /var/lib/pgsql/17/data/log/postgresql-$(date +%Y-%m-%d).log
```

সাধারণ কারণ: `postgresql.conf` এ কোনো value ভুল, অথবা `listen_addresses` এ এমন IP দেওয়া হয়েছে যেটা server এ নেই।

---

## ধাপ ৭ — postgres User এর Password Set করা এবং Application Setup

### postgres user এর password set করো

```bash
# postgres OS user হিসেবে psql এ ঢোকো
sudo -u postgres psql
```

Prompt `postgres=#` দেখাবে।

```sql
-- postgres superuser এর password set করো
ALTER USER postgres WITH PASSWORD '<strong_password>';

-- বের হও
\q
```

> **⚠️ এই password টা সরাসরি team password manager এ রাখো।**

### Application database এবং user তৈরি করো

> **⚠️ Backend team এর কাছ থেকে exact database name এবং user name জেনে নাও আগে।** নিচে pattern দেওয়া হলো।

```sql
sudo -u postgres psql
```

```sql
-- Application এর জন্য database তৈরি করো
CREATE DATABASE <app_database_name>;

-- Application এর জন্য dedicated user তৈরি করো
CREATE USER <app_user> WITH PASSWORD '<app_password>';

-- User কে database এ permission দাও
GRANT ALL PRIVILEGES ON DATABASE <app_database_name> TO <app_user>;

-- PG 15+ এ schema level permission ও দিতে হয়
\c <app_database_name>
GRANT ALL ON SCHEMA public TO <app_user>;

\q
```

**কেন আলাদা user:** `postgres` superuser দিয়ে application connect করালে সে সব database এ সব কাজ করতে পারবে — এটা security risk। `<app_user>` শুধু তার নিজের database এ কাজ করতে পারবে।

---

## ধাপ ৮ — Firewall Port Open করা

```bash
# Backend server থেকে 5432 port allow করো
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<backend-server-ip>/32" port protocol="tcp" port="5432" accept'

# তোমার DBA machine থেকে pgAdmin/psql এর জন্য
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<dba-machine-ip>/32" port protocol="tcp" port="5432" accept'

# Apply করো
firewall-cmd --reload

# Verify করো
firewall-cmd --list-all
```

---

## ধাপ ৯ — pgBackRest Install এবং Configure করা

### কী হচ্ছে এখানে?

pgBackRest PostgreSQL এর জন্য dedicated backup tool। এটা দুটো জিনিস করে:

**১. Continuous WAL Archiving:** PostgreSQL প্রতিটা transaction লেখার সময় WAL (Write-Ahead Log) file এ আগে লেখে। pgBackRest এই WAL files গুলো automatically একটা safe location এ copy করে রাখে। এতে করে যেকোনো নির্দিষ্ট সময়ে (minute পর্যন্ত) database restore করা যায় — এটাকে PITR বলে।

**২. Scheduled Base Backup:** সম্পূর্ণ database এর snapshot নেয়। WAL files এর সাথে মিলিয়ে যেকোনো point এ restore করা যায়।

### Install করো

```bash
dnf install -y pgbackrest
```

### Configuration তৈরি করো

```bash
vi /etc/pgbackrest/pgbackrest.conf
```

```ini
[global]
# Backup কোথায় রাখবে
repo1-path=/var/lib/pgbackrest

# কতটা full backup রাখবে (পুরনো মুছে দেবে)
repo1-retention-full=2

# কতটা differential backup রাখবে
repo1-retention-diff=7

# Log settings
log-level-console=info
log-level-file=detail
log-path=/var/log/pgbackrest

# Performance
start-fast=y
delta=y

[main]
# PostgreSQL data directory
pg1-path=/var/lib/pgsql/17/data
pg1-port=5432
pg1-user=postgres
```

**`repo1-retention-full=2`** — ২টা full backup রাখবে। তৃতীয়টা নেওয়ার সময় সবচেয়ে পুরনোটা automatically delete হবে।

**`start-fast=y`** — Backup শুরুর জন্য একটা checkpoint force করে — এতে backup আরো quickly শুরু হয়।

**`delta=y`** — Restore করার সময় শুধু যা পরিবর্তন হয়েছে সেটা overwrite করে — পুরো directory মুছে নতুন করে লিখতে হয় না।

### postgresql.conf এর archive_command verify করো

ধাপ ৪ এ এটা দিয়েছিলে:

```ini
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

এটা ঠিকঠাক আছে কিনা confirm করো।

### Stanza তৈরি করো এবং initialize করো

pgBackRest এ "stanza" মানে একটা PostgreSQL cluster এর configuration block — `[main]` section টাই stanza।

```bash
# postgres user হিসেবে stanza create করো
sudo -u postgres pgbackrest --stanza=main stanza-create

# postgresql.conf এর archive settings check করো
sudo -u postgres pgbackrest --stanza=main check
```

`check` command কোনো error না দিলে WAL archiving ঠিকমতো কাজ করছে।

> **⚠️ `check` command এ error আসলে:**
> - `postgresql.conf` এ `wal_level`, `archive_mode`, `archive_command` ঠিক আছে কিনা দেখো
> - Service restart হয়েছে কিনা দেখো
> - `/var/lib/pgbackrest` directory এর ownership ঠিক আছে কিনা: `ls -la /var/lib/pgbackrest`

### প্রথম full backup নাও

```bash
sudo -u postgres pgbackrest --stanza=main --type=full backup
```

কতক্ষণ লাগবে database size এর উপর নির্ভর করে। Successful হলে:

```bash
sudo -u postgres pgbackrest --stanza=main info
```

Backup এর details দেখাবে।

### Cron দিয়ে schedule করো

```bash
sudo -u postgres crontab -e
```

```cron
# প্রতি রবিবার রাত ২টায় full backup
0 2 * * 0 pgbackrest --stanza=main --type=full backup

# বাকি দিন রাত ২টায় differential backup
0 2 * * 1-6 pgbackrest --stanza=main --type=diff backup
```

**Full backup vs Differential backup:**

| Type | কী করে | Size | Speed |
|------|---------|------|-------|
| Full | সম্পূর্ণ database এর copy | বড় | ধীর |
| Differential | Last full backup এর পর যা পরিবর্তন হয়েছে | ছোট | দ্রুত |

WAL archiving continuous চলে — তাই full/diff backup এর মাঝের যেকোনো সময়েও restore করা যাবে।

---

# Part B — Migration

---

## Migration এর আগে যা করতে হবে

### ১. Source (Combined Server) এ data size দেখো

```bash
# Combined server এ
docker exec <pg-container-name> psql -U postgres -c "\l+"
```

এখানে প্রতিটা database এর size দেখাবে। Total size note করো — এতে বুঝতে পারবে dump এবং transfer কতক্ষণ লাগবে।

> **⚠️ `<pg-container-name>` জানতে:** `docker ps` — PostgreSQL container এর নাম দেখাবে।

### ২. New server এ disk space আছে কিনা দেখো

```bash
# New DB server এ
df -h /var/lib/pgsql
df -h /tmp
```

Dump file রাখার জন্য `/tmp` বা অন্য কোনো location এ যথেষ্ট space লাগবে। Source database এর size এর কমপক্ষে ২ গুণ free space রাখো।

### ৩. Network connectivity test করো

```bash
# New DB server থেকে combined server এ ping করো
ping <combined-server-ip>

# Combined server থেকে new DB server এ SSH করা যায় কিনা test করো
ssh user@<new-db-server-ip> "echo 'connection ok'"
```

### ৪. Application team কে জানাও

Migration শুরু করার আগে application team এবং stakeholders কে exact downtime window জানাও।

---

## Migration — Step by Step

### ওভারভিউ: কোন server এ কী করছো

```
Combined Server                    New DB Server
(চলছে)                             (ready, কিন্তু empty)
    │                                    │
    │── pre-check ──────────────────────▶│
    │                                    │
[Downtime শুরু]                          │
    │                                    │
    │── app containers বন্ধ             │
    │                                    │
    │── final pg_dump ──────────────────▶│ (transfer)
    │                                    │
    │   DB container চলছেই              │── pg_restore
    │   (rollback এর জন্য)              │
    │                                    │── data verify
    │                                    │
    │◀── backend reconnects ────────────│
    │                                    │
[Downtime শেষ]                           │
```

---

### Migration ধাপ ১ — Pre-Migration Dry Run (Downtime এর আগে)

**এটা actual downtime এর আগেই করো — practice run।**

Combined server এ app বন্ধ না করে একটা test dump নাও। এই dump টা production এ use হবে না, কিন্তু dump কতক্ষণ লাগে, transfer কতক্ষণ লাগে — সেটা জানা যাবে। এই তথ্য দিয়ে downtime window কতক্ষণ লাগবে estimate করতে পারবে।

```bash
# Combined server এ
time docker exec <pg-container-name> pg_dumpall \
  -U postgres \
  --clean \
  --if-exists \
  > /tmp/pg_test_dump.sql

# File size দেখো
ls -lh /tmp/pg_test_dump.sql

# Transfer speed test করো (new server এ পাঠাও, সময় মাপো)
time scp /tmp/pg_test_dump.sql user@<new-db-server-ip>:/tmp/pg_test_dump.sql
```

`time` command মোট সময় দেখায়। Dump + transfer এর total time জানলে downtime window plan করা সহজ হয়।

**Test dump টা new server এ restore করে দেখো সব ঠিক আছে কিনা:**

```bash
# New DB server এ
sudo -u postgres psql -f /tmp/pg_test_dump.sql 2>&1 | grep -i error | head -20
```

কোনো ERROR দেখালে সেটা note করো এবং Senior DBA কে জানাও। PG 18 specific syntax থাকলে এখানেই ধরা পড়বে — actual migration এর আগেই।

---

### Migration ধাপ ২ — Downtime Window শুরু

**সময় এসেছে। Application team কে জানাও।**

Combined server এ application containers বন্ধ করো — database container বন্ধ করবে না:

```bash
# Combined server এ
# শুধু frontend এবং backend বন্ধ করো
docker stop <frontend-container-name> <backend-container-name>

# Confirm করো বন্ধ হয়েছে
docker ps
# শুধু DB containers দেখাবে
```

**কেন DB container বন্ধ করছি না:** এই moment থেকে নতুন কোনো write আসবে না (app বন্ধ)। DB বন্ধ না করা মানে rollback সবসময় possible — সমস্যা হলে app আবার পুরনো DB এর দিকে point করলেই হবে।

---

### Migration ধাপ ৩ — Final Dump নেওয়া

App বন্ধ হওয়ার পর কোনো নতুন data আসছে না — এই মুহূর্তে dump নাও। এটাই authoritative final backup।

```bash
# Combined server এ

# Option A — সব database একসাথে dump (recommended)
docker exec <pg-container-name> pg_dumpall \
  -U postgres \
  --clean \
  --if-exists \
  > /tmp/pg_final_$(date +%Y%m%d_%H%M%S).sql

echo "Dump completed: $(ls -lh /tmp/pg_final_*.sql)"
```

**`pg_dumpall` কী করে:** সব database, সব user (role), সব permission একটাই SQL file এ বের করে আনে। New server এ এই একটা file restore করলেই সব পাওয়া যাবে।

**`--clean`:** Restore করার সময় আগে existing objects drop করে। নতুন server এ এটা দরকার নেই আসলে, কিন্তু idempotent রাখার জন্য ভালো।

**`--if-exists`:** `--clean` এর সাথে ব্যবহার করলে drop করার আগে object exist করে কিনা check করে — error এড়ায়।

> **যদি শুধু specific database dump করতে চাও** (নির্দিষ্ট DB আলাদা রাখতে চাইলে):
> ```bash
> docker exec <pg-container-name> pg_dump \
>   -U postgres \
>   -Fc \
>   -v \
>   -d <database_name> \
>   > /tmp/<database_name>_$(date +%Y%m%d_%H%M%S).dump
> ```
> `-Fc` = custom format (compressed, faster restore করা যায় `pg_restore` দিয়ে)

---

### Migration ধাপ ৪ — Dump File Transfer করা

```bash
# Combined server থেকে new DB server এ
scp /tmp/pg_final_*.sql user@<new-db-server-ip>:/tmp/

# Transfer সফল হয়েছে কিনা verify করো
ssh user@<new-db-server-ip> "ls -lh /tmp/pg_final_*.sql"
```

File size দুই জায়গায় same হওয়া উচিত — আলাদা হলে transfer incomplete।

---

### Migration ধাপ ৫ — New Server এ Restore করা

```bash
# New DB server এ

# Restore করো
sudo -u postgres psql -f /tmp/pg_final_<timestamp>.sql 2>&1 | tee /tmp/restore_log.txt

# Restore শেষে error আছে কিনা দেখো
grep -i "^error" /tmp/restore_log.txt
```

`tee` command একসাথে screen এ দেখায় এবং file এ লেখে — পরে দেখার জন্য।

**কিছু `ERROR` দেখলে panic করবে না।** কিছু common benign errors:

| Error | কারণ | Problem? |
|-------|------|----------|
| `role "postgres" already exists` | Postgres superuser already আছে | ✅ Normal |
| `ERROR: database "postgres" already exists` | Default database already আছে | ✅ Normal |
| `ERROR: must be owner of ...` | Permission hierarchy restore | ⚠️ Senior DBA কে জানাও |
| `ERROR: syntax error` | PG18-specific syntax | ❌ Critical — Senior DBA |

---

### Migration ধাপ ৬ — Data Verify করা

Restore শেষে data ঠিকমতো এসেছে কিনা confirm করো।

**New server এ:**

```bash
sudo -u postgres psql
```

```sql
-- সব database দেখো
\l

-- প্রতিটা important database এ ঢুকে table গুলো দেখো
\c <database_name>
\dt

-- Critical tables এর row count দেখো
SELECT COUNT(*) FROM <important_table_name>;

-- বের হও
\q
```

**Combined server এ একই count করো এবং compare করো:**

```bash
docker exec <pg-container-name> psql -U postgres -d <database_name> \
  -c "SELECT COUNT(*) FROM <important_table_name>;"
```

দুই জায়গায় count same হলে migration successful।

> **⚠️ যদি count আলাদা হয়:** Rollback করো — app containers combined server এ আবার উঠিয়ে দাও এবং investigation করো।

---

### Migration ধাপ ৭ — Application Reconnect করা

Data verify হয়ে গেলে backend team কে জানাও — তারা application এর connection string নতুন DB server এর দিকে point করবে।

```
পুরনো: postgresql://postgres:<pass>@<combined-server-ip>:5432/<dbname>
নতুন:  postgresql://<app_user>:<pass>@<new-db-server-ip>:5432/<dbname>
```

Application উঠিয়ে smoke test করো:
- Login করা যাচ্ছে কিনা
- Data দেখা যাচ্ছে কিনা
- কোনো DB error আসছে কিনা

সব ঠিক থাকলে **downtime শেষ।**

---

### Rollback Plan

যদি কোনো ধাপে সমস্যা হয়:

```bash
# Combined server এ — app containers আবার উঠিয়ে দাও
docker start <frontend-container-name> <backend-container-name>
```

Combined server এর DB container কখনো বন্ধ করা হয়নি — তাই rollback মানে শুধু app কে পুরনো connection এ ফিরিয়ে দেওয়া। কোনো data loss নেই।

---

## দরকারি Commands — Quick Reference

### Connection

```bash
# Local — postgres OS user দিয়ে (password ছাড়া)
sudo -u postgres psql

# Local — password দিয়ে
psql -U postgres -h 127.0.0.1

# Remote — নির্দিষ্ট database এ
psql -h <db-server-ip> -U <user> -d <database_name>

# pgAdmin দিয়ে connect করতে: host, port 5432, username, password
```

### psql shell এর ভেতরে

```sql
-- সব database দেখো  (MongoDB এর show dbs)
\l

-- Database switch করো  (MongoDB এর use dbname)
\c <database_name>

-- Tables দেখো  (MongoDB এর show collections)
\dt

-- Current database  (MongoDB এর db)
SELECT current_database();

-- Current user
SELECT current_user;

-- সব user দেখো
\du

-- Active connections দেখো
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity
WHERE state != 'idle';

-- একটা connection kill করো
SELECT pg_terminate_backend(<pid>);

-- বের হও
\q
```

### Backup — pgBackRest

```bash
# Full backup
sudo -u postgres pgbackrest --stanza=main --type=full backup

# Differential backup
sudo -u postgres pgbackrest --stanza=main --type=diff backup

# Backup list দেখো
sudo -u postgres pgbackrest --stanza=main info

# Verify করো সব ঠিক আছে কিনা
sudo -u postgres pgbackrest --stanza=main check
```

### User এবং Permission Management

```sql
-- Password change করো
ALTER USER <username> WITH PASSWORD '<new_password>';

-- User তৈরি করো
CREATE USER <username> WITH PASSWORD '<password>';

-- Database তৈরি করো
CREATE DATABASE <dbname> OWNER <username>;

-- Permission দাও
GRANT ALL PRIVILEGES ON DATABASE <dbname> TO <username>;
GRANT ALL ON SCHEMA public TO <username>;

-- User delete করো
DROP USER <username>;

-- Database delete করো (সতর্কতার সাথে)
DROP DATABASE <dbname>;
```

### Log দেখা

```bash
# Live log
tail -f /var/lib/pgsql/17/data/log/postgresql-$(date +%Y-%m-%d).log

# Slow queries
grep "duration:" /var/lib/pgsql/17/data/log/postgresql-$(date +%Y-%m-%d).log | sort -t= -k2 -rn | head -20

# Error গুলো
grep "^.*ERROR" /var/lib/pgsql/17/data/log/postgresql-$(date +%Y-%m-%d).log

# pgBackRest log
tail -f /var/log/pgbackrest/main-backup.log
```

### Service Management

```bash
systemctl start postgresql-17
systemctl stop postgresql-17
systemctl restart postgresql-17
systemctl reload postgresql-17   # config reload, service restart ছাড়া
systemctl status postgresql-17
```

---

## Key File Locations

| কী | কোথায় |
|----|--------|
| Main config | `/var/lib/pgsql/17/data/postgresql.conf` |
| Auth config | `/var/lib/pgsql/17/data/pg_hba.conf` |
| Data directory | `/var/lib/pgsql/17/data/` |
| Log directory | `/var/lib/pgsql/17/data/log/` |
| WAL directory | `/var/lib/pgsql/17/data/pg_wal/` |
| pgBackRest config | `/etc/pgbackrest/pgbackrest.conf` |
| pgBackRest backup | `/var/lib/pgbackrest/` |
| pgBackRest log | `/var/log/pgbackrest/` |
| Binary location | `/usr/pgsql-17/bin/` |

---

## Uninstall (দরকার হলে)

> **⚠️ এটা run করলে সব data চিরতরে মুছে যাবে।**

```bash
systemctl stop postgresql-17
systemctl disable postgresql-17
dnf remove -y postgresql17-server postgresql17
rm -rf /var/lib/pgsql/17/data
rm -rf /var/log/postgresql
```

---

## জিনিসগুলো যা তোমাকে আগে থেকে জেনে নিতে হবে

- [ ] **DB server এর নিজের IP** — `ip addr show` দিয়ে পাবে
- [ ] **Backend server এর IP** — DevOps/backend team এর কাছ থেকে নাও
- [ ] **তোমার DBA machine এর IP** — `curl ifconfig.me`
- [ ] **Combined server এ PG container এর নাম** — `docker ps`
- [ ] **কোন কোন database migrate করতে হবে** — Backend team confirm করবে
- [ ] **Application user এর name, password** — Backend team এর কাছ থেকে নাও অথবা নতুন তৈরি করো
- [ ] **Server এর RAM** — `free -h` দিয়ে দেখো, memory config এর জন্য
- [ ] **Timezone** — `timedatectl` দিয়ে দেখো
- [ ] **Memory config values** — Senior DBA এর কাছ থেকে company standard নাও
- [ ] **Authentication method (md5 vs scram-sha-256)** — Senior DBA preference
- [ ] **Downtime window** — Application team এর সাথে fix করো
- [ ] **Rollback decision maker** — Migration এ problem হলে কে rollback decide করবে আগেই fix করো

---

*Document শেষ — PostgreSQL 17 | Rocky Linux 9 | Migration from PG 18*
