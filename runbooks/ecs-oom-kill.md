# Runbook: ECS Task OOM Kill

**Alert name:** `ECSContainerOOMKilled`  
**Severity:** Warning → Critical (if rate > 3 per hour)  
**Trigger:** CloudWatch Metric `MemoryUtilization` > 90% sustained, OR task stop reason contains `OutOfMemoryError`  
**Team:** Platform SRE

---

## Confirm It's an OOM Kill

```bash
# Method 1: CloudWatch service events
aws ecs describe-services \
  --cluster prd-b2b \
  --services $SERVICE_NAME \
  --query 'services[0].events[:5].message'
# Look for: "task failed container health checks" or "exit code 137"

# Method 2: Stopped task exit code
aws ecs describe-tasks \
  --cluster prd-b2b \
  --tasks $TASK_ARN \
  --query 'tasks[0].containers[*].{name:name,exitCode:exitCode,reason:reason}'
# Exit code 137 = OOM kill (SIGKILL from kernel)

# Method 3: CloudWatch Container Insights
# Metric: MemoryUtilized
# Namespace: ECS/ContainerInsights
# Dimension: ServiceName
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name MemoryUtilized \
  --dimensions Name=ClusterName,Value=prd-b2b Name=ServiceName,Value=$SERVICE_NAME \
  --period 300 \
  --start-time $(date -u -v-1H +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --statistics Maximum \
  --unit Megabytes
```

---

## Immediate Mitigation

### Option A: Increase task memory (quickest)

```bash
# 1. Get current task definition
TASK_DEF=$(aws ecs describe-services --cluster prd-b2b --services $SERVICE_NAME --query 'services[0].taskDefinition' --output text)

# 2. Register new revision with more memory
# Edit the task definition JSON — increment memory (e.g. 1024 → 2048)
aws ecs describe-task-definition --task-definition $TASK_DEF --query 'taskDefinition' > /tmp/task-def.json
# Edit /tmp/task-def.json: change "memory" value
aws ecs register-task-definition --cli-input-json file:///tmp/task-def.json

# 3. Update service to use new task definition
aws ecs update-service \
  --cluster prd-b2b \
  --service $SERVICE_NAME \
  --task-definition $SERVICE_NAME  # picks up latest revision
```

### Option B: Reduce concurrent connections / requests

If the OOM is caused by a traffic spike, temporarily reduce ALB target group weight or enable circuit breaker in the upstream service.

---

## Root Cause Investigation

### Common OOM causes

| Cause | How to identify | Fix |
|-------|----------------|-----|
| Memory leak | Memory grows monotonically over hours, then OOM | Fix in code; add restart liveness probe as mitigation |
| Traffic spike | OOM correlates with request rate spike | Increase memory or add HPA |
| Large payload | Single request consuming all memory | Add request body size limit; stream instead of buffering |
| Goroutine/thread leak | CPU also high; heap dump shows abnormal allocations | Fix in code |

### Getting heap dump (if service is still alive)

```bash
# SSM exec into running task
aws ecs execute-command \
  --cluster prd-b2b \
  --task $TASK_ARN \
  --container $SERVICE_NAME \
  --command "kill -USR1 1" \  # if service supports it (e.g. Node.js --heapsnapshot-signal)
  --interactive

# Or: trigger /debug/pprof/heap endpoint if pprof is enabled (Go services)
curl http://localhost:6060/debug/pprof/heap > /tmp/heap.prof
go tool pprof /tmp/heap.prof
```

---

## Long-term Fix Checklist

- [ ] Add memory leak test to load test suite
- [ ] Set `memory_reservation` (soft limit) at 80% of `memory` (hard limit) to get CloudWatch warning before OOM
- [ ] Configure liveness probe to restart container if memory metric exceeds threshold (before kernel kills it)
- [ ] Review rightsizing — see `k8s-cost-optimizer/scripts/rightsizing-audit.sh` for memory utilization baselines
