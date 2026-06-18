# Data Validation Patterns

## Validation Strategy

Validation happens at three points:
1. **During migration** — DMS built-in validation (continuous)
2. **Pre-cutover** — Manual validation before switching traffic
3. **Post-cutover** — Application-level verification after go-live

---

## 1. DMS Built-in Validation

### Enable Validation in Task Settings

```json
{
  "ValidationSettings": {
    "EnableValidation": true,
    "ThreadCount": 5,
    "ValidationMode": "ROW_LEVEL",
    "FailureMaxCount": 10000,
    "HandleCollationDiff": "true",
    "RecordSuspendedState": "RECORD_ONLY",
    "SkipLobColumns": false,
    "TableFailureMaxCount": 1000,
    "ValidationOnly": false,
    "ValidationPartialLobSize": 0
  }
}
```

### Monitor Validation Status

```bash
# Check table-level validation status
aws dms describe-table-statistics \
  --replication-task-arn $TASK_ARN \
  --query 'TableStatistics[].{
    Table: TableName,
    State: ValidationState,
    Pending: ValidationPendingRecords,
    Failed: ValidationFailedRecords,
    Suspended: ValidationSuspendedRecords
  }' --output table
```

### Validation States

| State | Meaning | Action |
|-------|---------|--------|
| `Not enabled` | Validation not configured | Enable in task settings |
| `Pending records` | Records queued for validation | Wait — normal during active CDC |
| `Mismatched records` | Rows differ between source/target | Investigate — check LOB handling |
| `Suspended records` | Validation couldn't compare (e.g., no PK) | Add PK or accept limitation |
| `No primary key` | Table has no PK for row-level comparison | Add PK or use manual validation |
| `Table error` | Validation encountered an error | Check DMS logs |
| `Validated` | All rows match ✅ | Good to proceed |

### Common Validation Failures

| Failure Type | Cause | Resolution |
|-------------|-------|------------|
| Mismatched records (small count) | CDC latency — records changed after validation | Re-validate after CDC catches up |
| Mismatched records (LOB tables) | LOB mode truncation | Use Full LOB Mode or increase `LobMaxSize` |
| Suspended records | Table lacks primary key | Add PK, or accept and do manual validation |
| All records suspended | Permissions issue on source/target | Check DMS user grants |

---

## 2. Pre-Cutover Manual Validation

### 2.1 Row Count Comparison

**MySQL:**
```sql
-- Generate comparison queries dynamically
SELECT CONCAT(
  'SELECT ''', table_name, ''' AS tbl, COUNT(*) AS cnt FROM `', table_name, '` UNION ALL'
) FROM information_schema.tables
WHERE table_schema = 'your_db' AND table_type = 'BASE TABLE'
ORDER BY table_name;

-- Run the generated query on both source and target, compare results
```

**PostgreSQL:**
```sql
-- Generate comparison queries
SELECT format('SELECT %L AS tbl, COUNT(*) AS cnt FROM %I.%I UNION ALL',
  tablename, schemaname, tablename)
FROM pg_tables WHERE schemaname = 'public'
ORDER BY tablename;
```

**Automation script:**
```bash
#!/bin/bash
# Compare row counts between source and target
TABLES=$(mysql -h $SOURCE -u $USER -p$PASS -N -e \
  "SELECT table_name FROM information_schema.tables WHERE table_schema='$DB'")

echo "TABLE | SOURCE | TARGET | MATCH"
echo "------|--------|--------|------"
for TABLE in $TABLES; do
  SRC=$(mysql -h $SOURCE -u $USER -p$PASS -N -e "SELECT COUNT(*) FROM $DB.$TABLE")
  TGT=$(mysql -h $TARGET -u $USER -p$PASS -N -e "SELECT COUNT(*) FROM $DB.$TABLE")
  if [ "$SRC" = "$TGT" ]; then
    MATCH="✅"
  else
    MATCH="❌ (diff: $((TGT - SRC)))"
  fi
  echo "$TABLE | $SRC | $TGT | $MATCH"
done
```

### 2.2 Checksum Verification

**MySQL — pt-table-checksum (Percona Toolkit):**
```bash
# Install Percona Toolkit
apt-get install percona-toolkit

# Run checksum comparison
pt-table-checksum \
  --host=$SOURCE \
  --user=$USER \
  --password=$PASS \
  --databases=$DB \
  --replicate=percona.checksums \
  --no-check-binlog-format \
  --chunk-size=5000

# Check results
pt-table-sync --print --replicate percona.checksums \
  --host=$SOURCE --user=$USER --password=$PASS
```

**MySQL — Native CHECKSUM TABLE (simpler but locks tables):**
```sql
-- Quick checksums (READ lock during execution)
CHECKSUM TABLE orders, products, users, cart_items;
-- Compare output between source and target
```

**PostgreSQL — Hash-based verification:**
```sql
-- MD5 hash of critical columns for a table
SELECT md5(string_agg(
  md5(CAST(id AS text) || CAST(amount AS text) || CAST(status AS text) || CAST(created_at AS text)),
  '' ORDER BY id
)) AS table_hash
FROM orders;
-- Run on both source and target, compare hashes
```

