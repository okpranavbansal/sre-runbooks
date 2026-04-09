# Runbook: ArgoCD Sync Failed

**Severity:** P2 — new deployments blocked; running services unaffected  
**Alert:** ArgoCD app shows `OutOfSync` or `Degraded` for > 10 minutes

---

## Symptoms

- ArgoCD UI shows app status as `OutOfSync`, `Degraded`, or `Unknown`
- `kubectl get application -n argocd` shows `SyncFailed`
- Deployment PR was merged but pods were not updated

---

## Diagnostic Steps

```bash
# 1. Check application status
kubectl get application -n argocd

# 2. Get detailed sync status for a specific app
kubectl describe application billing-service-prd -n argocd

# 3. Check ArgoCD repo-server logs (KSOPS/SOPS failures show here)
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=50

# 4. Check ArgoCD application-controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=50
```

---

## Common Failures and Fixes

### Failure 1: SOPS/KSOPS decryption error

**Log:** `failed to decrypt secret: no matching keys`  
**Cause:** Age decryption key secret is missing or corrupted on the repo-server.

```bash
# Check if the key secret exists
kubectl get secret ksops-age-key -n argocd

# Recreate if missing (use the correct Age private key)
kubectl create secret generic ksops-age-key \
  --from-literal=age-key.txt="AGE-SECRET-KEY-PLACEHOLDER" \
  -n argocd

# Restart repo-server to pick up new secret
kubectl rollout restart deployment argocd-repo-server -n argocd
```

### Failure 2: NEG annotation drift (false OutOfSync)

**Log:** App shows OutOfSync but no manifest changes  
**Cause:** GKE added `cloud.google.com/neg` annotations not in Git (see migration playbook).

```bash
# Verify this is the cause
kubectl describe application billing-service-prd -n argocd | grep -A5 "Diff:"

# Fix: add ignoreDifferences to ApplicationSet (permanent fix)
# Temporary: force sync ignoring differences
argocd app sync billing-service-prd --force
```

### Failure 3: Server-side apply conflict

**Log:** `Apply failed: FieldValueInvalid`  
**Cause:** Conflicting field managers (e.g. a manual `kubectl apply` vs ArgoCD server-side apply).

```bash
# Reset field manager by force-applying
kubectl apply -f <manifest> --server-side --force-conflicts
# Then trigger ArgoCD sync
argocd app sync billing-service-prd
```

### Failure 4: StatefulSet volumeClaimTemplates immutable

**Log:** `The StatefulSet "clickhouse" is invalid: spec.volumeClaimTemplates`  
**Cause:** Kubernetes does not allow changing `volumeClaimTemplates` on existing StatefulSets.

```bash
# Backup data first, then:
# 1. Scale down to 0
kubectl scale statefulset clickhouse -n prd-copilot --replicas=0

# 2. Delete StatefulSet (but NOT the PVC - reclaimPolicy: Retain protects the data)
kubectl delete statefulset clickhouse -n prd-copilot

# 3. Re-sync from ArgoCD (will recreate with new spec)
argocd app sync clickhouse-prd

# 4. Scale back up
kubectl scale statefulset clickhouse -n prd-copilot --replicas=1
```

### Failure 5: Resource quota exceeded

**Log:** `exceeded quota: prd-copilot: pods`

```bash
# Check current quota usage
kubectl describe resourcequota -n prd-copilot

# Check what's consuming quota
kubectl get pods -n prd-copilot --field-selector=status.phase=Running | wc -l
```

---

## Escalation

If sync failure persists > 30 minutes and impacts a P1 service → escalate to Platform lead.
