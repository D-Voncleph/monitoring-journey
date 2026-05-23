# Upscale DV: Kubernetes Observability & Monitoring

This repository contains the standard operating procedures (SOPs) for deploying and accessing the enterprise observability stack within Upscale DV's Kubernetes environments.

## Architectural Overview

To achieve true system observability, we deploy the **kube-prometheus-stack**. This abstracts the complexity of monitoring into two core components:

1. **Prometheus:** A Time-Series Database (TSDB) that utilizes a pull-model to constantly scrape `/metrics` endpoints across all active Pods.
2. **Grafana:** The visualization layer that connects to Prometheus to translate raw PromQL data into dynamic, human-readable dashboards.

---

## 🚀 Deployment Runbook

Instead of managing hundreds of individual YAML manifests, we utilize Helm (the Kubernetes package manager) to deploy the entire stack in a single execution.

### 1. Install the Stack via Helm

```bash
# Add the official Prometheus community repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Deploy the stack into a dedicated namespace
helm install observability prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

### 2. Expose the Grafana UI (Sandbox / Bare-Metal)

By default, the Grafana service is isolated internally. For ephemeral sandbox testing (e.g., Killercoda), patch the service to expose a `NodePort`:

```bash
kubectl patch svc observability-grafana -n monitoring -p '{"spec": {"type": "NodePort"}}'

# Retrieve the assigned 5-digit external port
kubectl get svc observability-grafana -n monitoring
```

### 3. Decrypt the Admin Credentials

For security, the Helm chart generates a randomized administrator password and stores it in a Base64-encoded Kubernetes Secret. Do not attempt to log in with default credentials. Run this command to dynamically extract and decrypt the live password:

```bash
kubectl get secret observability-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

- **Username:** `admin`
- **Password:** (the output of the command above)

---

## 📊 Custom Application Dashboards (PromQL)

While the stack includes excellent out-of-the-box infrastructure dashboards, application-specific monitoring requires custom Prometheus Query Language (PromQL).

To monitor specific application tiers, build a custom Grafana dashboard using targeted queries.

### Example: FinTech Backend Memory Tracking

```promql
container_memory_working_set_bytes{namespace="fintech-app", pod=~"backend-api.*"}
```

This query filters out system noise, isolating the exact RAM utilization of the backend microservices to proactively detect memory leaks.
