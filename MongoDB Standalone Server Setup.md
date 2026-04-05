# MongoDB 8.0 — Production Setup Guide
**Rocky Linux 9 | Binary Install | Standalone Server**

---

> **এই document টা শুধু তোমার জন্য লেখা।**
> প্রতিটা step এ কী হচ্ছে, কেন হচ্ছে, কোথায় সতর্ক থাকতে হবে — সব বিস্তারিত আছে।
> যেখানে তোমার specific value দরকার (IP, password, database name), সেখানে `<এইভাবে>` placeholder দেওয়া আছে — run করার আগে অবশ্যই replace করতে হবে।

---

## পুরো কাজের ধারা (Overview)

```
Repository add
     ↓
MongoDB install
     ↓
mongod.conf configure
     ↓
Service start
     ↓
Admin user তৈরি  ←── authorization enable করার আগেই
     ↓
Authorization enable + restart
     ↓
Replica set initiate
     ↓
Firewall port open
     ↓
App database + user তৈরি
     ↓
Backup setup
```

---

## ধাপ ১ — Repository Add করা

### কী হচ্ছে এখানে?

Rocky Linux 9 এর নিজের default package list এ MongoDB নেই — বা থাকলেও অনেক পুরনো version। তাই MongoDB এর নিজস্ব official repository আলাদা করে add করতে হয়, যাতে `dnf install` করার সময় সে জানে কোথা থেকে MongoDB 8.0 নামাবে।

এই repository file টা `/etc/yum.repos.d/` folder এ রাখা হয় — এই folder এ যা থাকে `dnf` সবগুলো automatically পড়ে।

### Command

```bash
cat > /etc/yum.repos.d/mongodb-org-8.0.repo << 'EOF'
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-8.0.asc
EOF
```

### প্রতিটা line এর মানে

| Line | মানে |
|------|------|
| `[mongodb-org-8.0]` | Repository এর internal ID — যেকোনো unique নাম দেওয়া যায় |
| `name=MongoDB Repository` | শুধু human-readable label |
| `baseurl=...` | কোথা থেকে package download হবে — এখানে `/9/` মানে Rocky Linux 9 (EL9) |
| `gpgcheck=1` | Download করা package টা MongoDB এরই, কেউ tamper করেনি — এটা verify করবে |
| `gpgkey=...` | Verification এর জন্য MongoDB এর official public key |

### Verify করো

```bash
cat /etc/yum.repos.d/mongodb-org-8.0.repo
```

File এর content দেখাবে — ঠিকঠাক দেখালে পরের ধাপে যাও।

---

## ধাপ ২ — MongoDB Install করা

### কী হচ্ছে এখানে?

`mongodb-org` install করলে একসাথে অনেকগুলো component আসে:

| Component | কী করে |
|-----------|---------|
| `mongod` | MongoDB এর main server daemon — background এ চলে, সব data handle করে |
| `mongosh` | Interactive shell — এখানে commands type করে database manage করা যায় |
| `mongodump` | Backup নেওয়ার tool |
| `mongorestore` | Backup restore করার tool |
| `mongostat` | Live statistics দেখার tool |

### Command

```bash
dnf install -y mongodb-org
```

`-y` flag মানে "সব confirm এ automatically হ্যাঁ বলো" — manually Enter চাপতে হবে না।

### Install হয়েছে কিনা verify করো

```bash
mongod --version
mongosh --version
```

দুটোই version number দেখালে install সফল।

---

## ধাপ ৩ — mongod.conf Configure করা

### কী হচ্ছে এখানে?

`/etc/mongod.conf` হলো MongoDB এর main configuration file। Server start হওয়ার সময় এই file পড়ে সে জানে:
- Data কোথায় রাখবে
- Log কোথায় লিখবে
- কোন IP থেকে connection নেবে
- Password লাগবে কিনা
- Replica set এর নাম কী

**Service start করার আগেই এটা configure করতে হবে।**

### File open করো

```bash
vi /etc/mongod.conf
```

