# Task Manager GitOps configuration

This repository is the desired-state source watched by Argo CD. `apps/root.yaml` discovers the child Applications under `apps/children/`; each Application deploys a service/environment Helm release, monitoring stack, RBAC, or namespace manifests. All workloads use the shared `staging`, `production`, and `monitoring` namespaces.

## Structure

- `charts/`: reusable frontend, API, and worker Helm charts.
- `environments/`: Argo CD overlays for staging and production.
- `platform/namespaces/`: staging and production Namespace resources.
- `platform/rbac/`: developer and admin service accounts and bindings.
- `platform/monitoring/`: kube-prometheus-stack values and PrometheusRule/dashboard resources.
- `apps/`: root app-of-apps Application and child Applications.

## Workflow

1. Make a change to a chart or environment overlay and push it to `main`.
2. Argo CD's root Application discovers the changed child Application configuration.
3. Automated sync, pruning, self-healing, and namespace creation reconcile the cluster.
4. Check Argo CD health and Kubernetes rollouts; use Grafana's provisioned dashboards.

For kind, build and load the images named `frontend:v1.0.0`, `api:v1.0.0`, and `worker:v1.0.0` before syncing. The charts use `IfNotPresent`, so kind can use locally loaded images. A registry-backed production cluster must be able to pull the configured image repositories.

## Slack webhook secret (manual)

No real secret belongs in this repository. Before enabling Slack delivery, create the referenced Secret in the `monitoring` namespace:

```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
read -rsp 'Slack webhook URL: ' SLACK_WEBHOOK_URL
printf '\n'
kubectl -n monitoring create secret generic alertmanager-slack \
  --from-literal="slack_webhook_url=${SLACK_WEBHOOK_URL}"
unset SLACK_WEBHOOK_URL
```

Create the Secret before the monitoring Application syncs because the Alertmanager pod mounts it as a required volume. Supply the real URL through a protected terminal/secret manager, not shell history or Git. Rotate it with a secret-management tool or a safely sourced value, never by committing rendered YAML. Alertmanager reads the mounted file at runtime; the pinned stack chart uses the file-based Slack URL field.

## Notes

The API stores tasks in process memory; replicas do not share data. API HPA scales from CPU and memory requests and requires a working Kubernetes metrics API (metrics-server on kind). Grafana is accessible with `kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80`. Retrieve the generated admin password from the Grafana Secret rather than storing credentials here.
