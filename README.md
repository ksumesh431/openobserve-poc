# Basic k3s and OpenObserve Setup

Everything is IaC — no clickops. This repo stands up a local 3-node K3s cluster, OpenObserve,
an OpenTelemetry Collector, and pre-provisioned dashboards.

## Files

| File | Purpose |
|---|---|
| `k3d-cluster.yaml` | k3d cluster config (1 server + 2 agents) |
| `openobserve-iac.yaml` | OpenObserve (HelmChart) + Traefik Ingress |
| `otel-collector-iac.yaml` | OTel Collector DaemonSet (logs + host/kubelet metrics) |
| `o2-dashboard-cluster-iac.yaml` | Dashboard: K3s Cluster Overview |
| `o2-dashboard-pod-health-iac.yaml` | Dashboard: Pod & Workload Health |
| `o2-dashboard-pipeline-iac.yaml` | Dashboard: Telemetry Pipeline Health |

## 1. Install host prerequisites (Windows winget)

```powershell
winget install --id k3d.k3d --exact --silent --accept-package-agreements
winget install --id Kubernetes.kubectl --exact --silent --accept-package-agreements
```

## 2. Create the cluster

```powershell
k3d cluster create --config k3d-cluster.yaml
kubectl get nodes -o wide
```

## 3. Deploy OpenObserve + ingress

```powershell
kubectl apply -f openobserve-iac.yaml
kubectl rollout status statefulset/openobserve-standalone -n openobserve --timeout=180s
```

## 4. Deploy the OTel Collector

```powershell
kubectl apply -f otel-collector-iac.yaml
kubectl get pods -n openobserve -l app.kubernetes.io/name=opentelemetry-collector -o wide
```

## 5. Provision the dashboards

```powershell
kubectl apply -f o2-dashboard-cluster-iac.yaml -f o2-dashboard-pod-health-iac.yaml -f o2-dashboard-pipeline-iac.yaml
kubectl wait --for=condition=complete job/provision-o2-dashboard-cluster job/provision-o2-dashboard-pod-health job/provision-o2-dashboard-pipeline -n openobserve --timeout=300s
```

## Access

- UI: <http://o2.127.0.0.1.nip.io:8080> (port 8080 — the cluster maps host 8080 → Traefik 80)
- Login: `admin@local.dev` / `ComplexPassword123!`

## Re-applying a dashboard after editing its ConfigMap

Job specs are immutable, so delete the job first, then re-apply:

```powershell
kubectl delete job provision-o2-dashboard-cluster -n openobserve
kubectl apply -f o2-dashboard-cluster-iac.yaml
kubectl wait --for=condition=complete job/provision-o2-dashboard-cluster -n openobserve --timeout=300s
```

Jobs are idempotent — they update the dashboard if it already exists.

## Teardown

Deletes the cluster and everything in it (OpenObserve, collector, dashboards, registry):

```powershell
k3d cluster delete --config k3d-cluster.yaml
```

Verify with `k3d cluster list` (should be empty).
