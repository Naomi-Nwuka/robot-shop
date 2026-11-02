# Troubleshooting log

The main problems hit while building the Capstone 2 observability stack on EKS, each with the symptom, cause, fix and lesson.

## 1. CoreDNS never becomes ready (missing kube-proxy add-on)

**Symptom:** The two CoreDNS pods stayed `0/1 Running`. Readiness probes failed with HTTP 503 and the logs repeated:

~~~text
plugin/kubernetes: Failed to watch ... dial tcp 10.100.0.1:443: i/o timeout
~~~

**Cause:** `10.100.0.1` is the ClusterIP of the `kubernetes` service, a virtual address that kube-proxy turns into the real API server addresses on every node. `aws eks list-addons` showed only `coredns` and `vpc-cni`, so no kube-proxy ran and nothing translated the address. Pods still got IPs because the CNI was healthy.

**Fix:**

~~~bash
aws eks create-addon --cluster-name voting-app-capstone --addon-name kube-proxy --region us-east-1
kubectl get ds -n kube-system                      # kube-proxy should show 4/4
kubectl rollout restart deployment/coredns -n kube-system
~~~

CoreDNS reached `1/1 Running` within seconds.

**Lesson:** If CoreDNS cannot reach the API through the ClusterIP, check for kube-proxy first. `kubectl get ds -n kube-system` shows it in one command.

## 2. Services that started during the DNS outage kept dead connections

**Symptom:** After the fix every pod showed `1/1 Running`, but `web` had 193 restarts and `shipping` had 69. Later, "Add to cart" failed on Kubernetes although the same images worked in Docker.

**Cause:**
- `web` logged `host not found in upstream "catalogue"` and `shipping` logged a Hibernate error (it could not reach MySQL). Both started while DNS was down.
- For the cart, the browser's request reached cart as `/add/STAN-1/1`, with no user ID, so cart returned a plain 404. The ID comes from the `user` service, whose `/uniqueid` returned `{"command":"INCR","code":"NR_CLOSED"}`: a closed Redis connection from the outage. Its health check does not report Redis, so it looked healthy.
- My first test SKU (`HAL-1`) did not exist in the dataset, so "product not found" proved nothing. Use `curl http://catalogue:8080/products` to see real SKUs.

**Fix:** `kubectl rollout restart deployment web shipping user`. `/uniqueid` then returned `{"uuid":"anonymous-1"}` and checkout worked.

**Lesson:** Restart counters keep history and do not reset. After a cluster-wide network fix, restart every workload that started during the outage, because most apps do not recover broken connections on their own. An empty segment in a request URL (`/add//...`) points at a failed call earlier in the chain.

## 3. No request metrics to build the dashboard from

**Symptom:** `curl http://<service>:8080/metrics` showed only business counters on `cart` and `payment`, an HTML page on `catalogue` and `user`, and a 404 on `web`.

**Cause:** The Robot Shop services expose no HTTP request, error or latency metrics.

**Fix:** Install the Prometheus blackbox exporter and probe the shop's endpoints. `probe_success` and `probe_duration_seconds` feed the dashboard and the SLO. The probe URLs use fully qualified names (`web.default.svc.cluster.local`), because the exporter runs in the `monitoring` namespace and short names resolve only inside their own namespace.

**Lesson:** Check what a service actually exposes before designing a dashboard around it. Synthetic probes measure availability and latency, not real user traffic.

## 4. Grafana stuck at 2/3 and restarting

**Symptom:** The Grafana pod stayed at `2/3` with a rising restart count. Events showed `connection refused` on port 3000, then `Killing ... failed liveness probe`.

**Cause:** Grafana 13 loads many internal apps at startup and is slow on a `t3.small`. It had not opened port 3000 before the liveness probe killed it. There was no crash and no error in its logs.

**Fix:** `monitoring/grafana-slow-start.yaml` raises its CPU and sets a 300-second liveness delay. Apply it with `helm upgrade` and pass every earlier values file again. The new pod reached `3/3` with zero restarts.

**Lesson:** A pod killed by its liveness probe with no error in its logs is probably slow, not broken. Read the timestamps. When running `helm upgrade`, repeat all earlier `-f` files, or the earlier settings are lost.

## 5. Loki in CrashLoopBackOff

**Symptom:** `loki-0` crashed repeatedly with:

~~~text
mkdir /var/loki: read-only file system
error initialising module: ruler-storage
~~~

**Cause:** Persistence was off (the cluster has no EBS CSI driver), so Loki had no writable folder at `/var/loki` on its read-only container filesystem.

**Fix:** `monitoring/loki-storage-fix.yaml` mounts a temporary `emptyDir` volume at `/var/loki`, applied as an extra values file in `helm upgrade`. Loki became `1/1` and logs flowed.