> **vi editor এ কাজ করার basic:**
> - `i` চাপলে insert mode — তখন লেখা যাবে
> - `Esc` চাপলে insert mode থেকে বের হও
> - `:wq` লিখে Enter — save করে বের হও
> - `:q!` লিখে Enter — save না করে বের হও

### পুরো config (এটা দিয়ে replace করো)

```yaml
# ─── Storage ───────────────────────────────────────────────────
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true

# ─── System Log ────────────────────────────────────────────────
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

# ─── Network ───────────────────────────────────────────────────
net:
  port: 27017
  bindIp: 127.0.0.1,<db-server-er-nij-ip>

# ─── Security ──────────────────────────────────────────────────
security:
  authorization: enabled

# ─── Operation Profiling ───────────────────────────────────────
operationProfiling:
  slowOpThresholdMs: 100

# ─── Replication ───────────────────────────────────────────────
replication:
  replSetName: "rs0"
```

### প্রতিটা section এর বিস্তারিত ব্যাখ্যা

---

**`storage` section**

```yaml
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true
```

`dbPath` — MongoDB এর actual data files (BSON format) এখানে থাকবে। এই folder এ space যেন যথেষ্ট থাকে।

`journal: enabled: true` — Journal মানে হলো প্রতিটা write operation আগে একটা "log" এ লেখা হয়, তারপর actual data file এ। Server হঠাৎ crash করলে journal দেখে data recover করা যায়। Production এ এটা সবসময় `true` রাখতে হবে।

---

**`systemLog` section**

```yaml
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
```

`destination: file` — Log কোথায় যাবে। `file` মানে file এ লিখবে।

`logAppend: true` — Server restart হলে আগের log মুছে নতুন শুরু করবে না — আগেরটার সাথে যোগ করবে। এটা `true` না হলে restart এর আগের সব log হারিয়ে যাবে।

`path` — Log file এর location।

---

**`net` section**

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1,<db-server-er-nij-ip>
```

`port: 27017` — MongoDB এর default port। এটা সাধারণত পরিবর্তন করা হয় না।

`bindIp` — MongoDB কোন কোন IP address এ "কান পাতবে" (listen করবে)।
- `127.0.0.1` — localhost, মানে same server থেকে connect করতে পারবে
- `<db-server-er-nij-ip>` — DB server এর নিজের IP — এটা দিলে অন্য server থেকে (backend server) connect করা যাবে

> **⚠️ `<db-server-er-nij-ip>` replace করতে হবে।** DB server এর actual IP জানতে:
> ```bash
> ip addr show
> # অথবা
> hostname -I
> ```
> যে IP দেখাবে সেটা বসাও।

> **কেন `0.0.0.0` দেওয়া উচিত না:** `0.0.0.0` মানে সব IP থেকে connection নেবে — মানে internet থেকেও। Production এ specific IP দাও।

---

**`security` section**

```yaml
security:
  authorization: enabled
```

এটা MongoDB এর সবচেয়ে গুরুত্বপূর্ণ security setting। Default এ MongoDB **কোনো password ছাড়াই** যে কাউকে connect করতে দেয়। `authorization: enabled` দিলে valid username+password ছাড়া কেউ ঢুকতে পারবে না।

> **⚠️ Critical order:** এই setting টা আগে থেকেই config এ রাখো, কিন্তু **admin user তৈরি করার আগে** service restart করো না। নইলে lock হয়ে যাবে।
>
> এই guide এ সঠিক order মেনে চলা হয়েছে — admin user তৈরির পরে restart করা হবে।

---

**`operationProfiling` section**

```yaml
operationProfiling:
  slowOpThresholdMs: 100
```

100ms এর বেশি সময় নেওয়া query গুলো log এ mark করবে "slow operation" হিসেবে। এটা performance problem diagnose করতে কাজে লাগে।

---

**`replication` section**

```yaml
replication:
  replSetName: "rs0"
