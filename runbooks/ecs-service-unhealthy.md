# Runbook: ECS Service Unhealthy

**Alert name:** `ECSServiceHealthy`  
**Severity:** Critical  
**Trigger:** ECS service has 0 running tasks, OR desired count != running count for > 5 minutes  
**Team:** Platform SRE

---

## Quick Diagnosis (< 5 min)

```bash
# 1. Check service state
aws ecs describe-services \
  --cluster prd-b2b \
  --services $SERVICE_NAME \
  --query 'services[0].{desired:desiredCount,running:runningCount,pending:pendingCount,events:events[:5]}'

# 2. Check deployment state
aws ecs describe-services \
  --cluster prd-b2b \
  --services $SERVICE_NAME \
  --query 'services[0].deployments[*].{id:id,status:status,desired:desiredCount,running:runningCount,failed:failedTasks}'

# 3. Recent service events (most informative)
aws ecs describe-services \
  --cluster prd-b2b \
  --services $SERVICE_NAME \
  --query 'services[0].events[:10].message'
```

---

## Common Causes & Fixes

### Cause 1: Task failing health check

**Symptoms in events:** `task failed ELB health checks`

```bash
# Pull latest task logs
TASK_ID=$(aws ecs list-tasks --cluster prd-b2b --service-name $SERVICE_NAME --query 'taskArns[0]' --output text | awk -F/ '{print $NF}')
aws logs get-log-events \
  --log-group-name /ecs/prd-$SERVICE_NAME \
  --log-stream-name ecs/$SERVICE_NAME/$TASK_ID \
  --limit 100
```

**Fix:** Check application startup errors. Common issue is missing environment variable or SSM secret not found.

---

### Cause 2: Task stopped before health check passes

**Symptoms:** Service events show `task stopped` with `exit code 1`

```bash
# Get stopped task reason
aws ecs describe-tasks \
  --cluster prd-b2b \
  --tasks $TASK_ARN \
  --query 'tasks[0].{stoppedReason:stoppedReason,containers:containers[*].{name:name,exitCode:exitCode,reason:reason}}'
```

---

### Cause 3: Image pull failure

**Symptoms in events:** `CannotPullContainerError` or `ResourceInitializationError`

```bash
# Verify image exists in ECR
aws ecr describe-images \
  --repository-name $SERVICE_NAME \
  --image-ids imageTag=$IMAGE_TAG 2>&1 || echo "IMAGE NOT FOUND"
```

**Fix:** Re-push the image or roll back to previous tag via CI/CD.

---

### Cause 4: Fargate Spot interruption storm

**Symptoms:** Multiple tasks stopped simultaneously with `Host EC2 was terminated`

```bash
# Check how many tasks are On-Demand vs Spot
aws ecs list-tasks --cluster prd-b2b --service-name $SERVICE_NAME --desired-status RUNNING | \
  xargs aws ecs describe-tasks --cluster prd-b2b --tasks | \
  jq '.tasks[].capacityProviderName'
```

**Fix:** Temporarily force all tasks to FARGATE (On-Demand):
```bash
aws ecs update-service \
  --cluster prd-b2b \
  --service $SERVICE_NAME \
  --capacity-provider-strategy capacityProvider=FARGATE,weight=1,base=1
```

Revert after Spot capacity recovers.

---

### Cause 5: Deployment circuit breaker triggered

**Symptoms:** Service events show `Service was unable to perform a steady-state check, rolling back`

This means the rollback already happened. Confirm old version is running and investigate new version logs for root cause.

---

## Escalation

- **15 min**: No progress → page team lead
- **30 min**: All tasks 0 → initiate P1 bridge
- **Rollback threshold**: if P99 latency > 5s or error rate > 5% for > 10 min → rollback immediately
