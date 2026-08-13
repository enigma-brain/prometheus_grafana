# `kube-prometheus-stack` for Kubernetes Monitoring

## Background

The `kube-prometheus-stack` is deployed as a helm chart and is centered around Prometheus, which scrapes metrics from node exporters, and Alertmanager, which handles alerts sent by Prometheus and routes them to a receiver such as Slack. This chart also deploys Grafana for dashboards, but support for Grafana within this stack is more limited as the stack deploys operators for Prometheus and Alertmanager, but not Grafana. We could deploy the Grafana operator separately, but this would be managed by a separate stack with a different development cycle. Thus, we try to extensively use Prometheus and Alertmanager and use Grafana only for dashboards (and not for alerts, which it technically supports but is more clunky).

 NOTE: Prometheus seems to require local storage, so here we deploy a single instance of prometheus and grafana on a specific node (at-compute010) and use subdirectories within /mnt/md0 for these apps to store their data. If these pods ever crash, we have them deploy to the same node so that they still have access to their data within node-specific /mnt/md0.

## Prerequisites

Before deploying, ensure the following are in place on the target cluster:

- `kubectl` installed and configured with access to the cluster
- [Helm 3](https://helm.sh/docs/intro/install/) installed
- [nginx-ingress controller](https://kubernetes.github.io/ingress-nginx/) deployed in the cluster
- [cert-manager](https://cert-manager.io/docs/installation/) deployed in the cluster (for automatic TLS certificate provisioning)
- Local storage available on the target node (`at-compute010`) at `/mnt/md0`, with subdirectories `k8s_apps/prometheus` and `k8s_apps/grafana` already created

## Installation
- add prometheus helm repo and update
    - `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
    - `helm repo update`
- create monitoring namespace
    - `kubectl create namespace monitoring`
- statically provision local storage (PV and storage classes) by applying this file
    - `kubectl apply -f manifests/additional_kube_prometheus_config/persistentVolume_StorageClass.yaml`
    - if using local storage on /mnt/md0 of a local compute node, you need to also do `sudo chown -R 1000:2000 /mnt/md0/k8s_apps/prometheus` since prometheus runs with user id 1000 and group id 2000
- confirm resources created
    - `kubectl get pv`
    - `kubectl get sc`
- create the Alertmanager config secret — this must exist before `helm install` because `values.yaml` references it via `useExistingSecret`
    - use `manifests/alertmanager/config-test.yaml` as a template: copy it, fill in your Slack bot token in place of `'OMITTED'`, and apply it
    - `kubectl apply -f your-alertmanager-config.yaml`
    - do not commit the file with a real token
- create the MinIO/DirectPV scrape config secret — also required before `helm install`
    - follow the steps in [Additional Configuration](docs/additional-configuration.md#directpv-and-minio) to generate the scrape config and create the secret
- helm install kube-prom-stack, using `values.yaml` file to have helm create PersistentVolumeClaims for the PVs based on the storage classes
    - `helm install prometheus prometheus-community/kube-prometheus-stack -f values.yaml --namespace monitoring`
    - this `values.yaml` file also creates an additional servicemonitor for `nvidia-dcgm-exporter` for GPU monitoring


## Documentation

For detailed information on specific topics, see the documentation in the [docs](docs/) directory:

- **[Additional Configuration](docs/additional-configuration.md)** - Configure monitoring for specific Kubernetes components (kube-controller-manager, kube-scheduler, kube-proxy, etcd, NVIDIA DCGM exporter)
- **[Managing Alerts](docs/managing-alerts.md)** - Create alert rules and set up Slack notifications
- **[Dashboarding Workflow](docs/dashboards-and-alerts-workflow.md)** - Step-by-step workflow for creating and managing dashboards 
- **[Ray Monitoring](docs/ray-monitoring.md)** - PodMonitors, the ray-exporter and the RayCluster dashboard across the ray, ray-test and ray-prod namespaces
- **[Ingress Setup](docs/ingress-setup.md)** - Configure ingress for external access to Prometheus, Alertmanager, and Grafana GUIs
- **[Troubleshooting](docs/troubleshooting.md)** - Advanced topics including changing the admin password, subfolder configuration, and deleting default dashboards