```

এটা single server হলেও কেন দরকার:
- **Oplog চালু হয়** — Oplog মানে সব write operation এর একটা ordered log। এটা consistent backup এর জন্য দরকার।
- **ভবিষ্যতে** যদি আরেকটা server যোগ করে actual replica set বানাতে হয়, তখন data migration ছাড়াই করা যাবে।

`"rs0"` নামটা convention — পরিবর্তন করা যায়, কিন্তু পরে initiate করার সময় এই নামই match করতে হবে।

---

## ধাপ ৪ — Service Start করা

### কী হচ্ছে এখানে?

`systemctl` হলো Linux এর service manager। এটা দিয়ে service start/stop/restart করা যায়, এবং server boot হলে automatically চালু হবে কিনা সেটা set করা যায়।

### Commands

```bash
# Service চালু করো
systemctl start mongod

# Server reboot হলে automatically উঠবে — এটা না দিলে restart এর পর manually start করতে হবে
systemctl enable mongod

# Status দেখো
systemctl status mongod
```

### Status output কী দেখবে

```
● mongod.service - MongoDB Database Server
   Active: active (running) since ...
```

`active (running)` দেখলে সব ঠিক আছে।

### যদি `failed` দেখায়

```bash
# Log দেখো — সমস্যার কারণ এখানে থাকবে
tail -50 /var/log/mongodb/mongod.log
```

সাধারণ কারণ: `mongod.conf` এ yaml formatting ভুল, অথবা `bindIp` তে ভুল IP।

---

## ধাপ ৫ — Admin User তৈরি করা

### কী হচ্ছে এখানে?

এই মুহূর্তে `authorization: enabled` config এ আছে কিন্তু service এখনো এটা নিয়ে চলেনি (restart হয়নি)। তাই এখন password ছাড়াই mongosh দিয়ে connect করা যাচ্ছে। এই সুযোগে admin user তৈরি করে নিতে হবে।

Admin user তৈরি হয়ে গেলে service restart করব — তখন authorization fully active হবে।

### Shell এ ঢোকো

```bash
mongosh
```

Prompt পরিবর্তন হয়ে `test>` দেখাবে — মানে shell এ আছো।

### Admin user তৈরি করো

```javascript
use admin

db.createUser({
  user: "admin",
  pwd: "<admin_er_jonno_strong_password>",
  roles: [{ role: "root", db: "admin" }]
})
```

**প্রতিটা অংশের মানে:**

`use admin` — `admin` database এ switch করো। এটা MongoDB এর special system database — সব administrative user এখানেই থাকে।

`user: "admin"` — Username। যেকোনো নাম দেওয়া যায়।

`pwd: "..."` — Password। Production এ strong password দাও — uppercase, lowercase, number, special character সহ।

`roles: [{ role: "root", db: "admin" }]` — `root` role মানে সব database এ সব কাজ করার permission। এটা শুধু DBA এর জন্য — application user কে কখনো `root` দেওয়া হয় না।

### Success হলে দেখাবে

```
{ ok: 1 }
```

### Shell থেকে বের হও

```javascript
exit
```

---

## ধাপ ৬ — Authorization Activate করতে Restart

Config এ `authorization: enabled` আগে থেকেই আছে। এখন restart করলে সেটা কার্যকর হবে।

```bash
systemctl restart mongod
```

### Verify — authorization কাজ করছে কিনা

Password ছাড়া connect করার চেষ্টা করো — reject হওয়া উচিত:

```bash
mongosh --eval "db.adminCommand({ listDatabases: 1 })"
```

এটা এখন `Unauthorized` error দেবে — এটাই চাওয়া। মানে authorization ঠিকমতো কাজ করছে।

এবার admin password দিয়ে verify করো যে admin user ঢুকতে পারছে:

```bash
mongosh -u admin -p <password> --authenticationDatabase admin --eval "db.adminCommand({ listDatabases: 1 })"
```

Database list দেখালে সব ঠিক আছে।

---

## ধাপ ৭ — Replica Set Initiate করা

### কী হচ্ছে এখানে?

`mongod.conf` এ `replSetName: "rs0"` দিয়েছি — এটা বলেছে "আমি একটা replica set এর অংশ হতে চাই, নাম rs0"। কিন্তু শুধু config এ দিলেই হয় না, একবার explicitly initiate করতে হয়।

`rs.initiate()` run করলে:
- এই node PRIMARY হয়ে যায়
- Oplog চালু হয়
- Replica set officially তৈরি হয়

### Admin দিয়ে login করো

```bash
mongosh -u admin -p <password> --authenticationDatabase admin
```

### Initiate করো

```javascript
rs.initiate()
```

### Output দেখবে এরকম

```json
{
  ok: 1,
  operationTime: ...,
  ...
}
```

Prompt পরিবর্তন হয়ে `rs0 [direct: primary]>` হয়ে যাবে — এটাই confirmation যে replica set চালু হয়েছে এবং এই node PRIMARY।

### Status verify করো

```javascript
rs.status()
```

Output এ `"stateStr": "PRIMARY"` দেখলে সফল।

### Shell থেকে বের হও

```javascript
exit
```

---

## ধাপ ৮ — Firewall Port Open করা

### কী হচ্ছে এখানে?

Rocky Linux 9 এ `firewalld` default এ চালু থাকে এবং সব incoming connection block করে রাখে। MongoDB এর port 27017 explicitly open না করলে backend server থেকে connect করা যাবে না।

তবে সবার জন্য open করা নিরাপদ না — শুধু specific IP (backend server) এর জন্য open করতে হবে।

### Command

```bash
# শুধু backend server এর IP থেকে 27017 port এ connection allow করো
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<backend-server-ip>/32" port protocol="tcp" port="27017" accept'

