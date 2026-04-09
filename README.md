# SRE Runbooks
> Production incident response runbooks, postmortem templates, and SLO definitions derived from real incidents.

![SRE](https://img.shields.io/badge/SRE-7B42BC?style=flat) ![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white) ![Incident Response](https://img.shields.io/badge/Incident%20Response-EF7B4D?style=flat)

![SRE](https://img.shields.io/badge/SRE-7B42BC?style=flat) ![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white) ![Incident Response](https://img.shields.io/badge/Incident%20Response-EF7B4D?style=flat)

Production incident response runbooks, postmortem templates, and SLO definitions for a B2B SaaS platform running 37 ECS services + 23 GKE workloads.

> These runbooks are derived from real incidents. Service names are sanitized.

---

## Runbooks Index

| Runbook | Trigger | Severity |
|---------|---------|----------|
| [ECS Service Unhealthy](runbooks/ecs-service-unhealthy.md) | ECS task health check failing | P2 |
| [ECS OOM Kill](runbooks/ecs-oom-kill.md) | Container OOMKilled | P2 |
| [GKE Pod CrashLoop](runbooks/gke-pod-crashloop.md) | CrashLoopBackOff on GKE | P2 |
| [Kafka Consumer Lag](runbooks/kafka-consumer-lag.md) | Consumer lag > threshold | P2/P3 |
| [RDS Connection Exhaustion](runbooks/rds-connection-exhaustion.md) | MySQL max_connections hit | P1 |
| [Neo4j Disk Full](runbooks/neo4j-disk-full.md) | EBS volume at capacity | P1 |
| [ClickHouse Query Timeout](runbooks/clickhouse-query-timeout.md) | Slow query on GKE StatefulSet | P2 |
| [ArgoCD Sync Failed](runbooks/argocd-sync-failed.md) | ArgoCD app OutOfSync or Degraded | P2 |
| [ALB / Gateway 5xx Spike](runbooks/alb-5xx-spike.md) | Error rate > SLO threshold | P1 |
| [Certificate Expiry](runbooks/certificate-expiry.md) | ACM / GKE cert-map near expiry | P1 |

---

## Templates

- [Postmortem Template](templates/postmortem-template.md)
- [Incident Response Checklist](templates/incident-response-checklist.md)

## SLOs

- [SLO Definitions](slo/slo-definitions.yaml)
- [Error Budget Policy](slo/error-budget-policy.md)

## On-Call

- [Escalation Matrix](oncall/escalation-matrix.md)
- [Handoff Checklist](oncall/handoff-checklist.md)

---

## How to Use These Runbooks

1. **Alert fires** → Check alert name against the index above
2. **Open runbook** → Follow the diagnostic steps in order
3. **Escalate if needed** → Use the [Escalation Matrix](oncall/escalation-matrix.md)
4. **Resolve + document** → File a postmortem using the [template](templates/postmortem-template.md) for P1/P2 incidents
5. **Improve** → Add a "Prevention" entry to the runbook after each incident

## Author

**Pranav Bansal** — AI Infrastructure & SRE Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://linkedin.com/in/okpranavbansal)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/okpranavbansal)