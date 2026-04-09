# On-Call Handoff Checklist

> Complete this during the last 30 minutes of your on-call shift.  
> Share in `#oncall-handoff` Slack channel before handing off.

---

## Outgoing Engineer: Pre-Handoff

### 1. Open Incidents

- [ ] List all open incidents: paste PagerDuty incident list
- [ ] For each open incident:
  - Current status and owner
  - Last known state
  - Expected resolution time
  - Any ongoing mitigation keeping services up

### 2. Alert Noise

- [ ] Any alerts currently silenced/snoozed? Why? Expected end time?
- [ ] Any flapping alerts that have been acknowledged repeatedly in the last 24h?
- [ ] Any alerts that should be reviewed/updated based on this shift?

### 3. Pending Changes

- [ ] Any deployments in-flight or expected in the next few hours?
  ```bash
  gh run list --repo acme-org/inferno --status in_progress
  gh run list --repo acme-org/azmuth --status in_progress
  ```
- [ ] Any Terraform changes planned?
- [ ] Any database migrations scheduled?

### 4. Known Issues (Not Yet Incidents)

- [ ] Any services running degraded but within SLO? (e.g. elevated latency, non-critical queue backlog)
- [ ] Any external dependency issues? (Confluent Cloud, AWS, GCP, Postmark, HubSpot)

### 5. Context

- [ ] Any unusual traffic patterns expected? (campaign launches, customer demos, load tests)
- [ ] Any customers with elevated risk or recent escalations?

---

## Incoming Engineer: Post-Handoff

- [ ] Review the handoff message above
- [ ] Confirm PagerDuty schedule shows you as on-call
- [ ] Verify you can receive alerts (phone, Slack notifications enabled)
- [ ] Check current system health:
  ```bash
  # Quick health check
  for svc in azmuth bifrost merlin owlery floo pensieve scryer; do
    echo "=== $svc ===" && \
    aws ecs describe-services --cluster prd-b2b --services $svc \
      --query 'services[0].{desired:desiredCount,running:runningCount}' 2>/dev/null || \
    kubectl get deployment $svc -n prd-copilot 2>/dev/null
  done
  ```
- [ ] Confirm Grafana dashboard shows no active alerts
- [ ] Confirm you know where the runbooks are: `portfolio/sre-runbooks/runbooks/`
- [ ] Reply in `#oncall-handoff`: "Handoff received. I'm on-call."

---

## Escalation Contacts

| Role | Name | Contact | When to escalate |
|------|------|---------|-----------------|
| Team Lead | — | Slack / PagerDuty | S1-S2 after 15 min |
| Data Team | — | Slack | ClickHouse, Kafka, RDS issues |
| Infra Team | — | Slack | AWS/GCP infra, Terraform, networking |
| CTO | — | Phone | S1 only, > 30 min, or data loss risk |
| Vendors | Confluent Support | support.confluent.io | Kafka cluster issues |
| Vendors | AWS Support | Business Support | RDS, ECS, ACM issues |