# Rule টা apply করো
firewall-cmd --reload
```

> **⚠️ `<backend-server-ip>` replace করতে হবে।** যে server থেকে backend application MongoDB তে connect করবে সেই server এর IP। Backend team বা DevOps team এর কাছ থেকে নিশ্চিত করে নাও।

> **`/32` মানে:** এটা একটা single IP address — শুধু ঐ একটা IP থেকে connection আসবে। Range দিতে হলে `/24` ইত্যাদি ব্যবহার হয়।

### Verify করো

```bash
firewall-cmd --list-all
```

Output এ `rich rules` section এ তোমার rule দেখাবে।

### তোমার নিজের machine থেকে access দরকার হলে

DBA হিসেবে তোমার local machine থেকে যদি কোনো tool দিয়ে connect করতে চাও:

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<tomar-machine-ip>/32" port protocol="tcp" port="27017" accept'
firewall-cmd --reload
```

---

## ধাপ ৯ — Application Database এবং User তৈরি করা

### কী হচ্ছে এখানে?

Admin user দিয়ে application connect করানো উচিত না — security risk। প্রতিটা application বা service এর জন্য আলাদা user তৈরি করতে হয় যার শুধু নিজের database এ permission আছে।

### Login করো

```bash
mongosh -u admin -p <password> --authenticationDatabase admin
```

### Database এবং User তৈরি করো

> **⚠️ তোমার flight safety application এ কোন কোন database লাগবে সেটা backend team এর কাছ থেকে নিশ্চিত করে নাও আগে।** নিচে একটা example pattern দেওয়া হলো।

```javascript
// যে database এ user তৈরি করবে সেখানে switch করো
use <database_name>

// User তৈরি করো — শুধু এই database এ readWrite permission দিয়ে
db.createUser({
  user: "<app_user_name>",
  pwd:  "<app_user_password>",
  roles: [{ role: "readWrite", db: "<database_name>" }]
})
```

**`readWrite` role মানে:**
- Data পড়তে পারবে (SELECT এর equivalent)
- Data লিখতে পারবে (INSERT/UPDATE/DELETE এর equivalent)
- Database drop করতে পারবে না
- অন্য user তৈরি করতে পারবে না
- অন্য database দেখতে পারবে না

### Collection তৈরি করো (optional)

