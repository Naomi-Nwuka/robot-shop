# Robot Shop on EKS: Observability Capstone (Capstone 2)

This repository is my fork of [instana/robot-shop](https://github.com/instana/robot-shop), deployed on the Amazon EKS cluster built in Capstone 1 (`voting-app-capstone`, `us-east-1`). On top of the running shop I added metrics, logs, traces, an availability SLO and a Slack alert, so the service can be watched and trusted.

The shop's original Instana agent settings are removed. Everything runs on open-source tooling.

## What this delivers

| Requirement | Where it lives |
|---|---|
| Robot Shop running, all pods healthy, steady traffic | `K8s/helm` (shop), `K8s/load-deployment.yaml` (load-gen) |
| Metrics and one dashboard for requests, errors and latency | `monitoring/kube-prometheus-values.yaml`, `monitoring/blackbox-values.yaml`, `monitoring/dashboard.json` |
| Logs searchable by service and level | `monitoring/loki-values.yaml`, `monitoring/loki-storage-fix.yaml`, `monitoring/fluent-bit-values.yaml` |
| One request traced end to end | `monitoring/jaeger.yaml`, `monitoring/otel-collector-values.yaml`, `monitoring/shipping-otel-patch.yaml` |
| SLO with error budget | `docs/slo.md` |
| Alert to Slack with a linked runbook | `monitoring/probe-alert.yaml`, `monitoring/alertmanager-values.yaml`, `docs/runbook.md` |
| Blameless postmortem | `docs/postmortem.md` (completed after the incident drill) |

## Architecture

~~~text
                         EKS cluster voting-app-capstone (4 x t3.small nodes)

  namespace: default                                  namespace: monitoring
  ------------------                                  ---------------------
  load-gen --> web (nginx)                            blackbox exporter --probes--> web, cart, payment
                 |                                          |
                 +--> catalogue, cart, user, payment,       v
                      shipping, ratings, dispatch      Prometheus --> Grafana (dashboard)
                      |                                    |
        MongoDB, Redis, MySQL, RabbitMQ                    +--> Alertmanager --> Slack (+ runbook link)

  pod logs --> Fluent Bit (1 per node) -------------> Loki --> Grafana (Explore)

  shipping (OpenTelemetry Java agent) --> OTel Collector --> Jaeger (UI)
~~~

## Repository layout

~~~text
K8s/helm/                      Robot Shop Helm chart (release "robot-shop", namespace default)
K8s/load-deployment.yaml       Load generator (5 simulated users, deliberate errors on)
monitoring/                    All observability configuration (values files and manifests)
docs/runbook.md                What to do when the alert fires
docs/slo.md                    SLI, SLO and error budget policy
docs/postmortem.md             Blameless postmortem
TROUBLESHOOT.md                The main problems hit while building this, with the fix
~~~

## Prerequisites

- An EKS cluster with the **vpc-cni and kube-proxy add-ons** installed. Without kube-proxy, pods cannot reach the API server through its ClusterIP, CoreDNS never becomes ready and the whole cluster fails (see TROUBLESHOOT.md, problem 1).
- `kubectl`, `helm` and the AWS CLI configured for the cluster.
- A Slack workspace where you can create an incoming webhook.
- Capacity: the cluster uses `t3.small` nodes with an 11-pod limit each (44 in total). This build uses about 40 of them, so there is little spare room.

## Deploy

### 1. Robot Shop and load generator

~~~bash
helm install robot-shop ./K8s/helm
kubectl apply -f K8s/load-deployment.yaml
kubectl get pods
~~~

Expected: all shop pods `1/1 Running`. The shop is installed in the `default` namespace (there is no `robot-shop` namespace). The load generator runs 5 clients with `ERROR=1`, which deliberately makes a steady trickle of failing payment requests so the error panel has data.

### 2. Metrics: Prometheus, Grafana, Alertmanager

~~~bash
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack -n monitoring \
  -f monitoring/kube-prometheus-values.yaml \
  -f monitoring/grafana-slow-start.yaml \
  -f monitoring/alertmanager-values.yaml
~~~

The values are trimmed for small nodes. `grafana-slow-start.yaml` gives Grafana more CPU and a longer liveness delay, because it starts slowly on `t3.small` (TROUBLESHOOT.md, problem 4).

### 3. Request, error and latency data: blackbox exporter

~~~bash
helm install blackbox prometheus-community/prometheus-blackbox-exporter -n monitoring \
  -f monitoring/blackbox-values.yaml
~~~

### 4. Logs: Loki and Fluent Bit

~~~bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update
helm install loki grafana/loki -n monitoring \
  -f monitoring/loki-values.yaml -f monitoring/loki-storage-fix.yaml
helm install fluent-bit fluent/fluent-bit -n monitoring -f monitoring/fluent-bit-values.yaml
~~~

### 5. Traces: Jaeger, Collector and the instrumented service

~~~bash
kubectl apply -f monitoring/jaeger.yaml
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update
helm install otel-collector open-telemetry/opentelemetry-collector -n monitoring \
  -f monitoring/otel-collector-values.yaml
kubectl patch deployment shipping -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
kubectl patch deployment shipping --patch-file monitoring/shipping-otel-patch.yaml
~~~

The `shipping` service (Java) is instrumented by downloading the OpenTelemetry Java agent in an init container, so no image rebuild is needed. The `Recreate` strategy matters on these small nodes (TROUBLESHOOT.md, problem 6).

### 6. Alerting: Slack webhook and rule

The webhook URL is stored as a Kubernetes secret and is never committed to Git. The command reads it silently:

~~~bash
read -rs SLACK_URL && kubectl create secret generic slack-webhook -n monitoring \
  --from-literal=url="$SLACK_URL" && unset SLACK_URL
kubectl apply -f monitoring/probe-alert.yaml
~~~

`alertmanager-values.yaml` (applied in step 2) mounts the secret and routes alerts labelled `team="capstone"` to Slack.

### 7. Connect Grafana to Loki and import the dashboard

1. Port-forward Grafana: `kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80`.
2. Get the admin password: `kubectl get secret -n monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo`.
3. In Grafana: Connections, Data sources, add **Loki** with URL `http://loki.monitoring.svc.cluster.local:3100`.
4. Dashboards, New, Import: upload `monitoring/dashboard.json`.

## Using it

| Tool | Command | Address |
|---|---|---|
| Grafana | `kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80` | http://localhost:3000 |
| Jaeger | `kubectl port-forward -n monitoring svc/jaeger 16686:16686` | http://localhost:16686 |
| Prometheus | `kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090` | http://localhost:9090 |
| Shop | `kubectl port-forward svc/web 8082:8080` | http://localhost:8082 |

### Dashboard "Robot Shop RED"

| Panel | Query | Meaning |
|---|---|---|
| Latency (seconds) | `probe_duration_seconds` | Response time per probed endpoint |
| Failed probes | `1 - probe_success` | 0 is healthy, 1 is failing |
| Probe success rate | `avg(probe_success)` | Share of endpoints responding, shown as a percentage |

### Log queries (Grafana, Explore, Loki)

~~~text
{kubernetes_container_name="payment"}
{kubernetes_container_name="payment"} |= "ERROR"
{kubernetes_namespace_name="default"} |= "ERROR"
~~~

### Trace

In Jaeger choose service `shipping` and operation `GET /calc/{id}`. A trace shows 5 spans, including the MySQL query.

## SLO and alerting

- **SLO:** 99% of probes succeed over 30 days. **Error budget:** about 7.2 hours of failed probes per month. Full policy in `docs/slo.md`.
- **Alert:** `RobotShopProbeFailing` fires when any probe fails for 2 minutes. The Slack message names the failing endpoint and links to `docs/runbook.md`.
- **Tested:** scaling `payment` to zero produced a `FIRING` message in Slack within about 3 minutes, and scaling it back produced `RESOLVED`.

## Design decisions

- **Blackbox probes for requests, errors and latency.** The shop's services expose no request metrics (only business counters in `cart` and `payment`), so synthetic probes are the source for the dashboard. They measure availability and latency, not real user traffic. Real per-request detail comes from Loki and Jaeger.
- **One traced service.** Only `shipping` is instrumented, as the brief asks for one trace.
- **Lean everything.** Single-binary Loki, in-memory Jaeger, no ingress and small resource limits, because the nodes are `t3.small`.
- **No Instana.** No deployment carries Instana settings. The shop images still contain the Instana client library, which logs harmless connection errors. Fluent Bit filters those lines out.

## Known limitations

- **No persistent storage.** The cluster has no EBS CSI driver, so Prometheus, Loki and Jaeger keep data in temporary storage. Their data is lost if their pods restart.
- **Prometheus keeps 2 days of data** (`retention: 2d`), so the 30-day SLO window cannot be computed. A real deployment would extend retention.
- **Tight capacity.** Memory and the 11-pod limit per node leave little headroom. More nodes would be the first fix.
- **The load generator makes errors on purpose** (`partner-57` payments return HTTP 500), so a small steady error rate is expected.
- **Terraform state (Capstone 1):** the local state has a leftover failed node group from an earlier instance-type change. Clone the Capstone 1 repo fresh instead of reusing an old working folder.
- **kube-proxy was added by hand:** on the original cluster the add-on was installed with the AWS CLI. It is now also in the Capstone 1 Terraform `addons` block, so a rebuild includes it.

## Security

- No secrets are committed. The Slack webhook is a Kubernetes secret created by hand.
- The Grafana password is read from its secret and is never written down here.
- Grafana, Prometheus and Jaeger are reached by port-forward only and are not exposed publicly.

## Troubleshooting

See [TROUBLESHOOT.md](TROUBLESHOOT.md) for the main problems hit while building this and how each was fixed.
