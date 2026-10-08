# Incident Response

> **TL;DR:** Detect → triage → mitigate → resolve → postmortem. Have runbooks. Page on symptoms, not causes. Run blameless retros. Track DORA + MTTR + alert fatigue as KPIs.

## The Five-Phase Lifecycle

```
1. DETECT      — alarm fires, customer reports, internal observation
2. TRIAGE      — assess severity, page on-call, set up war room
3. MITIGATE    — restore service (rollback, scale, failover, flag-off)
4. RESOLVE     — root cause + permanent fix
5. POSTMORTEM  — blameless review, action items, share with team
```

**Mitigation > root cause during an incident.** Save root cause for after.

## Severity Levels

Pick 3-4 max. Example:

| Severity | Definition | Response |
|----------|------------|----------|
| **SEV1** | Service is down or major data integrity issue | Page IC + leadership; war room; status page |
| **SEV2** | Significant degradation; subset of users affected | Page IC; investigate immediately |
| **SEV3** | Minor degradation or workaround exists | Working-hours response |
| **SEV4** | Cosmetic / low-impact | Ticket queue |

Define them with concrete examples — "checkout 5xx > 5% for 2 minutes = SEV1".

## On-Call Rotation

- Weekly rotations, 1 primary + 1 secondary.
- Handoff at a defined time with a verbal review of open issues.
- **Follow-the-sun** for distributed teams reduces night-time pages.
- Compensate / give time-in-lieu for actual paged work — sustainable on-call is paid on-call.
- Onboard new on-calls with shadowing + a runbook tour.

### Limits to set

- Maximum N hours per week on-call.
- After a paged night, the engineer's day starts later (or off entirely).
- An incident in non-business hours is a tracked metric — push to reduce it.

## Paging Tools

| Tool | Notes |
|------|-------|
| PagerDuty | Industry default |
| Opsgenie | Atlassian's; tight Jira integration |
| Splunk On-Call (VictorOps) | Mature integrations |
| Grafana OnCall | Open source / managed in Grafana Cloud |
| BetterStack / Squadcast | Lighter weight |

Connect: alertmanager → pager service. Page selectively (use severities and time-of-day routing rules).

## Runbooks

A runbook is **the procedure to handle a specific alert / incident type**, written before the incident.

```markdown
# RUNBOOK: API High 5xx Rate

## Symptom
api 5xx > 2% over 10 min (alert: HighErrorRate)

## Likely causes
- Downstream DB unreachable
- Recent deploy regression
- Upstream provider (Stripe, S3) outage
- Memory leak / OOM kills

## Triage steps
1. Check Grafana dashboard: https://grafana.example.com/d/api
2. Check recent deploys: https://argo.example.com/applications/api
3. Check downstream services dashboard
4. Check error log query: `service=api level=error` (last 15 min)
5. Check Stripe + AWS status pages

## Mitigation
- **Recent bad deploy?** `kubectl rollout undo deploy/api -n prod`
- **DB slow?** Scale read replicas; check long-running queries
- **OOM?** Scale up `kubectl scale deploy/api --replicas=10`; investigate leak
- **Upstream outage?** Communicate; degrade gracefully (feature flag)

## Escalation
- After 15 min without mitigation, page the API team lead.
- After 30 min, declare SEV1, open status page incident.

## Postmortem owner
Whoever is IC at resolution time.
```

Runbooks live in the same repo as the service code (or a dedicated runbooks repo). Link from the alert annotation.

## Incident Command

For SEV1/SEV2, assign roles explicitly:

- **IC (Incident Commander):** runs the incident. Not technical lead — they coordinate.
- **Tech Lead:** drives technical mitigation.
- **Comms:** updates status page, internal Slack, customers.
- **Scribe:** captures timeline, decisions, actions in real time (for the postmortem).

Even for a small team, the IC role separates "drive the response" from "fix the bug."

## War Room Hygiene

- Dedicated channel (Slack `#incident-2026-05-20-api-5xx`). All comms there.
- Pin: current status, latest action, IC name.
- Update every 15-30 min even when nothing's new — silence breeds panic.
- Avoid voice-only — written record helps the scribe.

## Status Page

When customer-facing, update **early** and **often**. Customers tolerate honest "we're looking at it" far better than silence.

```
13:42 - Investigating elevated error rates on checkout.
13:55 - Identified — DB primary failover in progress. Some checkout requests will fail until ~14:10.
14:12 - Service restored. Monitoring.
14:30 - Resolved. Postmortem to follow within 5 business days.
```

Statuspage.io, BetterStack, Atlassian Statuspage, GitHub Status are common tools.

## Postmortems — Blameless

Within 5 business days, publish a postmortem. Template:

```markdown
# Postmortem — API 5xx Spike, 2026-05-20

## Summary
Between 13:42 and 14:10 UTC (28 min), the /api/checkout endpoint
returned 5xx for ~18% of requests. Root cause: a primary DB failover
triggered by an underlying EBS hardware issue. Mitigated by Aurora's
automatic failover. ~3000 checkout attempts impacted.

## Severity & Impact
SEV2 — 18% checkout failure for 28 minutes.

## Timeline (UTC)
13:42  Alert HighErrorRate fired
13:43  Page received by Kushal (primary on-call)
13:44  #incident-2026-05-20-api-5xx opened
13:48  IC identified DB connection errors in api logs
13:51  Confirmed primary failover via RDS console
14:08  Failover complete, errors trending down
14:10  Errors back to baseline
14:30  Status page resolved

## Root cause
AWS-side EBS issue triggered Aurora to failover the primary. Our app
connection pool didn't retry transient errors → user-visible 5xx instead
of automatic recovery.

## What went well
- Alert fired in 1 min of the spike
- On-call response within 1 min of page
- Aurora failover completed in <30s

## What went poorly
- No retry on transient DB errors in the connection pool
- Status page updated 13 min after incident start (too late)
- Runbook for "DB failover" was missing — we improvised

## Action items
- [ ] AI-1: Add retry-with-backoff for transient DB errors  (@kushal, by 2026-05-27)
- [ ] AI-2: Add "DB failover" runbook                       (@anil,   by 2026-05-25)
- [ ] AI-3: Update status page SLA: post within 5 min       (@team,   by 2026-05-23)
- [ ] AI-4: Chaos test: kill DB primary in staging monthly  (@kushal, by 2026-06-15)

## Lessons learned
Aurora's fast failover is great, but our app must cope with transient
connection errors. Resilience > avoidance.
```