MongoDB তে collection প্রথম insert এর সময় automatically তৈরি হয়। কিন্তু explicitly তৈরি করে confirm করা যায়:

```javascript
use <database_name>
db.createCollection("<collection_name>")
```

Collection মানে relational database এর table এর equivalent — তবে fixed schema নেই।

### App user এর connection verify করো

নতুন terminal এ বা exit করে:

```bash
mongosh -u <app_user_name> -p <app_user_password> --authenticationDatabase <database_name>
```

Connect হলে সফল। এই user দিয়ে অন্য database এ ঢোকার চেষ্টা করলে `Unauthorized` দেবে — এটাই expected।

---

## ধাপ ১০ — Backup Setup করা

### কী হচ্ছে এখানে?

`mongodump` MongoDB এর official backup tool। এটা সব data BSON format এ export করে — পরে `mongorestore` দিয়ে যেকোনো MongoDB তে restore করা যায়।

### Manual backup (যেকোনো সময় চালাতে পারবে)

```bash
mongodump \
  --authenticationDatabase admin \
  -u admin \
  -p <password> \
  --out /var/backups/mongodb/$(date +%Y%m%d_%H%M%S) \
  --gzip
```

`--gzip` — Output compress করে। Size অনেক কমে যায়।

`$(date +%Y%m%d_%H%M%S)` — Folder এর নামে timestamp বসে, যেমন `20250405_143022`।

### Automated daily backup script তৈরি করো

```bash
cat > /usr/local/bin/mongo_backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/var/backups/mongodb"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

mkdir -p "$BACKUP_DIR"

mongodump \
  --authenticationDatabase admin \
  -u admin \
  -p "<password>" \
  --out "$BACKUP_DIR/backup_$DATE" \
  --gzip

# পুরনো backup মুছো (7 দিনের বেশি পুরনো)
find "$BACKUP_DIR" -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;

echo "$(date): Backup completed — $BACKUP_DIR/backup_$DATE" >> /var/log/mongo_backup.log
EOF
```

> **⚠️ Script এ `<password>` replace করতে হবে।** Admin এর actual password বসাও।

Script কে executable করো:

```bash
chmod 700 /usr/local/bin/mongo_backup.sh
```

`chmod 700` — শুধু owner (root) read/write/execute করতে পারবে। অন্য কেউ এই file এর content দেখতে পারবে না — password এর জন্য জরুরি।

### Cron দিয়ে schedule করো

```bash
crontab -e
```

এই line যোগ করো:

```cron
0 3 * * * /usr/local/bin/mongo_backup.sh
```

**Cron syntax মানে:** `মিনিট ঘণ্টা দিন মাস সপ্তাহের_দিন`

`0 3 * * *` = প্রতিদিন রাত ৩:০০ তে। `*` মানে "সব"।

### Backup manually test করো

```bash
/usr/local/bin/mongo_backup.sh

# Log দেখো
cat /var/log/mongo_backup.log

# Backup folder দেখো
ls -lh /var/backups/mongodb/
```

### Restore করার পদ্ধতি (দরকার হলে)

```bash
mongorestore \
  --authenticationDatabase admin \
  -u admin \
  -p <password> \
  --drop \
  --gzip \
  /var/backups/mongodb/<backup_folder_name>
```

`--drop` — Restore করার আগে existing collection drop করে নেয়। এতে duplicate document আসে না। যদি existing data রেখে merge করতে চাও তাহলে `--drop` বাদ দাও।

---

## দরকারি Commands — Quick Reference

### Connection

```bash
# Admin হিসেবে login
mongosh -u admin -p <password> --authenticationDatabase admin

# App user হিসেবে login
mongosh -u <user> -p <password> --authenticationDatabase <database_name>

# Remote server থেকে connect
mongosh --host <db-server-ip> --port 27017 -u admin -p <password> --authenticationDatabase admin
```

### Shell এর ভেতরে

