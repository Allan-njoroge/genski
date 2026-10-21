# Alerting and Dashboards

Instrumentation is only useful if someone notices when it shows a problem. Alerts tell people when to act; dashboards help them understand what's happening. Both need careful design, because too much is as harmful as too little.

## Contents

1. Principles of good alerting
2. Symptoms and causes
3. Alerting on SLOs
4. Thresholds and noise
5. Severity and routing
6. Runbooks
7. Dashboards
8. Reviewing alerts

## 1. Principles of good alerting

Every alert that pages a person should be:

- **Urgent:** it needs attention now, not tomorrow.
- **Important:** users or the business are affected, or soon will be.
- **Actionable:** there is something the responder can do.
- **Real:** it reflects an actual problem, not a measurement glitch.

If an alert fails any of these, it should be a ticket, a dashboard item, or deleted. Alerts that fire and get ignored train people to ignore every alert, including the one that matters.

## 2. Symptoms and causes

**Symptoms** are what users experience: errors, slowness, failed purchases, delayed notifications. **Causes** are internal conditions that might lead to symptoms: high CPU, a full disk, a restarted instance.

**Page on symptoms:**

- Error rate for a critical endpoint or flow is high.
- Latency for a critical endpoint is above target.
- A critical business flow has stopped (no orders in the last 15 minutes during business hours).
- A job queue is growing, and the oldest item is older than acceptable.
- Messages are accumulating in the dead-letter queue.

**Use causes for diagnosis, not pages,** with exceptions for problems that will certainly cause an outage soon and can be prevented, such as a disk about to fill or a certificate about to expire.

Alerting on symptoms catches problems from any cause, including ones you didn't predict, with fewer alerts.

## 3. Alerting on SLOs

When the service has reliability targets (see `backend-reliability`), alert on how fast the error budget is being consumed (burn rate), rather than on raw thresholds:

- **Fast burn:** the budget would be exhausted within hours or days at the current rate → page someone.
- **Slow burn:** the budget would be exhausted within the period if this continues → create a ticket.

Combining a short and a longer time window for each condition avoids alerts for brief spikes while still catching sustained problems quickly. The Google *Site Reliability Workbook* chapter "Alerting on SLOs" describes the approach in detail.

## 4. Thresholds and noise

- **Base thresholds on real behaviour and targets,** not guesses. Look at the metric's history.
- **Require the condition to last** for a period (for example, 5 minutes), so brief spikes don't page anyone.
- **Use rates and ratios, not absolute counts:** 5% of requests failing, not 50 errors, which means different things at different traffic levels.
- **Account for low traffic:** a single failure out of two requests is 50%. Require a minimum volume, or use longer windows at quiet times.
- **Deduplicate and group** related alerts, so one incident produces one notification.
- **Alert on the absence of data** for critical signals: a service that stops reporting metrics may be completely down.

## 5. Severity and routing

| Severity | Meaning | Route to |
|---|---|---|
| Critical / page | Users significantly affected now; act immediately | On-call person, any time |
| Warning / ticket | Needs attention soon, not urgently | Team's ticket queue or channel, during working hours |
| Informational | Worth knowing; no action required | Dashboard or log, not a notification |

Use as few levels as possible, and route each to the people who can act on it.

## 6. Runbooks

Every paging alert links to a runbook: a short guide for the responder.

```markdown
# Alert: Checkout error rate high

## What it means
More than 2% of checkout requests have failed for 5 minutes.

## Impact
Customers can't complete purchases.

## First checks
1. Dashboard: [link]. Which errors? Which endpoint?
2. Recent deployments: [link]. Did this start after a release? → consider rolling back.
3. Payment provider status: [link]. Circuit breaker state on the dashboard.
4. Database: connection pool saturation and slow queries on the dashboard.

## Actions
- Bad deployment → roll back (procedure: [link]).
- Payment provider outage → confirm degraded-mode message is showing; notify [team/channel].
- Database saturation → [steps].

## Escalation
[Who to contact, and when.]
```

Keep runbooks next to the code or in the team's documentation, and update them after every incident in which they were used.

## 7. Dashboards

A good dashboard answers "is it healthy, and if not, where's the problem?" within seconds.

- **Start with a service overview:** request rate, error rate, and latency percentiles (RED) for the main operations, plus key business metrics.
- **Then dependencies:** latency and error rates for each database, cache, and external service; circuit breaker states.
- **Then resources and queues:** pool saturation, queue lengths, job failure rates.
- **Mark deployments** on graphs, so changes in behaviour can be matched to releases.
- **Show targets** (SLO thresholds) on relevant graphs.
- **Keep it focused.** A dashboard with fifty graphs is hard to read under pressure. Create detail dashboards for deeper investigation, linked from the overview.
- **Use consistent time ranges and units** across panels.
- Keep dashboards as code or configuration in version control where the platform supports it.

## 8. Reviewing alerts

Alerting needs regular maintenance:

- After each incident, check: did an alert fire? Was it early enough? Was the runbook useful?
- Periodically review alerts that fired: which led to action, and which were ignored or noisy? Tune or remove the noisy ones.
- When a new critical feature launches, add its alerts and runbook as part of the launch.