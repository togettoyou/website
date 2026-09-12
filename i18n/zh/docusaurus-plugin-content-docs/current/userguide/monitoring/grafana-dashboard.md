---
title: Grafana 仪表盘
sidebar_label: Grafana 仪表盘
---

HAMi Grafana Dashboard 把调度侧的资源分配数据和节点侧的 GPU 实时用量放在同一个视图中。除了集群容量、物理设备、调度分配和 vGPU 容器，它还可以接入 DCGM Exporter，展示 NVIDIA GPU 的硬件遥测数据。

![运行在 Grafana 中的 HAMi vGPU 监控面板，展示集群概览和物理 GPU 数据](/img/docs/common/userguide/monitoring/hami-grafana-dashboard-overview.png)

_统一后的 Dashboard 运行在 Grafana 中，数据采用 HAMi v2.10.0 的指标格式。_

## 兼容版本与数据来源

这份 Dashboard 面向 **HAMi v2.10.0 及以上版本**，要求 **Grafana 9.x 或更高版本**。之所以从 v2.10.0 开始支持，是因为这一版本为主机指标补充了 `node` 标签，同时调整了 `hami_node_gpu_overview` 的指标约定。所有查询均使用当前的 `hami_*` 指标名，因此应保持 `legacyMetrics: false`；这也是 v2.10.0 的默认值。

Prometheus 必须同时采集 HAMi 的两个指标端点：

| 组件                    | 默认 Service 端口 / NodePort | 提供的数据                          |
| ----------------------- | ---------------------------- | ----------------------------------- |
| Scheduler               | `31993`                      | 容量、分配、共享情况和设备清单      |
| 各节点上的 vGPU monitor | `31992`                      | 物理 GPU 和每个容器的 vGPU 实时用量 |

DCGM Exporter 是可选项。没有部署 DCGM Exporter 时，只有最后一个硬件遥测区域没有数据。

## 让 Prometheus 采集 HAMi

Helm Chart 默认创建 NodePort Service，可以先用下面两个命令检查连通性：

```bash
curl http://<kubernetes-node-ip>:31993/metrics
curl http://<kubernetes-node-ip>:31992/metrics
```

两个 Service 的行为并不相同。Scheduler Service 只选择当前活跃的 leader；vGPU monitor Service 则为每个 HAMi device plugin Pod 提供一个 endpoint。因此，不能只把一个 `31992` NodePort 地址配置为采集目标：Kubernetes 可能在多次请求间选择不同的 Pod，最终造成节点指标缺失。Prometheus 应当发现并直接采集每个 Service endpoint。

### 使用 Prometheus Operator

集群已经安装 Prometheus Operator 时，可以让 HAMi Chart 直接创建所需的 ServiceMonitor：

```bash
helm upgrade --install hami hami-charts/hami \
  --namespace kube-system \
  --set prometheus.enabled=true
```

Chart 会分别为 scheduler 和 vGPU monitor 创建一个 ServiceMonitor。模板要求集群中已经存在 `monitoring.coreos.com/v1/ServiceMonitor` CRD；vGPU monitor 还要求 `devicePlugin.enabled` 为开启状态。Prometheus 的 `serviceMonitorSelector` 和 namespace selector 必须能够选中这两个资源。

`prometheus.enabled=true` 只负责创建 ServiceMonitor，不会安装 Prometheus 或 Prometheus Operator。

### 不使用 Prometheus Operator

可以使用 Kubernetes 服务发现，同时找到 scheduler leader 和所有 vGPU monitor endpoint。下面的配置假设 HAMi 位于 `kube-system`；如果设置了 `namespaceOverride`，需要相应修改 namespace：

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

Prometheus 需要具备通过 Kubernetes API 发现 Service 和 Endpoint 的权限。

## 导入 Dashboard

下载 website 维护的 [gpu-dashboard.json](/grafana/gpu-dashboard.json)，然后在 Grafana 中进入 **Dashboards > New > Import**。上传 JSON 文件，选择已经采集 HAMi 指标的 Prometheus 数据源，即可完成导入。

Dashboard 顶部有三个变量。**Data source** 用于切换 Prometheus 实例，**Node** 用于筛选物理设备和调度面板，**Namespace** 用于筛选工作负载面板。Node 和 Namespace 均支持多选。

各区域按照数据来源和观察范围划分：

| 区域 | 内容 |
| --- | --- |
| Cluster overview | 物理 GPU 数量、总显存、已分配显存和共享容器数 |
| Physical GPUs (host) | 主机实时显存与利用率，以及分配量和实际用量对比 |
| Scheduler / allocation | 设备上限、分配比例、共享情况和 GPU 清单 |
| vGPU / container workloads | 每个容器的 vGPU 显存、上限和利用率 |
| NVIDIA hardware telemetry (optional DCGM Exporter) | XID 错误、温度、功耗和 SM 时钟频率 |

## 接入 NVIDIA 硬件遥测

最后一个区域使用四项标准 DCGM Exporter 指标：

- `DCGM_FI_DEV_XID_ERRORS`
- `DCGM_FI_DEV_GPU_TEMP`
- `DCGM_FI_DEV_POWER_USAGE`
- `DCGM_FI_DEV_SM_CLOCK`

查询通过 DCGM 的 `UUID` 标签与 HAMi 的 `device_uuid` 标签关联。部署 DCGM Exporter 后该区域仍为空时，可以在 Prometheus 中检查任意一条 DCGM 时间序列，确认它带有标准的 `UUID` 标签。

## 常见问题

- 集群汇总或设备清单为空，通常表示 Prometheus 没有采集 scheduler 的 `31993` 端点。
- 物理 GPU 面板为空，通常表示部分 vGPU monitor Service endpoint 没有被发现。
- 只有运行 vGPU 工作负载后，容器面板才会产生数据；同时需要确认 Namespace 变量选中了对应命名空间。
- **Node** 或 **Namespace** 选项为空，说明当前 Prometheus 数据源中没有相应的 `hami_*` 时间序列。
- DCGM 区域为空不会影响 HAMi 原生指标对应的其他面板。

各指标的完整约定请参见[集群设备分配](device-allocation)和[实时设备使用情况](real-time-device-usage)。
