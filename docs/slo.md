# SLO and error budget policy

**SLI:** probe success rate, `avg(avg_over_time(probe_success[30d]))`.
**SLO:** 99% of probes succeed over a rolling 30 days.
**Error budget:** 1% of 30 days, about 432 minutes (7.2 hours) of failed probes.

## Policy
- Budget above 50% remaining: ship changes normally.
- Budget below 50%: only low-risk changes; review recent incidents.
- Budget exhausted: freeze feature changes until availability recovers.

## Alert
`RobotShopProbeFailing` fires after 2 minutes of failure and posts to Slack with the runbook link.

## Known limit
Prometheus here keeps only 2 days of data (retention: 2d), so the 30-day query can't be computed. A real deployment would extend retention.
