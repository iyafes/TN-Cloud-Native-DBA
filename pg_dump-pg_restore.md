# pg_dump & pg_restore

## Environment Reference

| Node | IP | Role |
|------|----|------|
| pnode1 | 172.16.93.136 | Replica |
| pnode4 | 172.16.93.142 | Primary (Leader) |
| pnode5 | 172.16.93.144 | Replica |
| haproxy | 172.16.93.139 | Load Balancer |
| backup | 172.16.93.143 | Standalone PostgreSQL (restore target) |

**All dump commands run from the backup server (172.16.93.143)** — pg_dump connects to the cluster remotely over the network. You do not need to be on a cluster node to take a backup.

> **psql is not available on the haproxy node by default.** Always run psql and pg_dump/pg_restore from a db node (pnode1, pnode4, pnode5) or the backup server.

---

## 1. What is pg_dump

pg_dump is PostgreSQL's built-in logical backup tool. "Logical" means it does not copy the actual data files on disk — instead, it connects to a running PostgreSQL instance and exports the database contents as SQL statements or a binary representation.

A pg_dump backup looks like this internally:

```sql
CREATE SCHEMA app_schema;
CREATE TABLE app_schema.employees (...);
INSERT INTO app_schema.employees VALUES (...);
INSERT INTO app_schema.employees VALUES (...);
```

Running this output on any PostgreSQL server recreates the same database.

### What pg_dump includes

- Table definitions (DDL)
- Data (INSERT statements or binary COPY format)
- Sequences and their current values
- Indexes, constraints, triggers
- Views, functions, stored procedures
- Permission grants (ACL)
- Row Level Security policies

### What pg_dump does NOT include

- **Roles and users** — these are global (instance-level) objects, not database-level. Use `pg_dumpall --roles-only` to dump them separately.
- **Tablespace definitions**
- Other databases in the same instance

This is a critical point: if your database has GRANTs to roles like `readonly_role`, pg_dump will include those GRANT statements — but it will not create the roles themselves. Restoring to a new server without those roles will produce errors for the ACL entries, even though the data restores successfully.

### Advantages

- Works on a live database — no downtime required
- Works over the network — run from any machine with network access to PostgreSQL
- Selective — dump a full database, a specific schema, or just one table
- Cross-version compatible — a dump from PostgreSQL 14 can generally be restored on PostgreSQL 16
- Output is human-readable in plain SQL format

### Limitations

- **No PITR** — this is a point-in-time snapshot. Changes made after the dump are not captured.
- **Slow on large databases** — dumping and restoring a 500GB database can take hours
- **Logical only** — cannot be used to set up streaming replication

---

## 2. Output Formats

pg_dump supports four output formats. The format determines how you restore and what options are available.

| Format | Flag | File type | Restore tool | Selective restore | Compressed |
|--------|------|-----------|--------------|-------------------|------------|
| Plain SQL | `-F p` | `.sql` | `psql` | No | No |
| Custom | `-F c` | `.dump` | `pg_restore` | Yes | Yes |
| Directory | `-F d` | directory | `pg_restore` | Yes | Yes |
| Tar | `-F t` | `.tar` | `pg_restore` | Yes | No |

**Plain SQL** is human-readable. You can open it in a text editor and see exactly what will happen during restore. Good for small databases and quick inspection.

**Custom format** is the most commonly used in production. It is compressed (smaller file size), supports parallel restore, and allows selective restore of specific tables. Requires `pg_restore` instead of `psql`.

**Directory format** splits the dump into multiple files — one per table — which enables parallel dump (`-j` flag) and parallel restore. Best for very large databases.

---

## 3. Backup Server Setup

Before running any dumps, the backup server needs a working PostgreSQL instance to restore into, and it needs network access to the cluster.

### Initialize PostgreSQL on backup server (one-time setup)

```bash
# Initialize the data directory
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# Start and enable the service
sudo systemctl enable --now postgresql-16

# Verify it is running
sudo systemctl status postgresql-16
```

### Set postgres user password

```bash
sudo -u postgres psql -c "ALTER USER postgres WITH PASSWORD 'postgres123';"
```

### Allow remote connections

**Edit pg_hba.conf to allow connections from the cluster subnet:**

```bash
sudo tee -a /var/lib/pgsql/16/data/pg_hba.conf <<EOF
host    all             all             172.16.93.0/24          md5
EOF
```

**Edit postgresql.conf to listen on all interfaces:**

```bash
sudo sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" \
  /var/lib/pgsql/16/data/postgresql.conf
```

**Reload to apply changes:**