```javascript
// সব database দেখো
show dbs

// Database switch করো
use <database_name>

// Current database কোনটা
db

// Collections দেখো
show collections

// Replica set status
rs.status()

// Server statistics
db.serverStatus()

// Active operations দেখো
db.currentOp({ "active": true })

// Shell থেকে বের হও
exit
```

### User Management

```javascript
// সব user দেখো (admin database এ থেকে)
use admin
db.system.users.find().pretty()

// Password change করো
db.changeUserPassword("<username>", "<new_password>")

// User এর extra role দাও
db.grantRolesToUser("<username>", [{ role: "readWrite", db: "<another_db>" }])

// User delete করো
use <database_where_user_was_created>
db.dropUser("<username>")
```

### Log দেখা

```bash
# Live log
tail -f /var/log/mongodb/mongod.log

# শুধু slow operation গুলো
tail -f /var/log/mongodb/mongod.log | grep -i "slow"

# Error গুলো
grep -i "error\|failed" /var/log/mongodb/mongod.log

# Backup log
tail -f /var/log/mongo_backup.log
```

### Service Management

```bash
systemctl start mongod
systemctl stop mongod
systemctl restart mongod
systemctl status mongod
```

---

## MongoDB Roles — কোনটা কাকে দেবে

| Role | কী পারে | কাকে দেবে |
|------|---------|-----------|
| `root` | সব database এ সব কাজ | শুধু DBA admin user কে |
| `readWrite` | নির্দিষ্ট database এ read + write | Application user কে |
| `read` | নির্দিষ্ট database এ শুধু read | Reporting বা read-only tool কে |
| `dbAdmin` | Index তৈরি, schema management — data access নেই | Developer যে schema manage করে |
| `userAdmin` | নির্দিষ্ট database এ user তৈরি/manage | Secondary admin কে |

---

## Key File Locations

| কী | কোথায় |
|----|--------|
| Configuration file | `/etc/mongod.conf` |
| Data directory | `/var/lib/mongo` |
| Log file | `/var/log/mongodb/mongod.log` |
| Repository file | `/etc/yum.repos.d/mongodb-org-8.0.repo` |
| Backup script | `/usr/local/bin/mongo_backup.sh` |
| Backup output | `/var/backups/mongodb/` |
| Backup log | `/var/log/mongo_backup.log` |

---

## সম্পূর্ণ Uninstall (দরকার হলে)

> **⚠️ এটা run করলে সব data চিরতরে মুছে যাবে। আগে verified backup নাও।**

```bash
systemctl stop mongod
systemctl disable mongod
dnf erase -y $(rpm -qa | grep mongodb-org)
rm -rf /var/lib/mongo
rm -rf /var/log/mongodb
rm -f /etc/mongod.conf
```

---

## জিনিসগুলো যা তোমাকে আগে থেকে জেনে নিতে হবে

এগুলো এই guide এ দেওয়া সম্ভব না — তোমার specific environment এর উপর নির্ভর করে:

- [ ] **DB server এর নিজের IP** — `ip addr show` দিয়ে পাবে, `bindIp` তে বসাতে হবে
- [ ] **Backend server এর IP** — DevOps বা backend team এর কাছ থেকে নাও, firewall rule এ লাগবে
- [ ] **কোন কোন database লাগবে** — Backend team এর কাছ থেকে exact database name নাও
- [ ] **কোন কোন application user লাগবে** — Backend team এর কাছ থেকে user এবং কোন database এ access লাগবে জেনে নাও
- [ ] **Admin password কী হবে** — Strong password ঠিক করো এবং team password manager এ রাখো
- [ ] **Backup কতদিনের রাখতে হবে** — Script এ `RETENTION_DAYS` adjust করো (default 7 দিন রেখেছি)
- [ ] **Backup কখন চলবে** — Cron schedule বদলাও যদি রাত ৩টা suitable না হয়

---

*Document শেষ — MongoDB 8.0 | Rocky Linux 9*

---

---

# Part B — Migration (MongoDB)
**Combined Server (Docker) → New DB Server (Binary)**

---

## Migration এর আগে বুঝে নাও

