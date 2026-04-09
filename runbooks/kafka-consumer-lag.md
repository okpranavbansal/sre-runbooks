# Runbook: Kafka Consumer Lag

**Severity:** P2 (lag growing) / P3 (lag stable but above threshold)  
**Alert:** `kafka_consumer_lag_sum > 10000 for 15m` on any consumer group

---

## Symptoms

- Confluent Cloud dashboard shows consumer lag growing
- Processing latency increasing for async workflows
- Messages queuing in `prompt-ingestion` or `llm-response-events` topics
- Downstream services (analytics, audit) showing stale data

---

## Diagnostic Steps

```bash
# 1. Check consumer group lag via Confluent CLI
confluent kafka consumer group list \
  --environment <env-id> \
  --cluster <cluster-id>

confluent kafka consumer group describe <group-id> \
  --environment <env-id> \
  --cluster <cluster-id>

# 2. Check which partitions have the most lag
# (High lag on a single partition = likely one consumer is down)

# 3. Check consumer application logs
# On ECS:
aws ecs execute-command \
  --cluster prd-platform \
  --task <task-id> \
  --container llm-consumer \
  --interactive \
  --command "/bin/sh"

# On GKE:
kubectl logs -n prd-copilot -l app=llm-consumer --tail=100
```

---

## Root Cause Analysis

| Pattern | Likely Cause |
|---------|-------------|
| Lag growing on all partitions | Consumer crashed / not running |
| Lag growing on 1-2 partitions | Consumer stuck on poison pill message |
| Lag stable but high | Consumer throughput < producer throughput |
| Lag after deployment | Consumer group rebalance (normal, transient) |

---

## Mitigation Steps

### Option A: Consumer crashed — restart it

```bash
# ECS: force new deployment
aws ecs update-service \
  --cluster prd-platform \
  --service llm-consumer \
  --force-new-deployment

# GKE: restart deployment
kubectl rollout restart deployment llm-consumer -n prd-copilot
```

### Option B: Poison pill message — skip the bad offset

```bash
# Find the stuck offset
confluent kafka consumer group describe <group-id> \
  --environment <env-id> \
  --cluster <cluster-id>

# Reset offset past the bad message (use with caution in prd)
confluent kafka consumer group update <group-id> \
  --environment <env-id> \
  --cluster <cluster-id> \
  --reset-offsets \
  --topic <topic-name> \
  --partition <partition> \
  --offset <bad-offset + 1>
```

### Option C: Throughput insufficient — scale up consumers

```bash
# ECS: increase desired count
aws ecs update-service \
  --cluster prd-platform \
  --service llm-consumer \
  --desired-count 5

# GKE: scale deployment
kubectl scale deployment llm-consumer -n prd-copilot --replicas=5
```

---

## Prevention

- [ ] Dead letter topic (`llm.dlq`) for messages that fail 3 retries
- [ ] Alert at 5,000 lag (not 10,000) for prompt-ingestion topic
- [ ] Consumer `enable.auto.commit=false` — only commit after successful processing
- [ ] Add partition count increase plan: `prompt-ingestion` → 24 partitions at 2× current load

---

## Escalation

If lag exceeds 50,000 messages and is not recovering → escalate to backend lead + notify product (SLA impact).