```bash
sudo systemctl reload postgresql-16
```

### Verify local connection

```bash
# Use sudo -u postgres to avoid peer authentication issues
sudo -u postgres psql -c "\l"

# Or connect with password via TCP
psql -h 127.0.0.1 -U postgres -c "\l"
```

> **Common mistake:** Running `psql -U postgres` without `-h 127.0.0.1` uses a Unix socket where peer authentication applies — it checks if your OS username matches the PostgreSQL username. Since your OS user is `restore` (not `postgres`), it fails with `FATAL: Peer authentication failed`. Always use `-h 127.0.0.1` when connecting with a password, or use `sudo -u postgres psql` to match the OS user.

---

## 4. Hands-On Practice

All dump commands below run from the **backup server (172.16.93.143)**. The dumps connect to the cluster through HAProxy port 5000 (Primary).

### 4.1 — Plain SQL Format Dump

```bash
pg_dump -h 172.16.93.139 -p 5000 -U postgres \
  -F p \
  -f /tmp/practicedb_plain.sql \
  practicedb
```

**Parameters explained:**
- `-h 172.16.93.139 -p 5000` — connect through HAProxy to the Primary
- `-F p` — plain SQL format
- `-f /tmp/practicedb_plain.sql` — output file path
- `practicedb` — the database to dump

**Inspect the output:**

```bash
# Check file size
ls -lh /tmp/practicedb_plain.sql

# View the contents — you will see CREATE TABLE, INSERT, GRANT statements
cat /tmp/practicedb_plain.sql
```

---

### 4.2 — Custom Format Dump

```bash
pg_dump -h 172.16.93.139 -p 5000 -U postgres \
  -F c \
  -f /tmp/practicedb_custom.dump \
  practicedb
```

**Compare sizes:**

```bash
ls -lh /tmp/practicedb_plain.sql /tmp/practicedb_custom.dump
```

The custom format file will be significantly smaller due to compression.

**List contents of a custom format dump:**

```bash
pg_restore -l /tmp/practicedb_custom.dump
```

This lists every object in the dump (tables, sequences, data, ACLs) without actually restoring anything. Useful for inspecting what is inside before restoring.

---

### 4.3 — Specific Table Dump

```bash
pg_dump -h 172.16.93.139 -p 5000 -U postgres \
  -F c \
  -t app_schema.employees \
  -f /tmp/employees_only.dump \
  practicedb
```

**`-t app_schema.employees`** — always include the schema name with the table name. Without the schema prefix, pg_dump may not find the table if it is not in the `public` schema.

```bash
# Verify only employees is in this dump
pg_restore -l /tmp/employees_only.dump
```

---

### 4.4 — Schema Only Dump (structure, no data)

```bash
pg_dump -h 172.16.93.139 -p 5000 -U postgres \
  -F c \
  --schema-only \
  -f /tmp/practicedb_schema_only.dump \
  practicedb
```

Use this when you want to recreate the table structure on a new server without copying any rows.

---

### 4.5 — Data Only Dump (data, no structure)

```bash
pg_dump -h 172.16.93.139 -p 5000 -U postgres \
  -F c \
  --data-only \
  -f /tmp/practicedb_data_only.dump \
  practicedb
```

Use this when the target database already has the correct table structure and you only want to load data.

---

### 4.6 — pg_dumpall — Dump Everything Including Roles

`pg_dumpall` dumps all databases in the instance plus global objects (roles, tablespaces). Use this when you need to migrate an entire PostgreSQL server.

```bash
# Dump everything
pg_dumpall -h 172.16.93.139 -p 5000 -U postgres \
  -f /tmp/cluster_full.sql
```

**Dump roles only (no database data):**

```bash
pg_dumpall -h 172.16.93.139 -p 5000 -U postgres \
  --roles-only \
  -f /tmp/roles_only.sql

# Inspect — you will see all roles: dev_user, app_user, readonly_role, etc.
cat /tmp/roles_only.sql
```

---

### 4.7 — Full Restore (Custom Format)

**Step 1 — Create the target database:**

```bash
sudo -u postgres psql -c "CREATE DATABASE practicedb_restored;"
```

**Step 2 — Restore:**

```bash
pg_restore -h 127.0.0.1 -U postgres \
  -d practicedb_restored \
  -v \
  /tmp/practicedb_custom.dump
```

**`-v` (verbose)** shows each object being restored. Always use `-v` so you can see exactly what succeeded and what failed.

**Expected errors during restore:**