MongoDB migration এ PostgreSQL এর মতো version downgrade এর সমস্যা নেই — `mongodump` এবং `mongorestore` version এর ব্যাপারে অনেক flexible। তুমি যে version এর MongoDB থেকেই dump করো, latest MongoDB তে restore করা যায়।

এখানে method হলো:

```
mongodump (Combined Server Docker থেকে BSON বের করো)
       ↓
transfer to new server
       ↓
mongorestore (New DB Server এ restore করো)
```

এই method এ:
- ✅ কোনো data loss নেই
- ✅ Downtime controlled
- ✅ Rollback সহজ — combined server এর MongoDB container কখনো বন্ধ করা হয় না
- ✅ Version flexibility আছে

---

## Migration এর আগে যা করতে হবে

### ১. Source এ data size দেখো

```bash
# Combined server এ
docker exec <mongo-container-name> mongosh \
  -u admin -p <password> \
  --authenticationDatabase admin \
  --eval "db.adminCommand({ listDatabases: 1 })"
```

প্রতিটা database এর `sizeOnDisk` note করো — dump এবং transfer কতক্ষণ লাগবে আন্দাজ করতে পারবে।

### ২. New server এ disk space দেখো

```bash
df -h /var/backups
df -h /tmp
```

Source database এর total size এর কমপক্ষে ২ গুণ free space রাখো।

### ৩. Connectivity test করো

```bash
# New DB server থেকে combined server এ
ping <combined-server-ip>

# Combined server থেকে new server এ SSH test
ssh user@<new-db-server-ip> "echo 'ok'"
```

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
    │── final mongodump ────────────────▶│ (transfer)
    │                                    │
    │   DB container চলছেই              │── mongorestore
    │   (rollback এর জন্য)              │
    │                                    │── data verify
    │                                    │
    │◀── backend reconnects ────────────│
    │                                    │
[Downtime শেষ]                           │
```

---

### Migration ধাপ ১ — Pre-Migration Dry Run (Downtime এর আগে)

App বন্ধ না করে একটা test dump নাও। কতক্ষণ লাগে সেটা measure করো।

```bash
# Combined server এ
time docker exec <mongo-container-name> mongodump \
  --authenticationDatabase admin \
  -u admin \
  -p <password> \
  --out /tmp/mongo_test_dump \
  --gzip

# Dump folder এর size দেখো
docker exec <mongo-container-name> du -sh /tmp/mongo_test_dump

# Dump বের করে নাও container থেকে
docker cp <mongo-container-name>:/tmp/mongo_test_dump /tmp/mongo_test_dump

# New server এ transfer speed test করো
time scp -r /tmp/mongo_test_dump user@<new-db-server-ip>:/tmp/mongo_test_dump
```

**Test dump টা new server এ restore করে দেখো:**

```bash
# New DB server এ
mongorestore \
  --authenticationDatabase admin \
  -u admin \
  -p <password> \
  --drop \
  --gzip \
  /tmp/mongo_test_dump

# কোনো error আসলে দেখো
# Successful হলে collection count check করো
```

---

### Migration ধাপ ২ — Downtime Window শুরু

**Application team কে জানাও।**

```bash
# Combined server এ — শুধু app containers বন্ধ করো
docker stop <frontend-container-name> <backend-container-name>

# Confirm করো
docker ps
# শুধু DB containers দেখাবে
```

MongoDB container বন্ধ করবে না — rollback এর জন্য।

---

### Migration ধাপ ৩ — Final Dump নেওয়া

App বন্ধ, নতুন write আসছে না — এখন final dump নাও।

```bash
# Combined server এ

# Container এর ভেতরে dump করো
docker exec <mongo-container-name> mongodump \
  --authenticationDatabase admin \
  -u admin \
  -p <password> \
  --out /tmp/mongo_final_$(date +%Y%m%d_%H%M%S) \
  --gzip

