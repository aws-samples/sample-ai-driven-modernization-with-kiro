---
name: database-migration
description: AWS DMS configuration, dump/restore, data validation (record count/checksum), rollback procedures, Aurora/RDS connection switching. Reference during database migration tasks (Stage 3~5).
---
# Database Migration Guide

## Migration Strategy

### Migration Method Selection

**AWS DMS (Database Migration Service)**:
- Minimal downtime migration
- Continuous replication support
- Heterogeneous DB migration

**Dump and Restore**:
- Small databases
- Downtime acceptable
- Same DB engine

## Preparation

### 0. Engine-Specific Migration Readiness Checklist

Before starting migration, verify engine-specific prerequisites:

**MySQL / MariaDB**:
- [ ] Binary logging enabled: `log_bin = ON`
- [ ] Binlog format set: `binlog_format = ROW`
- [ ] Binlog row image: `binlog_row_image = FULL`
- [ ] Server ID set: `server_id` is unique and non-zero
- [ ] Log slave updates (for replicas): `log_slave_updates = ON`
- [ ] Binlog retention: `expire_logs_days` or `binlog_expire_logs_seconds` configured
- ⚠️ Changing `log_bin` requires DB restart. Plan maintenance window.

**PostgreSQL**:
- [ ] WAL level set: `wal_level = logical`
- [ ] Replication slots: `max_replication_slots >= 5` (DMS needs at least 1)
- [ ] WAL senders: `max_wal_senders >= 5`
- [ ] `rds.logical_replication = 1` (for RDS PostgreSQL)
- ⚠️ Changing `wal_level` requires DB restart. Plan maintenance window.

**Verification commands**:
```sql
-- MySQL: Check binlog settings
SHOW VARIABLES LIKE 'log_bin';
SHOW VARIABLES LIKE 'binlog_format';
SHOW VARIABLES LIKE 'binlog_row_image';
SHOW VARIABLES LIKE 'server_id';

-- PostgreSQL: Check WAL settings
SHOW wal_level;
SHOW max_replication_slots;
SHOW max_wal_senders;
```

### 1. Backup and Rollback Plan

### Critical DB Setting Changes — Script Review Process (MANDATORY)

For critical DB configuration changes (binlog, WAL, replication settings), follow this 3-step process:

**Step 1: Generate script** — AI creates a script that includes:
```bash
#!/bin/bash
# 1. Backup current settings
echo "=== Current Settings Backup ==="
mysql -h $DB_HOST -u $DB_USER -p -e "SHOW VARIABLES LIKE 'log_bin'; SHOW VARIABLES LIKE 'binlog_format';"

# 2. Apply changes (via Parameter Group for RDS)
echo "=== Applying Changes ==="
# aws rds modify-db-parameter-group ...

# 3. Verify changes
echo "=== Verification ==="
mysql -h $DB_HOST -u $DB_USER -p -e "SHOW VARIABLES LIKE 'log_bin'; SHOW VARIABLES LIKE 'binlog_format';"

# 4. Rollback command (if needed)
echo "=== Rollback Command (save for emergency) ==="
# aws rds modify-db-parameter-group --db-parameter-group-name ... --parameters "ParameterName=...,ParameterValue=...,ApplyMethod=pending-reboot"
```

**Step 2: User review and approval** — Present script to user for review. Do NOT execute without explicit approval.

**Step 3: Execute** — Run the approved script, verify results, log in audit.md.

⚠️ If the change requires DB restart, confirm maintenance window with user before proceeding.

### 2. Backup and Rollback Plan

**Create full backup**:
```bash
# MySQL backup
mysqldump -h source-host -u username -p \
  --single-transaction \
  --routines \
  --triggers \
  --all-databases > backup.sql

# Verify backup
mysql -h source-host -u username -p -e "SHOW DATABASES;"
```

**Establish rollback plan**:
- Store backup files in safe location
- Document restore procedures
- Perform restore test

### 2. Source Database Analysis

**Check database size**:
```sql
SELECT 
  table_schema AS 'Database',
  ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.tables
GROUP BY table_schema;
```

