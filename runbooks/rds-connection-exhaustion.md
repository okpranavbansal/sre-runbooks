# Runbook: RDS Connection Exhaustion

**Severity:** P1 — services cannot connect to database, widespread API failures  
**Alert:** `RDS_ConnectionCount > 450 for 5m` (for db.t3.large with max 500)

---

## Symptoms

- API services returning `500` with `Too many connections` in logs
- ECS tasks failing health checks and being replaced (creating more connections — feedback loop)
- CloudWatch metric `DatabaseConnections` near or at max

---

## Immediate Triage (< 5 minutes)

```bash
# 1. Check current connection count
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=prd-mysql \
  --start-time $(date -u -d '-15 minutes' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Maximum

# 2. Identify which services hold the most connections
# Connect to the DB (via bastion or ECS exec) and run:
SELECT user, count(*) as connections
FROM information_schema.processlist
GROUP BY user
ORDER BY connections DESC;

# 3. Kill long-running idle connections
SELECT concat('KILL ', id, ';')
FROM information_schema.processlist
WHERE command = 'Sleep'
AND time > 300
ORDER BY time DESC;
```

---

## Root Cause Analysis

Common causes in order of frequency:

1. **Connection pool misconfiguration** — service restarted with `pool_size` too high
2. **ECS task replacement storm** — OOM or health check failures causing rapid task cycling
3. **Slow query holding connection** — a locked table causing connections to queue
4. **Missing connection timeout** — connections not released after inactivity

---

## Mitigation Steps

### Option A: Reduce pool size (immediate, no downtime)

```bash
# Update SSM parameter and force ECS service redeploy
aws ssm put-parameter \
  --name /prd/billing-service/DB_POOL_SIZE \
  --value "5" \
  --type String \
  --overwrite

aws ecs update-service \
  --cluster prd-platform \
  --service billing-service \
  --force-new-deployment
```

### Option B: Temporarily scale down non-critical services

```bash
# Scale down low-priority services to free up connections
for svc in analytics-worker report-generator; do
  aws ecs update-service --cluster prd-platform --service $svc --desired-count 0
done
```

### Option C: Increase max_connections (requires DB restart — last resort for P1)

```bash
# Modify RDS parameter group (takes effect at next maintenance window or immediate reboot)
aws rds modify-db-parameter-group \
  --db-parameter-group-name prd-mysql-params \
  --parameters ParameterName=max_connections,ParameterValue=800,ApplyMethod=pending-reboot
```

---

## Prevention

- [ ] Enforce `DB_POOL_SIZE` via SSM parameter with max=10 per service instance
- [ ] Add RDS Proxy in front of the database (pools connections at proxy layer)
- [ ] Alert at 70% of `max_connections` (not 90%)
- [ ] Set `wait_timeout=300` and `interactive_timeout=300` on RDS

---

## Escalation

If connections cannot be brought below 80% within 15 minutes → escalate to DB lead + engineering manager.
