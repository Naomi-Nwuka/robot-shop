# Postmortem: [short title of the incident]

> **Status: template, not yet completed.** This will be filled in after the instructor runs the incident drill (step 6). The bracketed sections are placeholders.

**Date:** [date]  **Duration:** [start to end]  **Author:** Naomi Nwuka
**Status:** Resolved

## Summary
[2 to 3 sentences: what broke, who or what was affected, how it was fixed.]

## Impact
[What users or services could not do, and for how long. Which SLO probes failed.]

## Timeline (all times local)
| Time | Event |
|------|-------|
| [hh:mm] | Instructor introduced the fault (or first sign of trouble) |
| [hh:mm] | Slack alert RobotShopProbeFailing fired |
| [hh:mm] | Started triage using the runbook |
| [hh:mm] | Mitigation applied: [what] |
| [hh:mm] | Service recovered; alert RESOLVED in Slack |
| [hh:mm] | Root cause identified |

## Detection
[How we found out: Slack alert, dashboard panel, or both. Was it fast enough?]

## Mitigation
[What was done first to restore service, and the command used.]

## Root cause
[The actual underlying cause, with the evidence: a log line from Loki, a Jaeger trace, a dashboard panel.]

## What went well
- [e.g. the alert fired within 2 minutes and linked the runbook]

## What went poorly
- [e.g. no alert for X; had to search logs by hand]

## Action items
| Action | Owner | Due |
|--------|-------|-----|
| [e.g. add alert for pod restarts] | Naomi | [date] |

## Lessons learned
[What this teaches about the system. Describe causes in terms of systems and processes, not people.]
