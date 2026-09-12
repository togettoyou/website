---
id: grafana-dashboard
title: Grafana Dashboard
sidebar_label: Grafana Dashboard
---

The HAMi Grafana dashboard brings scheduler allocation data and node-side GPU usage into one view. It covers cluster capacity, physical devices, scheduler decisions, and individual vGPU containers; NVIDIA hardware telemetry can be added from DCGM Exporter.

![HAMi vGPU metrics dashboard running in Grafana with cluster and host GPU data](/img/docs/common/userguide/monitoring/hami-grafana-dashboard-overview.png)

_The unified dashboard running in Grafana with HAMi v2.10.0-compatible Prometheus series._

## Compatibility and data sources

Use this dashboard with **HAMi v2.10.0 or later** and **Grafana 9.x or later**. HAMi v2.10.0 is the compatibility boundary because the host metrics gained the `node` label and the `hami_node_gpu_overview` contract changed. The queries use the current `hami_*` metric names, so keep `legacyMetrics: false`, the v2.10.0 default.

Prometheus must collect both HAMi metrics endpoints:

| Component | Default Service port / NodePort | What it provides |
| --- | --- | --- |
| Scheduler | `31993` | Capacity, allocation, sharing, and device inventory |
| vGPU monitor on nodes | `31992` | Physical GPU and per-container vGPU usage |

DCGM Exporter is optional. Without it, only the final hardware telemetry row has no data.

## Connect Prometheus to HAMi

With the chart's default NodePort Services, these commands provide a quick reachability check:

```bash
curl http://<kubernetes-node-ip>:31993/metrics
curl http://<kubernetes-node-ip>:31992/metrics
```

The two Services behave differently. The scheduler Service selects the active leader, while the vGPU monitor Service has an endpoint for every HAMi device plugin Pod. A single `31992` NodePort target is therefore not enough: Kubernetes may route successive scrapes to different Pods, leaving gaps in per-node series. Prometheus should discover and scrape every Service endpoint.

### Prometheus Operator

When Prometheus Operator is already installed, let the HAMi chart create the required ServiceMonitors:

```bash
helm upgrade --install hami hami-charts/hami \
  --namespace kube-system \
  --set prometheus.enabled=true
```

The chart renders one ServiceMonitor for the scheduler and one for the vGPU monitor. The templates require the `monitoring.coreos.com/v1/ServiceMonitor` CRD; the vGPU monitor also requires `devicePlugin.enabled`. Check that the Prometheus `serviceMonitorSelector` and namespace selector admit both resources.

`prometheus.enabled=true` does not install Prometheus or Prometheus Operator.

### Prometheus without the Operator

Use Kubernetes service discovery to reach the scheduler leader and every vGPU monitor endpoint. This example assumes that HAMi runs in `kube-system`; change the namespace if `namespaceOverride` is set:

```yaml
scrape_configs:
  - job_name: hami
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
            - kube-system
    relabel_configs:
      - source_labels:
          - __meta_kubernetes_service_label_app_kubernetes_io_component
        regex: hami-(scheduler|device-plugin)
        action: keep
      - source_labels:
          - __meta_kubernetes_endpoint_port_name
        regex: monitor(port)?
        action: keep
```

Prometheus needs permission to discover Services and Endpoints through the Kubernetes API.

## Import the dashboard

Download the maintained [gpu-dashboard.json](/grafana/gpu-dashboard.json), then open **Dashboards > New > Import** in Grafana. Upload the file, select the Prometheus data source that collects HAMi metrics, and complete the import.

The toolbar has three variables. **Data source** switches Prometheus instances, **Node** narrows physical-device and scheduler panels, and **Namespace** narrows workload panels. Both filters support selecting multiple values.

The dashboard is arranged by the source and scope of the data:

| Row | Contents |
| --- | --- |
| Cluster overview | Physical GPU count, total and allocated memory, and shared containers |
| Physical GPUs (host) | Real-time host memory use and utilization, plus allocation comparisons |
| Scheduler / allocation | Device limits, allocation ratios, sharing, and GPU inventory |
| vGPU / container workloads | Per-container vGPU memory, limits, and utilization |
| NVIDIA hardware telemetry (optional DCGM Exporter) | XID errors, temperature, power usage, and SM clock |

## Add NVIDIA hardware telemetry

The optional hardware row reads four standard DCGM Exporter series:

- `DCGM_FI_DEV_XID_ERRORS`
- `DCGM_FI_DEV_GPU_TEMP`
- `DCGM_FI_DEV_POWER_USAGE`
- `DCGM_FI_DEV_SM_CLOCK`

The queries join the DCGM `UUID` label to HAMi's `device_uuid`. If this row remains empty after DCGM Exporter is added, inspect a DCGM series in Prometheus and confirm that it carries the standard `UUID` label.

## Troubleshooting

- Empty cluster totals or inventory usually mean that Prometheus is not scraping the scheduler endpoint on `31993`.
- Empty physical GPU panels usually mean that some vGPU monitor Service endpoints were not discovered.
- Container panels appear only when vGPU workloads are running; the selected namespace must also contain those workloads.
- Empty **Node** or **Namespace** selectors indicate that the selected Prometheus data source does not contain the corresponding `hami_*` series.
- An empty DCGM row does not affect the HAMi-native panels.

The complete metric contracts are documented under [Cluster device allocation](device-allocation) and [Real-time device usage](real-time-device-usage).