**Blameless** means: focus on systemic factors, not "Kushal pushed bad code." Humans operate in a system that allowed the error.

## Action Items — the Whole Point

A postmortem with no AIs is performative. AIs must:

- Have an owner.
- Have a deadline.
- Be tracked to completion (Jira / Linear).
- Be reviewed at follow-up retros.

If your team carries 100 stale AIs, you're not learning. Pick the highest-leverage ones, ship them, then plan next.

## SLO / SLI / SLA — How Reliability Connects

- **SLI (indicator):** what you measure (e.g., availability = successful requests / total).
- **SLO (objective):** internal target (99.9% over 30 days).
- **SLA (agreement):** external contract with penalty (99.5%).
- **Error budget = 1 − SLO.** 99.9% = 43 min/month of allowed downtime.

Track **burn rate**:
- 2% of budget burned in 1 hour → SEV2-level page.
- 5% in 6 hours → SEV2.
- 10% in 3 days → SEV3 ticket.

When budget is exhausted, freeze risky launches until it recovers. Treat reliability as a product feature with a budget like any other.

## Common Incident Categories

| Category | Mitigation |
|----------|------------|
| Bad deploy | Rollback (the #1 mitigation) |
| DB performance | Scale reads / kill slow queries / failover |
| Cache outage | Degrade gracefully / restart cache / fall back to DB |
| Upstream provider outage | Feature flag / queue / degrade |
| DDoS / abuse | Rate limit / WAF rules / block |
| Cert expiry | Renew + restart; monitor cert expiry proactively |
| DNS misconfig | Roll back DNS / wait TTL |
| Secrets leaked | Rotate + audit access + paper trail |

## Chaos Engineering — Reliability Before the Incident

Inject failures **on purpose** in controlled experiments:
- Kill a Pod (`chaos-mesh`, Litmus, Gremlin).
- Block network to a dependency.
- Latency injection.
- DB failover.
- Region failure.

Goal: validate your assumptions (rolling restarts work, autoscaler kicks in, retries handle blip). Cheaper than a real incident.

## Metrics to Track

- **MTTD** — Mean Time To Detect (alarm to detection).
- **MTTR** — Mean Time To Resolve.
- **Incident count by severity.**
- **DORA:** deployment frequency, lead time, change failure rate, MTTR.
- **% incidents with completed AIs.**
- **Alert noise:** pages per week per engineer; aim for <2.

## Interview Questions

**Q: Walk me through how you handle a production incident.**
A: Detect (alarm or report) → page on-call → triage (severity? scope?) → mitigate (rollback / scale / failover — don't root-cause yet) → resolve → blameless postmortem within 5 days with concrete action items. Update status page proactively, assign IC / Tech Lead / Comms / Scribe for SEV1/SEV2.

**Q: Why blameless postmortems?**
A: Punishing individuals discourages honest reporting and learning. Systems allowed the error — fix the system. Blame leads to hiding incidents, less psychological safety, lower reliability over time.

**Q: SLO vs SLA?**
A: SLO is internal target; SLA is external contract with financial penalty. SLA < SLO < internal goal — gives you a buffer. Error budget = 1 - SLO. Burn rate tells you to slow risky changes.

**Q: What's an action item that doesn't suck?**
A: Specific, owned, deadlined, measurable. "Improve reliability" — bad. "Add `retryOnTransientErrors` to the DB client by 2026-05-27, owned by Kushal, demoed in next sprint review" — good.

**Q: How do you reduce alert fatigue?**
A: (1) Alert on symptoms (RED), not causes. (2) Every page must be actionable; otherwise demote to ticket. (3) Tune `for:` durations to avoid flapping. (4) Group / inhibit related alerts. (5) Quarterly review: delete unused / flaky alerts.

**Q: How do you prepare for incidents that haven't happened yet?**
A: Runbooks for known classes (DB failover, region outage). Chaos engineering experiments. Game day exercises. Postmortems from past incidents stored for new on-calls to read. Onboarding includes shadowing on-call.

## Common Pitfalls

- Chasing root cause before mitigating — extended outage. Mitigate first.
- No runbook → improvising → mistakes → bigger blast radius.
- Silent status page → angry customers + lost trust.
- Postmortem with no concrete AIs — performative.
- Same incident repeating quarterly — AIs from last time were never done.
- Blameful retros → engineers hide problems → quiet decay.
- Hero culture — same engineer fixes every incident, burns out, leaves.

## Related

- [18-monitoring-and-observability.md](18-monitoring-and-observability.md)
- [19-logging-best-practices.md](19-logging-best-practices.md)
- [10-ci-cd-fundamentals.md](10-ci-cd-fundamentals.md)
- [21-security-devsecops.md](21-security-devsecops.md)
- [../serious-prep/](../serious-prep/)
