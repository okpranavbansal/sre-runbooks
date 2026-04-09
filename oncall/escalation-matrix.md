# On-Call Escalation Matrix

## Severity Definitions

| Severity | Definition | Response Time | Example |
|----------|-----------|---------------|---------|
| **P1** | Complete service outage or >5% error rate on user-facing APIs | 5 minutes | All API returning 500, DB connection exhaustion |
| **P2** | Partial outage or single service degraded | 30 minutes | One service CrashLoopBackOff, consumer lag growing |
| **P3** | Non-urgent degradation, no user impact | Next business day | Disk at 70%, deprecated API usage detected |

---

## Escalation Tiers

### Tier 1: On-Call Engineer (first response)

- **Who:** Rotating on-call, 24/7
- **P1 response:** 5 minutes
- **P2 response:** 30 minutes
- **Escalate to Tier 2 when:**
  - P1 not mitigated within 20 minutes
  - Root cause unknown and service impact ongoing
  - Incident requires infrastructure changes outside on-call's access

### Tier 2: Platform/SRE Lead

- **Who:** Senior SRE or Platform Lead
- **Availability:** Business hours primary; pager for P1 off-hours
- **Escalate to Tier 3 when:**
  - Data loss risk
  - Security incident
  - Coordinated response needed across teams

### Tier 3: Engineering Manager + CTO

- **Who:** EM + CTO
- **When:** P1 with significant customer impact, data breach, or when Tier 2 cannot resolve in 45 minutes

---

## Service → Team Matrix

| Service Area | Primary | Secondary |
|-------------|---------|-----------|
| API Gateway, Auth | Platform SRE | Backend lead |
| LLM Services (Bedrock) | AI Platform | Backend lead |
| Kafka / Confluent | Platform SRE | Data engineering |
| RDS, ClickHouse | Platform SRE | Backend lead |
| Neo4j | Platform SRE | Data engineering |
| GKE / ArgoCD / Infra | Platform SRE | DevOps lead |
| CloudFront / Route53 | DevOps lead | Platform SRE |
| Billing / Payments | Backend lead | EM |

---

## Communication Channels

| Channel | Used for |
|---------|---------|
| `#incidents` Slack | All active P1/P2 incidents; post status every 15 min |
| `#alerts` Slack | Automated alert feed (do not converse here) |
| `#postmortems` Slack | Share completed postmortem docs |
| Status Page | External user-facing status updates (P1 only) |
| PagerDuty | Pages on-call engineer for P1/P2 alerts |

---

## Incident Communication Template

Post in `#incidents` when you acknowledge a P1:

```
🚨 P1 INCIDENT — [service name]
Time: HH:MM UTC
Impact: [describe user impact]
Investigating: [your name]
Next update in: 15 minutes
```

Post every 15 minutes until resolved:

```
⏱ UPDATE [HH:MM UTC]
Status: Investigating / Mitigating / Resolved
Current theory: [what you think is happening]
Action taken: [what you tried]
ETA to resolution: X minutes / unknown
```