# Dump টা container থেকে host এ বের করে নাও
# (container এর ভেতরের path টা note করো আগে)
DUMP_NAME=$(docker exec <mongo-container-name> ls /tmp | grep mongo_final | tail -1)
docker cp <mongo-container-name>:/tmp/$DUMP_NAME /tmp/$DUMP_NAME

echo "Dump ready: /tmp/$DUMP_NAME"
ls -lh /tmp/$DUMP_NAME
```

**`--gzip` কেন:** Dump files compress হয় — size অনেক কমে, transfer দ্রুত হয়।

**`mongodump` কী করে:** প্রতিটা database এর প্রতিটা collection আলাদা `.bson` file এ export করে। Metadata (indexes, collection options) আলাদা `.json` file এ থাকে।

---

### Migration ধাপ ৪ — Dump Transfer করা

```bash
# Combined server থেকে new DB server এ
scp -r /tmp/$DUMP_NAME user@<new-db-server-ip>:/tmp/

# Transfer verify করো
ssh user@<new-db-server-ip> "du -sh /tmp/$DUMP_NAME"
```

দুই জায়গায় size same হওয়া উচিত।

---

### Migration ধাপ ৫ — New Server এ Restore করা

```bash
# New DB server এ
mongorestore \
  --authenticationDatabase admin \
  -u admin \
  -p <password> \
  --drop \
  --gzip \
  /tmp/<dump_folder_name> \
  2>&1 | tee /tmp/mongo_restore_log.txt

# Error আছে কিনা দেখো
grep -i "error\|failed" /tmp/mongo_restore_log.txt
```

**`--drop`:** Restore এর আগে existing collection drop করে। New server এ এখনো কোনো data নেই, তবু দেওয়া ভালো — idempotent হয়।

**Restore কী করে:** প্রতিটা `.bson` file থেকে documents insert করে, `.json` file দেখে indexes recreate করে।

---

### Migration ধাপ ৬ — Data Verify করা

**New server এ:**

```bash
mongosh -u admin -p <password> --authenticationDatabase admin
```

```javascript
// সব database দেখো
show dbs

// প্রতিটা important database এ যাও
use <database_name>

// Collections দেখো
show collections

// Critical collection এর document count দেখো
db.<collection_name>.countDocuments()

exit
```

**Combined server এ একই count করো:**

```bash
docker exec <mongo-container-name> mongosh \
  -u admin -p <password> \
  --authenticationDatabase admin \
  --eval "use <database_name>; db.<collection_name>.countDocuments()"
```

Count same হলে migration successful।

> **⚠️ যদি count আলাদা হয়:** Rollback করো — app containers combined server এ উঠিয়ে দাও এবং investigate করো।

---

### Migration ধাপ ৭ — Application Reconnect করা

Data verify হলে backend team কে জানাও। তারা connection string update করবে:

```
পুরনো: mongodb://admin:<pass>@<combined-server-ip>:27017/<dbname>?authSource=admin
নতুন:  mongodb://<app_user>:<pass>@<new-db-server-ip>:27017/<dbname>?authSource=<dbname>
```

Application উঠিয়ে test করো — login, data read, data write সব।

সব ঠিক থাকলে **downtime শেষ।**

---

### Rollback Plan

যেকোনো ধাপে সমস্যা হলে:

```bash
# Combined server এ — app আবার উঠিয়ে দাও
docker start <frontend-container-name> <backend-container-name>
```

Combined server এর MongoDB container কখনো বন্ধ করা হয়নি। Data সেখানে অক্ষত আছে।

---

## জিনিসগুলো যা Migration এর আগে জেনে নিতে হবে

- [ ] **Combined server এ MongoDB container এর নাম** — `docker ps`
- [ ] **Combined server এ MongoDB admin password**
- [ ] **Backend server এর IP** — DevOps team এর কাছ থেকে
- [ ] **কোন কোন database migrate করতে হবে** — Backend team confirm করবে
- [ ] **Downtime window কখন** — Application team এর সাথে fix করো
- [ ] **Rollback decision maker** — সমস্যা হলে কে decide করবে আগেই ঠিক করো

---

*MongoDB Migration section শেষ*