**Check table sizes**:
```sql
SELECT 
  table_name AS 'Table',
  ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)'
FROM information_schema.tables
WHERE table_schema = 'your_database'
ORDER BY (data_length + index_length) DESC;
```

**Check indexes**:
```sql
SHOW INDEX FROM your_table;
```

### 3. Target Database Preparation

**Create Aurora/RDS cluster**:
- Select appropriate instance type
- Multi-AZ configuration
- Parameter group settings
- Security group configuration

**Initial setup**:
```sql
-- Create database
CREATE DATABASE ecommerce CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create user and grant privileges
CREATE USER 'appuser'@'%' IDENTIFIED BY 'secure_password';
GRANT ALL PRIVILEGES ON ecommerce.* TO 'appuser'@'%';
FLUSH PRIVILEGES;
```

## AWS DMS Configuration

### 0. DMS Prerequisites (MANDATORY)

Before creating DMS resources, verify:
- [ ] **VPC Endpoint**: DMS VPC endpoint created (or DMS replication instance has internet access via NAT Gateway)
- [ ] **Engine-specific settings**: Binlog (MySQL) or WAL (PostgreSQL) configured per checklist above
- [ ] **Source/Target connectivity**: Security groups allow DMS replication instance to reach both source and target DBs
- [ ] **DMS IAM roles**: `dms-vpc-role` and `dms-cloudwatch-logs-role` exist

**Expected time estimates**:
| Step | Estimated Time |
|------|---------------|
| Replication instance creation | 10~15 minutes |
| Endpoint creation + connection test | 5 minutes each |
| Full load (per 10GB of data) | ~15~30 minutes (varies by instance size and network) |
| CDC (ongoing replication) | Continuous until cutover |

⚠️ Plan for total DMS setup time of 30+ minutes before migration tasks can begin.

### 1. Replication Instance Creation

**Instance sizing**:
- Small (< 10GB): dms.t3.medium
- Medium (10-100GB): dms.c5.large
- Large (> 100GB): dms.c5.xlarge

### 2. Source Endpoint Setup

```json
{
  "EndpointType": "source",
  "EngineName": "mysql",
  "ServerName": "source-db-host",
  "Port": 3306,
  "DatabaseName": "ecommerce",
  "Username": "dms_user",
  "Password": "secure_password"
}
```

**Verify source DB settings**:
```sql
-- Check binary logging enabled
SHOW VARIABLES LIKE 'log_bin';

-- Check binlog format (ROW required)
SHOW VARIABLES LIKE 'binlog_format';
```

### 3. Target Endpoint Setup

```json
{
  "EndpointType": "target",
  "EngineName": "aurora-mysql",
  "ServerName": "target-aurora-cluster.cluster-xxx.ap-northeast-2.rds.amazonaws.com",
  "Port": 3306,
  "DatabaseName": "ecommerce",
  "Username": "admin",
  "Password": "secure_password"
}
```

### 4. Replication Task Creation

**Task settings**:
```json
{
  "MigrationType": "full-load-and-cdc",
  "TableMappings": {
    "rules": [
      {
        "rule-type": "selection",
        "rule-id": "1",
        "rule-name": "include-all-tables",
        "object-locator": {
          "schema-name": "ecommerce",
          "table-name": "%"
        },
        "rule-action": "include"
      }
    ]
  }
}
```

**Migration types**:
- `full-load`: One-time full data copy
- `cdc`: Continuous change data capture only
- `full-load-and-cdc`: Full copy followed by continuous replication (recommended)

## Migration Execution

### 1. Test Environment Migration

**Steps**:
1. Run DMS Task in test environment
2. Validate data
3. Test application connectivity
4. Performance testing

### 2. Data Validation

**Record count comparison**:
```sql
-- Source DB
SELECT COUNT(*) FROM ecommerce.orders;

-- Target DB
SELECT COUNT(*) FROM ecommerce.orders;
```