### 2.3 Sample Record Deep Comparison

```sql
-- Compare the last N records (catching recent CDC-applied changes)
-- Source:
SELECT * FROM orders WHERE id IN (
  SELECT id FROM orders ORDER BY id DESC LIMIT 100
) ORDER BY id;

-- Target:
SELECT * FROM orders WHERE id IN (
  SELECT id FROM orders ORDER BY id DESC LIMIT 100
) ORDER BY id;

-- Export to CSV and diff:
-- mysql -h $SOURCE ... --batch --raw > /tmp/source_sample.csv
-- mysql -h $TARGET ... --batch --raw > /tmp/target_sample.csv
-- diff /tmp/source_sample.csv /tmp/target_sample.csv
```

### 2.4 Referential Integrity Check

Run on the TARGET to ensure FK relationships are intact:

```sql
-- MySQL: Find orphaned references
-- Orders referencing non-existent users
SELECT o.id, o.user_id FROM orders o
LEFT JOIN users u ON o.user_id = u.id
WHERE u.id IS NULL;

-- Order items referencing non-existent orders
SELECT oi.id, oi.order_id FROM order_items oi
LEFT JOIN orders o ON oi.order_id = o.id
WHERE o.id IS NULL;

-- Order items referencing non-existent products
SELECT oi.id, oi.product_id FROM order_items oi
LEFT JOIN products p ON oi.product_id = p.id
WHERE p.id IS NULL;
```

```sql
-- PostgreSQL: Check all FK constraints
SELECT conname AS constraint_name,
  conrelid::regclass AS table_name,
  confrelid::regclass AS referenced_table
FROM pg_constraint
WHERE contype = 'f'
  AND NOT convalidated;  -- Shows invalid (violated) constraints

-- Validate all constraints (will error if violations exist)
-- ALTER TABLE orders VALIDATE CONSTRAINT fk_user_id;
```

### 2.5 Schema Object Verification

```sql
-- MySQL: Compare stored procedure counts
SELECT ROUTINE_TYPE, COUNT(*) FROM information_schema.routines
WHERE ROUTINE_SCHEMA = 'your_db' GROUP BY ROUTINE_TYPE;

-- Compare trigger counts
SELECT COUNT(*) FROM information_schema.triggers WHERE TRIGGER_SCHEMA = 'your_db';

-- Compare view counts
SELECT COUNT(*) FROM information_schema.views WHERE TABLE_SCHEMA = 'your_db';

-- Compare index counts
SELECT TABLE_NAME, COUNT(*) AS index_count FROM information_schema.statistics
WHERE TABLE_SCHEMA = 'your_db' GROUP BY TABLE_NAME ORDER BY TABLE_NAME;
```

---

## 3. Post-Cutover Validation

### Application Smoke Tests

Run immediately after cutover:

```bash
# Health check
curl -sf https://your-app.example.com/health | jq '.database'

# API functional tests (read)
curl -sf https://your-app.example.com/api/products | jq '.[] | .id' | head -5

# API functional tests (write)
curl -sf -X POST https://your-app.example.com/api/orders \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "items": [{"product_id": 1, "quantity": 1}]}'

# Verify the write persisted
curl -sf https://your-app.example.com/api/orders?latest=1
```

### Performance Baseline Comparison

```sql
-- Run on Aurora after cutover — compare against source baseline
-- Query 1: Complex join (typical read pattern)
EXPLAIN ANALYZE SELECT o.id, o.total_amount, u.username
FROM orders o JOIN users u ON o.user_id = u.id
WHERE o.created_at > NOW() - INTERVAL 7 DAY
ORDER BY o.created_at DESC LIMIT 100;

-- Query 2: Aggregation (reporting pattern)
EXPLAIN ANALYZE SELECT DATE(created_at) AS dt, COUNT(*) AS orders, SUM(total_amount) AS revenue
FROM orders WHERE created_at > NOW() - INTERVAL 30 DAY
GROUP BY DATE(created_at);
```

### Monitoring Checklist (First 24 Hours)

- [ ] Application error rate < 0.1%
- [ ] P99 query latency within 2x of baseline
- [ ] No connection pool exhaustion
- [ ] No deadlocks or lock waits > 10s
- [ ] Aurora CPU < 70%
- [ ] Aurora FreeableMemory > 2 GB
- [ ] No replication lag on read replicas
- [ ] Scheduled jobs (cron, events) executing successfully
- [ ] Backup completed successfully (first automated backup)

---

## Validation Decision Matrix

| Validation Type | When | Criticality | Automated? |
|----------------|------|-------------|-----------|
| DMS built-in | During migration | High | ✅ Yes |
| Row count comparison | Pre-cutover, post-cutover | High | ✅ Scriptable |
| Checksum verification | Pre-cutover (final) | Medium | ✅ pt-table-checksum |
| Sample record comparison | Pre-cutover | Medium | Semi-auto |
| Referential integrity | Post-full-load, pre-cutover | High | ✅ Scriptable |
| Schema object count | After schema migration | Medium | ✅ Scriptable |
| Application smoke tests | Post-cutover | Critical | ✅ CI/CD |
| Performance comparison | Post-cutover (24h) | Medium | ✅ CloudWatch |
