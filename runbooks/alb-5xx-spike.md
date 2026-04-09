# Runbook: ALB / GKE Gateway 5xx Spike

**Severity:** P1 — user-facing errors, SLO burn rate accelerating  
**Alert:** `HTTPErrorRate_5xx > 5% for 5m` on any public endpoint

---

## Symptoms

- Users reporting errors in the application
- CloudWatch / Cloud Monitoring showing elevated 5xx rate
- Error budget burning faster than normal

---

## Triage (first 5 minutes)

```bash
# AWS: Check ALB 5xx metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=<alb-arn-suffix> \
  --start-time $(date -u -d '-15 minutes' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Sum

# GKE: Check which service is returning 5xx
kubectl top pods -n prd-copilot
kubectl get events -n prd-copilot --sort-by='.lastTimestamp' | tail -30

# Which HTTPRoute is affected?
# Check Cloud Monitoring: HTTP/Request count by backend_target_name
```

---

## Diagnosis by Pattern

### Pattern A: 5xx on specific service only

```bash
# Get logs from that service
kubectl logs -n prd-copilot -l app=billing-service --tail=100 | grep -i 'error\|panic\|exception'

# Check if pods are healthy
kubectl get pods -n prd-copilot -l app=billing-service
```

→ Follow [GKE Pod CrashLoop runbook](gke-pod-crashloop.md) if pods are crashing.

### Pattern B: 5xx spiked after a deployment

```bash
# Identify what changed in the last 30 minutes
# Check ArgoCD sync history
argocd app history billing-service-prd

# Rollback via Git revert and push, OR use ArgoCD to roll back:
argocd app rollback billing-service-prd <revision-id>
```

### Pattern C: 5xx across all services (infrastructure problem)

**Possible causes:** GKE Gateway NEG issue, node pool problem, cert expiry.

```bash
# Check Gateway status
kubectl describe gateway prd-copilot-gateway -n prd-copilot

# Check if all backend services have healthy NEGs
gcloud compute backend-services list --project=prj-acme-prd

# Check GKE node health
kubectl get nodes
kubectl describe nodes | grep -E 'Condition|NotReady'
```

### Pattern D: 5xx from Fargate Spot interruption (ECS environments)

Spot task interrupted while handling requests → ALB targets briefly unhealthy.

```bash
# Check ECS service events
aws ecs describe-services \
  --cluster prd-platform \
  --services billing-service \
  --query 'services[0].events[:5]'

# Add SIGTERM handler to drain connections gracefully (see k8s-cost-optimizer)
```

---

## SLO Impact Assessment

```
Error rate: X%
SLO target: 99.9% availability (43.8 min/month error budget)
Burn rate: X× (at this rate, budget exhausted in Y hours)
```

If burn rate > 14× → wake up the on-call engineer immediately.

---

## Prevention

- [ ] Canary deployments (10% traffic to new version before 100%)
- [ ] ALB `deregistration_delay = 30` (not 300) for faster Spot task drain
- [ ] Health check grace period: `HealthCheckGracePeriodSeconds = 60` on ECS
- [ ] PodDisruptionBudget on GKE services
- [ ] Error budget alerting at 5% consumed per hour (not just raw error rate)