**Lesson:** Turning off persistence also removes the writable directory, so provide an `emptyDir`. The cost is that logs are lost when the pod restarts.

## 6. shipping evicted when adding OpenTelemetry

**Symptom:** After patching `shipping` to load the OpenTelemetry Java agent, the old pod went to `Error` and a replacement from the old version appeared. The rollout looked stuck.

**Cause:** `kubectl describe pod` showed:

~~~text
Evicted: The node was low on resource: memory. Threshold quantity: 100Mi, available: 70248Ki.
Container shipping was using 961296Ki, request is 500Mi
~~~

Exit code 137 was an external kill, not an app crash. The default rolling update ran two Java processes at once on a `t3.small` (about 1.4 GB usable), and the pod furthest over its memory request was evicted.

**Fix:**
- `kubectl patch deployment shipping -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'`, so the old pod stops before the new one starts.
- `kubectl delete pod <evicted pod>` to remove the leftover record.

The agent then logged `opentelemetry-javaagent - version: 2.33.1` and Jaeger listed `shipping`.

**Lesson:** On small nodes, a rolling update of a memory-heavy app can evict it. Use `Recreate` for these, and read the Events of `kubectl describe pod` first.

## 7. New Prometheus rule not visible

**Symptom:** `RobotShopProbeFailing` was missing from the Prometheus rules page, and a query through the Kubernetes API proxy failed with `dial tcp ...:9090: connect: connection timed out`.

**Cause:**
- The API proxy route needs the control plane to reach the pod on port 9090, which the node security group does not allow. That failure says nothing about the rule.
- The browser page was stale. Querying Prometheus directly showed the rule loaded (`"duration":120`, label `team: capstone`).

**Checks that helped:** the rule's labels matched Prometheus's `ruleSelector` (`release=monitoring`), the Prometheus logs had no errors, and `kubectl get cm -n monitoring -o yaml | grep -c RobotShopProbeFailing` returned 1, showing the operator had written the rule.

**Fix:** Use a port-forward and ask Prometheus directly: `curl -s localhost:9090/api/v1/rules`. A test (`kubectl scale deployment payment --replicas=0`) then produced the Slack alert, and scaling back produced `RESOLVED`.

**Lesson:** When something is "not there", ask the component itself before changing config.

## 8. Runbook link in the alert returned 404

**Symptom:** The link in the Slack alert opened a GitHub 404 page.

**Cause:** I had written the link with branch `main`. The fork's branch is `master` (`git branch --show-current`), inherited from the upstream repo.

**Fix:** Change `/blob/main/` to `/blob/master/` in `monitoring/probe-alert.yaml`, run `kubectl apply -f`, commit and push.

**Lesson:** Check the real branch name (`git ls-remote --heads origin`) before writing links, and open the link once after the alert exists.

## 9. Payment returns HTTP 500 (expected, not a fault)

**Symptom:** Every `POST /api/payment/pay/partner-57` returned 500, with `'NoneType' object is not iterable` in the payment log.

**Cause:** The load generator runs with `ERROR=1` and sends a cart with no `items` for that user, on purpose. Real users' payments in the same log had real totals, a successful PayPal call (`returned 200`) and `queue order`, and `dispatch` received them.

**Fix:** None needed. I kept `ERROR=1` so the errors panel and the SLO alert have something to measure.

**Lesson:** Check a real checkout, not only load-gen traffic, before treating errors as a fault.

## 10. Instana errors in the logs

**Symptom:** `cart`, `catalogue` and `dispatch` logs filled with `instana: ERROR: cannot connect to the agent`.

**Cause:** No deployment carries Instana settings (`kubectl get deploy -o yaml | grep -i instana` prints nothing). The Instana client library is built into the Robot Shop images and keeps looking for an agent, harmlessly.

**Fix:** A grep filter in the Fluent Bit config (`Exclude log instana`) keeps these lines out of Loki.

## Quick reference: commands that found the answer

| Question | Command |
|---|---|
| Why is a pod not ready? | `kubectl describe pod <name>` (read the Events) |
| Why did it crash? | `kubectl logs <pod> --previous` |
| Which add-ons are installed? | `aws eks list-addons --cluster-name voting-app-capstone --region us-east-1` |
| What does a service expose? | `kubectl run t --rm -it --image=curlimages/curl --restart=Never -- curl -s http://<svc>:8080/metrics` |
| Are nodes out of room? | `kubectl get pods -A -o custom-columns=NODE:.spec.nodeName --no-headers \| sort \| uniq -c` |
| Is a Helm release current? | `helm list -A` |
