# Lab 05: Monitoring Argo CD and Applications with Prometheus & Grafana

## Objectives

In this lab you will:

- Deploy a monitoring stack (Prometheus and Grafana) to your Kubernetes cluster
- Collect metrics from Argo CD and, optionally, from cluster nodes and workloads
- Visualize Argo CD application health and sync status in Grafana dashboards

## Conceptual Overview

A production-ready GitOps setup must be observable. This lab introduces a basic monitoring stack:

- **Prometheus** – collects and stores time-series metrics (CPU, memory, requests, Argo CD metrics, etc.)
- **Grafana** – visualizes metrics via dashboards
- **ServiceMonitor / Scrape configuration** – tells Prometheus where to collect metrics from (for example, Argo CD metrics endpoint)

High-level flow:

```text
Argo CD and workloads expose metrics
        ↓
Prometheus scrapes metrics periodically
        ↓
Grafana queries Prometheus and renders dashboards
```

## Architecture

```text
┌──────────────────────────────────────┐
│     Argo CD & Applications           │
│  - Metrics endpoints (HTTP)         │
└──────────────┬──────────────────────┘
               │ scraped by
               ↓
┌──────────────────────────────────────┐
│            Prometheus                │
│  - Time-series metrics store         │
└──────────────┬──────────────────────┘
               │ queried by
               ↓
┌──────────────────────────────────────┐
│              Grafana                 │
│  - Dashboards & visualizations       │
└──────────────────────────────────────┘
```

## Repository Contents

Typical files for this lab include:

- `monitoring-app.yaml` – Argo CD `Application` that deploys Prometheus, Grafana, and related resources
- `cleanup.sh` – optional script for tearing down the monitoring stack
- `README.md` – this guide

The exact structure may vary; check this lab directory for concrete manifests or Helm charts used.

## Step-by-Step Instructions

### 1. Deploy the Monitoring Stack via Argo CD

Apply the monitoring application manifest:

```bash
microk8s kubectl apply -f monitoring-app.yaml
```

This will typically deploy:

- Prometheus server
- Grafana
- ServiceMonitors for scraping Argo CD metrics (and possibly other components)

Check the status of the monitoring application:

```bash
argocd app get lab-05-monitoring
```

Verify that the application is **Synced** and **Healthy**.

### 2. Access Grafana

Port-forward the Grafana Service to your local machine:

```bash
microk8s kubectl port-forward -n lab-05-monitoring svc/grafana 3000:80
```

Open your browser at <http://localhost:3000>.

Default credentials (if not changed by the chart):

- Username: `admin`
- Password: `admin` (you may be prompted to change it on first login)

### 3. Configure Prometheus as a Data Source in Grafana

Inside Grafana:

1. Go to **Configuration → Data sources**.
2. Click **Add data source** and select **Prometheus**.
3. Set the URL to the Prometheus Service, for example:

   ```text
   http://prometheus-server.lab-05-monitoring.svc.cluster.local
   ```

4. Click **Save & test** to verify connectivity.

### 4. Import an Argo CD Dashboard

Grafana has many community dashboards that visualize Argo CD metrics. One widely used dashboard has ID `14584`.

To import it:

1. In Grafana, go to **Dashboards → Import**.
2. Enter the dashboard ID: `14584`.
3. Choose the Prometheus data source you added earlier.
4. Click **Import**.

You should now see a dashboard with:

- Application sync status
- Health states
- Sync durations
- Number of syncs per application

### 5. Explore Metrics

Some useful Argo CD metrics include:

- `argocd_app_health_status`
  - Indicates the health of each application (Healthy, Progressing, Degraded)
- `argocd_app_sync_status`
  - Indicates whether an application is Synced or OutOfSync
- `argocd_app_sync_total`
  - Total number of sync operations per application

You can query these directly in Prometheus or explore them via Grafana panels.

## Troubleshooting

### Prometheus is not scraping Argo CD metrics

1. Ensure that a `ServiceMonitor` or scrape configuration for Argo CD is deployed.
2. Port-forward Prometheus and inspect targets:

   ```bash
   microk8s kubectl port-forward -n lab-05-monitoring svc/prometheus-server 9090:80
   ```

   Open <http://localhost:9090>, go to **Status → Targets**, and check whether the Argo CD target is up.

3. Verify that the Argo CD metrics endpoint is reachable:

   ```bash
   microk8s kubectl port-forward -n argocd svc/argocd-metrics 8082:8082
   curl http://localhost:8082/metrics
   ```

### Grafana dashboard shows no data

1. Confirm the Prometheus data source is configured correctly and passes the **Test**.
2. Make sure the time range in Grafana (top-right) covers the period where metrics are available (for example, `Last 1 hour`).
3. Use the Prometheus UI to verify that metrics like `argocd_app_info` or `argocd_app_health_status` exist.

### Monitoring application is not healthy in Argo CD

1. Check the Argo CD application details:

   ```bash
   argocd app get lab-05-monitoring
   ```

2. Inspect the Pods in the monitoring namespace:

   ```bash
   microk8s kubectl get pods -n lab-05-monitoring
   microk8s kubectl describe pod <pod-name> -n lab-05-monitoring
   microk8s kubectl logs <pod-name> -n lab-05-monitoring
   ```

## Cleanup

Use the provided `cleanup.sh` script if present:

```bash
chmod +x cleanup.sh
./cleanup.sh
```

Or clean up manually:

```bash
# Delete the monitoring application
argocd app delete lab-05-monitoring

# Optionally delete the monitoring namespace
microk8s kubectl delete namespace lab-05-monitoring
```

## Success Criteria

This lab is successful when:

- The monitoring application in Argo CD is **Synced** and **Healthy**
- You can access the Grafana UI
- Prometheus is reachable as a data source from Grafana
- An Argo CD-focused dashboard shows meaningful metrics for your applications

## Next Steps

Continue with:

- **Lab 06** – Configuring Argo CD notifications and alerts for key events.

Further ideas:

- Add node and Pod-level metrics (for example, using kube-state-metrics or node exporters)
- Create custom Grafana dashboards per team or environment
- Configure alerting rules in Prometheus and connect them to an alert manager.