You will see errors like:
```
pg_restore: error: could not execute query: ERROR:  role "readonly_role" does not exist
Command was: GRANT USAGE ON SCHEMA app_schema TO readonly_role;
```

**This is expected and not a critical failure.** The dump includes GRANT statements for roles that exist on the source cluster. The backup server does not have those roles, so the grants fail. The table structure and data restore successfully regardless.

The warning at the end:
```
pg_restore: warning: errors ignored on restore: 7
```
confirms that only the ACL (permission) entries failed — not the data.

**Step 3 — Verify:**

```bash
psql -h 127.0.0.1 -U postgres -d practicedb_restored \
  -c "SELECT * FROM app_schema.employees;"

psql -h 127.0.0.1 -U postgres -d practicedb_restored \
  -c "SELECT * FROM app_schema.departments;"

psql -h 127.0.0.1 -U postgres -d practicedb_restored \
  -c "\dt app_schema.*"
```

---

### 4.8 — Correct Way to Restore With Roles

To avoid ACL errors, restore roles first, then restore the database.

**Step 1 — Restore roles on the backup server:**

```bash
psql -h 127.0.0.1 -U postgres -f /tmp/roles_only.sql
```

**Step 2 — Drop and recreate the target database (clean state):**

```bash
sudo -u postgres psql -c "DROP DATABASE IF EXISTS practicedb_restored;"
sudo -u postgres psql -c "CREATE DATABASE practicedb_restored;"
```

**Step 3 — Restore the database:**

```bash
pg_restore -h 127.0.0.1 -U postgres \
  -d practicedb_restored \
  -v \
  /tmp/practicedb_custom.dump
```

This time there will be no ACL errors because the roles exist.

---

### 4.9 — Selective Table Restore

Restoring a specific table from a full database dump requires the target schema to exist first. pg_restore with `-t` only restores the table — it does not create the schema automatically.

**Step 1 — Create database and schema:**

```bash
sudo -u postgres psql -c "CREATE DATABASE practicedb_selective;"

psql -h 127.0.0.1 -U postgres -d practicedb_selective \
  -c "CREATE SCHEMA app_schema;"
```

**Step 2 — Restore only the employees table:**

```bash
pg_restore -h 127.0.0.1 -U postgres \
  -d practicedb_selective \
  -t employees \
  -v \
  /tmp/practicedb_custom.dump
```

**Step 3 — Verify only employees was restored:**

```bash
psql -h 127.0.0.1 -U postgres -d practicedb_selective \
  -c "\dt app_schema.*"
# Only employees appears — departments is not there

psql -h 127.0.0.1 -U postgres -d practicedb_selective \
  -c "SELECT * FROM app_schema.employees;"
```

> **Why does `-t employees` work without the schema prefix here?** When using `-t` in pg_restore, you specify just the table name (not `app_schema.employees`). pg_restore finds it by name within the dump. However, the schema must already exist in the target database — pg_restore will not create it automatically when doing selective restore.

---

### 4.10 — Plain SQL Restore

Plain SQL format uses `psql` for restore, not `pg_restore`. The SQL file is executed line by line.

**Step 1 — Create the target database:**

```bash
sudo -u postgres psql -c "CREATE DATABASE practicedb_plain_restored;"
```

**Step 2 — Restore using psql:**

```bash
psql -h 127.0.0.1 -U postgres \
  -d practicedb_plain_restored \
  -f /tmp/practicedb_plain.sql
```

You will see the same role-related errors as with custom format restore. These are expected.

**Step 3 — Verify:**

```bash
psql -h 127.0.0.1 -U postgres -d practicedb_plain_restored \
  -c "\dt app_schema.*"

psql -h 127.0.0.1 -U postgres -d practicedb_plain_restored \
  -c "SELECT * FROM app_schema.employees;"
```

**Plain SQL vs Custom format for restore:**

Plain SQL restore executes every statement visibly — errors appear immediately on screen as they happen. Custom format with `pg_restore -v` also shows errors, but the `-v` flag must be explicitly added. For debugging, plain SQL can be easier to follow because the output is line-by-line SQL.

---

### 4.11 — Verify All Dumps

```bash
ls -lh /tmp/*.sql /tmp/*.dump
```

Expected output shows all dump files with their sizes. Custom format files are smaller than plain SQL due to compression.

---

## 5. Common Mistakes and How to Avoid Them

**Mistake 1 — Peer authentication failure**

```
psql: error: FATAL: Peer authentication failed for user "postgres"
```

Cause: Running `psql -U postgres` without `-h 127.0.0.1` uses a Unix socket. The OS username (`restore`) does not match the PostgreSQL username (`postgres`).