**Checksum comparison**:
```sql
-- Source DB
SELECT MD5(GROUP_CONCAT(id ORDER BY id)) AS checksum
FROM ecommerce.orders;

-- Target DB
SELECT MD5(GROUP_CONCAT(id ORDER BY id)) AS checksum
FROM ecommerce.orders;
```

**Sample data comparison**:
```sql
-- Compare recent data
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
```

### 3. Production Migration

**Minimal downtime procedure**:
1. Start DMS Task (full-load-and-cdc)
2. Wait for full data copy completion
3. Continuous replication via CDC
4. Switch read traffic to target DB
5. Verify data consistency
6. Switch write traffic to target DB
7. Monitor source DB and keep on standby

## Application Connection Switching

### 1. Update Connection Information

**Environment variable method**:
```bash
# Before
DB_HOST=ec2-instance-ip
DB_PORT=3306

# After
DB_HOST=aurora-cluster.cluster-xxx.ap-northeast-2.rds.amazonaws.com
DB_PORT=3306
```

**Parameter Store method**:
```bash
aws ssm put-parameter \
  --name /ecommerce/prod/db/host \
  --value "aurora-cluster.cluster-xxx.ap-northeast-2.rds.amazonaws.com" \
  --type String \
  --overwrite
```

### 2. Connection Pool Adjustment

```typescript
// TypeORM example
{
  type: 'mysql',
  host: process.env.DB_HOST,
  port: 3306,
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: 'ecommerce',
  poolSize: 10,
  connectTimeout: 10000,
  acquireTimeout: 10000
}
```

### 3. Read Replica Usage

```typescript
// Read/write separation
const writeConnection = {
  host: 'aurora-cluster.cluster-xxx.ap-northeast-2.rds.amazonaws.com'
};

const readConnection = {
  host: 'aurora-cluster.cluster-ro-xxx.ap-northeast-2.rds.amazonaws.com'
};
```

## Monitoring and Validation

### DMS Task Monitoring

**CloudWatch metrics**:
- `FullLoadThroughputRowsSource`: Rows copied per second
- `CDCLatencySource`: Replication latency
- `CDCThroughputRowsSource`: CDC throughput

**Check task status**:
```bash
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=arn:aws:dms:... \
  --query 'ReplicationTasks[0].Status'
```

### Database Performance Monitoring

**Aurora metrics**:
- CPU utilization
- Connection count
- Read/Write IOPS
- Replication lag

**Check slow queries**:
```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;

-- Check slow queries
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 10;
```

## Rollback Procedures

### Emergency Rollback

**Step 1: Restore application connection**:
```bash
# Restore connection to source DB
aws ssm put-parameter \
  --name /ecommerce/prod/db/host \
  --value "ec2-instance-ip" \
  --overwrite

# Restart application
kubectl rollout restart deployment/backend
```

**Step 2: Verify data synchronization**:
- Check if DMS Task is still running
- Verify latest data exists in source DB

### Data Restoration

**Restore from backup**:
```bash
# MySQL restore
mysql -h source-host -u username -p < backup.sql

# Restore specific database only
mysql -h source-host -u username -p ecommerce < backup.sql
```

## Post-Migration

### 1. Resource Cleanup

- Stop and delete DMS Replication Task
- Delete DMS Replication Instance
- Stop source EC2 instance (do NOT delete immediately)

### 2. Optimization

**Index optimization**:
```sql
ANALYZE TABLE orders;
OPTIMIZE TABLE orders;
```

**Parameter group tuning**:
- `max_connections`
- `innodb_buffer_pool_size`
- `query_cache_size`

### 3. Verify Backup Settings

- Enable automatic backups
- Set backup retention period (7 days recommended)
- Create snapshot

## Best Practices

### DO
- ✅ Full backup before migration
- ✅ Validate in test environment first
- ✅ Run data validation queries
- ✅ Establish rollback plan
- ✅ Gradual traffic switching
- ✅ Enhanced monitoring

### DON'T
- ❌ Migrate without backup
- ❌ Test directly in production
- ❌ Skip data validation
- ❌ Proceed without rollback plan
- ❌ Switch all traffic at once
- ❌ Delete source DB immediately
