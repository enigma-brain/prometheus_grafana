# Workflow for Grafana Dashboards 

### Quick notes
- If you create a config map for dashboards that already exist and they have the same uids, the configmap will overwrite the existing ones
- If you've provisioned an alert, you can no longer use those evaluation groups when making new alerts
- The `data` key (the `*.json` filename) must be **unique across all dashboard ConfigMaps in the same Grafana folder**. The sidecar writes one file per data key per folder, so two ConfigMaps both using `dashboard1.json` silently overwrite each other and only one gets provisioned. Use `<configmap-name>.json`.
- Since Grafana 12, the dashboard JSON's datasource variable must hold the datasource **uid** (`prometheus` here) — name-based lookup ("Prometheus") and the `"default"` pseudo-uid no longer resolve and every panel shows *No data*.
- ConfigMaps larger than 256KiB (e.g. the ceph-mixin bundle) must be applied with `kubectl apply --server-side`.

## Dashboards

### Workflow

Create the dashboard in Grafana, within a folder.

When the dashboard is mature, export the particular dashboard as JSON (can't export an entire folder), and put in a configmap, specifying that it's a dashboard and the folder in the config map:

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: monitoring
  name: test-storage
  labels:
    app: test-storage
    grafana_dashboard: "1"
  annotations:
    grafana_folder: "Test"
data: 
  test-storage.json: |-
    YAML/JSON
```

The data key must be unique within the Grafana folder (see Quick notes above) — name it after the ConfigMap.

if you ever need to make adjustments, you can copy the JSON content and import as a new dashboard. **You'll have to change the UID**. Make the edits you need, export the JSON to your clipboard, and replace the content in the configmap.

NOTE: I experienced problems with `kubectl create -f manifest_dir/`, which creates all dashboard config maps at the same time. Looking at the logs of the grafana server it mentioned both `error="A dashboard with the same uid already exists"`, perhaps because I didn't delete the dashboard previously, but even when I did, I saw `error="[password-auth.failed] too many consecutive incorrect login attempts for user - login for user temporarily blocked"`. Creating the configmaps one by one, and waiting until the dashboard was visible in the GUI, seemed to work well.