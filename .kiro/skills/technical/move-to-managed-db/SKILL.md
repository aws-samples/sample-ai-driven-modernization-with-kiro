---
name: move-to-managed-db
description: |
  Guide end-to-end migration from self-managed databases (MySQL, MariaDB, PostgreSQL on EC2
  or on-premises) to AWS managed services (Amazon Aurora, Amazon RDS). Covers compatibility
  assessment, migration method selection, execution, validation, and cutover.
  Always use when: migrating databases off EC2, evaluating Aurora vs RDS, assessing migration
  feasibility, selecting a migration method, planning cutover, or troubleshooting migration issues.
version: 1
metadata:
  service: [rds, aurora, dms, ec2, secretsmanager, cloudwatch, vpc, kms, s3]
  task: [migrate, configure, deploy, debug, optimize, validate, assess]
  persona: [developer, devops, architect, dba, sa]
  workload: [database, migration, modernization]
---

# Move to Managed DB

Migrate self-managed databases to Amazon Aurora or Amazon RDS — from compatibility assessment through validated cutover.

> **Language**: Respond in the user's language (Korean → Korean, English → English). Code, CLI, CDK, scripts, parameter-group entries, and resource names stay in English regardless of conversation language.

> **Working artifact — keep a live migration plan (`migration-plan.md`).** Real migrations span many sessions and decision points. At the start of an engagement, create a `migration-plan.md` in the working directory with one section per phase (0–9) and a checklist of the steps below. **Update it as each result lands** — record the source assessment values, the chosen method and *why*, every discovered DB client (Phase 7.5), the cutover timeline, validation evidence (row counts, checksums, processlist), and rollback notes. This is the running record of the migration: a step is "done" only when its result is written into the plan. Treat it the way the Kiro modernization flow updates its steering plan as work progresses — the document is the source of truth, not chat scrollback.

---

## Phase 0: Scope & Coverage (Read First)

**Engines covered.** This skill covers migration to **all RDS/Aurora engines**: MySQL, MariaDB, PostgreSQL, Oracle, SQL Server, Db2, Aurora MySQL, Aurora PostgreSQL.

- **Homogeneous** (same engine family, e.g. EC2 MySQL → Aurora MySQL, **Oracle → RDS Oracle, SQL Server → RDS SQL Server**): the native-tool fast paths below (Phases 1–9). DMS is *not* the default. Oracle and SQL Server lift-and-shift have **dedicated native paths** — Oracle Data Pump (Phase 5 §"Oracle Data Pump") and SQL Server native backup/restore via S3 (Phase 5 §"SQL Server Native Backup/Restore"). The matrix in Phase 3 covers them.
- **Heterogeneous** (Oracle/SQL Server → **Aurora/PostgreSQL/MySQL**, i.e. the engine family *changes*): requires schema/code conversion. See **[references/heterogeneous-migration.md](references/heterogeneous-migration.md)** — SCT / DMS Schema Conversion / Babelfish, PL/SQL→PL/pgSQL challenges, license implications. **Oracle → RDS Oracle and SQL Server → RDS SQL Server are NOT heterogeneous** — stay in this skill.
- **Korean-market source engines** (Tibero, CUBRID, Altibase): **no native AWS DMS/SCT tooling** — PoC + JDBC paths. See heterogeneous-migration.md §5.

