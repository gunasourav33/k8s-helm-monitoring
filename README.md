# k8s-helm-monitoring

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg) ![Kubernetes](https://img.shields.io/badge/kubernetes-1.24+-blue.svg) ![Helm](https://img.shields.io/badge/helm-3.x-informational.svg) ![Prometheus](https://img.shields.io/badge/prometheus-grafana-orange.svg)

Production-grade Kubernetes deployment using Helm with full observability — Prometheus metrics, Grafana dashboards, and alerting rules baked in.

## What's in here

| Path | Description |
|------|-------------|
| `helm/` | Helm chart for the web application |
| `helm/templates/` | Kubernetes manifests (Deployment, Service, HPA, ServiceAccount) |
| `manifests/` | Prometheus stack configuration (ServiceMonitor, alerts, Grafana dashboard) |

## Prerequisites

- Kubernetes 1.24+
- Helm 3.10+
- `kube-prometheus-stack` installed in the `monitoring` namespace
- kubectl configured for your cluster

## Quick start

```bash
# Add the prometheus community chart repo (if not already added)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install the monitoring stack (skip if already installed)
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# Deploy the webapp chart
helm install webapp ./helm --namespace default -f helm/values.yaml
```

## Configuration

Key values in `helm/values.yaml`:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `replicaCount` | `2` | Number of pod replicas |
| `image.repository` | `gcr.io/google-samples/hello-app` | Container image |
| `image.tag` | `2.0` | Image tag |
| `autoscaling.enabled` | `true` | Enable HPA |
| `autoscaling.minReplicas` | `2` | Minimum replicas |
| `autoscaling.maxReplicas` | `10` | Maximum replicas |
| `autoscaling.targetCPUUtilizationPercentage` | `70` | CPU threshold for scale-up |
| `prometheus.serviceMonitor.enabled` | `true` | Create Prometheus ServiceMonitor |

Override values at deploy time:

```bash
helm upgrade webapp ./helm \
  --set replicaCount=3 \
  --set image.tag=2.1 \
  --namespace default
```

## Monitoring setup

Apply the Prometheus ServiceMonitor and alert rules:

```bash
kubectl apply -f manifests/prometheus-stack.yaml
```

This creates:
- **ServiceMonitor** — tells Prometheus to scrape `/metrics` on port 8080
- **PrometheusRule** — three alert conditions (see below)
- **Grafana ConfigMap** — auto-provisions a dashboard in Grafana

### Alert rules

| Alert | Condition | Severity |
|-------|-----------|----------|
| `HighErrorRate` | HTTP 5xx rate > 5% for 5 min | warning |
| `PodMemoryUsageHigh` | Memory > 85% of limit for 10 min | warning |
| `PodCrashLooping` | Restart rate > 0.5/min for 5 min | critical |

## Security

All pods run with a hardened security context:

- Non-root user (UID 1000)
- Read-only root filesystem
- `allowPrivilegeEscalation: false`
- Only `/tmp` and `/app/cache` are writable (mounted as `emptyDir`)

## Upgrade and rollback

```bash
# Upgrade to a new image version
helm upgrade webapp ./helm --set image.tag=2.2

# Roll back to the previous release if something breaks
helm rollback webapp 0
```

## Common issues

**Pods stuck in `Pending`** — check node resource availability with `kubectl describe pod <pod>`.

**ServiceMonitor not picked up** — verify `kube-prometheus-stack` was deployed with `serviceMonitorSelectorNilUsesHelmValues: false` or that your release labels match the Prometheus selector.

**Grafana dashboard missing** — confirm the `grafana-dashboards` ConfigMap label `grafana_dashboard: "1"` matches your Grafana sidecar config.
