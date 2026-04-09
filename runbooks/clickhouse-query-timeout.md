# Runbook: ClickHouse Query Timeout

**Alert name:** `ClickHouseQueryTimeout`  
**Severity:** Warning → Critical  
**Trigger:** ClickHouse query latency P99 > 10s OR error rate > 5% with `TIMEOUT` errors in logs  
**Team:** Data / Platform SRE

---

## Quick Diagnosis

```bash
# SSH into ClickHouse host (or kubectl exec for GKE StatefulSet)
# --- AWS EC2 ---
aws ssm start-session --target $INSTANCE_ID

# --- GKE StatefulSet ---
kubectl exec -it clickhouse-0 -n prd-copilot -- bash

# Inside the node:
clickhouse-client --user admin --password "$CH_PASSWORD"
```

```sql
-- 1. What queries are currently running?
SELECT
    query_id,
    elapsed,
    read_rows,
    read_bytes,
    memory_usage,
    query
FROM system.processes
ORDER BY elapsed DESC
LIMIT 10;

-- 2. Are there slow queries in the last hour?
SELECT
    query_start_time,
    query_duration_ms / 1000 AS duration_sec,
    read_rows,
    result_rows,
    exception,
    query
FROM system.query_log
WHERE query_start_time >= now() - INTERVAL 1 HOUR
  AND query_duration_ms > 5000
ORDER BY query_duration_ms DESC
LIMIT 20;

-- 3. Merge backlog (high backlog → slow SELECT queries)
SELECT
    table,
    count() AS num_parts,
    sum(rows) AS total_rows,
    sum(bytes_on_disk) AS total_bytes
FROM system.parts
WHERE active AND database = 'default'
GROUP BY table
ORDER BY num_parts DESC;
```

---

## Common Causes & Fixes

### Cause 1: Full table scan (missing partition key filter)

**Symptom:** `read_rows` in `system.processes` shows billions of rows

**Fix:** Add `WHERE date_column >= today() - 7` to limit partition range. Work with the application team to fix the query. Do NOT kill it immediately — killing mid-aggregation can corrupt intermediate results.

```sql
-- Kill a specific stuck query:
KILL QUERY WHERE query_id = 'abc123-...';
```

### Cause 2: Too many parts (merge backlog)

**Symptom:** `system.parts` shows > 1000 active parts for a table

```sql
-- Manually trigger merge for a specific table
OPTIMIZE TABLE analytics_events FINAL;
-- Warning: FINAL is expensive on large tables; run during off-peak
```

### Cause 3: Disk I/O saturation

```bash
# Check disk utilization
iostat -x 1 10
# Look for: %util > 80% on the ClickHouse data disk

# Check ClickHouse disk usage
du -sh /var/lib/clickhouse/data/*

# If disk full → see neo4j-disk-full runbook for general pattern
df -h /var/lib/clickhouse
```

### Cause 4: Memory pressure causing spill to disk

```sql
-- Check memory usage per query
SELECT query_id, memory_usage, query
FROM system.processes
ORDER BY memory_usage DESC
LIMIT 5;
```

Increase `max_memory_usage` setting or add `max_bytes_before_external_group_by` to spill gracefully.

---

## Emergency: Kill All Long-Running Queries

```sql
-- Kill all queries running more than 60 seconds (use with caution in prd)
KILL QUERY WHERE elapsed > 60;
```

---

## Escalation

- **10 min**: Query still running, no progress → kill it, alert app team
- **20 min**: Multiple queries timing out → possible infrastructure issue, page data team
- **30 min**: ClickHouse unresponsive → restart service (see restart procedure below)

```bash
# Restart ClickHouse (GKE)
kubectl rollout restart statefulset clickhouse -n prd-copilot

# Restart ClickHouse (EC2)
sudo systemctl restart clickhouse-server
```
