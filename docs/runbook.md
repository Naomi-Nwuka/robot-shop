# Runbook: RobotShopProbeFailing

**Meaning:** a blackbox probe against a Robot Shop endpoint has failed for 2 minutes.

## Triage
1. Open the Grafana dashboard "Robot Shop RED" and see which probe is failing.
2. `kubectl get pods` - look for pods not 1/1 Running.
3. `kubectl logs deployment/<service> --tail=50`, or search Loki: `{kubernetes_container_name="<service>"} |= "ERROR"`.
4. Open Jaeger (service `shipping`) to check for slow or failing requests.

## Mitigate first
- Pod down or crash-looping: `kubectl rollout restart deployment <service>`.
- Bad release: `helm rollback <release>`.
- Scaled to zero: `kubectl scale deployment <service> --replicas=1`.

## After mitigation
Find the root cause, then write a blameless postmortem in docs/postmortem.md.
