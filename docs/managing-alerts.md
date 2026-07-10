# Alerts

Flow: **PrometheusRule → Prometheus → Alertmanager → Slack**

- **PrometheusRule** CRDs define the alert conditions (PromQL expressions). Prometheus automatically discovers these based on label selectors.
- **Prometheus** evaluates the rules and fires alerts to Alertmanager when conditions are met. 
- **Alertmanager** receives alerts and routes them based on its configuration, which is where you define Slack as a receiver. Alertmanager is automatically included and connected to prometheus within the kube-prometheus-stack.

## Creating alerts via PrometheusRule CRD

Here is a simple example of an alert for overheating GPUs:

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gpu-alerts
  namespace: monitoring
  labels:
    release: prometheus  
spec:
  groups:
    - name: gpu.rules
      rules:
        - alert: GPUHighTemp
          expr: DCGM_FI_DEV_GPU_TEMP >= 92
          for: 1m
          labels:
            severity: critical
            team: enigma
          annotations:
            summary: "Node {{ $labels.Hostname }} has high GPU temp"
            description: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} has temp {{ $value }}C"
```

- `metadata.labels` needs to be `release: prometheus` in order for prometheus to detect it.

- `spec.groups.name.rules.alert.labels` dictate which slack channel the alert gets routed to (see below). A severity of critical gets sent to the `critical` channel, warning to the `warning` channel, and info goes to the info channel. Some rules are given a severity of `none` so that they aren't sent to Slack.

## Alerts from existing sources

- The `kube-prometheus-stack` has many useful alerting rules
- The website [Awesome Prometheus Alerts](https://samber.github.io/awesome-prometheus-alerts/) also has many useful rules

## Ceph alert rules live in the rook repo

Ceph alerting rules do **not** come from `manifests/promrules/` in this repo. The
`prometheus-ceph-rules` PrometheusRule is created by the `rook-ceph-cluster` Helm
chart in the `rook-ceph` namespace, and its lifecycle (the `release=prometheus`
label and the `CephNodeRootFilesystemFull` removal patch) is managed from the rook
repo as part of Rook upgrades. The `storage-rules.yaml` rules here contain no
Ceph-capacity alerts — the rook-provided rules are the only Ceph alert coverage,
so don't remove or duplicate them from this repo.

# Alertmanager

To configure AlertManager for slack integration, there is a config section in the Helm chart that details which alerts get sent to which slack channels, and a bot token is needed. This config section is too long to put here, but see a full example at [`manifests/alertmanager/config-test.yaml`](../manifests/alertmanager/config-test.yaml).

For inspiration about new alerts, see this repository [awesome prometheus alerts](https://samber.github.io/awesome-prometheus-alerts/rules) of commonly used alerts

## How to get a slack token.

Following [this tutorial](https://docs.slack.dev/tools/python-slack-sdk/tutorial/uploading-files/), 
1. Create a Slack app and install it in the workspace.
2. Go to "OAuth & Permissions" section, scroll down to Scopes -> Bot Token Scopes, and add the `chat:write` OAuth Scope from the drop down menu.
3. Go to "Install App" section, and install the app to the work space.
4. After this you should see a new OAuth Token. Copy this and save it.
5. Create a channel in slack
6. Once created, go to the message bar and type @AlertManager, or whatever name you chose for your bot, press enter, and then press the invite button to invite the bot to the channel.

This bot token can be used for multiple slack channels. Just put it in the appropriate place in the config section of the helm chart.

If you ever need to see this token again, visit `api.slack.com/apps`, click on the AlertManager app, and go to "OAuth and Permissions". The token should be displayed on this page. 

### Changing slack token

To change the token in the case it gets compromised, go to `api.slack.com/apps`. Click on the app, go to "OAuth and Permissions", scroll down to the bottom and click "Revoke Tokens". Then reinstall the app to the workspace, on the same page. Reinvite the bot to the slack channels.