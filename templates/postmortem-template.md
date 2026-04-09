# Postmortem: [Incident Title]

**Date:** YYYY-MM-DD  
**Severity:** P1 / P2  
**Duration:** HH:MM  
**Services affected:** [list of services]  
**Author:** [name]  
**Status:** Draft / Final  

---

## Summary

_One paragraph: what happened, what was the user impact, how was it resolved._

---

## Impact

| Metric | Value |
|--------|-------|
| Duration | X hours Y minutes |
| Users affected | ~N (or % of traffic) |
| Requests errored | ~N |
| SLO error budget consumed | X% of monthly budget |
| Revenue impact | Estimated $X (if applicable) |

---

## Timeline

All times in UTC.

| Time | Event |
|------|-------|
| HH:MM | Alert fired: `<alert name>` |
| HH:MM | On-call engineer acknowledged |
| HH:MM | [First diagnostic step taken] |
| HH:MM | Root cause identified |
| HH:MM | Mitigation applied |
| HH:MM | Services recovering (error rate < 1%) |
| HH:MM | Incident resolved |

---

## Root Cause

_Describe the technical root cause. Be specific: what failed, why it failed, what conditions triggered it._

---

## Contributing Factors

_List factors that made the incident worse or harder to detect/resolve:_

- Missing alert at 70% threshold (only alerted at 95%)
- No runbook for this failure mode
- On-call engineer unfamiliar with the affected service

---

## What Went Well

- Alert fired within 3 minutes of impact
- Rollback procedure was documented and executed in < 5 minutes
- Communication to stakeholders was timely

---

## What Went Poorly

- Root cause took 45 minutes to identify (logs not surfaced in single dashboard)
- Runbook was out of date
- Mitigation required manual steps that could have been automated

---

## Action Items

| Action | Owner | Due date | Priority |
|--------|-------|----------|----------|
| Add alert at 70% threshold | SRE team | YYYY-MM-DD | High |
| Write runbook for X failure mode | [owner] | YYYY-MM-DD | Medium |
| Automate Y mitigation step | [owner] | YYYY-MM-DD | Medium |
| Update architecture diagram | [owner] | YYYY-MM-DD | Low |

---

## Lessons Learned

_What would you tell your past self to prevent this incident?_

---

## Appendix

### Relevant Logs

```
[paste relevant log snippets here]
```

### Graphs

_Link to CloudWatch / Grafana dashboard screenshots from the incident window._
