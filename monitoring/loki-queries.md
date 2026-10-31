# Loki queries (Grafana Explore, data source: Loki)

Logs for one service:
{kubernetes_container_name="payment"}

Error lines for one service:
{kubernetes_container_name="payment"} |= "ERROR"

Errors across all Robot Shop services:
{kubernetes_namespace_name="default"} |= "ERROR"