**Korean enterprise scenarios (CHECK EARLY).** Korean enterprises almost always wrap the DB in a domestic **access-control/audit appliance** (Chakra Max, DBSafer, Petra) and a **DB encryption** product (Petra Cipher, D'Amo, CUBE-One). These break on managed RDS in their sniffing/agent/plug-in modes and are the **#1 cause of stalled migrations**. Before choosing a method, read **[references/third-party-db-security.md](references/third-party-db-security.md)** and **[references/regulatory-compliance.md](references/regulatory-compliance.md)** (PIPA encryption mandates, 망분리, ISMS-P, audit-log retention). The rule: **access control → vendor gateway mode in the VPC + Database Activity Streams; encryption → vendor API mode or KMS at-rest + app-side column encryption.**

> **Downtime expectation.** For EC2 → RDS/Aurora migrations, this skill targets a **10–30 second write-pause** at cutover (app restart or connection-pool refresh). True zero-downtime (0 lost requests) isn't achievable on this path without an intermediary proxy already in place — the skill optimizes for the *shortest* pause via CDC catch-up + coordinated cutover. To minimize further: pre-set connection-pool `maxLifetime` to 30s; prefer a coordinated app restart over Secrets Manager rotation (≈10s vs up to 5 min for pool TTL); deploy RDS Proxy on the **target** so future failovers are sub-second even though the initial cutover still pauses briefly. Blue/Green (< 1s with RDS Proxy) needs the source *already* on RDS — it applies to RDS→Aurora upgrades, not initial EC2→RDS moves. See Phase 8 "Minimize the write-pause window".

---

## Phase 1: Compatibility Assessment (CRITICAL — Do First)

Before choosing a migration method or target, assess whether the source workload is COMPATIBLE with managed services. The following are common **blockers** and **adjustments** that must be resolved first.

### 1.1 Blockers — Must Resolve Before Migration

| Category | Blocker | Impact | Resolution |
|----------|---------|--------|------------|
| **Encryption** | MySQL TDE (Transparent Data Encryption) enabled | Data encrypted at tablespace level won't transfer | Decrypt before migration → re-encrypt with AWS KMS at rest |
| **Encryption** | Keyring plugins (keyring_file, keyring_aws for MySQL) | Plugin not available on RDS/Aurora | Decrypt → use AWS KMS volume encryption instead |
| **Encryption** | Third-party agents (Vormetric, Thales CipherTrust) | Host-level encryption agents can't run on managed instances | Remove agent → use AWS KMS + column-level app encryption |
| **Encryption (KR)** | Korean DB encryption in **plug-in / OS-volume mode** (Petra Cipher plug-in, D'Amo DE/KE, CUBE-One) | Engine plug-ins and OS-volume agents can't run on managed RDS | Switch to vendor **API/app-side mode**, or KMS at-rest + app-side column encryption (SEED/ARIA at app layer if mandated). See `references/third-party-db-security.md` |
| **Access control (KR)** | Korean DB access/audit in **sniffing or host-agent mode** (Chakra Max, DBSafer agent, Petra sniffing) | No SPAN/port-mirror and no OS access on managed RDS | Move to vendor **gateway/proxy mode** in the VPC + **Database Activity Streams** for audit. See `references/third-party-db-security.md` |
| **Storage Engine** | MyISAM tables (Aurora target) | Aurora is InnoDB-only | Convert: `ALTER TABLE t ENGINE=InnoDB` before migration |
| **Storage Engine** | FEDERATED engine | Not supported on RDS or Aurora | Redesign as application-level cross-DB queries |
| **Storage Engine** | Custom engines (TokuDB, RocksDB, Spider) | Not available on managed services | Convert to InnoDB |
| **Auth** | PAM / LDAP authentication plugins | Not supported | Switch to native auth, IAM DB auth, or Kerberos (AD) |
| **Auth** | Custom auth plugins | Not loadable | Switch to native password or IAM auth |
| **Features** | C-compiled UDFs (User Defined Functions) | Cannot install custom .so files | Rewrite as stored functions or move logic to app layer |
| **Features** | Galera Cluster / Group Replication topology | Not available on Aurora | Redesign using Aurora Multi-AZ + read replicas |
| **PostgreSQL** | C-language extensions (custom compiled) | Cannot install on Aurora | Check `pg_available_extensions`; rewrite or remove |
| **PostgreSQL** | Direct pg_hba.conf access | No filesystem access | Use RDS security groups + IAM auth + SSL settings |
| **Compressed tables** | InnoDB compressed row_format (Aurora) | Not supported on Aurora | `ALTER TABLE t ROW_FORMAT=DYNAMIC` before migration |

### 1.2 Adjustments — Require Changes But Not Blocking

| Category | Issue | What Happens | Workaround |
|----------|-------|-------------|------------|
| **Privileges** | Code uses SUPER privilege | Will fail on RDS/Aurora | Use `rds_superuser_role` (RDS) or session-level alternatives |
| **Privileges** | DEFINER clauses in stored procs/views | Fails if definer user doesn't exist | Strip DEFINER or recreate user on target |
| **File I/O** | `LOAD DATA INFILE` (server-side) | Blocked on RDS/Aurora | Use `LOAD DATA LOCAL INFILE` (client-side) or S3 import |
| **File I/O** | `SELECT INTO OUTFILE` | Blocked | Use `SELECT ... INTO` with S3 (Aurora) or client-side export |
| **Replication** | Multi-source replication | Only on RDS MySQL 8.0.35+, NOT Aurora | Redesign consolidation approach |
| **Charset** | `lower_case_table_names` set to 2 | Not supported on Linux-based RDS/Aurora | Must be 0 (case-sensitive) or 1 (lowercase) |
| **Timezone** | Non-UTC timezone_data differences | Can corrupt TIMESTAMP columns during migration | Set `time_zone` parameter explicitly on target |
| **Versions** | MySQL 5.6 or earlier | Cannot migrate directly to Aurora MySQL 3.x (8.0) | Upgrade to 5.7 first, then migrate |
| **OS** | Cron jobs / shell scripts on DB host | No OS access on managed | Move to EventBridge Scheduler, Lambda, or ECS tasks |

### 1.3 Assessment Queries

**Check for TDE (MySQL):**
```sql
SELECT * FROM information_schema.INNODB_TABLESPACES WHERE ENCRYPTION = 'Y';
SHOW VARIABLES LIKE 'early-plugin-load';  -- keyring plugin loaded?
```

**Check storage engines (MySQL/MariaDB):**
```sql
SELECT TABLE_NAME, ENGINE FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_db' AND ENGINE != 'InnoDB';
```

**Check compressed tables (Aurora blocker):**
```sql
SELECT TABLE_NAME, ROW_FORMAT FROM information_schema.TABLES
WHERE TABLE_SCHEMA = 'your_db' AND ROW_FORMAT = 'Compressed';
```

**Check auth plugins:**
```sql
SELECT user, host, plugin FROM mysql.user WHERE plugin NOT IN ('mysql_native_password', 'caching_sha2_password');
```

**Check DEFINER clauses:**
```sql
SELECT ROUTINE_NAME, DEFINER FROM information_schema.ROUTINES WHERE ROUTINE_SCHEMA = 'your_db';
SELECT TABLE_NAME, DEFINER FROM information_schema.VIEWS WHERE TABLE_SCHEMA = 'your_db';
```

**Check UDFs:**
```sql
SELECT * FROM mysql.func;  -- Lists all installed UDFs
```

**Check PostgreSQL extensions:**
```sql
SELECT extname, extversion FROM pg_extension;
-- Compare against Aurora's available extensions:
SELECT name FROM pg_available_extensions ORDER BY name;
```

### 1.4 Oracle → RDS for Oracle — Blockers & Compatibility Checks

RDS for Oracle is managed: **no OS/SSH, no RAC, no ASM (EBS-backed), no SYS/SYSDBA, no custom patches.** The master user gets the `DBA` role **minus** `ALTER DATABASE`, `ALTER SYSTEM`, `CREATE ANY DIRECTORY`, `DROP ANY DIRECTORY`, `GRANT ANY PRIVILEGE`, `GRANT ANY ROLE` — admin tasks go through `rdsadmin` packages and parameter groups instead. Assess these before choosing a method:

| Category | Blocker / Adjustment | Impact | Resolution |
|----------|----------------------|--------|------------|
| **Engine version** | Only **19c and 21c** are supported/creatable on managed RDS. 10.2 / 11g / 12c / 18c are gone. | Cannot land on managed RDS | Upgrade to 19c+ first, or use **RDS Custom for Oracle** (BYOL/EE; note end-of-support 2027-03-31) / EC2 for deprecated majors. |
| **Edition / licensing** | License Included = **SE2 only**; Enterprise Edition = **BYOL only**. SE2 caps at 16 vCPU / 128 GiB, no Data Guard / read replicas. | Wrong target edition blocks creation | Decide LI-SE2 vs BYOL-EE up front; EE features (TDE, partitioning options) need BYOL-EE. |
| **Character set** | DB character set + NCHAR set are **fixed at instance creation, cannot change after**. CDB DB charset is always `AL32UTF8` (set non-default only on the PDB). | Mismatch corrupts data, unfixable post-create except recreate | Match target `--character-set-name` / `--nchar-character-set-name` to source (or proper superset) **before** loading. e.g. `KO16MSWIN949` for legacy Korean. |
| **Architecture** | RAC topology / ASM storage | Not available | Redesign as single-instance + Multi-AZ; storage is EBS, no ASM tuning. |
| **Disallowed import modes** | Data Pump **FULL mode** import; importing SYS/SYSTEM/RDSADMIN-owned objects (incl. Scheduler objects) | Can damage the data dictionary; unsupported | Use **schema or table mode** only; `EXCLUDE` system Scheduler objects (see Phase 5). |
| **Transportable tablespaces import** | Dump files made with `TRANSPORT_TABLESPACES`/`TRANSPORTABLE`/`TRANSPORT_FULL_CHECK` via the *regular* impdp path | Not supported on the standard path | Use the dedicated `rdsadmin.rdsadmin_transport_util` XTTS path (EE-only — see Phase 3). |
| **TDE wallet** | Customer-managed Oracle Wallet | Cannot import your own wallet into managed RDS | RDS manages the wallet/master key. Export with Data Pump `ENCRYPTION_MODE=PASSWORD` (TRANSPARENT mode unsupported), import into a TDE-enabled target whose wallet AWS generates. See §1.4 checks below. |
| **OS-file dependencies** | BFILE, external tables, `UTL_FILE`, FTP/SFTP, Messaging Gateway, Oracle Text File/URL datastores | No general filesystem; several features unsupported | Re-stage through RDS-managed directories (`DATA_PUMP_DIR` via S3); rework BFILE/external tables; Text must use non-File/URL datastores. |
| **Unsupported features** | Data Guard, Database Vault, Flashback Database, RAS, Unified Auditing Pure Mode, Workspace Manager (WMSYS), hybrid partitioned tables, OEM repository | Won't migrate | Inventory dependencies; redesign or drop. Data Guard target → EC2/RDS Custom only. |
| **File/block limits** | 16 TiB per single file (ext4); `DB_BLOCK_SIZE` fixed at creation; up to ~64 TiB instance storage | Large bigfile datafiles / block-size changes blocked | Plan tablespace layout; set block size at creation. |

**Oracle pre-migration inventory queries** (run on source, resolve before migrating):
```sql
-- Directory objects (master user can't CREATE ANY DIRECTORY on RDS)
SELECT owner, directory_name, directory_path FROM dba_directories;
-- External tables (depend on directory objects / OS files)
SELECT owner, table_name, default_directory_name FROM dba_external_tables;
-- Network/file ACL-dependent packages (UTL_HTTP/SMTP/TCP need re-granted ACLs + VPC egress)
SELECT * FROM dba_network_acls;
-- Database links (recreate; need VPC routing + SG rules)
SELECT owner, db_link, host FROM dba_db_links;
-- Scheduler jobs (recreate app-owned; never import SYS/SYSTEM-owned)
SELECT owner, job_name, schedule_type FROM dba_scheduler_jobs WHERE owner NOT IN ('SYS','SYSTEM','RDSADMIN');
-- Java in the DB
SELECT owner, COUNT(*) FROM dba_objects WHERE object_type LIKE 'JAVA%' GROUP BY owner;
-- TDE in use at source?
SELECT * FROM v$encryption_wallet;
SELECT tablespace_name, encrypted FROM dba_tablespaces WHERE encrypted='YES';
```

### 1.5 SQL Server → RDS for SQL Server — Blockers & Compatibility Checks

RDS for SQL Server is managed: **no OS/RDP, no `sysadmin` server role, no `RESTORE FROM DISK`.** The master user belongs only to `processadmin`, `public`, `setupadmin`; a DB creator gets `db_owner`. The file-based migration path is **native backup/restore via S3** (Phase 5). Assess:

| Category | Blocker / Adjustment | Impact | Resolution |
|----------|----------------------|--------|------------|
| **Engine version / restore direction** | Native restore accepts a `.bak` from an **equal-or-lower** engine version, **never higher**. Supported majors: 2016/2017/2019/2022. | Higher-version `.bak` won't restore | Pick target RDS engine version ≥ source. EOS versions (2014 and older) are auto-upgraded; can keep older DB `COMPATIBILITY_LEVEL`. |
| **Edition / licensing** | License Included (Enterprise/Standard/Web/Express) or **BYOM** (Ent/Std/Developer) via License Mobility + active Software Assurance. Standard capped at 24 cores / 128 GB by MS limits. | Wrong edition blocks features | Choose edition for the features you need (TDE, Multi-AZ). |
| **TDE edition gating** | TDE needs Enterprise on 2016/2017; **Standard or Enterprise** on 2019/2022. | TDE unavailable on Web/Express | Pick Standard+ / Enterprise. |
| **Security model** | Code needing `sysadmin`, `xp_cmdshell`, `CONTROL SERVER`, `UNSAFE`/`EXTERNAL_ACCESS` assembly, `CREATE ENDPOINT`, server-level triggers, `TRUSTWORTHY` | Roles/permissions not grantable on RDS | Refactor; move logic out of DB; CLR `SAFE` only on ≤2016, **not supported 2017+**. |
| **FILESTREAM** | `.bak` containing a FILESTREAM filegroup | Native restore rejects it | Remove FILESTREAM/FileTable; redesign as BLOB columns or S3. |
| **Unsupported features** | Log Shipping, Replication (as publisher/distributor), Maintenance Plans, Service Broker cross-instance endpoints, MSDTC/distributed transactions, PolyBase, Stretch DB, ML/R Services, backup to Azure Blob | Won't migrate | Emulate log shipping via native full+diff+log restores; recreate jobs; cross-instance messaging won't work. |
| **Server-level objects** | Logins, SQL Agent jobs, linked servers live in `master`/`msdb` — **not** carried by a user-DB `.bak` (can't import `msdb`) | Orphaned users + missing jobs after restore | Script logins (preserve SID + HASHED password), Agent jobs, and linked servers separately; recreate on RDS. See Phase 6. |
| **Max databases / Multi-AZ** | Per-instance DB limit depends on instance class + AZ mode; Multi-AZ native restore requires **FULL recovery model**; no native log *backups* from RDS | Restore/convert fails if over limit | Check the per-class limit before sizing; keep source in FULL recovery for minimal-downtime restores. |
| **Collation** | Instance/server default collation set at creation | Server-level collation can't change later | Match instance collation to source; DB/column collations ride in the `.bak`. |

**SQL Server pre-migration inventory queries** (run on source):
```sql
-- Recovery model (must be FULL for Multi-AZ native restore / log restores)
SELECT name, recovery_model_desc FROM sys.databases;
-- FILESTREAM filegroups (blocker for native restore)
SELECT DB_NAME(database_id) db, type_desc, name FROM sys.master_files WHERE type_desc='FILESTREAM';
-- CLR assemblies (unsupported 2017+)
SELECT name, permission_set_desc FROM sys.assemblies WHERE is_user_defined=1;
-- Logins to recreate on RDS (SID + hash preserved later — see Phase 6)
SELECT name, type_desc FROM sys.server_principals WHERE type IN ('S','U','G') AND name NOT LIKE '##%##';
-- SQL Agent jobs to recreate
SELECT name, enabled FROM msdb.dbo.sysjobs;
-- Linked servers to recreate
SELECT name, product, provider FROM sys.servers WHERE is_linked=1;
-- TDE-encrypted DBs (need certificate migration — see Phase 5)
SELECT DB_NAME(database_id) db, encryption_state FROM sys.dm_database_encryption_keys;
```

---

## Phase 2: Source Database Assessment

### Execution Location — How Will You Reach the Source DB? (Decide First)

The source DB is almost always in a **private subnet**, and your execution environment (laptop, CloudShell, an automation runner, this skill's host) is frequently **in a different VPC or has no `mysql`/`psql` client installed**. Settle the access path *before* running any assessment query — it determines how every command in Phases 2, 5, 7, and 8 is invoked.

| Option | When | How |
|--------|------|-----|
| **Direct** | Execution host is in the **same VPC/subnet** and has a client installed | Standard `mysql -h <db> …` / `psql` |
| **Bastion (SSH tunnel)** | A bastion host exists in the DB's VPC | `ssh -L 3306:<db-endpoint>:3306 ec2-user@bastion` then connect to `127.0.0.1:3306` |
| **SSM Port Forwarding** | DB host (or a host in-VPC) is an SSM-managed instance; you have a local client | `aws ssm start-session --target <instance-id> --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters '{"host":["<db-endpoint>"],"portNumber":["3306"],"localPortNumber":["13306"]}'` then connect to `127.0.0.1:13306` |
| **SSM Send-Command** | **No local client / cross-VPC** — run the query *on the DB host itself* (or another in-VPC SSM-managed host that has a client) | `aws ssm send-command --instance-ids <id> --document-name AWS-RunShellScript --parameters 'commands=["mysql -h 127.0.0.1 -e \"…\""]'` |

> **Default recommendation for an isolated/private-subnet source: SSM.** Port forwarding when you have a local client; **Send-Command when you don't** (run the client that already exists on the DB host). This avoids opening the DB to new networks just to migrate it. The same path is reused for the `mysqldump`/`pg_dump` export in Phase 5 and the processlist cross-check in Phases 7–8.

### Credential Handling — Never Put Passwords in `argv`

Every command below (assessment, dump, cutover) needs DB credentials. **Passwords in command-line arguments are visible in `ps -ef`, shell history, and — when run via SSM/SSH — in CloudTrail and SSM command history.** Rules:

- **Never** pass `-p<password>` or `--password=<pw>` on the command line.
- Use **`MYSQL_PWD`** env var (`export MYSQL_PWD=...; mysql -h … -u …`) or a **`--defaults-extra-file`** with `[client] password=...` (chmod 600).
- **Preferred:** fetch the secret **on the DB host** using the instance's IAM role, so the plaintext never transits your machine or appears in argv:
  ```bash
  # Run on the DB host (e.g. via SSM Send-Command); password stays on the host, out of argv
  export MYSQL_PWD=$(aws secretsmanager get-secret-value \
    --secret-id ecommerce-demo/db-credentials --query SecretString --output text \
    | python3 -c 'import sys,json;print(json.load(sys.stdin)["password"])')
  mysql -h 127.0.0.1 -u admin -e "SELECT VERSION();"
  ```
- PostgreSQL: use `PGPASSWORD` or a `~/.pgpass` (chmod 600) the same way.

### Information to Collect

| Category | What | How |
|----------|------|-----|
| Engine & version | MySQL 5.7/8.0, MariaDB 10.x, PostgreSQL 12-16 | `SELECT VERSION();` |
| Total size | Data + indexes in GB | See queries below |
| Table count | Number of tables, top 10 largest | `information_schema.tables` |
| Schema objects | Stored procs, triggers, views, events, functions | `information_schema.routines`, `pg_proc` |
| LOB columns | BLOB/TEXT/JSON columns, max sizes | Profile actual data sizes |
| Replication readiness | Binary logging (MySQL) / wal_level (PostgreSQL) | `SHOW VARIABLES LIKE 'log_bin'` / `SHOW wal_level` |
| Current connections | Typical/peak concurrent connections | `SHOW STATUS LIKE 'Threads_connected'` |
| Source location | EC2 (same VPC? same region?), on-premises, other cloud? | User input |
| **Network bandwidth** | Usable Mbps on the path source→AWS (Direct Connect / VPN / internet egress). Use the *usable* figure, not the link's rated speed. | User input / `iperf3` to an EC2 box in target region |
| **Transfer window** | Hours of acceptable downtime (offline copy) *or* hours available for bulk load before CDC catch-up | User input |

### Sizing Queries

**MySQL/MariaDB:**
```sql
SELECT table_schema,
  ROUND(SUM(data_length + index_length) / 1024 / 1024 / 1024, 2) AS size_gb,
  COUNT(*) AS table_count
FROM information_schema.tables
WHERE table_schema = 'your_db'
GROUP BY table_schema;
```

**PostgreSQL:**
```sql
SELECT pg_size_pretty(pg_database_size('your_db')) AS db_size;
SELECT count(*) AS table_count FROM pg_tables WHERE schemaname = 'public';
```

**Oracle:**
```sql
-- Total DB size (datafiles + temp)
SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) AS size_gb FROM dba_segments;
-- Per-schema size + object counts
SELECT owner, ROUND(SUM(bytes)/1024/1024/1024,2) AS size_gb
FROM dba_segments WHERE owner NOT IN ('SYS','SYSTEM','RDSADMIN') GROUP BY owner;
SELECT owner, object_type, COUNT(*) FROM dba_objects
WHERE owner NOT IN ('SYS','SYSTEM','RDSADMIN') GROUP BY owner, object_type ORDER BY 1,2;
```

**SQL Server:**
```sql
-- Per-DB size
SELECT DB_NAME(database_id) AS db, SUM(size)*8/1024 AS size_mb FROM sys.master_files GROUP BY database_id;
-- Object counts + per-table rows (run in the target DB context)
SELECT type_desc, COUNT(*) FROM sys.objects GROUP BY type_desc ORDER BY type_desc;
```

### Throughput Estimation (run BEFORE choosing a method)

A method is only viable if the bulk data can physically move within the transfer window.
Estimate the over-the-wire transfer time from DB size and usable bandwidth:

```
estimated_hours = db_size_gb / (bandwidth_mbps * 0.125 * 0.7)
```

- `* 0.125` converts Mbps → MB/s (8 bits per byte).
- `* 0.7` is a 70% real-world efficiency factor (TCP overhead, encryption, contention, restart
  retries). For a clean same-region 10 GbE path you can use 0.8; for busy shared internet egress
  use 0.5.

**Worked examples** (usable bandwidth, not rated link speed):

| DB size | Usable bandwidth | Estimated transfer | Implication |
|---------|------------------|--------------------|-------------|
| 50 GB | 1 Gbps (≈940 Mbps usable) | ~0.2 hr (~12 min) | Online copy trivially fits any window |
| 500 GB | 200 Mbps (typical site VPN) | ~10 hr | Needs an overnight window or CDC catch-up |
| 2 TB | 200 Mbps | ~40 hr | Wire transfer infeasible in a normal window → **Snow Family** |
| 2 TB | 1 Gbps Direct Connect | ~8 hr | Feasible with DX + CDC; without DX, use Snow |
| 10 TB | 500 Mbps | ~127 hr (>5 days) | **Snow Family mandatory** |

**Decision rule:**

1. Compute `estimated_hours`.
2. If `estimated_hours` ≤ transfer window → online method (dump/restore, XtraBackup+S3, DMS) is fine.
3. If `estimated_hours` > transfer window **and** source is on-prem/other-cloud **and** size > 1 TB
   → **flag it** and route to the offline-seed branch below. Do not silently pick a method that
   can't finish in time.
4. If only modestly over window → DMS Full Load + CDC: the bulk full-load can run for days while
   the app stays up, and CDC closes the gap — the *downtime* is just the final CDC drain, not the
   whole transfer.

### Offline-Seed Branch — Snow Family / DataSync (on-prem + low bandwidth + > 1 TB)

When the wire can't carry the data in time, seed Aurora/RDS from a physical/offline copy, then
catch up the delta with CDC:

| Condition | Approach |
|-----------|----------|
| > 1 TB, on-prem, bandwidth-bound, hard cutover deadline | **AWS Snowball Edge**: export dump/XtraBackup to the device → ship to AWS → load into S3 → `restore-db-cluster-from-s3` (MySQL) or import → then **DMS CDC** from on-prem to close the delta accumulated since the export LSN/binlog position. |
| 100 GB – 1 TB, on-prem, slow but no hard deadline | **AWS DataSync** over DX/VPN to land the dump in S3 (managed, resumable, checksummed), then restore + CDC catch-up. |
| Continuous/repeated file sync from on-prem | **DataSync** scheduled tasks. |

**Critical for Snow + CDC**: record the source's binlog file+position (MySQL) or LSN/replication
slot (PostgreSQL) **at the moment the offline export is taken**, so the subsequent DMS CDC task
starts exactly from that point. Mismatched start position = duplicate or missing rows.

---

## Phase 3: Migration Method Selection

**This is a decision the user must make (or approve).** Present the options with trade-offs.

### Decision Input — Ask the User

1. **Homogeneous or heterogeneous?** Same engine family (MySQL→MySQL/Aurora MySQL, PostgreSQL→PostgreSQL/Aurora PG, MariaDB→MariaDB, **Oracle→RDS Oracle, SQL Server→RDS SQL Server**) = **homogeneous**, stay in this skill (Phases 4–9). Engine family *changes* (Oracle/SQL Server/Tibero/CUBRID → Aurora/PostgreSQL/MySQL, MySQL→PostgreSQL) = **heterogeneous**, go to [references/heterogeneous-migration.md](references/heterogeneous-migration.md) first (schema/code conversion), then return for cutover.
2. **What is your source?** EC2 MySQL / EC2 MariaDB / EC2 PostgreSQL / EC2/on-prem Oracle / EC2/on-prem SQL Server / RDS MySQL / RDS MariaDB / RDS PostgreSQL / On-premises / Other cloud (Azure DB, Cloud SQL)
3. **What is your target?** Aurora MySQL / Aurora PostgreSQL / RDS MySQL / RDS MariaDB / RDS PostgreSQL / **RDS Oracle** / **RDS SQL Server**
4. **Downtime tolerance?** Zero / Seconds / Minutes / Hours (maintenance window)
5. **Database size?** < 10 GB / 10-100 GB / 100 GB - 1 TB / > 1 TB
6. **Network bandwidth (source → AWS)?** Same-VPC/region / Direct Connect (state Gbps) / VPN (state Mbps) / internet egress (state Mbps) — needed for the throughput check above.
7. **Do you need stored procedures, triggers, views migrated?** Yes / No / Don't know
8. **Migrating more than one database?** If yes, see "Multi-database scenarios" below.
9. **Do you have a preferred method?** (User may already know what they want)
10. **Number of databases on the source host?** (If >1, each migrates independently. Flag schema name collisions.)
11. **Can the application code be modified?** (Gates: app-side encryption, connection-string change vs Secrets Manager, dual-read capability)
12. **RPO (Recovery Point Objective)?** (Acceptable data loss if rollback needed. Zero RPO requires reverse replication.)
13. **Downstream CDC consumers?** (Debezium/Kafka, BI replicas, ETL jobs reading binlog — these break at cutover and need re-pointing.)
14. **Encryption requirement at creation time?** (KMS key type: AWS-managed vs CMK. Cannot change after cluster creation.)
15. **Cross-region or cross-account?** (Routes to different networking/KMS/DMS setup.)

### Method Decision Matrix

> **Homogeneous only.** This matrix is for same-engine-family migrations, including **Oracle → RDS
> Oracle** (rows 13–15) and **SQL Server → RDS SQL Server** (rows 16–18). If the engine family
> *changes* (Oracle/SQL Server/Tibero/CUBRID → Aurora/PostgreSQL/MySQL, or MySQL ↔ PostgreSQL), stop
> and use [references/heterogeneous-migration.md](references/heterogeneous-migration.md) — you need
> schema/code conversion (SCT / DMS Schema Conversion / Babelfish) before any data move.
>
> **Deterministic read order.** Rows are evaluated **top to bottom; take the FIRST row whose
> Source + Target + Size + Downtime + Bandwidth all match.** Conditions are mutually exclusive
> within a source, so exactly one row applies. "Bulk transfer fits window?" refers to the
> Phase 2 throughput estimate (`estimated_hours` vs transfer window).

| # | Source | Target | Size | Downtime tolerance | Bulk transfer fits window? | **Recommended Method** | Why |
|---|--------|--------|------|--------------------|----------------------------|------------------------|-----|
| 1 | RDS MySQL | Aurora MySQL | Any | < 1 min | n/a (same region) | **Aurora Read Replica promotion** | Built-in, seconds of downtime, migrates everything |
| 2 | RDS MySQL **or** RDS MariaDB | RDS MySQL / RDS MariaDB / Aurora MySQL | Any | < 1 min | n/a | **Blue/Green Deployment** | Managed switchover w/ guardrails (MySQL↔Aurora MySQL cross-engine supported) |
| 3 | RDS PostgreSQL | Aurora PostgreSQL | Any | < 1 min | n/a | **Aurora Read Replica promotion** | Built-in |
| 4 | EC2/on-prem MySQL or MariaDB | Aurora MySQL / RDS MySQL / RDS MariaDB | < 10 GB | Yes (< 1 hr) | Yes | **mysqldump** (`--routines --triggers --events`) | Simplest, migrates ALL objects |
| 5 | EC2/on-prem MySQL or MariaDB | Aurora MySQL / RDS MySQL / RDS MariaDB | 10 GB – 1 TB | Yes (hours) | Yes | **Percona XtraBackup + S3** | 3–7× faster than DMS, physical copy incl. all objects |
| 6 | EC2/on-prem MySQL or MariaDB | Aurora MySQL / RDS MySQL / RDS MariaDB | > 1 TB | Minimal (minutes) | Yes | **XtraBackup + S3 seed → binlog/DMS CDC catch-up** | Fast physical bulk, then drain delta; cutover = final drain |
| 7 | EC2/on-prem MySQL or MariaDB | Aurora MySQL / RDS MySQL / RDS MariaDB | Any | No (zero/seconds) | Yes | **DMS Full Load + CDC** | Only near-zero-downtime path from EC2/on-prem (see schema-objects note) |
| 8 | On-prem MySQL/MariaDB/PostgreSQL | Aurora / RDS (same family) | > 1 TB | Any | **No** (bandwidth-bound) | **Snow Family seed + DMS CDC** (or DataSync for 100 GB–1 TB) | Wire can't carry it in time — see Offline-Seed Branch |
| 9 | EC2/on-prem PostgreSQL | Aurora PostgreSQL / RDS PostgreSQL | < 10 GB | Yes | Yes | **pg_dump / pg_restore** (`-Fd -j`) | Complete (all objects), simple, parallel |
| 10 | EC2/on-prem PostgreSQL | Aurora PostgreSQL / RDS PostgreSQL | 10 GB – 1 TB | Yes (hours) | Yes | **pg_dump/restore (parallel)** | No physical-backup S3 path for PG; parallel dump is the bulk tool |
| 11 | EC2/on-prem PostgreSQL | Aurora PostgreSQL / RDS PostgreSQL | Any | No (zero/seconds) | Yes | **Native logical replication** (preferred) or **DMS CDC** | PG-native handles more DDL; DMS simpler to operate |
| 12 | Other cloud (Azure DB for MySQL/PostgreSQL, Cloud SQL) | Aurora / RDS (same family) | Any | No (near-zero) | per estimate | **DMS Full Load + CDC** | No native cross-cloud replica; DMS connects over public/peered endpoint |
| 13 | EC2/on-prem **Oracle** | **RDS Oracle** | < 1 TB | Yes (hours) | Yes | **Oracle Data Pump** (schema/table mode, via S3 integration) | AWS-recommended logical method; migrates schema + data. No FULL mode. |
| 14 | EC2/on-prem **Oracle** (EE) | **RDS Oracle** (EE) | > 1 TB | Minimal (minutes) | Yes | **Transportable tablespaces (XTTS via RMAN)** → optional DMS CDC catch-up | Physical, fastest for very large EE DBs; EE-only, no encrypted tablespaces, source not 11g. |
| 15 | EC2/on-prem **Oracle** | **RDS Oracle** | Any | No (zero/near-zero) | Yes | **Data Pump bulk load → AWS DMS CDC from recorded SCN** (or GoldenGate) | Only near-zero-downtime path; Data Pump seeds, DMS/GoldenGate drains delta. |
| 16 | EC2/on-prem **SQL Server** | **RDS SQL Server** | Any | Yes (hours) | Yes | **Native backup/restore** (.bak via S3, `rds_restore_database`) | AWS-recommended lift-and-shift; migrates the user DB. Logins/jobs separate (Phase 6). |
| 17 | EC2/on-prem **SQL Server** | **RDS SQL Server** | > 1 TB | Minimal (minutes) | Yes | **Native FULL + DIFFERENTIAL + LOG** (restore WITH NORECOVERY, final log WITH RECOVERY at cutover) | Shrinks cutover to tail-log restore. Source must be FULL recovery model. |
| 18 | EC2/on-prem **SQL Server** | **RDS SQL Server** | Any | No (zero/near-zero) | Yes | **AWS DMS Full Load + CDC** | Near-zero downtime; DMS moves data only — migrate logins/jobs/perms separately. |

**Notes on overlap resolution:**
- Rows 13–15 (Oracle): prefer **Data Pump** (row 13) for the common downtime-OK case; **XTTS** (row 14) only when EE + very large + no encrypted tablespaces + source ≥ 12c; **DMS/GoldenGate CDC** (row 15) when downtime must be near-zero. RMAN whole-DB physical restore is **NOT** supported into managed RDS Oracle (EC2/RDS Custom only).
- Rows 16–18 (SQL Server): prefer **native backup/restore** (row 16); use **full+diff+log** (row 17) to minimize the cutover window; **DMS** (row 18) only for near-zero downtime. Log Shipping, Replication, and `RESTORE FROM DISK` are not available on RDS.
- Rows 1 vs 2 for RDS MySQL → Aurora MySQL: prefer **Read Replica promotion** (row 1) for the
  lowest downtime; choose **Blue/Green** (row 2) when you want staged validation + one-command
  switchover, or when also doing a version upgrade.
- Snapshot migration (RDS-source, minutes of downtime) is a fallback only when Read Replica /
  Blue/Green are unavailable for the engine version — not a first choice, so it's omitted from the
  primary tree.
- The old "RDS Console Auto-Migrate / Any-Any" catch-all has been **removed**: it is just a console
  wrapper around DMS and applied to scenarios it doesn't fit (on-prem, >1 TB, heterogeneous). Use the
  specific row above; if you want the console UX, that maps to row 7/12 (DMS) behind the scenes.

### Binlog State Gate (MySQL/MariaDB — check BEFORE committing to a method)

Every near-zero-downtime path (DMS CDC, binlog replication, reverse replication for lossless rollback) **requires `log_bin=ON` with `binlog_format=ROW`** on the source. Carry the Phase 2 `SHOW VARIABLES LIKE 'log_bin'` result *directly* into method selection — do not assume CDC is available:

| Source `log_bin` | DB size | Recommendation |
|------------------|---------|----------------|
| **ON** (`binlog_format=ROW`) | Any | CDC methods available — matrix rows 6–8 (XtraBackup+CDC, DMS) are on the table. |
| **OFF** | Small (rough guide: **< ~1–2 GB**, fits a brief maintenance window per the Phase 2 throughput estimate) | **Prefer a brief dump cutover** (mysqldump → import → repoint app). Enabling binlog requires a **source restart = downtime anyway**; for a tiny DB the one-shot dump cutover causes *less total disruption* than "restart to enable binlog, then stand up DMS." |
| **OFF** | Large | You must **enable binlog first** (`log_bin=ON`, `binlog_format=ROW`, adequate `binlog retention`) — and **acknowledge the restart cost** with the user — before any CDC method. There is no zero-downtime path while binlog is off. |

> **The contradiction to surface explicitly:** "zero-downtime" and `log_bin=OFF` are mutually exclusive until you take the restart to enable binlog. State this trade-off to the user; for small DBs the honest answer is often a 1–2 minute dump cutover, not a CDC project. This also means **reverse replication may be impossible** — see Phase 8 "When reverse replication is NOT possible."

### Multi-Database Scenarios

When consolidating or moving more than one database:

- **Migrate sequentially, not in parallel**, unless each DB has its own DMS replication instance
  and target — a shared DMS instance saturates and CDC latency climbs across all tasks.
- **Schema/name collisions**: two source DBs with the same schema name cannot co-locate on one
  target without renaming. Decide up-front: separate RDS/Aurora clusters, or rename schemas via DMS
  table-mapping transformation rules (`rename` actions).
- **Cross-database queries / FKs** between the source DBs block consolidation onto separate clusters
  — those joins must move to the app layer or the DBs must land on the same cluster.
- **Shared users/grants**: migrate the global `mysql.user` / `pg_roles` grants once, deduplicated,
  not per-DB (each method's schema-objects step omits grants — see Phase 6).
- **Sequence the cutovers**: cut over the least-dependent DB first; keep reverse replication
  (Phase 8) running per-DB so each can roll back independently.

### Method Summary

| Method | Migrates Schema Objects? | Downtime | Complexity | Best Size Range |
|--------|------------------------|----------|-----------|-----------------|
| mysqldump / pg_dump | ✅ ALL (procs, triggers, views, events) | Minutes-hours | Low | < 50 GB |
| Percona XtraBackup + S3 | ✅ ALL (physical copy) | Minutes-hours | Medium | 10 GB - multi-TB |
| DMS Full Load + CDC | ❌ Data only (no procs/triggers/views) | Near-zero | Medium-High | Any size |
| Aurora Read Replica | ✅ ALL | Seconds | Low | Any (RDS source only) |
| Snapshot migration | ✅ ALL | Minutes | Low | Any (RDS source only) |
| Binlog replication | ❌ Data only | Near-zero | High | Any (MySQL/MariaDB) |
| S3 import (LOAD DATA FROM S3) | ❌ Data only | Depends on load time | Medium | Flat file import |
| PG logical replication | ❌ Data + DDL (no procs/grants) | Near-zero | Medium | Any (PG 10+) |
| Blue/Green Deployment | ✅ ALL | < 1 minute | Low | Any (RDS/Aurora source) |
| Oracle Data Pump (schema/table) | ✅ ALL in-schema objects (procs, triggers, views, sequences); NOT SYS-owned/Scheduler | Minutes-hours | Medium | < 1 TB (Oracle→RDS Oracle) |
| Oracle transportable tablespaces (XTTS) | ❌ Data/tablespaces only (recreate PL/SQL, views, users via metadata) | Minimal | High | > 1 TB EE (Oracle→RDS Oracle) |
| SQL Server native backup/restore | ✅ User DB objects; NOT logins/Agent jobs/linked servers (server-level) | Minutes-hours | Low | Any ≤ 64 TiB (SQL Server→RDS SQL Server) |

### Critical Note on Schema Objects

**DMS, binlog replication, and S3 import do NOT migrate:**
- Stored procedures, functions
- Triggers
- Views
- Events (MySQL)
- Sequences (PostgreSQL)
- User grants/permissions
- Custom data types (PostgreSQL)

**If user needs these migrated → must use dump/restore for schema + chosen method for data**, or use a method that does physical copy (XtraBackup, snapshot, Read Replica).

### Edge-Case Scenarios

**Multi-Database**: If multiple databases on one host, migrate sequentially. Each is a separate skill invocation. Check for cross-database queries (db1.table JOIN db2.table) — these break when databases are on separate hosts. Resolution: consolidate to one Aurora cluster with multiple schemas, or refactor the queries to use application-level joins.

**Cross-Region**: Requires DMS with cross-region replication. Set up VPC peering or Transit Gateway between source and target regions. Use region-specific KMS keys (encryption keys don't cross regions). Consider Aurora Global Database as the target for ongoing cross-region reads post-migration.

**Cross-Account**: Source and target in different AWS accounts. DMS endpoints use cross-account VPC peering plus IAM roles for access. KMS key policy must grant the target account access for encrypted snapshots. Snapshot sharing requires explicit account grant via modify-db-snapshot-attribute.

**>10 TB or Low Bandwidth (on-prem)**: If estimated transfer time exceeds 7 days, use AWS Snow Family (Snowball Edge) or DataSync for the baseline data. Critical: capture the binlog position or PostgreSQL LSN at the exact point of data export. After the Snow device is loaded into AWS and data is in S3, restore to Aurora and set up CDC replication from the captured position to catch up with changes that occurred during transit.

---

## Phase 4: Target Provisioning

### Aurora vs RDS Selection

| Factor | Aurora | RDS |
|--------|--------|-----|
| Availability | 99.99% (6 copies across 3 AZs) | 99.95% (Multi-AZ synchronous standby) |
| Auto-scaling storage | ✅ Up to 128 TB | ❌ Must provision EBS |
| Read replicas | 15 (single reader endpoint) | 15 (individual endpoints) |
| Failover time | < 30 seconds | 60-120 seconds |
| Serverless option | ✅ Aurora Serverless v2 | ❌ |
| Cost (minimum) | ~$60/mo (db.t4g.medium) | ~$13/mo (db.t4g.micro) |
| Global Database | ✅ Cross-region < 1s lag | ❌ |
| Backtrack (MySQL) | ✅ Point-in-time rewind | ❌ |

### Instance Sizing During Migration

Size LARGER during migration for import throughput, then scale down:

| DB Size | Migration Instance | Steady-State Instance |
|---------|-------------------|----------------------|
| < 50 GB | db.r6g.large | db.r6g.medium or Serverless v2 |
| 50-500 GB | db.r6g.xlarge | db.r6g.large |
| 500 GB - 2 TB | db.r6g.2xlarge | db.r6g.xlarge |
| > 2 TB | db.r6g.4xlarge | db.r6g.2xlarge |

### Oracle / SQL Server Target Provisioning — Settings Fixed at Creation

These cannot be changed after the instance is created — get them right up front (they're a frequent cause of "recreate the whole instance" rework):

| Engine | Set-at-creation, immutable | CLI flag | Notes |
|--------|----------------------------|----------|-------|
| **RDS Oracle** | DB character set | `--character-set-name` | Default `AL32UTF8`. Match source (e.g. `KO16MSWIN949`). CDB DB charset is always `AL32UTF8` — set non-default on the PDB only. |
| **RDS Oracle** | NCHAR character set | `--nchar-character-set-name` (CLI v2) | `AL16UTF16` (default) or `UTF8`. |
| **RDS Oracle** | `DB_BLOCK_SIZE` | parameter group at create | Default 8 KB; cannot change later. |
| **RDS Oracle** | Edition + license model | `--engine oracle-ee`/`oracle-se2`, `--license-model` | LI = SE2 only; EE = BYOL only. |
| **RDS SQL Server** | Server/instance collation | `--collation` (via parameter or console) | DB/column collations ride in the `.bak`; the *instance* default can't change later. |
| **RDS SQL Server** | Edition + license model | `--engine sqlserver-ee`/`-se`/`-web`/`-ex`, `--license-model` | License-Included or BYOM (License Mobility + Software Assurance). |

**Option groups (required for the native paths):** RDS Oracle Data Pump via S3 needs the **`S3_INTEGRATION`** option + an IAM role on `S3_INTEGRATION`; RDS SQL Server native backup/restore needs the **`SQLSERVER_BACKUP_RESTORE`** option with an IAM role; TDE on either engine needs the **`TDE`** option (permanent + persistent — cannot be removed once attached). Set these on the option group before Phase 5.

### RDS Proxy on the Target (provision BEFORE cutover for minimal-downtime / future failover)

For minimal-downtime workloads, stand up **RDS Proxy in front of the target cluster** during provisioning — before Phase 8 — so the app connects through the proxy endpoint from the moment of cutover. The proxy does **not** remove the brief *initial* EC2→RDS cutover pause, but once in place it makes every subsequent failover/maintenance event a **< 1s reconnect with no app restart and no pool refresh** (the proxy holds client connections and re-points to the new writer), and it absorbs the reconnect storm when the app's pool refreshes at cutover.

```bash
# Create a proxy targeting the new cluster; creds come from Secrets Manager (the same secret you rotate at cutover)
aws rds create-db-proxy \
  --db-proxy-name your-app-proxy \
  --engine-family MYSQL \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:REGION:ACCT:secret:your-app/db-credentials","IAMAuth":"DISABLED"}]' \
  --role-arn arn:aws:iam::ACCT:role/rds-proxy-secrets-role \
  --vpc-subnet-ids subnet-aaa subnet-bbb \
  --require-tls
aws rds register-db-proxy-targets \
  --db-proxy-name your-app-proxy \
  --db-cluster-identifier your-aurora-cluster
```

Then point clients at the **proxy endpoint** (`your-app-proxy.proxy-xxxx.<region>.rds.amazonaws.com`) in Phase 7.5/8, not the cluster endpoint. PostgreSQL: use `--engine-family POSTGRESQL`. (RDS Proxy requires the target to be RDS/Aurora — it's a target-side construct, so it works for EC2→RDS even though the *source* can't be proxied.)

### TLS-Enforcement Gate (check the target parameter group BEFORE loading or cutover)

Compliance baselines (K-ISMS, PCI, internal hardening) commonly set **`require_secure_transport=ON`** (MySQL/MariaDB) or **`rds.force_ssl=1`** (PostgreSQL) on the target parameter group. When enforced, **every** connection must use TLS — and that breaks two things if you don't plan for it:

1. **The migration load tool** — `mysqldump`/`mysql` import and `psql` restore must connect with TLS, or the import fails at connect time.
2. **The application connector** — the app's JDBC/driver string must request TLS *and* must be a version that supports the parameter, or the app can't reconnect after cutover.

**Check first:**
```sql
SHOW VARIABLES LIKE 'require_secure_transport';   -- MySQL/MariaDB: should be ON if enforced
-- PostgreSQL: parameter rds.force_ssl = 1
```
Then verify both the load tool and the app connector are configured for TLS. Connector parameter reference:

| Connector | TLS parameter |
|-----------|---------------|
| `mysql` / `mariadb` CLI | `--ssl` (optionally `--ssl-ca=rds-combined-ca-bundle.pem` to verify the server cert) |
| Connector/J 8.x–9.x | `sslMode=REQUIRED` (use `VERIFY_CA` + `trustCertificateKeyStoreUrl` to validate the cert) |
| Connector/J 5.1.x | `useSSL=true&requireSSL=true` |
| Node `mysql2` | `ssl: { rejectUnauthorized: false }` (or `{ ca: fs.readFileSync('rds-combined-ca-bundle.pem') }` to verify) |
| Python `mysqlclient` | `ssl={'ca': '/path/to/rds-combined-ca-bundle.pem'}` |
| `psql` / libpq | `sslmode=require` (or `verify-ca` / `verify-full` with `sslrootcert=`) |

> **Watch the default mismatch.** Connector/J ≥ 8.0.13 defaults to `sslMode=PREFERRED` — it *tries* TLS but **silently falls back to plaintext if the server allowed it**. With `require_secure_transport=ON` the server refuses plaintext, so PREFERRED can still connect — but make the intent explicit with `sslMode=REQUIRED` so behavior doesn't depend on negotiation. The `rds-combined-ca-bundle.pem` is downloadable from the AWS docs (region-specific bundles also available).

---

## Phase 5: Execute Migration (Method-Specific)

### If mysqldump / pg_dump (small DBs, downtime OK)

```bash
# MySQL: Full export including all schema objects
mysqldump --single-transaction --routines --triggers --events \
  --set-gtid-purged=OFF --column-statistics=0 \
  -h $SOURCE_HOST -u admin -p your_db > full_dump.sql

# Import to Aurora
mysql -h $AURORA_ENDPOINT -u admin -p your_db < full_dump.sql
```

> **Logical dump = a clean major-version-upgrade opportunity.** Because mysqldump/pg_dump re-import the data from scratch, the target can be a **newer major version** than the source (e.g. MariaDB 10.5 → 10.11, MySQL 5.7 → 8.0, PostgreSQL 13 → 16) in the *same* migration — no separate upgrade project. Validate with `CHECKSUM TABLE` that bytes are identical across the version gap (they were 10.5→10.11 in the reference migration). **But version gaps introduce behavioral changes** — reserved-word additions, deprecated/removed functions, changed defaults (charset, sql_mode, auth plugin). Review **[references/version-upgrades.md](references/version-upgrades.md)** before choosing the target version. (Physical methods — XtraBackup, snapshot, Read Replica — cannot skip major versions; only logical dump can.)

```bash
# PostgreSQL: Parallel dump/restore
pg_dump -Fd -j 8 -h $SOURCE_HOST -U postgres -d your_db -f /backup/
pg_restore -Fd -j 8 -h $AURORA_ENDPOINT -U postgres -d your_db /backup/
```

### If Percona XtraBackup + S3 (large MySQL, physical)

```bash
# 1. Full backup
xtrabackup --backup --target-dir=/backup --user=admin --password=$PASS

# 2. Prepare backup
xtrabackup --prepare --target-dir=/backup

# 3. Upload to S3
aws s3 sync /backup/ s3://your-bucket/xtrabackup/ --sse aws:kms

# 4. Restore to Aurora (via console or CLI)
aws rds restore-db-cluster-from-s3 \
  --db-cluster-identifier your-aurora-cluster \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.07.1 \
  --s3-bucket-name your-bucket \
  --s3-prefix xtrabackup/ \
  --s3-ingestion-role-arn arn:aws:iam::ACCOUNT:role/aurora-s3-role \
  --source-engine mysql \
  --source-engine-version 8.0.36 \
  --master-username admin \
  --master-user-password $NEW_PASS
```

**Requirements:**
- Source: MySQL 5.7 (XtraBackup 2.4) or MySQL 8.0 (XtraBackup 8.0)
- `innodb_file_per_table` must be enabled
- NO encrypted tablespaces (TDE must be off)
- InnoDB page size must be 16 KB (default)

### If DMS Full Load + CDC (zero-downtime)

See [references/dms-best-practices.md](references/dms-best-practices.md) for complete DMS configuration including:
- Replication instance sizing
- Source/target endpoint configuration
- Task settings (Full Load + CDC, parallel load, batch apply, LOB handling)
- Table mappings
- Monitoring

**Key prerequisite for CDC:**
- MySQL/MariaDB: `binlog_format=ROW`, `binlog_row_image=FULL`, `log_bin=ON`
- PostgreSQL: `wal_level=logical`, available replication slots

### If Aurora Read Replica (RDS MySQL/PG → Aurora)

```bash
# From RDS MySQL → Aurora MySQL Read Replica
aws rds create-db-cluster \
  --db-cluster-identifier aurora-replica-cluster \
  --engine aurora-mysql \
  --replication-source-identifier arn:aws:rds:REGION:ACCOUNT:db:source-rds-instance

# Wait for replica to sync, then promote
aws rds promote-read-replica-db-cluster \
  --db-cluster-identifier aurora-replica-cluster
```

**Only works from RDS (not EC2 directly).** For EC2 → Aurora, use DMS or XtraBackup.

### If Blue/Green Deployment (RDS/Aurora in-place upgrade)

```bash
aws rds create-blue-green-deployment \
  --blue-green-deployment-name "migrate-to-aurora" \
  --source "arn:aws:rds:REGION:ACCOUNT:db:source-instance" \
  --target-engine-version "8.0.mysql_aurora.3.07.1"

# Wait for green to be AVAILABLE and synchronized, then:
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier $BGD_ID \
  --switchover-timeout 300
```

### If Oracle Data Pump (Oracle → RDS Oracle, primary method)

RDS Oracle has **no OS shell** — you cannot run `impdp`/`expdp` on the host. You either call **`DBMS_DATAPUMP`** from a SQL client, or run **`impdp` from a remote Oracle Instant Client** against the RDS endpoint. The dump moves to RDS via **S3 integration** (most common) or **`DBMS_FILE_TRANSFER` over a DB link**.

> **Schema/table mode only — never FULL mode**, and never import SYS/SYSTEM/RDSADMIN-owned objects: a full-mode import can damage the data dictionary. RDS does not grant SYS/SYSDBA.

**One-time setup — S3 integration:** attach an IAM policy (`s3:GetObject`, `s3:ListBucket`, `s3:PutObject`; add `s3:AbortMultipartUpload`+`s3:ListMultipartUploadParts` for ≥100 MB files; add `kms:Decrypt`/`Encrypt`/`GenerateDataKey`/`DescribeKey` for SSE-KMS buckets — bucket must be **same Region**, SSE-C not supported), associate the role for `S3_INTEGRATION`, and add the `S3_INTEGRATION` option to the option group:
```bash
aws rds add-role-to-db-instance --db-instance-identifier my-oracle-target \
  --feature-name S3_INTEGRATION --role-arn arn:aws:iam::ACCT:role/rds-s3-integration-role
aws rds add-option-to-option-group --option-group-name myoptiongroup \
  --options OptionName=S3_INTEGRATION,OptionVersion=1.0 --apply-immediately
```

```sql
-- 1. On TARGET (as master): create the schema + grants (schema mode)
CREATE USER schema_1 IDENTIFIED BY "<password>";
GRANT CREATE SESSION, RESOURCE TO schema_1;
ALTER USER schema_1 QUOTA UNLIMITED ON users;
```

```sql
-- 2. On SOURCE: export with DBMS_DATAPUMP (schema mode). EXCLUDE Scheduler objects
--    owned by system schemas (importing those into RDS is unsupported).
DECLARE v_hdnl NUMBER;
BEGIN
  v_hdnl := DBMS_DATAPUMP.OPEN(operation=>'EXPORT', job_mode=>'SCHEMA', job_name=>NULL);
  DBMS_DATAPUMP.ADD_FILE(v_hdnl,'sample.dmp','DATA_PUMP_DIR',NULL,dbms_datapump.ku$_file_type_dump_file);
  DBMS_DATAPUMP.ADD_FILE(v_hdnl,'sample_exp.log','DATA_PUMP_DIR',NULL,dbms_datapump.ku$_file_type_log_file);
  DBMS_DATAPUMP.METADATA_FILTER(v_hdnl,'SCHEMA_EXPR','IN (''SCHEMA_1'')');
  DBMS_DATAPUMP.START_JOB(v_hdnl);
END;
/
-- (expdp equivalent: expdp user/pwd@src DIRECTORY=DATA_PUMP_DIR DUMPFILE=sample.dmp SCHEMAS=SCHEMA_1 PARALLEL=4)
-- For a TDE source, export with ENCRYPTION_MODE=PASSWORD (TRANSPARENT mode is NOT supported by RDS).
-- Each S3 object must be ≤ 5 TiB — use PARALLEL to split larger dumps into multiple files.
```

```sql
-- 3. Upload dump to S3 (from source, if source is also RDS; else use `aws s3 cp` from the OS)
SELECT rdsadmin.rdsadmin_s3_tasks.upload_to_s3(
  p_bucket_name=>'amzn-s3-demo-bucket', p_directory_name=>'DATA_PUMP_DIR') AS task_id FROM dual;

-- 4. On TARGET (as master): download dump from S3 into DATA_PUMP_DIR (async; returns a task id)
SELECT rdsadmin.rdsadmin_s3_tasks.download_from_s3(
  p_bucket_name=>'amzn-s3-demo-bucket', p_directory_name=>'DATA_PUMP_DIR') AS task_id FROM dual;
-- confirm the file landed
SELECT * FROM TABLE(rdsadmin.rds_file_util.listdir('DATA_PUMP_DIR')) ORDER BY mtime;
```

```sql
-- 5. On TARGET (as master): import via DBMS_DATAPUMP. Add METADATA_REMAP for tablespace/schema
--    remap; set TABLE_EXISTS_ACTION=>'REPLACE' to re-run.
DECLARE v_hdnl NUMBER;
BEGIN
  v_hdnl := DBMS_DATAPUMP.OPEN(operation=>'IMPORT', job_mode=>'SCHEMA', job_name=>NULL);
  DBMS_DATAPUMP.ADD_FILE(v_hdnl,'sample.dmp','DATA_PUMP_DIR',NULL,dbms_datapump.ku$_file_type_dump_file);
  DBMS_DATAPUMP.ADD_FILE(v_hdnl,'sample_imp.log','DATA_PUMP_DIR',NULL,dbms_datapump.ku$_file_type_log_file);
  DBMS_DATAPUMP.METADATA_FILTER(v_hdnl,'SCHEMA_EXPR','IN (''SCHEMA_1'')');
  DBMS_DATAPUMP.START_JOB(v_hdnl);
END;
/
```

```bash
# 5-alt. Or run impdp from a REMOTE Instant Client (bastion/EC2) against the RDS endpoint:
impdp admin@//RDS-ENDPOINT:1521/DBNAME \
  directory=DATA_PUMP_DIR dumpfile=sample.dmp logfile=sample_imp.log \
  schemas=SCHEMA_1 table_exists_action=replace
```

```sql
-- 6. Cleanup — dump files are NOT auto-purged and consume the same EBS volume as datafiles
EXEC UTL_FILE.FREMOVE('DATA_PUMP_DIR','sample.dmp');
```

**Alternative transfer (no S3): `DBMS_FILE_TRANSFER` over a DB link** — create a DB link from source to the RDS endpoint, then `DBMS_FILE_TRANSFER.PUT_FILE(...)` to push `sample.dmp` into the target `DATA_PUMP_DIR`; import as in step 5. Requires VPC routing + security-group ingress between source and target.

**Best practice:** transfer the dump → take a **DB snapshot** → test the import. If objects get invalidated, delete and recreate from the snapshot (the staged dump is included).

**Transportable tablespaces (XTTS, very large EE DBs):** use the dedicated `rdsadmin.rdsadmin_transport_util` package (not the regular impdp path). EE-only, source ≥ 12c, Linux only, target release ≥ source, **no encrypted tablespaces**, cannot transport `SYSTEM`/`SYSAUX` or non-data objects (recreate PL/SQL/views/users/sequences via Data Pump metadata), and the instance must have no read replicas. S3 file limit 5 TiB (EFS recommended for larger). See [references/aws-official-migration-methods.md](references/aws-official-migration-methods.md).

### If SQL Server Native Backup/Restore (SQL Server → RDS SQL Server, primary method)

RDS SQL Server has **no OS access** and **no `RESTORE FROM DISK`** — you restore a `.bak` staged in **S3** via the `msdb.dbo.rds_*` procedures, enabled by the **`SQLSERVER_BACKUP_RESTORE`** option.

**One-time setup:** create an S3 bucket in the **same Region**, an IAM role (trust `rds.amazonaws.com`, scoped with `aws:SourceArn` for the DB instance + option group), and add the option:
```bash
aws rds add-option-to-option-group --apply-immediately --option-group-name mybackupgroup \
  --options "OptionName=SQLSERVER_BACKUP_RESTORE,OptionSettings=[{Name=IAM_ROLE_ARN,Value=arn:aws:iam::ACCT:role/rds-backup-restore-role}]"
aws rds modify-db-instance --db-instance-identifier mydbinstance \
  --option-group-name mybackupgroup --apply-immediately   # no restart required
```
The IAM permissions policy needs `s3:ListBucket`,`s3:GetBucketLocation` on the bucket and `s3:GetObject`,`s3:PutObject`,`s3:ListMultipartUploadParts`,`s3:AbortMultipartUpload`,`s3:GetObjectAttributes` on `bucket/*`; add `kms:DescribeKey`/`GenerateDataKey`/`Encrypt`/`Decrypt` on a **symmetric** key for encrypted backups.

```sql
-- 1. On SOURCE: take a native backup, then upload .bak to S3 (aws s3 cp from the source host)
BACKUP DATABASE mydatabase TO DISK = 'D:\backups\mydb_full.bak' WITH INIT, FORMAT, COMPRESSION;
```
```bash
aws s3 cp D:\backups\mydb_full.bak s3://my-bucket/sqlbackups/mydb_full.bak
```
```sql
-- 2. On RDS: restore (single file → DB comes online; @with_norecovery defaults to 0 for FULL)
exec msdb.dbo.rds_restore_database
  @restore_db_name='mydatabase',
  @s3_arn_to_restore_from='arn:aws:s3:::my-bucket/sqlbackups/mydb_full.bak';

-- 3. Monitor (status refreshes ~every 5%; history kept 36 days)
exec msdb.dbo.rds_task_status @db_name='mydatabase';
-- cancel:  exec msdb.dbo.rds_cancel_task @task_id=<n>;   (cannot cancel FINISH_RESTORE)
```

**Multifile backup (large DBs, ≤10 files, parallel throughput):** the `*` is expanded to `1-of-N`, etc.:
```sql
exec msdb.dbo.rds_backup_database @source_db_name='mydatabase',
  @s3_arn_to_backup_to='arn:aws:s3:::my-bucket/out/backup*.bak',
  @number_of_files=4, @max_transfer_size=4194304, @buffer_count=10;
-- restore by giving the common prefix + '*'
exec msdb.dbo.rds_restore_database @restore_db_name='mydatabase',
  @s3_arn_to_restore_from='arn:aws:s3:::my-bucket/out/backup*';
```

**Minimal-downtime sequence (FULL + DIFFERENTIAL + LOG)** — source must be in **FULL recovery model**. Restore the big backups ahead of time `WITH NORECOVERY`, apply the final log at cutover `WITH RECOVERY`:
```sql
exec msdb.dbo.rds_restore_database @restore_db_name='mydatabase',
  @s3_arn_to_restore_from='arn:aws:s3:::my-bucket/mydb_full.bak', @type='FULL', @with_norecovery=1;
exec msdb.dbo.rds_restore_database @restore_db_name='mydatabase',
  @s3_arn_to_restore_from='arn:aws:s3:::my-bucket/mydb_diff.bak', @type='DIFFERENTIAL', @with_norecovery=1;
exec msdb.dbo.rds_restore_log @restore_db_name='mydatabase',
  @s3_arn_to_restore_from='arn:aws:s3:::my-bucket/mydb_log1.trn', @with_norecovery=1;
-- final log at cutover brings the DB online (rds_restore_log defaults to NORECOVERY=1, so set 0)
exec msdb.dbo.rds_restore_log @restore_db_name='mydatabase',
  @s3_arn_to_restore_from='arn:aws:s3:::my-bucket/mydb_logN.trn', @with_norecovery=0;
-- or, if the last task was left NORECOVERY:
exec msdb.dbo.rds_finish_restore @db_name='mydatabase';
```
`rds_restore_log` supports `@stopat='2026-06-04 03:57:09'` for point-in-time. Drop a stuck restore: `exec msdb.dbo.rds_drop_database @db_name='mydatabase';`

**Constraints:** S3 bucket same Region as instance; cannot restore over an existing DB name; 5 TB per file, native restore up to 64 TiB (Express 10 GB); up to 2 concurrent tasks; **cannot back up to a `.bak` from RDS for log shipping (no native log backups from RDS)**; a `.bak` from a *higher* engine version won't restore; FILESTREAM filegroups rejected; Multi-AZ native restore requires FULL recovery model; not supported with cross-Region read replicas; KMS must be symmetric; procedures can't run inside a transaction. **Logins, SQL Agent jobs, and linked servers are NOT in a user-DB `.bak`** — migrate them separately (Phase 6).

---

## Phase 6: Schema Object Migration (If Method Doesn't Include Them)

If you used DMS, binlog replication, or S3 import — schema objects must be migrated separately:

```bash
# MySQL: Export schema objects only (no data)
mysqldump --routines --triggers --events --no-data --no-create-info \
  -h $SOURCE -u admin -p your_db > schema_objects.sql

# Remove DEFINER clauses (they break on Aurora)
sed -i 's/DEFINER=[^*]*\*/\*/g' schema_objects.sql

# Import to target
mysql -h $TARGET -u admin -p your_db < schema_objects.sql
```

```bash
# PostgreSQL: Functions, triggers, views, types
pg_dump --schema-only --no-owner --no-privileges \
  -h $SOURCE -U postgres your_db | \
  grep -v 'COMMENT ON EXTENSION' > schema.sql

psql -h $TARGET -U postgres -d your_db -f schema.sql
```

### Oracle — Objects NOT Carried by a Schema-Mode Data Pump

Schema-mode Data Pump brings in-schema objects (procs, triggers, views, sequences). It does **not** bring SYS/SYSTEM-owned Scheduler jobs (intentionally excluded), nor will arbitrary directory objects / ACLs work as-is on RDS:
- **Scheduler jobs**: recreate **app-owned** `DBMS_SCHEDULER` jobs on the target (migrate any legacy `DBMS_JOB` to `DBMS_SCHEDULER`). Never recreate SYS/SYSTEM-owned jobs.
- **Network ACLs** (for `UTL_HTTP`/`UTL_SMTP`/`UTL_TCP`): re-grant with `DBMS_NETWORK_ACL_ADMIN` on the target and confirm VPC egress.
- **Database links**: recreate; they need VPC routing + security-group rules and updated TNS descriptors.
- **Directory objects / external tables / BFILE**: re-stage through RDS-managed directories (the master user lacks `CREATE ANY DIRECTORY`).

### SQL Server — Server-Level Objects (logins, Agent jobs, linked servers)

A user-DB `.bak` carries **database users** but not **server logins** → SQL-auth users are **orphaned** after restore (login SID mismatch). Fastest fix: recreate logins on RDS with the **same SID + HASHED password** so the orphan auto-resolves.

```sql
-- On SOURCE: generate CREATE LOGIN statements preserving hash + SID
SELECT 'CREATE LOGIN ' + QUOTENAME(p.name) +
  CASE WHEN p.type_desc='SQL_LOGIN'
    THEN ' WITH PASSWORD = ' + CONVERT(NVARCHAR(MAX),l.password_hash,1) +
         ' HASHED, SID = ' + CONVERT(NVARCHAR(MAX),p.sid,1) + ';'
    ELSE ' FROM WINDOWS;' END
FROM sys.server_principals p
LEFT JOIN sys.sql_logins l ON p.principal_id=l.principal_id
WHERE p.type_desc IN ('SQL_LOGIN','WINDOWS_LOGIN','WINDOWS_GROUP')
  AND p.name NOT LIKE '##%##' AND p.name <> 'sa'
  AND p.name NOT LIKE 'NT SERVICE%' AND p.name NOT LIKE 'NT AUTHORITY%';
```
```sql
-- On RDS, after restore: if a user is still orphaned, relink (preferred over deprecated sp_change_users_login)
USE [mydatabase];
EXEC sp_change_users_login 'Report';     -- list orphans
ALTER USER [appuser] WITH LOGIN = [appuser];
```
- **SQL Agent jobs** live in `msdb` (not importable) — script them out on the source and recreate (no CmdExec/PowerShell/replication steps, no email/alerts on RDS).
- **Linked servers** are server-level — recreate manually (Oracle OLEDB has a dedicated RDS option).
- **CLR**: `SAFE` assemblies only on ≤2016; not supported 2017+ — refactor.

### TDE-Encrypted Source — Bringing the Certificate / Re-encrypting

- **Oracle**: you **cannot import your own wallet**. Export with Data Pump `ENCRYPTION_MODE=PASSWORD`, import into a TDE-enabled target whose wallet **AWS generates** (`SELECT * FROM v$encryption_wallet;` to confirm). Create encrypted tablespaces normally: `CREATE TABLESPACE enc_ts ENCRYPTION USING 'AES256' DEFAULT STORAGE(ENCRYPT);`. The `TDE` option is permanent — to remove it you must export to a non-TDE instance.
- **SQL Server**: bring the **source TDE certificate** in first via `rds_restore_tde_certificate` (cert name must start with `UserTDECertificate_`). On the source, `BACKUP CERTIFICATE ... WITH PRIVATE KEY`, where the private-key password is the **plaintext** of a KMS data key (`aws kms generate-data-key --key-spec AES_256`); upload `.cer`/`.pvk` to S3 and tag the `.pvk` object with `x-amz-meta-rds-tde-pwd` = the KMS `CiphertextBlob`:
  ```sql
  EXECUTE msdb.dbo.rds_restore_tde_certificate
    @certificate_name='UserTDECertificate_mycert',
    @certificate_file_s3_arn='arn:aws:s3:::cert-bucket/tde-cert.cer',
    @private_key_file_s3_arn='arn:aws:s3:::cert-bucket/tde-key.pvk',
    @kms_password_key_arn='arn:aws:kms:us-west-2:ACCT:key/<key-id>';
  ```
  Then `rds_restore_database` the TDE `.bak`; RDS **auto-rekeys** the restored DB to an RDS-managed `RDSTDECertificate` before it becomes available. Constraints: both `SQLSERVER_BACKUP_RESTORE` + `TDE` options required; **TDE cert restore not supported on Multi-AZ** (do it Single-AZ, then convert); max 10 user certs; no cross-account KMS keys.

---

## Phase 6.5: Migration Rehearsal (Recommended Before Production)

Before executing against production, perform a full dry-run:

1. **Create a source clone**: Snapshot the source EC2, launch a clone in same VPC.
2. **Run the full migration against the clone** (all phases).
3. **Measure actual time**: Record duration of each phase.
4. **Validate cutover procedure**: Practice end-to-end.
5. **Verify rollback**: Test the rollback procedure works.
6. **Destroy the clone**: Delete all rehearsal resources.

This de-risks production by: confirming time estimates, catching permission/network/compatibility issues, giving team confidence, and providing a realistic timeline for stakeholders.

---

## Phase 7: Validation

### Required Checks

| Check | Method | When |
|-------|--------|------|
| Row count comparison | Query both source and target | After full load |
| DMS validation | Built-in (if using DMS) | Continuous during migration |
| Checksum (critical tables) | `CHECKSUM TABLE` or `pt-table-checksum` | Pre-cutover |
| Referential integrity | FK orphan queries on target | Pre-cutover |
| Schema object count | Compare proc/trigger/view counts | After schema import |
| Application smoke test (read-only) | Point test instance at target | Pre-cutover |
| Stored procedure execution | Run key procs on target | Pre-cutover |
| **Bidirectional cutover verification** | App health (`/actuator/health` db=UP) **AND** target processlist shows expected client IPs + conn count | Post-cutover (Phase 8 step 10) |

### Quick Validation Script

```bash
#!/bin/bash
echo "Table | Source | Target | Match"
for TABLE in $(mysql -h $SOURCE -u $USER -p$PASS -N -e \
  "SELECT table_name FROM information_schema.tables WHERE table_schema='$DB'"); do
  SRC=$(mysql -h $SOURCE -u $USER -p$PASS -N -e "SELECT COUNT(*) FROM $DB.$TABLE")
  TGT=$(mysql -h $TARGET -u $USER -p$PASS -N -e "SELECT COUNT(*) FROM $DB.$TABLE")
  [ "$SRC" = "$TGT" ] && M="✅" || M="❌ (diff: $((TGT-SRC)))"
  echo "$TABLE | $SRC | $TGT | $M"
done
```

### Application-Level Validation

Row counts and checksums prove the *bytes* moved; these checks prove the *application* still
behaves identically against the target. Run them pre-cutover, on BOTH source and target, and diff.

1. **Dual-read / shadow-traffic.** Configure the app to read from **both** source and target and
   compare results before trusting the target. Spring Boot: route via `AbstractRoutingDataSource`.
   Node.js: a read-through proxy that fans the read to both and diffs. Discrepancies surface
   collation, timezone, or replication-lag issues no row-count check would catch.

2. **Collation / sort-order check.** A different default collation silently reorders results and
   breaks `ORDER BY`-dependent pagination and `=`/`LIKE` matching:
   ```sql
   SELECT id, name FROM products ORDER BY name LIMIT 100;
   SHOW VARIABLES LIKE 'collation%';
   ```
   Run on BOTH, diff the results.

3. **AUTO_INCREMENT / sequence high-water-mark.** Confirm the target's next-value counter sits
   above the current max — otherwise the first inserts collide with existing PKs (see Phase 8 step 6):
   ```sql
   -- MySQL
   SELECT TABLE_NAME, AUTO_INCREMENT FROM information_schema.tables WHERE TABLE_SCHEMA = 'your_db';
   SELECT MAX(id) FROM orders;  -- Must be < AUTO_INCREMENT

   -- PostgreSQL
   SELECT schemaname, sequencename, last_value FROM pg_sequences;
   ```

4. **Aggregate fidelity.** Cheaper than a full checksum, catches truncated/duplicated rows and
   numeric-type drift:
   ```sql
   SELECT COUNT(*) AS cnt, SUM(total_amount) AS total, MIN(created_at) AS earliest, MAX(id) AS max_id FROM orders;
   ```
   Run on BOTH, compare.

5. **Timezone-shift detection.** A target with a different `time_zone` parameter shifts every
   TIMESTAMP — easy to miss until reports are off by hours:
   ```sql
   SELECT id, created_at FROM orders ORDER BY id DESC LIMIT 10;
   ```
   Any shift between source and target = timezone parameter mismatch (see Phase 1.2).

6. **Regression test suite.** Run your application's full test suite against the target database.
   If no test suite exists: for each major table, create 1 record, read it back, update it, and
   delete it — the minimum CRUD round-trip that proves the app's data layer works end-to-end.

### Post-Version-Upgrade Validation

> **Trigger — run this whole subsection ONLY when the source and target are different MAJOR
> versions** (e.g. MySQL 5.7 → 8.0, MySQL 8.0 → 8.4, MariaDB 10.x → 11.x, PostgreSQL 13 → 15). A
> logical dump can cross majors (Phase 5 note), but a version gap is a **behavioral-change surface**:
> the bytes can match (row counts + `CHECKSUM TABLE` pass) while the *engine behaves differently*. The
> checks above prove source↔target equivalence; these prove the workload survives the **version gap**
> itself. Read [references/version-upgrades.md](references/version-upgrades.md) for the exact
> per-version changes, then run the relevant checks below. Confirm the gap first — `SELECT VERSION();`
> on both — and record findings in `migration-plan.md`.

**1. Collation / sort-order ALGORITHM change (beyond the param check above).** The general collation
check compares the *variable*; a major upgrade can change the default collation *algorithm* itself —
MySQL 5.7 `utf8mb4_general_ci` vs 8.0 `utf8mb4_0900_ai_ci` sort and *compare* differently (accent/case
weighting) even for identical charset and data. Two risks: silent re-ordering, and **new
unique-key/PK collisions** (values that were distinct under the old collation become "equal").
```sql
-- run on BOTH; the new default collation may reorder rows even with the same charset
SELECT id, name FROM products ORDER BY name, id LIMIT 200;        -- diff the two outputs
-- what did each table/column actually resolve to after import?
SELECT TABLE_NAME, TABLE_COLLATION FROM information_schema.TABLES WHERE TABLE_SCHEMA='your_db';
SELECT TABLE_NAME, COLUMN_NAME, COLLATION_NAME FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA='your_db' AND COLLATION_NAME IS NOT NULL;
-- prove comparison semantics changed:
SELECT 'a' = 'A' COLLATE utf8mb4_0900_ai_ci AS ai_match;          -- 1 under the 8.0 default
-- new collision risk on a string unique key / PK — must return ZERO rows:
SELECT name, COUNT(*) c FROM products GROUP BY name HAVING c > 1;
```

**2. Auth-plugin compatibility (the app's real connector, not the `mysql` CLI).** 8.0 default became
`caching_sha2_password`; **8.4 ships `mysql_native_password` OFF at startup.** Old connectors
(Connector/J 5.1, libmysqlclient < 8.0, legacy PHP/Node drivers) then **fail to authenticate** even
though the import succeeded.
```sql
-- TARGET: which plugin did each account land on after import?
SELECT user, host, plugin FROM mysql.user;
```
```bash
# Authenticate from a host running the APP's actual connector version (the newer mysql CLI hides this)
mysql -h "$TARGET" -u appuser -p -e "SELECT CURRENT_USER(), CONNECTION_ID();"
```
Fix: upgrade the connector, **or** recreate the user on the legacy plugin
(`ALTER USER 'appuser'@'%' IDENTIFIED WITH mysql_native_password BY '…'`; on 8.4 first set
`mysql_native_password=ON` in the target parameter group — see version-upgrades.md MySQL 8.0→8.4).

**3. `sql_mode` strictness — catch queries that worked before but fail now.** A newer major defaults
to a stricter `sql_mode` (8.0 enables `ONLY_FULL_GROUP_BY`, `STRICT_TRANS_TABLES`, `NO_ZERO_DATE`,
`NO_ZERO_IN_DATE`). `SELECT`s and writes that the source tolerated now **error**.
```sql
SELECT @@GLOBAL.sql_mode;   -- run on SOURCE and TARGET, diff the flag set
```
Drive the app's real workload (the regression suite, item 6) under the target `sql_mode`, and probe
the known-risky shapes directly:
```sql
-- ONLY_FULL_GROUP_BY: a non-aggregated, non-grouped column now errors (1055) instead of returning arbitrary rows
SELECT customer_id, order_date, SUM(total) FROM orders GROUP BY customer_id;
-- NO_ZERO_DATE + STRICT: a zero/invalid date now errors instead of warning
INSERT INTO events (created_at) VALUES ('0000-00-00');
```
Also scan existing data that strict mode will reject on the *next* UPDATE:
```sql
SELECT COUNT(*) FROM events WHERE created_at = '0000-00-00' OR created_at IS NULL;
```
Goal is to fix the queries/data; only as a temporary bridge, drop the offending flag from the target
`sql_mode` parameter.

**4. Reserved-word collisions in the existing schema.** A new major reserves identifiers your schema
may already use (8.0: `RANK`, `ROW`, `ROWS`, `GROUPS`, `LEAD`, `LAG`, `OVER`, `CUME_DIST`,
`DENSE_RANK`, `FIRST_VALUE`, `NTILE`, `PERCENT_RANK`, `RECURSIVE`, `SYSTEM`; 8.4: `PARALLEL`,
`QUALIFY`, `TABLESAMPLE`, `MANUAL`; MariaDB 10.7: `ROW_NUMBER`, 11.6: `VECTOR`). The dump imports fine
(mysqldump back-quotes identifiers) — but **unquoted application SQL referencing those names breaks at
runtime.** Scan the migrated schema for collisions:
```sql
SELECT TABLE_NAME, COLUMN_NAME FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA='your_db'
  AND UPPER(COLUMN_NAME) IN ('RANK','ROW','ROWS','GROUPS','LEAD','LAG','OVER','CUME_DIST',
      'DENSE_RANK','FIRST_VALUE','LAST_VALUE','NTILE','PERCENT_RANK','RECURSIVE','SYSTEM',
      'PARALLEL','QUALIFY','TABLESAMPLE','MANUAL','VECTOR','ROW_NUMBER');
SELECT TABLE_NAME FROM information_schema.TABLES
WHERE TABLE_SCHEMA='your_db'
  AND UPPER(TABLE_NAME) IN ('RANK','ROW','ROWS','GROUPS','LEAD','LAG','OVER','CUME_DIST',
      'DENSE_RANK','FIRST_VALUE','LAST_VALUE','NTILE','PERCENT_RANK','RECURSIVE','SYSTEM',
      'PARALLEL','QUALIFY','TABLESAMPLE','MANUAL','VECTOR','ROW_NUMBER');
```
Any hit → `grep` the application for unquoted use of that identifier and add back-quotes, or rename
the column. (PostgreSQL: query `information_schema.columns` the same way against the new major's
reserved list; `SELECT quote_ident('row');` shows whether a name now needs quoting.)

**5. Deprecated/removed functions & syntax in stored routines, triggers, views.** A logical dump
*creates* routines, but a function or syntax **removed** in the new major fails at **call time, not
import time** — so a green import hides it. Pull every body, scan for removed constructs, then
actually execute each one:
```sql
SELECT ROUTINE_NAME, ROUTINE_TYPE, ROUTINE_DEFINITION FROM information_schema.ROUTINES
WHERE ROUTINE_SCHEMA='your_db';
SELECT TRIGGER_NAME, ACTION_STATEMENT FROM information_schema.TRIGGERS WHERE TRIGGER_SCHEMA='your_db';
SELECT TABLE_NAME, VIEW_DEFINITION FROM information_schema.VIEWS WHERE TABLE_SCHEMA='your_db';
```
Grep those bodies for things removed across the gap — e.g. 5.7→8.0: `PASSWORD()` (removed),
`GROUP BY … ASC/DESC` (removed), `ENCODE`/`DECODE`/`DES_ENCRYPT`/`DES_DECRYPT` (removed),
`SQL_CALC_FOUND_ROWS`/`FOUND_ROWS()` (deprecated). Then **exercise each routine** (the regression
suite, item 6, must drive every routine/trigger/view path — MySQL has no bulk recompile):
```sql
CALL your_proc(/* representative args */);
SELECT * FROM your_view LIMIT 1;
```
(Oracle's invalid-object scan + `UTL_RECOMP` is in the Oracle Validation block below; SQL Server: see
`sys.dm_os_performance_counters … 'Deprecated Features'` in version-upgrades.md, then re-run modules.)

**6. Default character-set change — verify data RENDERS, not just that counts match.** 5.7→8.0 default
charset went `latin1`→`utf8mb4`. Row counts and even `CHECKSUM TABLE` can pass while text is
**mojibake** if a `latin1` column's bytes were re-interpreted (double-encoded) during dump/reload.
Verify actual rendering of multibyte / Korean / emoji data:
```sql
-- run on BOTH with a utf8mb4 client connection so the CLI itself doesn't mangle the comparison:
--   mysql --default-character-set=utf8mb4 …
SELECT id, name, HEX(name) AS name_hex, LENGTH(name) AS bytes, CHAR_LENGTH(name) AS chars
FROM products WHERE name REGEXP '[^ -~]' LIMIT 50;     -- rows containing non-ASCII
SELECT T.TABLE_NAME, CCSA.CHARACTER_SET_NAME
FROM information_schema.TABLES T
JOIN information_schema.COLLATION_CHARACTER_SET_APPLICABILITY CCSA
  ON CCSA.COLLATION_NAME = T.TABLE_COLLATION
WHERE T.TABLE_SCHEMA='your_db';
```
`CHAR_LENGTH` (characters) **must match on both**; `bytes` differing is expected if encoding differs,
but the rendered glyphs must be identical. Garbled Korean/emoji = a charset bug in the dump step —
re-dump with `--default-character-set` matching the source's *true* storage charset and reload.

**7. Optimizer / query-plan regression.** A new major changes the cost model (MariaDB 11.0 time-based
optimizer; MySQL 8.0 histograms + new CTE/derived-table handling) and plans can **regress**. Baseline
`EXPLAIN` on the **source before cutover** and diff against the target.
```sql
-- SOURCE, pre-cutover: pull the hottest queries to baseline
SELECT DIGEST_TEXT, COUNT_STAR, SUM_TIMER_WAIT
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 20;
EXPLAIN FORMAT=JSON SELECT /* each top query */ … ;     -- capture access type, key, rows, cost
-- TARGET (after ANALYZE TABLE so stats are fresh — Phase 9): same queries, compare
EXPLAIN ANALYZE SELECT … ;                              -- MySQL 8.0.18+ / MariaDB 10.1+: real timing
```
A plan that drops an index, changes join order, or balloons `rows`/cost vs the baseline = regression
→ pin with an index/optimizer hint, or (MariaDB 11.x) tune the new cost variables
(`optimizer_disk_read_cost`, …). **PostgreSQL:** baseline with `EXPLAIN (ANALYZE, BUFFERS)` on both
sides and pull the hot list from `pg_stat_statements`. See
[references/version-upgrades.md](references/version-upgrades.md) "After the upgrade" item 4.

### Oracle Validation (Oracle → RDS Oracle)

```sql
-- Monitor the running Data Pump job
SELECT job_name, operation, job_mode, state FROM dba_datapump_jobs WHERE owner_name = USER;
-- Read the import log (no OS access — use the rdsadmin helper)
SELECT * FROM TABLE(rdsadmin.rds_file_util.read_text_file('DATA_PUMP_DIR','sample_imp.log'));
-- Object counts by type (compare source vs target)
SELECT object_type, COUNT(*) FROM dba_objects WHERE owner='SCHEMA_1' GROUP BY object_type ORDER BY 1;
-- Invalid objects — then recompile (utlrp.sql can't run; no shell)
SELECT object_name, object_type FROM dba_objects WHERE status='INVALID' AND owner='SCHEMA_1';
EXEC UTL_RECOMP.RECOMP_PARALLEL(4,'SCHEMA_1');   -- or DBMS_UTILITY.COMPILE_SCHEMA('SCHEMA_1')
-- Row counts (gather stats first, or COUNT(*) for exactness)
EXEC DBMS_STATS.GATHER_SCHEMA_STATS('SCHEMA_1');
SELECT table_name, num_rows FROM dba_tables WHERE owner='SCHEMA_1' ORDER BY table_name;
```
Common import errors: `ORA-39083` (object create failed — grant/tablespace; fix grants or `METADATA_REMAP`), `ORA-31693` (table data load failed — quota/space), `ORA-39166` (object not in dump / wrong schema).

### SQL Server Validation (SQL Server → RDS SQL Server)

```sql
-- Integrity (you have db_owner on RDS); or use the AWSSQLServer-DBCC SSM automation document
DBCC CHECKDB ('mydatabase') WITH NO_INFOMSGS, ALL_ERRORMSGS;
-- Object counts (compare source vs target)
SELECT type_desc, COUNT(*) FROM sys.objects GROUP BY type_desc ORDER BY type_desc;
-- Row counts per table
SELECT s.name, t.name, SUM(p.rows) AS row_count
FROM sys.tables t JOIN sys.schemas s ON t.schema_id=s.schema_id
JOIN sys.partitions p ON t.object_id=p.object_id AND p.index_id IN (0,1)
GROUP BY s.name, t.name ORDER BY 1,2;
-- Orphaned users (fix with ALTER USER ... WITH LOGIN — see Phase 6)
USE [mydatabase]; EXEC sp_change_users_login 'Report';
-- Compare logins source vs target
SELECT name, type_desc, sid FROM sys.server_principals
WHERE type IN ('S','U','G') AND name NOT LIKE '##%##' ORDER BY name;
-- TDE state (3 = encrypted)
SELECT DB_NAME(database_id) db, encryption_state FROM sys.dm_database_encryption_keys;
```

---

## Phase 7.5: Pre-cutover — Discover DB Clients (MANDATORY)

> **"You moved the DB, but if the application can't find the new DB, that's the biggest incident of all."** This is the **#1 operational risk** of a migration and the most-skipped step. Before changing anything at cutover, enumerate **EVERY** client that connects to the source DB. Miss one and you get **split data** (the missed client keeps writing to the old DB) or an **outage** (the missed client can't reach the new one). Record every client found here in `migration-plan.md`.

### Step 1 — Trace via Security Group

The source DB's security-group ingress reveals *who is allowed to connect*. Start there:
```bash
# Find the ingress sources (SGs/CIDRs) allowed to the DB port
aws ec2 describe-security-groups --group-ids <db-sg> \
  --query "SecurityGroups[].IpPermissions[?ToPort==\`3306\`].[UserIdGroupPairs[].GroupId, IpRanges[].CidrIp]"
```
For each allowed **source SG**, list every resource attached to it — EC2, ENIs (covers ECS tasks, Lambda-in-VPC, NLB targets), etc.:
```bash
aws ec2 describe-instances --filters Name=instance.group-id,Values=<client-sg> \
  --query "Reservations[].Instances[].[InstanceId,PrivateIpAddress,Tags[?Key=='Name'].Value|[0]]"
aws ec2 describe-network-interfaces --filters Name=group-id,Values=<client-sg> \
  --query "NetworkInterfaces[].[PrivateIpAddress,Description,InterfaceType]"
```
A descriptive ingress rule (e.g. *"Allow MySQL from Backend"*) is a strong hint — but verify against actual attached resources; descriptions drift.

### Step 2 — Inspect each client's connection config

The DB host/credentials can live in **many** places, and a later source **overrides** an earlier one. Check **all** of them, in override order, and use the **Connection Config Location Checklist**:

| Priority (highest wins) | Location | How to check |
|------------------------|----------|--------------|
| 1. **Process args** | Runtime flags override everything bundled | `ps -eo args \| grep -iE "jdbc:\|DB_HOST\|datasource\|--spring"` |
| 2. **Env files / env vars** | systemd `EnvironmentFile=`, container env, `.env` | `cat <EnvironmentFile>`; `systemctl show <svc> -p Environment` |
| 3. **systemd units** | `ExecStart` often carries hardcoded `--spring.datasource.url=...` | `grep -rn "jdbc:\|DB_HOST\|datasource" /etc/systemd/system/` |
| 4. **App config files** | `application*.properties` / `.yml`, `config.js/json`, `database.yml` | grep the deploy dir for the connection key |
| 5. **Secrets Manager** | The credential secret — **but does it even contain `host`?** | `aws secretsmanager get-secret-value --secret-id <id> --query SecretString` |
| 6. **Hardcoded IPs** | Last-resort sweep for the source private IP anywhere | `grep -rn "<source-private-ip>" /etc /opt /home /srv 2>/dev/null` |

> **Command-line args / env override bundled config.** A Spring Boot service launched with `--spring.datasource.url=jdbc:mysql://10.0.0.5:3306/db` **ignores** the `${DB_HOST}` in its bundled `application-prod.properties`. Changing the properties file alone will *not* repoint the app. **Find the highest-priority source and change that one** (often the systemd `ExecStart` line — back up the unit first: `cp backend.service backend.service.bak.$(date +%s)` — then `systemctl daemon-reload`).

> **Verify the secret actually contains `host`.** Many secrets hold only `username`/`password` (no `host`/`port`/`dbname`). If so, the "Secrets Manager rotation" cutover **will not repoint the app** — rotating the secret changes credentials but the host still comes from config/systemd. You must change the config/unit *and* (recommended) backfill `host`/`port`/`dbname`/`engine` into the secret so future rotation works.

> **ALWAYS use the RDS DNS endpoint, never a resolved IP.** Repoint clients to `your-db.xxxx.<region>.rds.amazonaws.com`, not the private IP it currently resolves to. **Multi-AZ failover keeps the DNS name but moves to a different standby IP** — a hardcoded IP breaks on the first failover. (The source itself having a hardcoded IP, as above, is exactly the anti-pattern to not carry forward.)

> **ORM auto-DDL check (do this here).** If a client uses an ORM with schema auto-management enabled — Spring/Hibernate `spring.jpa.hibernate.ddl-auto=update` (or `create`/`create-drop`), Rails `db:migrate` on boot, Django auto-migrate, Sequelize `sync({alter:true})` — its **first connection to the new DB may attempt schema modifications** and drift the freshly-migrated schema. For production migrations, set it to **`validate`** or **`none`** before cutover so the app verifies the schema instead of mutating it.

> **Connection-pool prep (do this here, for minimal-downtime cutovers).** For each client, pre-set the pool to recycle fast so a cutover repoint takes effect in seconds without a hard restart — HikariCP `maxLifetime=30000`, PgBouncer low `server_lifetime`, etc. The window-minimization rationale and the full per-pool table live in Phase 8 "Minimize the Write-Pause Window"; record the chosen values per client in `migration-plan.md`.

### Step 3 — Cross-check live connections from the DB side

SG rules show who is *allowed*; the processlist shows who is *actually connected right now*. Reconcile the two — an active client that isn't in your Step 1/2 inventory is a client you were about to miss:
```sql
-- MySQL/MariaDB
SELECT user, SUBSTRING_INDEX(host,':',1) AS client_ip, db, command, COUNT(*) AS conns
FROM information_schema.processlist GROUP BY user, client_ip, db, command;
-- PostgreSQL
SELECT usename, client_addr, datname, state, COUNT(*) FROM pg_stat_activity
GROUP BY usename, client_addr, datname, state;
```
Confirm every client IP/count matches what Steps 1–2 found. Investigate any unexpected IP before proceeding.

### Completion criterion

**EVERY discovered client is repointed to the new RDS DNS endpoint and verified connected** (see Phase 8 bidirectional verification). The migration is not "done" until the inventory in `migration-plan.md` is fully checked off — repointed *and* confirmed on both the app side and the DB-side processlist.

---

## Phase 8: Cutover

### Method Selection (Ask User)

| Method | Best For | Downtime |
|--------|----------|----------|
| **Secrets Manager rotation** | Apps using Secrets Manager for DB creds | 0-30s (pool refresh) |
| **DNS CNAME swap** (Route 53) | Apps using DNS hostname for DB | TTL (set to 60s before) |
| **Blue/Green switchover** | Already on RDS/Aurora | < 1 minute (managed) |
| **Application restart** | Simple apps, acceptable brief outage | Seconds-minutes |

### Minimize the Write-Pause Window

For "absolute minimal downtime" EC2 → RDS/Aurora, the cutover pause is the time between *freezing the source* and *the app writing to the target*. CDC catch-up (matrix rows 6–8: DMS Full Load + CDC, or XtraBackup/Data-Pump seed + binlog/CDC catch-up) is what makes this a **pause measured in seconds, not the whole transfer** — the bulk load runs while the app stays up, and only the final CDC drain happens inside the window. Shrink that window with these prep steps:

**1. Connection-pool settings — set BEFORE cutover (not at cutover).** A stale pool keeps connections open to the old DB long after you repoint, stretching the effective pause. Tune the pool so it recycles fast:

| Pool | Pre-cutover setting | Effect |
|------|---------------------|--------|
| **HikariCP** (Spring Boot) | `maxLifetime=30000` (30s), `keepaliveTime=30000`, `validationTimeout=5000` | Connections recycle within 30s, so a Secrets-Manager/DNS repoint takes effect in ≤30s without a restart. Also set `connectionTimeout` low so failover fails fast rather than hanging. |
| **HikariCP — force immediate** | `dataSource.evictConnections()` / `HikariPoolMXBean.softEvictConnections()` via JMX at cutover | Drops idle connections now instead of waiting out `maxLifetime` — turns the 30s wait into ~immediate. |
| **PgBouncer** | `RECONNECT;` then `RELOAD;` on the admin console (or `server_lifetime` low) after changing the upstream `host=` in `pgbouncer.ini` | New server connections go to the target; existing ones drain. App pool need not restart. |
| **Node (`mysql2`/`pg`) pool** | low `idleTimeoutMillis` / `maxLifetime`; call `pool.end()` + recreate at cutover | Forces re-resolution of the new endpoint. |

Document the chosen values in `migration-plan.md` as a Phase 7.5 / pre-cutover prep item.

**2. Coordinated vs. rolling restart — pick deliberately.**

- **Coordinated (all-at-once) restart** — stop every write client, repoint, start them all. **Fastest total pause (~10s)** and there is **no split-brain window** because no client is writing during the swap. Use this when a brief full write-pause is acceptable (the default for minimal-downtime cutovers). This is faster than waiting on Secrets Manager pool TTL (up to ~5 min) — change the config/unit and restart directly.
- **Rolling restart** (restart instances one at a time behind a load balancer) — keeps *some* capacity serving, so no hard outage, **but** during the roll some instances point at the old DB and some at the new one → **split-brain writes**. Only safe if the source is already frozen read-only (Phase 8 step 3) *before* the roll begins, so stragglers can't write the old DB. Prefer coordinated restart for write workloads; reserve rolling for read-heavy/stateless tiers.

**3. Deploy RDS Proxy on the TARGET before cutover.** The initial EC2 → RDS/Aurora cutover still incurs the brief pause above — **RDS Proxy does not eliminate the *initial* cutover pause** — but standing it up *before* cutover pays off two ways: (a) point the app at the **proxy endpoint** at cutover instead of the cluster endpoint, so all *future* failovers/maintenance are handled by the proxy (it holds the client connections and reconnects to the new writer in **< 1s, no app restart, no pool refresh**); and (b) the proxy multiplexes the reconnect storm at cutover, so the pool refresh doesn't hammer the new DB. After this migration, failovers become effectively zero-downtime even though *this* cutover paused briefly. (Provision it in Phase 4 — see "RDS Proxy on the target".) Note: true Blue/Green sub-second switchover requires the source to **already be on RDS/Aurora** — it's for RDS→Aurora upgrades, not this initial EC2→RDS move.

### Cutover Procedure (Secrets Manager — Recommended)

> **Order matters.** The source must be frozen (read-only *or* all write clients stopped — step 3)
> *before* you repoint the app, or late writes land on the source and are silently lost
> (**split-brain**). If reverse replication is feasible (see "When reverse replication is NOT
> possible" below), it must be running *before* cutover, or the 7-day rollback loses every write
> that hit Aurora. When reverse replication is **not** possible, use an alternative rollback
> strategy and get an explicit RPO acknowledgment — do not silently proceed as if rollback is lossless.

```bash
# 1. Verify CDC caught up (if using DMS)
#    CDCLatencySource must = 0, CDCLatencyTarget must = 0

# 2. Stop application writes (maintenance mode / disable write endpoints)

# 3. FREEZE THE SOURCE — prevents split-brain (late writes hitting source).
#    PREREQUISITE: SET GLOBAL read_only requires SUPER or READ_ONLY ADMIN privilege.
#    The RDS/app master account often does NOT have it (a plain `admin` on a self-managed
#    source frequently lacks SUPER) → `SET GLOBAL read_only=ON` returns:
#      ERROR 1227 (42000): Access denied; you need (at least one of) the SUPER, READ_ONLY ADMIN privilege(s)
#
#    3a. PREFERRED (if you HAVE the privilege): freeze the engine read-only:
mysql -h $SOURCE_HOST -u admin -p -e \
  "SET GLOBAL read_only = ON; SET GLOBAL super_read_only = ON;"
#        (super_read_only also blocks users with SUPER, incl. the replication account's writes.)
#        PostgreSQL: ALTER DATABASE your_db SET default_transaction_read_only = on;  (new sessions only)
#
#    3b. FALLBACK (NO privilege — first-class, equally valid): QUIESCE BY STOPPING WRITE CLIENTS.
#        Stop ALL write clients found in Phase 7.5 FIRST (this is why client discovery is mandatory),
#        then PROVE writers = 0 from the DB side before trusting the freeze:
#        # stop each write client (app), e.g.:  sudo systemctl stop backend.service
mysql -h $SOURCE_HOST -u admin -p -e "
  SELECT id, user, SUBSTRING_INDEX(host,':',1) AS ip, db, command, info
  FROM information_schema.processlist
  WHERE command NOT IN ('Sleep','Daemon','Binlog Dump') AND info IS NOT NULL;"
#        Expect ZERO active write/DML threads. If any remain, find and stop that client before
#        continuing. Stopping the clients (not read_only) becomes the freeze mechanism.

# 4. Wait 30s for final CDC drain, then confirm CDCLatencySource = CDCLatencyTarget = 0
sleep 30

# 5. Stop the forward DMS / replication task (source → target)
aws dms stop-replication-task --replication-task-arn $TASK_ARN

# 6. RESET TARGET HIGH-WATER MARKS before enabling writes on Aurora.
#    DMS/binlog copy data but NOT the AUTO_INCREMENT counter — new inserts on the target
#    can collide with existing PKs. Re-seed every auto-increment / sequence above the max.
#    MySQL/MariaDB — generate & run per table:
mysql -h $AURORA_ENDPOINT -u admin -p -N -e "
  SELECT CONCAT('ALTER TABLE \`', t.TABLE_NAME, '\` AUTO_INCREMENT = ',
                IFNULL((SELECT MAX(\`', k.COLUMN_NAME, '\`) FROM \`', t.TABLE_NAME, '\`),0)+1, ';')
  FROM information_schema.TABLES t
  JOIN information_schema.COLUMNS k
    ON k.TABLE_SCHEMA = t.TABLE_SCHEMA AND k.TABLE_NAME = t.TABLE_NAME
   AND k.EXTRA LIKE '%auto_increment%'
  WHERE t.TABLE_SCHEMA = 'your_db';" | mysql -h $AURORA_ENDPOINT -u admin -p your_db
#    PostgreSQL — re-seed sequences to their column max:
#    SELECT setval(pg_get_serial_sequence(quote_ident(t)||'.'||quote_ident(c), c),
#                  COALESCE((SELECT MAX(c) FROM t), 1)) ...  (see cutover-procedures.md)

# 7. START REVERSE REPLICATION (Aurora → source) so rollback is lossless.
#    This task was CREATED and tested pre-cutover (see "Reverse replication" below);
#    here you just resume it now that Aurora is the write leader.
aws dms start-replication-task \
  --replication-task-arn $REVERSE_TASK_ARN \
  --start-replication-task-type start-replication

# 8. Update secret to point at Aurora
aws secretsmanager update-secret \
  --secret-id "your-app/db-credentials" \
  --secret-string '{"host":"'$AURORA_ENDPOINT'","port":"3306","username":"admin","password":"'$PASS'","dbname":"your_db"}'

# 9. Trigger application credential refresh
#    (ECS: force new deployment / Spring: wait for maxLifetime / Lambda: auto)
#    NOTE: if the host was hardcoded in config/systemd (Phase 7.5), the secret rotation in step 8
#    is NOT enough — you must also change the unit/config and restart the client here.

# 10. Verify connectivity BIDIRECTIONALLY (single-direction curl is insufficient):
#     (a) App side — health endpoint reports the DB as up:
curl -s https://your-app.example.com/actuator/health | jq '.components.db.status'   # expect "UP"
#     (b) DB side — the NEW DB's processlist shows the expected client IPs + connection count:
mysql -h $AURORA_ENDPOINT -u admin -p -e "
  SELECT SUBSTRING_INDEX(host,':',1) AS client_ip, db, COUNT(*) AS conns
  FROM information_schema.processlist GROUP BY client_ip, db;"
#     Confirm each Phase 7.5 client IP appears with a sane connection count (e.g. the backend's
#     pool size). A green /health alone can mask a client that never repointed.

# 11. Resume traffic (clear maintenance mode)
```

### Reverse Replication (set up BEFORE cutover)

A 7-day rollback window is meaningless if rolling back loses every write Aurora took after
cutover. **Before** the cutover window, establish a CDC channel in the reverse direction
(target → source) so the source stays a warm, current standby you can fail back to.

```bash
# Pre-cutover: create (do NOT start) a reverse DMS task with Aurora as source endpoint
# and the original DB as target endpoint. Use the same table mappings as the forward task.
aws dms create-replication-task \
  --replication-task-identifier reverse-aurora-to-source \
  --source-endpoint-arn $AURORA_ENDPOINT_ARN \
  --target-endpoint-arn $SOURCE_ENDPOINT_ARN \
  --replication-instance-arn $DMS_INSTANCE_ARN \
  --migration-type cdc \
  --table-mappings file://table-mappings.json \
  --replication-task-settings file://reverse-task-settings.json
# Validate it can connect and apply by test-running against a scratch table, then stop it.
# It is RESUMED in step 7 of the cutover, the instant Aurora becomes the write leader.
```

- **MySQL/MariaDB reverse CDC** requires `binlog_format=ROW` + `log_bin=ON` on **Aurora**
  (set in the cluster parameter group) so the source can consume Aurora's binlog.
- **PostgreSQL** requires `rds.logical_replication=1` on the Aurora parameter group and a
  spare replication slot on the source.
- Keep reverse replication running for the full rollback window. If you roll back, the source
  is already current — just re-freeze Aurora read-only and repoint the secret.

### When Reverse Replication is NOT Possible (do not require it as the only rollback path)

Reverse replication is the *ideal* lossless-rollback mechanism, but several real conditions make it **impossible** — recognize them and switch to an alternative rather than blocking the migration:

| Condition | Why reverse CDC fails |
|-----------|----------------------|
| **Source `log_bin=OFF`** (and you didn't enable it) | The source cannot act as a replication source/target for binlog-based CDC. The new Aurora/RDS → source channel has nothing to consume on the source side, and DMS CDC into the source still needs the source reachable as a binlog target. |
| **Major version downgrade** (target newer than source, e.g. 10.11 → 10.5) | Replicating *back* from a newer major version to an older one is unsupported/unreliable — binlog/replication formats and feature sets differ. |
| **Insufficient privileges** | No `REPLICATION SLAVE`/`SUPER` (or PostgreSQL replication role) to establish the channel. |

**Alternative rollback strategies (pick per RPO):**

1. **Application-level write logging.** Have the app append every post-cutover write (or its inputs) to a durable log — an append-only audit table on Aurora, an SQS/Kinesis stream, or structured request logs. On rollback, replay that log against the source. Closest to lossless without reverse CDC.
2. **Shortened sync window.** Minimize the time between the last source sync and cutover so the *loss window* (writes that would be stranded on a rollback) is as small as possible — e.g. cut over immediately after the final dump/validation, during a low-traffic window.
3. **Accept the risk with explicit RPO acknowledgment** (low-traffic / non-critical only). If reverse replication is impossible and the above are overkill, **state plainly** that rolling back loses every write Aurora accepted after cutover, get the user's explicit sign-off on that RPO, and keep the cutover window short. Record the acknowledgment in `migration-plan.md`.

> Do **not** present reverse replication as the only rollback path. When it's off the table, choosing one of the above (and saying so) is the correct, honest move.

### Rollback

Keep source DB running **and receiving reverse replication** for **7 days** after cutover.
Rollback = (1) set Aurora `read_only`/`super_read_only` ON, (2) confirm reverse CDC drained
to zero lag, (3) revert the secret/DNS to the source, (4) reset AUTO_INCREMENT/sequences on
the **source** above its new max. Because reverse replication kept the source current, no
data is lost. See [references/cutover-procedures.md](references/cutover-procedures.md).

---

## Phase 9: Post-Migration

1. **Run `ANALYZE TABLE`** on all tables (rebuilds statistics for query optimizer)
2. **Set up CloudWatch alarms** (CPU, memory, connections, latency, replica lag)
3. **Scale down** instance to steady-state size
4. **Review parameter group** — tune for workload (not migration throughput)
5. **Enable automated backups** (if not already — default 7-day retention)
6. **Decommission source** after 7-day rollback period:
   - Stop DMS replication instance
   - Terminate source EC2 (or stop if keeping as cold backup)
   - Delete DMS resources

---

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| DMS: `binlog truncated` | Binlogs expired before CDC read them | Increase `expire_logs_days`, restart full load |
| DMS: Out of memory | Instance undersized or LOBs too large | Scale up instance, use Limited LOB Mode |
| DMS: Increasing CDC latency | Target can't keep up | Enable `BatchApplyEnabled`, scale target |
| Aurora: `Access denied for user` | DEFINER clause references non-existent user | Strip DEFINERs from schema objects |
| Aurora: Stored proc fails | Uses SUPER privilege or unsupported syntax | Rewrite proc without SUPER, fix syntax |
| Slow queries after migration | Missing optimizer statistics | `ANALYZE TABLE` on all tables |
| Connection failures | Security group misconfiguration | Verify SG rules between app → Aurora |
| `ERROR 1227` on `SET GLOBAL read_only` | Account lacks SUPER / READ_ONLY ADMIN | Use the freeze fallback: stop all write clients, verify writers=0 via processlist (Phase 8 step 3b) |
| App can't reach new DB after secret rotation | DB host hardcoded in systemd/config, not in the secret | Change the systemd `ExecStart`/config (highest-priority source wins); backfill host into the secret (Phase 7.5) |
| Migration load or app fails with TLS/SSL error | `require_secure_transport=ON` on target | Add TLS params to load tool + connector (Phase 4 TLS-Enforcement Gate) |
| Schema drift on first app connection to new DB | ORM `ddl-auto=update`/auto-migrate | Set `validate`/`none` before cutover (Phase 7.5) |
| MyISAM error on Aurora | MyISAM not supported | Convert to InnoDB before migration |
| TDE error | Encrypted tablespaces | Decrypt before migration |
| Oracle: `ORA-39083` on import | Missing privilege or target tablespace | Grant on target user; `METADATA_REMAP` tablespace |
| Oracle: `ORA-31693` table load failed | Tablespace quota / space | `ALTER USER ... QUOTA UNLIMITED`; grow storage |
| Oracle: invalid objects after import | Dependencies / compile order | `UTL_RECOMP.RECOMP_PARALLEL` (no `utlrp.sql` — no shell) |
| Oracle: can't import (FULL mode) | RDS blocks FULL mode | Use schema/table mode; exclude SYS-owned Scheduler objects |
| Oracle: TDE dump won't import | `ENCRYPTION_MODE=TRANSPARENT` | Re-export with `ENCRYPTION_MODE=PASSWORD` |
| SQL Server: restore fails, higher version | `.bak` from newer engine | Target RDS engine version must be ≥ source |
| SQL Server: orphaned users post-restore | Login SID mismatch | Recreate login w/ same SID, or `ALTER USER ... WITH LOGIN` |
| SQL Server: FILESTREAM restore rejected | FILESTREAM filegroup in `.bak` | Remove FILESTREAM; redesign as BLOB/S3 |
| SQL Server: missing Agent jobs/logins | Server-level objects not in user `.bak` | Script + recreate separately (Phase 6) |

---

## References (Research Backing)

This skill is backed by extensive research. All detailed procedures live in `references/`:

- **`aws-official-migration-methods.md`** — Every AWS-documented migration method across all engines.
- **`rds-aurora-limitations.md`** — Full blocker/adjustment catalog with assessment queries (encryption, storage engines, auth, privileges).
- **`dms-best-practices.md`** — DMS sizing, endpoints, task settings, LOB handling, engine-specific gotchas.
- **`validation-patterns.md`** — Row counts, checksums, referential integrity, smoke tests.
- **`cutover-procedures.md`** — Secrets Manager rotation, DNS swap, Blue/Green switchover; DB-client discovery, freeze fallback (no SUPER), bidirectional verification, reverse-replication-impossible alternatives.
- **`version-upgrades.md`** — Major version upgrade considerations when migrating via logical dump (reserved words, deprecated functions, default/charset/sql_mode changes) for MySQL, MariaDB, and PostgreSQL.
- **`heterogeneous-migration.md`** — All RDS/Aurora engines; Oracle/SQL Server → Aurora via SCT / DMS Schema Conversion / Babelfish; Tibero/CUBRID/Altibase (no native tooling) paths; PL/SQL conversion; license implications.
- **`third-party-db-security.md`** — Migration-blocker playbook for Korean DB access-control/audit (Chakra Max, DBSafer, Petra) and encryption (Petra Cipher, D'Amo, CUBE-One) tools; on-prem mode → RDS survival → AWS-native replacement (DAS, RDS Proxy, KMS).
- **`regulatory-compliance.md`** — PIPA encryption mandates, 전자금융감독규정 + 2026 망분리 reform, ISMS-P, audit-log retention, SEED/ARIA/AES guidance, and the RDS/Aurora design implications of each.

### Integration with the Kiro modernization flow

This skill is the deep reference behind the database track of the [AI-Driven Modernization with Kiro](https://github.com/aws-samples/sample-ai-driven-modernization-with-kiro) prompt set. It maps onto the stages as:

| Kiro stage | Prompt | This skill supplies |
|------------|--------|---------------------|
| Stage 1 — As-Is Analysis | `database-analysis-prompt.md` | Engine/version/size assessment queries; Korean-security-tool discovery questions |
| Stage 4 — Make Task Lists | `move-to-managed-database-prompt.md` | Method decision matrix; all-engine + heterogeneous task phases; compliance-to-task mapping |
| Stage 5 — Risk Analysis | `2-risk-analysis-database-prompt.md` | Korean-security-tool & compliance risk checkpoints |
| Stage 6 — Task Actions | `start-move-to-managed-database-prompt.md` | Method-specific execution procedures, validation, cutover, rollback |
