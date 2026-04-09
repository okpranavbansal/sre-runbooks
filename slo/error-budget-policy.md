# Error Budget Policy

## What Is an Error Budget?

An error budget is the allowed amount of downtime or errors before an SLO is breached.

```
Error Budget = 1 - SLO target
Example: 99.9% SLO → 0.1% error budget → 43.8 min/month allowed downtime
```

---

## Our SLOs and Error Budgets

Measurement window: **30-day rolling**. See `slo/slo-definitions.yaml` for the full YAML definitions.

| Service | SLI | Target | Error Budget/Month |
|---------|-----|--------|--------------------|
| Copilot API (availability) | Request success rate (non-5xx) | 99.9% | 43.8 min |
| Copilot API (P99 latency) | P99 < 2s | 99.5% | 3.6 hours |
| LLM Orchestrator (availability) | Service availability | 99.5% | 3.6 hours |
| LLM Orchestrator (latency) | First-token P95 < 5s | 99.5% | 3.6 hours |
| Email delivery (Owlery) | Successful delivery rate | 99.5% | 3.6 hours |
| Data ingestion pipeline | Message processing success | 99.0% | 7.2 hours |

---

## Error Budget Consumption Rates

We track error budget consumption on a **30-day rolling window** (consistent with SLO definitions).

### Severity vocabulary

Alerts use P-levels; incident templates use S-levels. They map as follows:

| Alert severity | Incident severity | Meaning |
|---------------|-------------------|---------|
| P1 | S1 | Full outage — page everyone, stop feature work |
| P2 | S2 | Major degradation — page on-call, freeze deploys |
| P3 | S3 | Minor, no user impact — investigate async |

### Budget burn rate thresholds

| Burn Rate | Alert | Action |
|-----------|-------|--------|
| > 14.4x normal | Page immediately (P1/S1) | Stop all non-critical work, focus on reliability |
| > 6x normal | Page (P2/S2) | Freeze feature deployments, investigate |
| > 1x normal | Warning in Slack | Review deployment velocity |

**Burn rate formula:**
```
burn_rate = (error_rate_in_window / error_budget) / (window_duration / SLO_period)
```

---

## Policy: When Error Budget Is Low

### Budget > 50% remaining
- Normal deployment velocity
- Feature development proceeds as usual

### Budget 25-50% remaining
- Review deployment practices
- Add extra caution to risky changes
- Require second approval for production deploys

### Budget 10-25% remaining
- **Deployment freeze**: no new features to production
- SRE team takes ownership of all production changes
- Daily error budget review in team standup

### Budget < 10% remaining
- Full deployment freeze
- Page team lead immediately
- Root cause analysis required before any production change
- Error budget review in weekly leadership sync

---

## Error Budget Recovery

When budget is exhausted:

1. Conduct blameless postmortem within 48 hours
2. Create reliability improvement tickets (minimum 3 action items)
3. Require at least 50% budget recovery before resuming feature velocity
4. Add tracking issue for recurring causes

---

## PrometheusRule for Budget Burn Alert

```yaml
groups:
  - name: error-budget
    rules:
      - alert: ErrorBudgetBurnFast
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[1h])) /
            sum(rate(http_requests_total[1h]))
          ) / 0.001 > 14.4
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Fast error budget burn: {{ $value | humanizePercentage }} error rate"
          description: "At this rate, monthly error budget exhausted in < 1 hour"

      - alert: ErrorBudgetBurnMedium
        expr: |
          (
            sum(rate(http_requests_total{status=~"5.."}[6h])) /
            sum(rate(http_requests_total[6h]))
          ) / 0.001 > 6
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Moderate error budget burn over 6h window"
```
