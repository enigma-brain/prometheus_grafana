# Ray (KubeRay) Monitoring

Ray clusters are monitored across three namespaces: `ray` (the long-running shared
clusters `op-pipe-kuberay` and `edi-etl-kuberay`), `ray-test` and `ray-prod` (the
per-environment `op-sync-*-kuberay` and `op-spikesort-*-kuberay` clusters).

## The three pieces

| Piece | File | Provides |
|---|---|---|
| PodMonitors | `manifests/additional_kube_prometheus_config/ray_podmonitors.yaml` | Ray's own metrics endpoint: `ray_node_*`, `ray_resources`, `ray_component_*`, `autoscaler_*` |
| ray-exporter | `manifests/ray_exporter/` | Dashboard/state-API metrics Ray does not export to Prometheus: `ray_job_*`, `ray_tasks_*`, `ray_pipeline_tasks_*`, `ray_cluster_up` |
| Dashboard | `manifests/dashboard_config_maps/apps/ray-clusters-preview.yaml` | Grafana "RayCluster (preview)", uid `ray-cluster-dashboard-preview` |

`manifests/dashboard_config_maps/apps/ray-clusters.yaml` is the unmodified upstream
KubeRay dashboard (uid `ray-cluster-dashboard`), kept for reference. It is not
namespace-aware.

### Why a separate exporter

Ray publishes per-job resource reservation, per-task-name counts and job metadata
only through the head pod's dashboard/state API (`/api/v0/tasks`, `/nodes/<id>`,
`/api/jobs/`) — none of it reaches the Prometheus endpoint the PodMonitors scrape.
`ray-exporter` polls those APIs on each scrape and translates them into metrics.

It runs as a single Deployment in the `ray` namespace and covers every namespace
listed in its `RAY_NAMESPACE` env var. Each emitted series carries a `namespace`
label naming the namespace of the Ray cluster it describes.

> **`honorLabels: true` on the exporter's ServiceMonitor is load-bearing.** Without
> it Prometheus overwrites the exporter's `namespace` label with the exporter pod's
> own namespace, and every Ray cluster is reported as living in `ray`.

## The dashboard's variable chain

`$namespace` is the top-level variable. `$Cluster`, `$SessionName`, `$Instance` and
`$pvc` are all chained off it, and every `ray_*` / `autoscaler_*` selector carries an
explicit `namespace=~"$namespace"` matcher.

`$pvc` exists because PVC names differ per environment — `pipeline-data-hot` /
`pipeline-data-cold` in `ray`, `pipeline-cache-hot` / `pipeline-data` in `ray-test`
and `ray-prod` — so the "Persistent Volume Fill (%)" panel discovers them rather
than hardcoding names. A PVC only appears once a running pod mounts it, since the
underlying `kubelet_volume_stats_*` metrics come from the kubelet.

A namespace likewise only appears in `$namespace` once its Ray pods are actually
being scraped.

## Adding a new Ray namespace

All four steps are required; missing the first means no metrics are collected at all.

1. Add it to **both** `namespaceSelector.matchNames` lists in
   `manifests/additional_kube_prometheus_config/ray_podmonitors.yaml`.
2. Add a `Role` + `RoleBinding` for it in `manifests/ray_exporter/ray-exporter.yaml`
   (copy the `ray-test` block — the RoleBinding subject stays the `ray-exporter`
   ServiceAccount in namespace `ray`).
3. Add it to `RAY_NAMESPACE` on the Deployment in the same file.
4. Apply and restart:
   ```bash
   kubectl apply -f manifests/additional_kube_prometheus_config/ray_podmonitors.yaml
   kubectl apply -f manifests/ray_exporter/ray-exporter.yaml
   kubectl rollout restart deploy/ray-exporter -n ray
   ```

The dashboard needs no change — `$namespace` picks the new namespace up on its next
variable refresh.

## Applying

```bash
# PodMonitors and the exporter stack: ordinary apply
kubectl apply -f manifests/additional_kube_prometheus_config/ray_podmonitors.yaml
kubectl apply -f manifests/ray_exporter/ray-exporter-code.yaml
kubectl apply -f manifests/ray_exporter/ray-exporter.yaml

# The exporter code is a volume-mounted ConfigMap, so applying it does NOT reload
# the running process:
kubectl rollout restart deploy/ray-exporter -n ray

# The dashboard JSON is ~630KB, over the 256KiB limit of the
# kubectl.kubernetes.io/last-applied-configuration annotation that plain apply uses.
# --force-conflicts takes field ownership from the earlier ad-hoc `kubectl replace`.
kubectl apply --server-side --force-conflicts \
  -f manifests/dashboard_config_maps/apps/ray-clusters-preview.yaml
```

## Verifying

```bash
# Ray metrics are arriving from every namespace
kubectl -n monitoring exec sts/prometheus-prometheus-kube-prometheus-prometheus -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=count%20by%20(namespace)%20(ray_node_cpu_count)'

# The exporter reached every configured namespace (1 per namespace)
kubectl -n monitoring exec sts/prometheus-prometheus-kube-prometheus-prometheus -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=ray_exporter_namespace_up'

# Every Ray head answered the exporter (1 per cluster)
kubectl -n monitoring exec sts/prometheus-prometheus-kube-prometheus-prometheus -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/query?query=ray_cluster_up'
```

`ray_exporter_namespace_up{namespace="..."} 0` means head discovery failed there —
usually a missing `Role`/`RoleBinding` from step 2 above. Check the exporter log:

```bash
kubectl logs -n ray deploy/ray-exporter
```