Fix: Always use `-h 127.0.0.1` when connecting with a password:
```bash
psql -h 127.0.0.1 -U postgres -d mydb
```
Or switch to the postgres OS user:
```bash
sudo -u postgres psql
```

---

**Mistake 2 — Role does not exist errors during restore**

```
ERROR: role "readonly_role" does not exist
```

Cause: pg_dump includes GRANT statements for roles, but does not include role definitions. The roles exist on the source cluster but not on the restore target.

Fix: Always restore roles first using `pg_dumpall --roles-only`, then restore the database:
```bash
pg_dumpall -h 172.16.93.139 -p 5000 -U postgres --roles-only -f /tmp/roles_only.sql
psql -h 127.0.0.1 -U postgres -f /tmp/roles_only.sql
pg_restore -h 127.0.0.1 -U postgres -d target_db /tmp/backup.dump
```

---

**Mistake 3 — Schema does not exist during selective restore**

```
ERROR: schema "app_schema" does not exist
```

Cause: When using `-t tablename` with pg_restore, only the specified table is restored. The schema it belongs to is not created automatically.

Fix: Create the schema manually before selective restore:
```bash
psql -h 127.0.0.1 -U postgres -d target_db -c "CREATE SCHEMA app_schema;"
pg_restore -h 127.0.0.1 -U postgres -d target_db -t employees /tmp/backup.dump
```

---

**Mistake 4 — Using psql to restore a custom format dump**

```
psql: error: /tmp/backup.dump: not a valid SQL file
```

Cause: Custom format (`.dump`) is a binary format — `psql` cannot read it.

Fix: Use `pg_restore` for custom, directory, and tar formats. Only plain SQL (`.sql`) can be restored with `psql`.

---

**Mistake 5 — Forgetting `-t` schema prefix in pg_dump**

```bash
# This may not find the table if it is not in public schema
pg_dump -t employees ...

# Always include the schema name
pg_dump -t app_schema.employees ...
```

---

**Mistake 6 — Dumping from a replica instead of primary**

Dumps from replicas are generally safe for read-only data, but sequences may not reflect the latest values since sequence updates are not always replicated immediately. Always dump from the Primary (port 5000 via HAProxy) for consistent results.

---

## 6. Format Comparison Summary

| Scenario | Recommended format | Restore command |
|----------|--------------------|-----------------|
| Quick backup, human-readable | Plain SQL (`-F p`) | `psql -f file.sql` |
| Production backup, compressed | Custom (`-F c`) | `pg_restore -d dbname file.dump` |
| Large database, parallel restore | Directory (`-F d`) | `pg_restore -j 4 -d dbname dir/` |
| Specific table restore | Custom (`-F c`) | `pg_restore -t tablename file.dump` |
| Full instance migration | `pg_dumpall` | `psql -f file.sql` |
| Roles only | `pg_dumpall --roles-only` | `psql -f roles.sql` |

---

## 7. Quick Reference

```bash
# Dump — full database, plain SQL
pg_dump -h HOST -p PORT -U postgres -F p -f output.sql dbname

# Dump — full database, custom format (recommended)
pg_dump -h HOST -p PORT -U postgres -F c -f output.dump dbname

# Dump — specific table
pg_dump -h HOST -p PORT -U postgres -F c -t schema.tablename -f table.dump dbname

# Dump — schema only (no data)
pg_dump -h HOST -p PORT -U postgres -F c --schema-only -f schema.dump dbname

# Dump — data only (no structure)
pg_dump -h HOST -p PORT -U postgres -F c --data-only -f data.dump dbname

# Dump — all databases + roles
pg_dumpall -h HOST -p PORT -U postgres -f full.sql

# Dump — roles only
pg_dumpall -h HOST -p PORT -U postgres --roles-only -f roles.sql

# List contents of a custom dump (no restore)
pg_restore -l file.dump

# Restore — custom format
pg_restore -h HOST -U postgres -d dbname -v file.dump

# Restore — selective table (schema must exist first)
pg_restore -h HOST -U postgres -d dbname -t tablename -v file.dump

# Restore — plain SQL
psql -h HOST -U postgres -d dbname -f file.sql

# Restore — roles first, then database (correct production workflow)
psql -h HOST -U postgres -f roles.sql
pg_restore -h HOST -U postgres -d dbname -v file.dump
```

---

*Environment: Rocky Linux 9.7 | PostgreSQL 16 | Backup server: 172.16.93.143 | Cluster Primary via HAProxy: 172.16.93.139:5000*
