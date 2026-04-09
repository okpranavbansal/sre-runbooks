# Runbook: GKE Pod CrashLoopBackOff

**Severity:** P2 — service degraded; dependent on number of replicas affected  
**Alert:** `kube_pod_container_status_restarts_total > 5 in 10m`

---

## Symptoms

- `kubectl get pods -n prd-copilot` shows `CrashLoopBackOff` status
- ArgoCD app shows `Degraded`
- Service returning 503 (if all replicas are crashing)

---

## Diagnostic Steps

```bash
# 1. Identify crashing pods
kubectl get pods -n prd-copilot | grep -E 'CrashLoop|Error|OOMKilled'

# 2. Get last few log lines before crash
kubectl logs -n prd-copilot <pod-name> --previous --tail=50

# 3. Describe pod for events (OOM, image pull errors, probe failures)
kubectl describe pod <pod-name> -n prd-copilot

# 4. Check recent events in the namespace
kubectl get events -n prd-copilot --sort-by='.lastTimestamp' | tail -20
```

---

## Common Causes

### Cause 1: OOMKilled (memory limit exceeded)

**Signs:** `kubectl describe pod` shows `OOMKilled`, exit code 137

```bash
# Temporary: increase memory limit in the overlay
# apps/billing-service/overlays/prd/deployment.yaml
resources:
  limits:
    memory: "2Gi"   # increase from 1Gi

# Push to Git → ArgoCD auto-syncs

# Also investigate: is this a memory leak or under-provisioning?
# Check memory growth trend in Cloud Monitoring
```

### Cause 2: Failed readiness/liveness probe

**Signs:** `Liveness probe failed: Get "http://...": connection refused`

```bash
# Check if the app started correctly
kubectl logs <pod-name> -n prd-copilot | grep -i 'listen\|start\|ready\|error'

# If app starts slowly: increase initialDelaySeconds
livenessProbe:
  initialDelaySeconds: 60  # was 30
```

### Cause 3: Missing secret or ConfigMap

**Signs:** `Error: secret "billing-service-secret" not found`

```bash
# Check if the secret exists
kubectl get secret billing-service-secret -n prd-copilot

# If missing: the ArgoCD KSOPS sync may have failed
# Check ArgoCD repo-server logs for SOPS errors
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=30
```

### Cause 4: Bad image / image pull error

**Signs:** `ImagePullBackOff` or `ErrImagePull`

```bash
# Check image reference
kubectl describe pod <pod-name> | grep Image:

# Verify image exists in Artifact Registry
gcloud artifacts docker images list \
  asia-southeast1-docker.pkg.dev/prj-acme-prd/copilot/billing-service

# Fix: revert to last known good image tag in Git
```

### Cause 5: Stakater Reloader triggered hot reload

**Signs:** Multiple pods restarting simultaneously after a secret/config change

This is expected behavior. Reloader detects the secret change and rolls the pods. Verify the new config is correct and wait for the rollout to complete.

```bash
kubectl rollout status deployment/billing-service -n prd-copilot
```

---

## Prevention

- [ ] PodDisruptionBudget on all production services (minimum 1 always available)
- [ ] `resources.requests` and `resources.limits` set on all containers
- [ ] Liveness probe `failureThreshold: 3` before restart (not 1)
- [ ] Resource alerts: alert on OOM kill events, not just pod restarts
