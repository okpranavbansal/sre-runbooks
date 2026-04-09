# Incident Response Checklist

> Copy this checklist into the incident Slack channel or Confluence page at incident start.  
> Check each item as you go. Do not skip steps under pressure.

---

## T+0: Alert fires

- [ ] Acknowledge the alert in PagerDuty (stops escalation timer)
- [ ] Confirm alert is real — not a flapping sensor or test
- [ ] Declare severity: **S1** (complete outage) / **S2** (partial degradation) / **S3** (no user impact yet)

## T+5: Triage

- [ ] Open incident channel: `#incident-YYYY-MM-DD-<slug>`
- [ ] Assign roles:
  - [ ] **Incident Commander (IC)**: coordinates, communicates, keeps timeline
  - [ ] **Tech Lead**: drives investigation and fix
  - [ ] **Comms**: updates status page and stakeholder Slack
- [ ] Post first status update in `#incidents-updates`: "We are investigating X. Impact: Y. ETA: unknown."

## T+10: Diagnose

- [ ] Check dashboards: Grafana / CloudWatch for anomalies in the 30 min before alert
- [ ] Check recent deploys: any CI/CD runs in the last 2 hours?
  ```bash
  # GitHub Actions
  gh run list --repo acme-org/$REPO --limit 10

  # ArgoCD
  argocd app history $APP_NAME
  ```
- [ ] Check external dependencies: Confluent Cloud, AWS status page, GCP status page
- [ ] Identify blast radius: which users/accounts are affected?

## T+15: Contain

- [ ] If deploy caused it → rollback immediately (don't debug in prd)
  ```bash
  # ECS rollback
  aws ecs update-service --cluster prd-b2b --service $SERVICE --task-definition $PREV_TASK_DEF

  # ArgoCD rollback
  argocd app rollback $APP_NAME $REVISION
  ```
- [ ] If traffic spike → enable rate limiting or shed load
- [ ] If data corruption risk → take snapshot before any writes
  ```bash
  aws rds create-db-snapshot --db-instance-identifier prd-wyzard --db-snapshot-identifier incident-$(date +%Y%m%d%H%M)
  ```

## T+30: Mitigate

- [ ] Verify mitigation is working: error rate decreasing? Latency recovering?
- [ ] Post update: "Mitigation deployed. Monitoring recovery."
- [ ] Confirm no residual impact (check all affected services)

## T+Resolution: Close

- [ ] Post all-clear: "Incident resolved. Impact: [duration]. Root cause: [summary]."
- [ ] Update status page to resolved
- [ ] Schedule postmortem within 48 hours
- [ ] Create Jira ticket for follow-up action items

---

## Severity Definitions

| Level | User Impact | Response Time | Escalation |
|-------|------------|---------------|-----------|
| S1 | Full outage (all users) | Immediate | IC + CTO + all team leads |
| S2 | Major feature down (>50% users) | < 15 min | IC + team lead |
| S3 | Minor degradation (<20% users) | < 1 hour | On-call only |
| S4 | No user impact (internal only) | Best effort | Ticket only |
